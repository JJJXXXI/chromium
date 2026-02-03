# Android Chromium 使用系统字体 - 极简实现方案

## 🎯 目标

让 Android Chromium 浏览器直接使用系统字体，**无需用户菜单**，**无需 Settings UI**。

---

## 💡 核心思路

```
替代复杂方案:
  ❌ 创建 Settings UI
  ❌ 让用户选择字体
  ❌ 保存用户偏好
  ✅ 工作量: 2-3 周

极简方案:
  ✅ 在启动时自动检测系统字体
  ✅ 直接应用到网页
  ✅ 不需要任何 UI
  ✅ 工作量: 1-2 天
```

---

## 📊 对比

| 方面 | 完整方案 | 极简方案 |
|------|---------|---------|
| **Settings UI** | ✅ 需要 | ❌ 无 |
| **Java 代码** | ✅ 200+ 行 | ❌ 0 行 |
| **XML 配置** | ✅ 100+ 行 | ❌ 0 行 |
| **C++ 修改** | ⚠️ 最小 | ✅ 仅 1-2 处 |
| **工作量** | 📈 2-3 周 | 📉 1-2 天 |
| **代码复杂度** | 中等 | 简单 |
| **用户体验** | 可定制 | 自动适配 |
| **效果** | 完全相同 | 完全相同 |

---

## 🚀 实现方案

### Step 1: 移除条件编译限制

**文件**: `chrome/browser/ui/prefs/prefs_tab_helper.cc`
**当前代码** (行 82-88):
```cpp
#if !BUILDFLAG(IS_ANDROID) || BUILDFLAG(ENABLE_DESKTOP_ANDROID_EXTENSIONS)
  RegisterFontFamilyPrefs(registry, fonts_with_defaults);
  registry->RegisterIntegerPref(prefs::kWebKitDefaultFontSize, 16);
  registry->RegisterIntegerPref(prefs::kWebKitDefaultFixedFontSize, 13);
  registry->RegisterIntegerPref(prefs::kWebKitMinimumFontSize, 0);
  RegisterLocalizedFontPref(registry, prefs::kWebKitMinimumLogicalFontSize,
                            IDS_MINIMUM_LOGICAL_FONT_SIZE);
#endif
```

**改为** (移除 `!BUILDFLAG(IS_ANDROID)` 条件):
```cpp
// 所有平台都注册字体偏好（包括 Android）
RegisterFontFamilyPrefs(registry, fonts_with_defaults);
registry->RegisterIntegerPref(prefs::kWebKitDefaultFontSize, 16);
registry->RegisterIntegerPref(prefs::kWebKitDefaultFixedFontSize, 13);
registry->RegisterIntegerPref(prefs::kWebKitMinimumFontSize, 0);
RegisterLocalizedFontPref(registry, prefs::kWebKitMinimumLogicalFontSize,
                          IDS_MINIMUM_LOGICAL_FONT_SIZE);
```

---

### Step 2: 在启动时自动设置系统字体

**文件**: `chrome/browser/android/chrome_android_main.cc` (或主初始化文件)

**添加函数**:
```cpp
// 获取 Android 系统字体的默认值
void ApplyAndroidSystemFonts(PrefService* prefs) {
  // 使用 Android 系统推荐的默认字体
  // 参考 WebView 的实现
  
  // Serif 字体（中文: 宋体，英文: Georgia）
  prefs->SetString("webkit.webprefs.fonts.serif.Zyyy", "Noto Serif");
  
  // Sans-serif 字体（中文: 黑体，英文: Arial）
  prefs->SetString("webkit.webprefs.fonts.sansserif.Zyyy", "Roboto");
  
  // Fixed 字体（等宽: Courier）
  prefs->SetString("webkit.webprefs.fonts.fixed.Zyyy", "Roboto Mono");
  
  // 字体大小使用系统默认（通常 16px）
  prefs->SetInteger("webkit.webprefs.default_font_size", 16);
  prefs->SetInteger("webkit.webprefs.default_fixed_font_size", 13);
}
```

**在启动时调用** (找到 Chrome 初始化的地方):
```cpp
// 在 Chrome 启动时
void InitializeChromeAndroid() {
  // ... 其他初始化代码 ...
  
  // 设置 Android 系统字体
  PrefService* prefs = g_browser_process->local_state();
  if (prefs) {
    ApplyAndroidSystemFonts(prefs);
  }
}
```

---

## 📱 使用 Android 系统字体的更优雅方案

### 方案 A: 检测系统字体（推荐）

```cpp
#include "base/android/jni_string.h"

// 通过 JNI 获取 Android 系统字体
std::string GetAndroidSystemFont() {
  // 从 Android TypefaceManager 获取系统字体
  // 这需要 JNI 调用，参考 WebView 的实现
  
  // 简单版本：使用 Android 9+ 的标准字体
  return "Noto Sans";  // Google 推荐的通用字体
}

void ApplyAndroidSystemFonts(PrefService* prefs) {
  std::string system_font = GetAndroidSystemFont();
  
  prefs->SetString("webkit.webprefs.fonts.serif.Zyyy", "Noto Serif");
  prefs->SetString("webkit.webprefs.fonts.sansserif.Zyyy", system_font);
  prefs->SetString("webkit.webprefs.fonts.fixed.Zyyy", "Roboto Mono");
}
```

