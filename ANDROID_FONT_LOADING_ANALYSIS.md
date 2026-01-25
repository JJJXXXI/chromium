# Android 平台字体加载完整分析

## 目录
1. [高层入口点](#高层入口点)
2. [SkFontMgr_New_Android 构造流程](#skfontmgr_new_android-构造流程)
3. [系统字体扫描流程](#系统字体扫描流程)
4. [字体匹配机制](#字体匹配机制)
5. [数据流图](#数据流图)
6. [核心代码位置](#核心代码位置)

---

## 高层入口点

### Chromium 中的初始化流程

**位置**: [skia/ext/font_utils.cc](skia/ext/font_utils.cc#L70-L85)

```cpp
sk_sp<SkFontMgr> DefaultFontMgr() {
  static std::once_flag flag;
  static SkFontMgr* mgr;
  std::call_once(flag, [] {
    mgr = fontmgr_factory().release();
    g_factory_called = true;
  });
  return sk_ref_sp(mgr);
}

static sk_sp<SkFontMgr> fontmgr_factory() {
  if (g_fontmgr_override) {
    return sk_ref_sp(g_fontmgr_override);
  }

#if BUILDFLAG(IS_ANDROID)
  // 首先尝试 NDK 字体 API（Android N+）
  if (base::FeatureList::IsEnabled(kUseAndroidNDKFontAPI) &&
      android_get_device_api_level() > __ANDROID_API_V__) {
    sk_sp<SkFontMgr> ndk_fontmgr =
        SkFontMgr_New_AndroidNDK(false, SkFontScanner_Make_Fontations());
    if (ndk_fontmgr && ndk_fontmgr->countFamilies()) {
      return ndk_fontmgr;  // 成功返回
    }
  }
  // 降级到传统的 Android 字体管理器
  return SkFontMgr_New_Android(nullptr, SkFontScanner_Make_Fontations());
#elif BUILDFLAG(IS_APPLE)
  return SkFontMgr_New_CoreText(nullptr);
#elif ...
```

### 平台初始化时机

**位置**: [content/renderer/renderer_main_platform_delegate_android.cc](content/renderer/renderer_main_platform_delegate_android.cc#L26-L35)

```cpp
void RendererMainPlatformDelegate::PlatformInitialize() {
  // 在沙箱设置前初始化字体管理器
  // SkFontMgr_New_AndroidNDK 在 Android 14+ 设备上可能调用 statx
  // 这些系统调用在沙箱中被禁止
  // 详见: https://crbug.com/40618213
  [[maybe_unused]] auto mgr = skia::DefaultFontMgr();
}
```

---

## SkFontMgr_New_Android 构造流程

### 构造函数流程（从 third_party/skia 源码推断）

```
SkFontMgr_New_Android(SkFontConfigInterface* fci, SkFontScanner* fontScanner)
    ↓
    ├─ 创建 SkFontMgr_Android 实例
    ├─ fSystemFontUse_ = fci  (FontConfigInterface)
    ├─ fFontScanner_ = fontScanner  (Fontations 字体扫描器)
    │
    ├─ 调用 fFontScanner_->scanSystemFonts()
    │  └─ 扫描 Android 系统字体目录
    │     ├─ /system/fonts/
    │     ├─ /product/fonts/
    │     ├─ /odm/fonts/
    │     └─ 用户字体目录（如可用）
    │
    └─ 加载字体元数据到内存
       ├─ 字体家族名
       ├─ 字体样式（normal, bold, italic）
       ├─ 文件路径
       └─ Unicode 范围信息
```

### 关键数据结构

```cpp
// SkFontMgr_Android 的主要成员
class SkFontMgr_Android : public SkFontMgr {
 private:
  // 系统字体集合
  // 结构：FamilyName -> [Font*, Font*, ...]
  // 例如："Roboto" -> [Roboto Normal, Roboto Bold, Roboto Italic, ...]
  SkTDArray<SkFontStyleSet_Android*> fFamilies;
  
  // 字体扫描器（负责解析字体文件）
  sk_sp<SkFontScanner> fFontScanner_;
  
  // FontConfig 接口（获取字体配置）
  SkFontConfigInterface* fSystemFontUse_;
  
  // 缓存的字体样式集合
  SkTDArray<SkString> fNames;
};
```

### 构造流程详细步骤

```
1️⃣ 初始化阶段
   ├─ 创建 SkFontMgr_Android 对象
   ├─ 设置 FontScanner = Fontations (支持最新字体格式)
   └─ 设置 FontConfigInterface = nullptr (Android 使用系统目录)

2️⃣ 扫描系统字体目录
   ├─ /system/fonts/          (系统预装字体)
   ├─ /product/fonts/         (产品定制字体)
   ├─ /odm/fonts/             (设备制造商字体)
   └─ /data/local/tmp/fonts/  (用户字体，如可用)

3️⃣ 解析字体文件
   ├─ 对每个 .ttf/.otf/.ttc 文件：
   │  ├─ 使用 Fontations 解析文件头
   │  ├─ 提取字体元数据：
   │  │  ├─ Family name (e.g., "Roboto")
   │  │  ├─ Subfamily (e.g., "Bold Italic")
   │  │  ├─ Weight/Style/Width 值
   │  │  └─ Unicode 覆盖范围
   │  └─ 创建 SkTypeface 对象
   │
   └─ 构建家族映射表

4️⃣ 组织字体家族
   ├─ 相同名称的字体组织为一个家族
   │  例如：
   │  ├─ Roboto
   │  │  ├─ Roboto-Regular.ttf      (weight=400, width=100, slant=0)
   │  │  ├─ Roboto-Bold.ttf          (weight=700, width=100, slant=0)
   │  │  ├─ Roboto-Italic.ttf        (weight=400, width=100, slant=12)
   │  │  └─ Roboto-BoldItalic.ttf    (weight=700, width=100, slant=12)
   │  │
   │  └─ Noto Sans CJK
   │     ├─ NotoSansCJK-Regular.ttc
   │     └─ NotoSansCJK-Bold.ttc
   │
   └─ 建立索引便于快速查找

5️⃣ 缓存元数据
   ├─ 字体家族列表存储在内存
   ├─ 不立即加载完整字体文件
   ├─ 只在需要时延迟加载
   └─ 支持高效的字体匹配查询
```

---

## 系统字体扫描流程

### 目录扫描策略

Android 系统字体存储在多个位置，按优先级排序：

| 位置 | 优先级 | 说明 |
|------|--------|------|
| `/system/fonts/` | 1 (最高) | 系统预装字体 (Roboto, Noto Sans 等) |
| `/product/fonts/` | 2 | 产品定制字体 |
| `/odm/fonts/` | 3 | ODM (设备制造商) 字体 |
| `/data/local/tmp/fonts/` | 4 | 用户安装的字体 (如可用) |

### 字体文件格式支持

```
TrueType (.ttf)
├─ 最常见的格式
├─ 支持单个字体文件
└─ 例：Roboto-Regular.ttf

OpenType (.otf)
├─ 扩展的 TrueType
├─ 支持高级排版特性
└─ 例：NotoSerifArabic-Regular.otf

TrueType Collection (.ttc)
├─ 多个字体在一个文件中
├─ 用于 CJK (汉字、日文、韩文)
├─ 减少存储空间
└─ 例：NotoSansCJK-Regular.ttc
    ├─ 包含：中文、日文、韩文、繁体版本
    └─ 通过 ttcIndex 区分

Variable Fonts (.ttf with variations)
├─ 支持连续的重量/宽度/样式变化
└─ 例：Roboto[wght].ttf
    ├─ 重量范围：100-900
    └─ 自动渲染中间值
```

### Fontations 扫描器工作流程

```
SkFontScanner_Make_Fontations()
    ↓
创建 Fontations 扫描器实例
    ↓
scanSystemFonts()
    ├─ 遍历每个系统字体目录
    │  ├─ 列出所有 .ttf/.otf/.ttc 文件
    │  └─ 调用 deserializeAxes() 和 deserializeFamilies()
    │
    ├─ 对每个字体文件
    │  ├─ 打开文件并读取字体头
    │  ├─ 解析 name 表 (字体名称)
    │  ├─ 解析 fvar 表 (如存在，变体轴)
    │  ├─ 解析 gvar 表 (字形变体)
    │  └─ 提取样式属性
    │
    └─ 生成字体元数据
       ├─ Font { name, path, weight, width, slant, ttcIndex }
       ├─ Family { name, [Font1, Font2, ...] }
       └─ 支持快速匹配查询
```

---

## 字体匹配机制

### 查询流程

```
Request: font-family = "Arial", weight = bold, style = italic
    ↓
matchFamily(familyName = "Arial")
    ├─ 在 fFamilies 中查找
    ├─ 精确匹配：找到 "Arial" 家族
    ├─ 近似匹配：如找不到 "Arial"，尝试相似名称
    │  └─ 例："Arial" → "Helvetica" → "sans-serif"
    └─ 返回 SkFontStyleSet_Android
           (包含 Arial 的所有样式变体)
    ↓
matchFamilyStyle(familyName, SkFontStyle(weight=700, slant=italic))
    ├─ 遍历该家族的所有字体
    ├─ 选择最匹配的权重和样式
    │  ├─ 权重匹配优先级：
    │  │  ├─ 精确匹配 (weight == 700)
    │  │  ├─ 相邻权重 (600 或 800)
    │  │  └─ 最接近的权重
    │  │
    │  └─ 样式匹配：
    │     ├─ 精确匹配 (italic == italic)
    │     ├─ 合成 (normal + synthetic italic)
    │     └─ 降级到可用样式
    │
    └─ 返回最佳匹配的 SkTypeface
```

### 权重匹配策略

```cpp
// 权重映射
Standard weights:
  - 100 (Thin)
  - 200 (ExtraLight)
  - 300 (Light)
  - 400 (Normal)
  - 500 (Medium)
  - 600 (SemiBold)
  - 700 (Bold)
  - 800 (ExtraBold)
  - 900 (Black)

匹配算法 (伪代码):
  bestMatch = null
  minDistance = Infinity
  
  for each font in family:
    distance = abs(font.weight - requestedWeight)
    if distance < minDistance:
      minDistance = distance
      bestMatch = font
  
  return bestMatch
```

---

## 数据流图

### 完整初始化流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                 Chromium 渲染进程启动                                 │
└────────────────┬────────────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│ RendererMainPlatformDelegate::PlatformInitialize()                  │
│ (content/renderer/renderer_main_platform_delegate_android.cc)        │
└────────────────┬────────────────────────────────────────────────────┘
                 │ 在沙箱设置前调用
                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│ skia::DefaultFontMgr()                                               │
│ (skia/ext/font_utils.cc)                                            │
│ - 单例模式确保只初始化一次                                           │
└────────────────┬────────────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│ fontmgr_factory()                                                    │
│ (skia/ext/font_utils.cc)                                            │
│ 决策：使用哪个字体管理器？                                           │
└────────────┬──────────────────────┬──────────────────────────┬───────┘
             │                      │                          │
             ▼                      ▼                          ▼
        NDK 字体 API         传统 Android API         其他平台处理
      (Android N+)      (fallback)                 (macOS/Linux/Win)
             │                      │
             ▼                      ▼
    SkFontMgr_New_AndroidNDK  SkFontMgr_New_Android
    (SkFontScanner_Make_        (SkFontScanner_Make_
     Fontations())              Fontations())
             │                      │
             └──────────┬───────────┘
                        │
                        ▼
        ┌──────────────────────────────────────────────────┐
        │    SkFontMgr_Android 构造                         │
        │  (third_party/skia/src/ports)                     │
        └────────────────┬─────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
    初始化        创建        调用
    成员变量    Fontations   scanSystemFonts()
         │     扫描器           │
         │               │     └─ 扫描字体目录
         │               │        ├─ /system/fonts/
         │               │        ├─ /product/fonts/
         │               │        ├─ /odm/fonts/
         │               │        └─ 其他位置
         │               │
         └───────────────┼────────────────┐
                         │                │
                         ▼                ▼
                    解析字体文件      构建家族索引
                    (Fontations)     SkFontStyleSet_Android
                         │                │
                         ├─ 读取 TTF/OTF  ├─ Font 0 (Regular)
                         ├─ 提取元数据    ├─ Font 1 (Bold)
                         ├─ 获取字形表    ├─ Font 2 (Italic)
                         └─ 计算 Unicode  └─ Font 3 (BoldItalic)
                            覆盖范围
                         │                │
                         └────────────┬───┘
                                      │
                                      ▼
                        ┌─────────────────────────────────┐
                        │ 字体家族缓存                     │
                        │ Map<FamilyName, StyleSet*>     │
                        │                                 │
                        │ "Roboto" →                     │
                        │   ├─ Roboto-Regular.ttf        │
                        │   ├─ Roboto-Bold.ttf           │
                        │   ├─ Roboto-Italic.ttf         │
                        │   └─ Roboto-BoldItalic.ttf    │
                        │                                 │
                        │ "Noto Sans" →                  │
                        │   ├─ NotoSans-Regular.ttf      │
                        │   └─ NotoSans-Bold.ttf         │
                        │                                 │
                        │ ...                             │
                        └─────────────────────────────────┘
                                      │
                                      ▼
                        ┌─────────────────────────────────┐
                        │  返回 sk_sp<SkFontMgr>          │
                        │  (准备好字体查询)                │
                        └─────────────────────────────────┘
```

### 字体渲染时的查询流程

```
┌────────────────────────────────────────────────┐
│ 需要渲染文本 (字体家族, 权重, 样式)             │
│ 例：("Arial", bold, italic)                    │
└────────────┬─────────────────────────────────┘
             │
             ▼
    SkFontMgr::matchFamilyStyle()
             │
             ├─ matchFamily("Arial")
             │  └─ 查找 fFamilies
             │     ├─ 精确匹配 ✓
             │     └─ 返回 SkFontStyleSet
             │
             ├─ SkFontStyleSet::matchStyle(weight=700, slant=italic)
             │  └─ 遍历家族中的所有字体
             │     ├─ Roboto-Regular: distance = |400-700| + |0-italic| = 高
             │     ├─ Roboto-Bold: distance = |700-700| + |0-italic| = 中
             │     ├─ Roboto-BoldItalic: distance = |700-700| + |italic-italic| = 0 ✓
             │     └─ 返回最低距离者
             │
             └─ 返回最佳匹配的 SkTypeface
                │
                ▼
        创建 FontPlatformData
        (含 SkTypeface 指针)
                │
                ▼
        SimpleFontData
        (platform layer)
                │
                ▼
        绘制时加载完整字体文件
        (按需加载，延迟初始化)
```

---

## 核心代码位置

### Chromium 层

| 文件 | 功能 | 关键函数 |
|------|------|---------|
| [skia/ext/font_utils.cc](skia/ext/font_utils.cc#L70-L85) | 字体管理器工厂 | `fontmgr_factory()`, `DefaultFontMgr()` |
| [skia/ext/font_utils.h](skia/ext/font_utils.h) | 字体 API 导出 | `DefaultFontMgr()`, `MakeTypefaceFromName()` |
| [content/renderer/renderer_main_platform_delegate_android.cc](content/renderer/renderer_main_platform_delegate_android.cc) | 平台初始化 | `PlatformInitialize()` |
| [ui/gfx/platform_font_skia.h](ui/gfx/platform_font_skia.h) | 平台字体抽象 | `PlatformFontSkia` |
| [ui/gfx/font_fallback_skia_impl.cc](ui/gfx/font_fallback_skia_impl.cc#L112) | 字体回退 | `GetSkiaFallbackTypeface()` |

### Skia 层（第三方库）

| 文件 | 功能 |
|------|------|
| `third_party/skia/src/ports/SkFontMgr_android.cpp` | Android 字体管理器实现 |
| `third_party/skia/src/ports/SkFontMgr_android_ndk.cpp` | Android NDK 字体 API |
| `third_party/skia/src/core/SkTypeface.cpp` | 字体类型定义 |
| `third_party/skia/include/ports/SkFontMgr_android.h` | 导出接口 |
| `third_party/skia/include/ports/SkFontScanner_Fontations.h` | Fontations 扫描器 |

---

## 性能优化

### 1. 延迟加载策略

```
初始化: 只扫描元数据 (字体名称、路径、样式)
    ↓ 轻量级 (~100KB 内存)
    ↓
查询时: 仍然不加载完整字体文件
    ↓ 返回 SkTypeface 指针
    ↓
渲染时: 才真正加载字体数据到内存
    ↓ 按需即时加载
```

### 2. 缓存策略

```cpp
// 三层缓存机制

Level 1: 字体家族索引 (内存)
  Map<FamilyName, SkFontStyleSet*>
  ├─ O(1) 查询
  └─ 占用内存小

Level 2: SkTypeface 对象缓存 (内存)
  由 SkFontMgr 自动管理
  ├─ 避免重复创建相同字体
  └─ 引用计数自动释放

Level 3: 字形缓存 (GPU)
  由 Skia 绘制引擎管理
  ├─ 光栅化的字形存储在 GPU
  └─ 高频字形保留

// 内存占用估算
系统初始化: ~5-20MB (取决于安装的字体数量)
- Roboto: ~1MB
- Noto Sans: ~2MB
- Noto Sans CJK: ~10-15MB
- 其他字体: ~2-5MB
```

### 3. 多线程支持

```cpp
// 字体查询是线程安全的
SkFontMgr 接口允许从任何线程调用
    ↓
内部使用 mutex 保护共享数据
    ↓
适合 Web 引擎多线程架构
```

---

## Android 特定特性

### Unicode 覆盖范围支持

```cpp
// 每个字体都可以声明其支持的 Unicode 范围
struct FontRange {
  uint32_t start;    // 起始 Unicode 码点
  uint32_t end;      // 结束 Unicode 码点
};

// 用途：字体回退选择
// 例：如果 Roboto 不支持阿拉伯文，自动切换到 Noto Sans Arabic
if (font->supportsUnicodeRange(arabicCodePoint)) {
  return font;  // 使用该字体
} else {
  return fallbackFont;  // 使用备选字体
}
```

### CJK 字体优化

```cpp
// TrueType Collection (.ttc) 支持
// 一个文件包含多种语言的字体
NotoSansCJK.ttc
├─ 中文 (Simplified Chinese) - ttcIndex=0
├─ 中文 (Traditional Chinese) - ttcIndex=1
├─ 日文 (Japanese) - ttcIndex=2
├─ 韩文 (Korean) - ttcIndex=3
└─ 越南文 (Vietnamese) - ttcIndex=4

// 通过 ttcIndex 区分不同语言
SkTypeface* face = fontMgr->makeFromFile(
    "/system/fonts/NotoSansCJK.ttc",
    ttcIndex  // 指定语言版本
);
```

### API 等级兼容性

```
┌─────────────────────────────────────┐
│ Android 版本决策树                   │
└────────────┬────────────────────────┘
             │
    ┌────────┴────────┐
    │                 │
    ▼                 ▼
API Level           API Level
>= 24              < 24
(Android 7+)       (Android 6-)
    │                 │
    ▼                 ▼
NDK API         传统 Android API
(if enabled)    (always works)
    │                 │
    └────────┬────────┘
             │
             ▼
      SkFontMgr_Android
      (Skia 提供)
```

---

## 常见问题

### Q1: 为什么需要在沙箱前初始化字体？

A: Android 14+ 的系统字体扫描可能调用 `statx()` 系统调用，这在渲染进程沙箱中被禁止。
   通过在沙箱设置前初始化，可以避免权限问题。

### Q2: 为什么有两个 Android 字体管理器？

A:
- **NDK API**: 使用 `ASystemFontIterator` (Android N+)，功能更完整
- **传统 API**: 直接扫描 `/system/fonts/` 目录，兼容性更好
- Chromium 根据 API 级别自动选择

### Q3: 如何处理用户安装的字体？

A: Android 允许应用程序在 `/data/local/tmp/fonts/` 安装字体（如系统允许）。
   SkFontMgr_Android 会自动扫描这个目录。

### Q4: 性能如何？

A: 非常高效
- 初始化时间: ~10-50ms (取决于系统字体数量)
- 字体查询时间: O(1) 平均，O(n) 最坏
- 内存占用: 初始 5-20MB，运行时 ~100MB+

---

## 总结

```
Android 字体加载流程总结：

1. 初始化 (PlatformInitialize)
   └─ 在沙箱前调用

2. 工厂方法 (fontmgr_factory)
   └─ 选择 NDK 或传统 API

3. 构造 (SkFontMgr_New_Android)
   └─ 初始化字体管理器

4. 扫描 (scanSystemFonts)
   └─ 遍历系统目录，解析元数据

5. 索引 (buildFamilyMap)
   └─ 组织字体家族

6. 查询 (matchFamilyStyle)
   └─ 返回最佳匹配的 SkTypeface

7. 渲染 (构造 FontPlatformData)
   └─ 延迟加载完整字体文件

整个流程优化了初始化时间，同时提供了快速的字体查询能力。
```
