# Android Chromium：从 HTML 到字体获取的完整链路

## 总体流程图

```
HTML 解析
  ↓
CSS 样式应用
  ↓
FontDescription 创建
  ↓
FontBuilder 处理 (设置基础字体)
  ↓
FontSelector 字体选择 (查询用户配置或系统字体)
  ↓
FontFallbackList 字体列表遍历 (按优先级查找)
  ↓
FontCache 查询系统字体 (调用 SkFontMgr)
  ↓
SkFontMgr_New_Android 查询 /system/etc/fonts.xml
  ↓
实际字体文件加载
  ↓
渲染到屏幕
```

---

## 详细链路分析

### 第一阶段：HTML 和 CSS 解析

**入口**：`HTMLParser` → `CSSPropertyParser`

**代码位置**：
- `third_party/blink/renderer/core/html/parser/`
- `third_party/blink/renderer/core/css/`

**流程**：
```html
<p>Hello World</p>
<!-- 无 style 属性 -->
```

1. **HTML 解析器** 创建 DOM 节点
2. **CSS 解析器** 应用样式表规则
3. **没有 CSS font-family** → 继承父元素或使用初始值

---

### 第二阶段：FontDescription 构建

**关键类**：`FontDescription`

**代码位置**：`third_party/blink/renderer/platform/fonts/font_description.h`

```cpp
class FontDescription {
  FontFamily family_;           // 用户指定的字体列表（如 "Arial, sans-serif"）
  GenericFamilyType generic_family_;  // 标记：kStandardFamily/kSerifFamily 等
  float size_;                  // 字体大小
  FontWeight weight_;           // 粗细
  FontStyle style_;             // 斜体
  // ... 其他属性
};
```

**当 CSS 无字体指定时**：
- `family_` = 空字符串
- `generic_family_` = `kStandardFamily`（标记为"标准"通用族）

---

### 第三阶段：FontBuilder 处理

**关键函数**：`FontBuilder::StandardFontFamily()`

**代码位置**：`third_party/blink/renderer/core/css/resolver/font_builder.cc:61-76`

```cpp
FontFamily FontBuilder::StandardFontFamily() const {
  const AtomicString& standard_font_family = StandardFontFamilyName();
  LOG(ERROR) << "[FONT DEBUG] STANDARD:" << standard_font_family;
  return FontFamily(standard_font_family,
                    FontFamily::InferredTypeFor(standard_font_family));
}

AtomicString FontBuilder::StandardFontFamilyName() const {
  if (document_) {
    Settings* settings = document_->GetSettings();
    if (settings) {
      return settings->GetGenericFontFamilySettings().Standard();
      // ↑ 查询用户配置
    }
  }
  return AtomicString();
}
```

**流程**：
```
FontBuilder::StandardFontFamilyName()
  ↓
document_->GetSettings()
  ↓
Settings::GetGenericFontFamilySettings()
  ↓
GenericFontFamilySettings::Standard(script)
  ↓
GenericFontFamilyForScript(standard_font_family_map_)
  ↓
查询 standard_font_family_map_[script]
  ↓
如果为空 → 返回 g_empty_atom
```

**问题所在**：
- `standard_font_family_map_` 来自 `WebPreferences`
- `WebPreferences` 的默认值是硬编码的 **Times New Roman**（web_preferences.cc:25）
- 不是系统配置，也不是用户在 Android 设置里改的字体

---

### 第四阶段：WebPreferences 默认值（核心问题！）

**代码位置**：`third_party/blink/common/web_preferences/web_preferences.cc:20-38`

```cpp
WebPreferences::WebPreferences() {
  standard_font_family_map[web_pref::kCommonScript] = u"Times New Roman";
  // ↑ 硬编码的默认值，对所有平台都一样！
  
  serif_font_family_map[web_pref::kCommonScript] = u"Times New Roman";
  sans_serif_font_family_map[web_pref::kCommonScript] = u"Arial";
  fixed_font_family_map[web_pref::kCommonScript] = u"Courier New";
  cursive_font_family_map[web_pref::kCommonScript] = u"Script";
  fantasy_font_family_map[web_pref::kCommonScript] = u"Impact";
  math_font_family_map[web_pref::kCommonScript] = u"Latin Modern Math";
}
```