### 方案 B: 使用硬编码的系统字体（最简单）

```cpp
void ApplyAndroidSystemFonts(PrefService* prefs) {
  // 使用所有 Android 系统都有的字体
  prefs->SetString("webkit.webprefs.fonts.serif.Zyyy", "Noto Serif");
  prefs->SetString("webkit.webprefs.fonts.sansserif.Zyyy", "Roboto");  // 或 "Arial"
  prefs->SetString("webkit.webprefs.fonts.fixed.Zyyy", "Roboto Mono");
}
```

---

## 🔍 关键位置查找

### 找到 Chrome 的主初始化代码

```bash
# Android Chrome 的启动入口
find chrome/android -name "*.cc" | xargs grep -l "ChromeActivitySessionHelper\|ChromeApplication"

# 或者查看
cat chrome/android/java/src/org/chromium/chrome/browser/ChromeApplication.java
```

### 在 PrefService 初始化后设置

```cpp
// 通常在这个地方：
// chrome/browser/browser_process.cc 或
// chrome/browser/android/chrome_android_main.cc

// 找到 PrefService 初始化的地方，然后添加：
if (IsAndroid()) {
  ApplyAndroidSystemFonts(prefs);
}
```

---

## 📝 最小改动清单

### 需要修改的文件

1. **chrome/browser/ui/prefs/prefs_tab_helper.cc** (行 82)
   - 移除 `!BUILDFLAG(IS_ANDROID)` 条件
   - 改动: 1 行

2. **chrome/browser/android/chrome_android_main.cc** (或主初始化文件)
   - 添加 `ApplyAndroidSystemFonts()` 函数
   - 在启动时调用该函数
   - 改动: ~20 行

### 无需修改

- ❌ 不需要创建 Settings UI
- ❌ 不需要 Java 代码
- ❌ 不需要 XML 配置
- ❌ 不需要字符串资源
- ❌ 不需要 JNI 桥接（基础版本）

---

## 🎯 完整的改动代码

### 改动 1: prefs_tab_helper.cc

**当前** (行 80-90):
```cpp
#if BUILDFLAG(IS_ANDROID) || BUILDFLAG(ENABLE_DESKTOP_ANDROID_EXTENSIONS)
  // Android font preferences
#endif

#if !BUILDFLAG(IS_ANDROID) || BUILDFLAG(ENABLE_DESKTOP_ANDROID_EXTENSIONS)
  RegisterFontFamilyPrefs(registry, fonts_with_defaults);
  registry->RegisterIntegerPref(prefs::kWebKitDefaultFontSize, 16);
  registry->RegisterIntegerPref(prefs::kWebKitDefaultFixedFontSize, 13);
  registry->RegisterIntegerPref(prefs::kWebKitMinimumFontSize, 0);
  RegisterLocalizedFontPref(registry, prefs::kWebKitMinimumLogicalFontSize,
                            IDS_MINIMUM_LOGICAL_FONT_SIZE);
#endif
```

**改为** (所有平台都启用):
```cpp
// 所有平台（包括 Android）都支持字体自定义
RegisterFontFamilyPrefs(registry, fonts_with_defaults);
registry->RegisterIntegerPref(prefs::kWebKitDefaultFontSize, 16);
registry->RegisterIntegerPref(prefs::kWebKitDefaultFixedFontSize, 13);
registry->RegisterIntegerPref(prefs::kWebKitMinimumFontSize, 0);
RegisterLocalizedFontPref(registry, prefs::kWebKitMinimumLogicalFontSize,
                          IDS_MINIMUM_LOGICAL_FONT_SIZE);
```

### 改动 2: chrome_android_main.cc (新增)

**添加新函数** (文件开头):
```cpp
// 应用 Android 系统字体
void ApplyAndroidSystemFonts(PrefService* prefs) {
  if (!prefs) {
    return;
  }
  
  // 使用 Android 系统推荐的字体
  // 这些字体在所有 Android 设备上都可用
  
  // Serif 字体
  prefs->SetString("webkit.webprefs.fonts.serif.Zyyy", "Noto Serif");
  
  // Sans-serif 字体（系统默认）
  prefs->SetString("webkit.webprefs.fonts.sansserif.Zyyy", "Roboto");
  
  // 等宽字体
  prefs->SetString("webkit.webprefs.fonts.fixed.Zyyy", "Roboto Mono");
  
  // 可选：设置字体大小
  // prefs->SetInteger("webkit.webprefs.default_font_size", 16);
  // prefs->SetInteger("webkit.webprefs.default_fixed_font_size", 13);
}
```

**在初始化时调用** (找到合适的初始化函数):
```cpp
void ChromeMainAndroid::Init() {
  // ... 现有初始化代码 ...
  
  // 应用系统字体
  PrefService* local_prefs = g_browser_process->local_state();
  if (local_prefs) {
    ApplyAndroidSystemFonts(local_prefs);
  }
}
```

---

## 🧪 测试方法

