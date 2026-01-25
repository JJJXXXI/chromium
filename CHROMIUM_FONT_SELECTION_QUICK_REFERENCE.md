# Chromium 无 CSS 字体选择 - 快速参考

## 一句话总结
无 CSS 时,字体经过 **初始值 → 通用族 → 系统查询 → 延迟加载** 四个阶段。

---

## 代码追踪速查表

### 🟢 Stage 1: CSS 解析与转换

```
HTML 元素 (无 font-family CSS)
  └─ StyleResolver::ResolveStyle()
      └─ StyleCascade::Apply()
          ├─ ✓ CSS 有 font-family
          │   └─ StyleBuilder::ApplyProperty(kFontFamily, ...)
          │       └─ StyleBuilderConverter::ConvertFontFamily()
          │           └─ CSSValue → FontDescription::FamilyDescription
          │
          └─ ✗ CSS 无 font-family
              └─ FontDescription 初始值保留 (kNoFamily)
```

**关键代码行**:
- `style_builder_converter.cc:510` - `ConvertFontFamily()` 入口
- `style_builder_converter.cc:470` - `ConvertFontFamilyName()` 转换单个值
- `style_builder_converter.cc:590` - 构建 FontFamily 链表

---

### 🟠 Stage 2: 初始值应用

```
FontBuilder::CreateFont()
  ├─ FontDescription description = builder.GetFontDescription()
  ├─ UpdateFontDescription()
  │   └─ 若 generic_family == kNoFamily
  │       └─ SetGenericFamily(InitialGenericFamily())  // → kStandardFamily
  └─ builder.SetFont(new Font(description, font_selector))
```

**关键代码行**:
- `font_builder.cc:653` - `CreateFont()` 函数
- `font_builder.h:138` - `InitialGenericFamily()` 返回 `kStandardFamily`
- `font_builder.cc:680` - `ComputeFontSelector()` 选择 FontSelector

---

### 🟡 Stage 3: 通用族映射

```
FontSelector::FamilyNameFromSettings()
  ├─ 按脚本查询 GenericFontFamilySettings
  │   └─ 若表有条目
  │       └─ 返回字体名 (如 "DejaVu Sans" on Linux)
  │
  └─ 若表为空 (Android 情况)
      └─ FontCache::GetGenericFamilyNameForScript()
          └─ SkFontMgr_android::matchFamilyStyle()
              └─ 解析 fonts.xml → "sans-serif" → "Roboto"
```

**关键代码行**:
- `font_selector.cc:28` - `FamilyNameFromSettings()` 入口
- `font_selector.cc:68` - 查询 GenericFontFamilySettings
- `font_cache_android.cc:*` - Android 系统查询
- `third_party/skia/src/ports/SkFontMgr_android.cpp` - Skia fonts.xml 解析

---

### 🔵 Stage 4: 延迟字体匹配

```
Paint/Layout 调用 Font::LineHeight()
  └─ Font::PrimaryFont()
      ├─ 若 !font_list_
      │   └─ FontFallbackList::Create(this, description_, font_selector_)
      │       └─ 初始化 FontFallbackList::primary_font_
      │
      └─ FontFallbackList::PrimaryFont()
          └─ FontFallbackList::GetFontData(family)
              └─ CSSFontSelector::GetFontData()
                  ├─ FontFaceCache::Get()  // @font-face
                  └─ FontCache::GetFontData()  // 系统字体
                      └─ 加载 Roboto.ttf → SimpleFontData
```

**关键代码行**:
- `font.h:*` - `Font::PrimaryFont()` 定义
- `font_fallback_list.cc:*` - `Create()`, `PrimaryFont()` 实现
- `css_font_selector.cc:*` - `GetFontData()` 查询逻辑
- `font_cache.cc:*` - `GetFontData()` 加载逻辑

---

## GDB 调试命令

```bash
# 打印 FontDescription 内容
(gdb) p *description
(gdb) p description->generic_family_

# 打印 Font 对象内容
(gdb) p *font
(gdb) p font->font_selector_
(gdb) p font->font_list_

# 打印 FontFallbackList
(gdb) p *font->font_list_
(gdb) p font->font_list_->primary_font_

# 在关键函数设置断点
(gdb) break FontBuilder::CreateFont
(gdb) break CSSFontSelector::GetFontData
(gdb) break FontCache::GetFontData

# 条件断点: 仅当 generic_family == kNoFamily 时停止
(gdb) break font_builder.cc:680 if description->generic_family_ == 0
```

---

## LOG 调试输出