**这就是根本问题**：
- 无论用户在 Android 设置里改什么，Chromium 仍然用这些硬编码默认值
- Android UI 字体跟随系统，但网页字体用的是硬编码的 Times New Roman 等

---

### 第五阶段：FontSelector 字体选择

**关键函数**：`FontSelector::FamilyNameFromSettings()`

**代码位置**：`third_party/blink/renderer/platform/fonts/font_selector.cc:20-102`

```cpp
AtomicString FontSelector::FamilyNameFromSettings(
    const GenericFontFamilySettings& settings,
    const FontDescription& font_description,
    const FontFamily& generic_family,
    UseCounter* use_counter) {
  
  // Android 特殊处理！
  #if BUILDFLAG(IS_ANDROID)
    if (font_description.GenericFamily() == FontDescription::kStandardFamily) {
      return FontCache::GetGenericFamilyNameForScript(
          font_family_names::kWebkitStandard,
          GetFallbackFontFamily(font_description),  // ← 这里的 fallback 就是 "Times New Roman"
          font_description);
      // ↑ 直接调用 Android 特殊路径，绕过 settings！
    }
  #else
    // 其他平台：查询 settings
    if (font_description.GenericFamily() == FontDescription::kStandardFamily)
      return settings.Standard(script);
  #endif
}
```

**Android vs 其他平台**：
- **Android**：`FontCache::GetGenericFamilyNameForScript()` → 系统字体查询
- **其他平台**：`settings.Standard()` → 用户配置查询

---

### 第六阶段：FontCache::GetGenericFamilyNameForScript() - Android 特殊路径

**代码位置**：`third_party/blink/renderer/platform/fonts/android/font_cache_android.cc:231-287`

```cpp
AtomicString FontCache::GetGenericFamilyNameForScript(
    const AtomicString& family_name,              // = "webkit-standard"
    const AtomicString& generic_family_name_fallback,  // = "Times New Roman"
    const FontDescription& font_description) {
  
  // Step 1: 检查 locale
  const LayoutLocale* content_locale = font_description.Locale();
  if (!content_locale)
    return generic_family_name_fallback;  // 返回 "Times New Roman"
  
  // Step 2: 如果是 CJK，用示例字符查询
  UChar32 exampler_char;
  switch (content_locale->GetScript()) {
    case USCRIPT_SIMPLIFIED_HAN:  // 中文
    case USCRIPT_TRADITIONAL_HAN:
    case USCRIPT_KATAKANA_OR_HIRAGANA:  // 日文
      exampler_char = 0x4E00;  // "中" 字
      break;
    case USCRIPT_HANGUL:  // 韩文
      exampler_char = 0xAC00;
      break;
    default:
      return generic_family_name_fallback;  // 非 CJK → 返回 "Times New Roman"
  }
  
  // Step 3: 用 SkFontMgr 查询能否显示该字符的字体
  Bcp47Vector locales = GetBcp47LocaleForRequest(...);
  sk_sp<SkTypeface> typeface(
      skia::DefaultFontMgr()->matchFamilyStyleCharacter(
          nullptr,                    // 族名为 NULL
          SkFontStyle(),
          locales.data(),
          locales.size(),
          exampler_char));            // 查找支持 0x4E00 的字体
  
  if (!typeface) {
    return generic_family_name_fallback;  // 找不到 → 返回 "Times New Roman"
  }
  
  // Step 4: 获取查询到的字体名
  SkString skia_family_name;
  typeface->getFamilyName(&skia_family_name);
  return ToAtomicString(skia_family_name);  // 返回 "Noto Sans CJK SC" 或类似
}
```

**关键问题**：
1. **CJK 脚本**：用 0x4E00 查询 → Skia 在 fonts.xml 中找到支持的字体 → **NotoSansCJK**
2. **非 CJK 脚本**：直接返回 `generic_family_name_fallback` → **Times New Roman**
3. **Times New Roman 在 Android 上不存在** → 查询失败 → 回退到系统默认（Roboto 或 Noto Sans）