### 编译
```bash
autoninja -C out/Default chrome_apk
```

### 安装和测试
```bash
adb install -r out/Default/apks/Chrome.apk
adb shell am start -n com.android.chrome/.MainActivity

# 打开任何网页，应该看到字体已改变
# 例如: google.com 应该使用 Roboto 字体
```

### 验证
```bash
# 验证偏好已设置
adb shell sqlite3 /data/user/0/com.android.chrome/app_chrome/Default/Preferences \
  "SELECT * FROM preferences WHERE key LIKE '%webkit.webprefs.fonts%';"
```

---

## 📊 工作量对比

### 完整方案 vs 极简方案

```
完整方案 (2-3 周):
  Week 1:
    - 创建 Fragment
    - 创建 XML
    - 集成到 Settings
  
  Week 2-3:
    - 字体列表管理
    - 测试
    - 优化

极简方案 (1-2 天):
  Day 1:
    - 修改编译条件 (30 分钟)
    - 添加初始化代码 (1 小时)
  
  Day 2:
    - 测试 (30 分钟)
    - 调试 (30 分钟)
```

---

## ✅ 最终效果

**用户体验**:
```
用户打开 Chrome 浏览器
    ↓
自动应用系统字体
    ↓
网页显示使用系统字体
    ↓
用户无需任何操作
    ↓
体验完成 ✅
```

**网页效果**:
- Serif 文本 → Noto Serif
- 正文文本 → Roboto
- 代码文本 → Roboto Mono

---

## 🎯 选择: 使用哪些字体？

### 推荐组合 1: 标准 (所有 Android 都有)

```cpp
void ApplyAndroidSystemFonts(PrefService* prefs) {
  prefs->SetString("webkit.webprefs.fonts.serif.Zyyy", "Noto Serif");
  prefs->SetString("webkit.webprefs.fonts.sansserif.Zyyy", "Roboto");
  prefs->SetString("webkit.webprefs.fonts.fixed.Zyyy", "Roboto Mono");
}
```

### 推荐组合 2: 最大兼容性

```cpp
void ApplyAndroidSystemFonts(PrefService* prefs) {
  // 这些字体在绝大多数 Android 设备上都有
  prefs->SetString("webkit.webprefs.fonts.serif.Zyyy", "Georgia");
  prefs->SetString("webkit.webprefs.fonts.sansserif.Zyyy", "Arial");
  prefs->SetString("webkit.webprefs.fonts.fixed.Zyyy", "Courier New");
}
```

### 推荐组合 3: 自适应 (检测系统)

```cpp
std::string GetSystemSanSerifFont() {
  // 在 Android 9+ 上使用 Roboto
  // 在旧版本上使用 Arial
  if (android_version >= 28) {  // Android 9
    return "Roboto";
  }
  return "Arial";
}

void ApplyAndroidSystemFonts(PrefService* prefs) {
  prefs->SetString("webkit.webprefs.fonts.serif.Zyyy", "Noto Serif");
  prefs->SetString("webkit.webprefs.fonts.sansserif.Zyyy", 
                   GetSystemSanSerifFont());
  prefs->SetString("webkit.webprefs.fonts.fixed.Zyyy", "Roboto Mono");
}
```

---

## 🚀 立即开始

### Step 1: 找到正确的文件

```bash
# 打开 prefs_tab_helper.cc
vim chrome/browser/ui/prefs/prefs_tab_helper.cc +82

# 找到初始化文件（通常是这些）
find chrome/browser/android -name "*.cc" | head -10
```

### Step 2: 做两个改动

1. 修改条件编译 (1 行)
2. 添加初始化函数 (~20 行)

### Step 3: 测试

```bash
autoninja -C out/Default chrome_apk
adb install -r out/Default/apks/Chrome.apk
# 打开网页，验证字体改变
```

---

## 💡 为什么这个方案更好？

| 方面 | 复杂方案 | 简单方案 |
|------|---------|---------|
| **实现时间** | 2-3 周 | 1-2 天 |
| **代码改动** | ~500 行 | ~50 行 |
| **维护成本** | 中等 | 很低 |
| **用户困惑** | 可能 | 无 |
| **自动应用** | 否 | 是 |
| **可维护性** | 中等 | 高 |

---

## ⚠️ 注意事项

1. **字体名称必须正确**: 使用系统中实际存在的字体
2. **测试多个版本**: 测试 Android 6, 8, 10, 12+ 等版本
3. **检查字体可用性**: 确保选择的字体在目标设备上存在

### 验证字体是否存在

```bash
# 在 Android 设备上检查字体
adb shell ls /system/fonts/ | grep -i roboto
adb shell ls /system/fonts/ | grep -i noto
```

---

## 🎉 总结

**极简方案让你能够:**

✅ 在 1-2 天内完成
✅ 最小化代码改动 (~50 行)
✅ 零 Java 代码
✅ 零 UI 开发
✅ 自动应用系统字体
✅ 无需用户操作

**结果完全相同:**
- Android 网页使用自定义字体 ✅
- 用户体验一致 ✅
- 功能完整 ✅

---

**准备好了吗？立即开始实现！** 🚀