```cpp
// 在 style_builder_converter.cc
LOG(INFO) << "ConvertFontFamily: CSSValue=" 
          << (font_family_value ? font_family_value->Value() : "generic");

// 在 font_builder.cc CreateFont()
LOG(INFO) << "FontBuilder::CreateFont: "
          << "generic_family=" << description.GenericFamily()
          << ", family=" << (description.GenericFamily() == FontDescription::kNoFamily 
                             ? "INITIAL" : "SET");

// 在 font_selector.cc
LOG(INFO) << "FamilyNameFromSettings: script=" << script
          << ", mapped_name=" << mapped_name;

// 在 font_fallback_list.cc
LOG(INFO) << "FontFallbackList::PrimaryFont: "
          << "first_call=" << (!primary_font_)
          << ", family=" << family_name;

// 在 css_font_selector.cc
LOG(INFO) << "CSSFontSelector::GetFontData: "
          << "family=" << family_name
          << ", found_web_font=" << (web_font ? "yes" : "no")
          << ", fallback_to_system=" << (system_font ? "yes" : "no");
```

---

## 变量定义快查

| 变量 | 类型 | 定义文件 | 含义 |
|------|------|---------|------|
| `kNoFamily` | enum | `font_description.h` | "未指定通用族" |
| `kStandardFamily` | enum | `font_description.h` | "标准族" (sans-serif) |
| `kSerifFamily` | enum | `font_description.h` | "衬线族" |
| `FamilyDescription` | struct | `font_description.h` | 族信息容器 |
| `FontFamily` | class | `font_family.h` | 字体名链表节点 |
| `CSSFontSelector` | class | `css_font_selector.h` | CSS 字体选择器 |
| `FontFallbackList` | class | `font_fallback_list.h` | 字体降级链 |
| `SimpleFontData` | class | `font_data.h` | 实际字体文件数据 |

---

## 常见断点位置

| 调试目标 | 文件 | 函数 | 目的 |
|---------|------|------|------|
| CSS 是否被应用 | `style_builder_converter.cc` | `ConvertFontFamily` | 看 CSSValue 的类型和值 |
| 初始值是否被设置 | `font_builder.cc` | `CreateFont` | 检查 `generic_family_` 的值变化 |
| 映射表是否为空 | `font_selector.cc` | `FamilyNameFromSettings` | 看 `mapped_name` 是否为空 |
| 系统查询是否触发 | `font_cache_android.cc` | `GetGenericFamilyNameForScript` | 看是否进入 Android 特定路径 |
| 字体是否加载 | `font_cache.cc` | `GetFontData` | 看 `SimpleFontData` 是否为 null |
| 延迟匹配何时触发 | `font_fallback_list.cc` | `PrimaryFont` | 看 `!font_list_` 条件何时为真 |
| Fallback 链构建 | `font_fallback_list.cc` | `GetFontData` | 看是否调用 fallback 机制 |

---

## Android 特定路径

### fonts.xml 位置
- `/system/etc/fonts.xml` (系统级)
- `/product/etc/fonts.xml` (产品级)

### 解析流程
```
fonts.xml
  ├─ <alias name="standard" to="sans-serif" />
  └─ <family name="sans-serif">
      └─ <font weight="400" style="normal">Roboto-Regular.ttf</font>
```

### 代码路径
```
SkFontMgr_android::matchFamilyStyle("standard", style)
  └─ 加载并解析 fonts.xml
      ├─ 查找 <alias name="standard">
      └─ 追踪到 <family name="sans-serif">
          └─ 选择最匹配的字体文件
              └─ 返回 Roboto-Regular.ttf 的 SkTypeface
```

---

## 性能分析

| 操作 | 发生时机 | 成本 | 优化建议 |
|------|---------|------|---------|
| CSS 解析 | Style Calculation | 低 | 缓存 CSSValue |
| GenericFamilySettings 查询 | Style Calculation | 低 | 预加载到内存 |
| SkFontMgr fonts.xml 查询 | Paint Time (延迟) | **中** | 缓存 fonts.xml 结果 |
| TTF 文件加载 | Paint Time (延迟) | **高** | 系统级字体缓存 |
| HarfBuzz Shaping | Paint Time (延迟) | **中** | 缓存 ShapeResult |
| Skia 光栅化 | Paint Time (延迟) | **中** | GPU 加速 |

---

## 参考资源

- **架构**: `third_party/blink/renderer/platform/fonts/README.md`
- **完整流程**: `COMPLETE_FONT_PROCESSING_FLOW.md`
- **详细文档**: `NO_CSS_FONT_SELECTION_DETAIL.md`
- **本文档**: `CHROMIUM_FONT_SELECTION_QUICK_REFERENCE.md`