---

### 第七阶段：SkFontMgr 系统字体查询

**代码位置**：`skia/ext/font_utils.cc:67-83`

```cpp
static sk_sp<SkFontMgr> fontmgr_factory() {
  #if BUILDFLAG(IS_ANDROID)
    // 优先使用 NDK API（Android 14+）
    if (base::FeatureList::IsEnabled(kUseAndroidNDKFontAPI) &&
        android_get_device_api_level() > __ANDROID_API_V__) {
      sk_sp<SkFontMgr> ndk_fontmgr =
          SkFontMgr_New_AndroidNDK(false, SkFontScanner_Make_Fontations());
      if (ndk_fontmgr && ndk_fontmgr->countFamilies()) {
        return ndk_fontmgr;
      }
    }
    // 回退到传统 Android API
    return SkFontMgr_New_Android(nullptr, SkFontScanner_Make_Fontations());
    // ↑ 读取 /system/etc/fonts.xml
  #endif
}
```

**SkFontMgr_New_Android 做什么**：
1. 扫描 `/system/etc/fonts.xml`
2. 构建族名 → 字体文件的映射
3. 当调用 `matchFamilyStyleCharacter(nullptr, ..., 0x4E00)` 时：
   - 族名为 NULL → 使用默认族（通常是 sans-serif）
   - 查找支持 0x4E00 的字体
   - 从上到下遍历 fallback 链
   - 找到 → 返回 `NotoSansCJK-Regular.ttc`（根据 locale 返回对应 index）

---

### 第八阶段：FontFallbackList 最后的降级

**代码位置**：`third_party/blink/renderer/platform/fonts/font_fallback_list.cc:149-197`

```cpp
const FontData* FontFallbackList::GetFontData(
    const FontDescription& font_description) {
  
  // 遍历所有指定的 font-family
  for (curr_family = &font_description.Family(); curr_family; ...) {
    // 尝试查询 FontSelector
    result = font_selector_->GetFontData(font_description, *curr_family);
    
    // 如果 FontSelector 失败，尝试 FontCache
    if (!result && !curr_family->FamilyName().empty()) {
      result = FontCache::Get().GetFontData(font_description,
                                            curr_family->FamilyName());
    }
    
    if (result) return result;  // 找到 → 返回
  }
  
  // 都失败了？尝试通用族
  if (font_selector_) {
    FontFamily generic_family(font_family_names::kWebkitStandard,
                              FontFamily::Type::kGenericFamily);
    if (const FontData* data =
        font_selector_->GetFontData(font_description, generic_family)) {
      return data;
    }
  }
  
  // 最后的后备方案
  return FontCache::Get().GetLastResortFallbackFont(font_description);
}
```

---

## 数据流总结

### 情况 1：无 CSS 字体 + CJK 内容（如中文）

```
<p>你好</p>  （无 style）
  ↓
FontDescription::family = ""
FontDescription::generic_family = kStandardFamily
  ↓
FontBuilder::StandardFontFamilyName()
  → Settings.Standard() 
  → "Times New Roman"
  ↓
FontSelector::FamilyNameFromSettings() [Android]
  → FontCache::GetGenericFamilyNameForScript()
  → 检测到 CJK，用 0x4E00 查询
  → SkFontMgr->matchFamilyStyleCharacter(nullptr, 0x4E00)
  ↓
SkFontMgr 扫描 fonts.xml
  → 找到支持 0x4E00 的字体
  → 返回 "Noto Sans CJK SC"（或 JP/KR 版本）
  ↓
加载 NotoSansCJK-Regular.ttc
```

**结果**：网页显示 **Noto Sans CJK SC**（不是 Times New Roman！）

### 情况 2：无 CSS 字体 + 非 CJK 内容（如英文）

```
<p>Hello World</p>  （无 style）
  ↓
FontDescription::family = ""
FontDescription::generic_family = kStandardFamily
  ↓
FontBuilder::StandardFontFamilyName()
  → Settings.Standard() 
  → "Times New Roman"
  ↓
FontSelector::FamilyNameFromSettings() [Android]
  → FontCache::GetGenericFamilyNameForScript()
  → 检测到非 CJK，直接返回
  → "Times New Roman"
  ↓
FontCache::GetFontData("Times New Roman")
  → SkFontMgr->matchFamily("Times New Roman")
  → fonts.xml 中没有 "Times New Roman"
  → 查询失败，返回 nullptr
  ↓
FontFallbackList 降级
  → 尝试 serif 族（XML 中 serif 是 NotoSerif）
  或
  → GetLastResortFallbackFont() → Roboto
  ↓
加载 NotoSerif-Regular.ttf 或 Roboto-Regular.ttf
```

**结果**：网页显示 **NotoSerif** 或 **Roboto**（不是 Times New Roman！）

---

## 修改方案

### ✅ 方案 1：改 WebPreferences 默认值（推荐）

**文件**：`third_party/blink/common/web_preferences/web_preferences.cc:25`

```cpp
#if BUILDFLAG(IS_ANDROID)
  standard_font_family_map[web_pref::kCommonScript] = u"sans-serif";
#else
  standard_font_family_map[web_pref::kCommonScript] = u"Times New Roman";
#endif
```

**效果**：
- Android：网页使用 `sans-serif` → Roboto（和 UI 一致）
- 其他平台：保持原有行为

---

### ✅ 方案 2：改 FontCache::GetGenericFamilyNameForScript() - 跳过 CJK Hack

**文件**：`third_party/blink/renderer/platform/fonts/android/font_cache_android.cc:231`

```cpp
// 直接返回传入的 fallback，不跑 CJK 查询
return generic_family_name_fallback;
```

**效果**：
- 跳过 CJK 特殊处理
- 尊重 WebPreferences 的配置

---

## 完整调用栈示例

```
LayoutObject::StyleDidChange()
  → ComputedStyle::Font() 
  → FontBuilder::CreateFont()
  → FontBuilder::GetComputedFont()
  → FontBuilder::StandardFontFamily()  ← 关键！
    → StandardFontFamilyName()
    → Settings::GetGenericFontFamilySettings().Standard()
    → GenericFontFamilySettings::GenericFontFamilyForScript()
    → 返回 "Times New Roman" (来自 WebPreferences)
  → FontSelector::GetFontData()  ← Android 特殊路径
    → FontSelector::FamilyNameFromSettings()
    → FontCache::GetGenericFamilyNameForScript()
      → content_locale->GetScript() 检测脚本
      → SkFontMgr->matchFamilyStyleCharacter(nullptr, ..., 0x4E00)
      → 返回 "Noto Sans CJK SC"（如果是 CJK）
      → 或返回 "Times New Roman"（如果是非 CJK，但查询失败）
  → FontCache::GetFontData()
    → GetFontPlatformData()
    → 加载字体文件
  → SimpleFontData 创建
  → 渲染
```

---

## 关键发现

| 阶段 | 发现 | 影响 |
|------|------|------|
| WebPreferences | 硬编码 Times New Roman 等 | **所有平台**都用这些默认值 |
| Android 特殊路径 | CJK Hack 用 0x4E00 查询 | CJK 内容强制用 NotoSansCJK |
| fonts.xml | "Times New Roman" 不存在 | 非 CJK 查询失败，降级到 Noto/Roboto |
| Chromium UI | 单独的配置（未调查） | UI 字体和网页字体不一致 |

---

## 要点总结

1. **没有 CSS 字体 → FontDescription.family 为空 → 查询 kStandardFamily**
2. **kStandardFamily 的值来自 WebPreferences，不是用户设置**
3. **Android 的 CJK Hack 会强制匹配 CJK 特定字体（NotoSansCJK）**
4. **其他内容（英文等）则用 Times New Roman，但它在 Android 上不存在，导致降级**
5. **要让网页字体跟随系统，需要修改 WebPreferences 的默认值或改 CJK Hack 逻辑**
