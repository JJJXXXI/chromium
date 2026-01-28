# Chromium 未设置 CSS 字体时的 Family 和字体获取流程

## 快速总结

| 步骤 | 变量名 | 内容 | 代码位置 |
|------|------|------|--------|
| 1️⃣ **CSS 解析** | `FontDescription::Family()` | 如果 HTML 无 `style="font-family:..."` | `CSSPropertyParser::ParseFontFamily()` |
| 2️⃣ **默认通用字体** | `FontDescription::GenericFamily()` | 返回 `kStandardFamily` (标准/sans-serif) | `CSSPropertyParser` |
| 3️⃣ **查询系统设置** | `settings.Standard()` / `settings.SansSerif()` | 查询用户配置的字体名 (空= 返回 `g_empty_atom`) | `GenericFontFamilySettings::Standard()` |
| 4️⃣ **获取系统字体 (Android)** | `FontCache::GetGenericFamilyNameForScript()` | 调用 `skia::DefaultFontMgr()->matchFamilyStyleCharacter(nullptr, ...)` | `font_cache_android.cc:231` |
| 5️⃣ **SkFontMgr 查询系统** | `SkTypeface` | 从 `/system/etc/fonts.xml` 读取系统字体 (如 "Roboto") | `skia/ext/font_utils.cc:83` |
| 6️⃣ **最终 family 名** | `family_name` (AtomicString) | 系统字体名称，如 `"Roboto"`, `"Noto Sans CJK"` | 返回给 `FontFallbackList` |
| 7️⃣ **获取字体数据** | `SimpleFontData` | 从 SkTypeface 读取字体文件，创建字体对象 | `FontCache::GetFontData()` |

---

## 详细流程图

### 第一阶段：CSS 解析 (HTML → FontDescription)

```
<html>
  <p>Hello World</p>  ← 无 style="font-family:..." 
</html>
    ↓
CSSPropertyParser::ParseFontFamily() 
    ↓
FontDescription::SetFamily() 被调用
    ↓
FontDescription::Family().FamilyName() = ""  ← 空字符串！
FontDescription::GenericFamily() = kStandardFamily  ← 标记为"标准"通用字体
```

**代码位置**: `third_party/blink/renderer/core/css/properties/css_property_parser_helpers.cc`

**关键点**: 
- 如果 HTML 没有指定 `font-family`, 那么 `FontDescription::Family()` 返回 **空字符串**
- 但 `FontDescription::GenericFamily()` 被设置为 `kStandardFamily` 来标记"这是一个通用字体族"

---

### 第二阶段：字体选择 (FontDescription → FontSelector → FontFallbackList)

#### 2a. FontFallbackList::GetFontData() - 遍历 family 列表

**文件**: `font_fallback_list.cc:149`

```cpp
const FontData* FontFallbackList::GetFontData(
    const FontDescription& font_description) {
  const FontFamily* curr_family = &font_description.Family();
  
  // 遍历所有指定的 font-family
  for (; curr_family; curr_family = curr_family->Next()) {
    if (!curr_family->FamilyName().empty()) {  // ← 跳过空字符串！
      // 查询具体的字体
    }
  }
  
  // 没有找到？使用通用字体族
  FontFamily generic_family(font_family_names::kWebkitStandard,
                           FontFamily::Type::kGenericFamily);
  if (const FontData* data = 
      font_selector_->GetFontData(font_description, generic_family)) {
    return data;  // ← 找到通用字体族的实现
  }
}
```

**注意**: 因为 `FontDescription::Family().FamilyName()` 为空，所以上面的循环会跳过它，直接跳到**通用字体族处理**。

---

#### 2b. FontSelector::GetFontData() → FamilyNameFromSettings()

**文件**: `font_selector.cc:20`

当调用 `GetFontData()` 且 `curr_family->FamilyIsGeneric()` 为 true 时：

```cpp
AtomicString FontSelector::FamilyNameFromSettings(
    const GenericFontFamilySettings& settings,
    const FontDescription& font_description,
    const FontFamily& generic_family,      // ← 例如 kWebkitStandard
    UseCounter* use_counter) {
    
  // Android 特殊逻辑
  #if BUILDFLAG(IS_ANDROID)
    if (font_description.GenericFamily() == FontDescription::kStandardFamily) {
      return FontCache::GetGenericFamilyNameForScript(
          font_family_names::kWebkitStandard,
          GetFallbackFontFamily(font_description),
          font_description);
      // ↑ 返回系统字体名，例如 "Roboto"
    }
  #else  // macOS, Windows, Linux
    UScriptCode script = font_description.GetScript();
    if (font_description.GenericFamily() == FontDescription::kStandardFamily)
      return settings.Standard(script);  
      // ↑ 查询用户配置，如果为空返回 g_empty_atom
  #endif
}
```

**关键差异**:
- **Android**: 直接调用系统 API (`skia::DefaultFontMgr`)，绕过 `GenericFontFamilySettings`
- **其他平台**: 查询 `GenericFontFamilySettings::Standard()`，如果为空则返回空

---

### 第三阶段：Android 的系统字体获取 (GetGenericFamilyNameForScript)

**文件**: `font_cache_android.cc:231`

```cpp
AtomicString FontCache::GetGenericFamilyNameForScript(
    const AtomicString& family_name,           // = "webkit-standard"
    const AtomicString& generic_family_name_fallback,
    const FontDescription& font_description) {
  
  // 对于 CJK 文字，查找特定的 CJK 字体
  UChar32 exampler_char = 0x4E00;  // "中" 字
  
  Bcp47Vector locales = GetBcp47LocaleForRequest(...);
  
  // ← 关键：查询系统字体管理器
  sk_sp<SkTypeface> typeface(
      skia::DefaultFontMgr()->matchFamilyStyleCharacter(
          nullptr,                    // ← 族名为 NULL（查询默认字体）
          SkFontStyle(),
          locales.data(),
          locales.size(),
          exampler_char               // ← 示例字符
      )
  );
  
  if (!typeface) {
    return g_empty_atom;
  }
  
  // 获取字体名
  SkString skia_family_name;
  typeface->getFamilyName(&skia_family_name);
  return ToAtomicString(skia_family_name);  // ← 例如返回 "Roboto"
}
```

**系统字体查询流程**:

```
skia::DefaultFontMgr()->matchFamilyStyleCharacter(nullptr, ...)
    ↓
SkFontMgr_New_Android() 初始化 (来自 skia/ext/font_utils.cc:83)
    ↓
扫描 /system/etc/fonts.xml (Android < 14) 或使用 ASystemFontIterator (Android ≥ 14)
    ↓
查找支持示例字符的字体
    ↓
返回最合适的字体 Typeface
```

---

### 第四阶段：获取字体数据 (family name → SimpleFontData)

**文件**: `font_cache.cc:152`

```cpp
const SimpleFontData* FontCache::GetFontData(
    const FontDescription& font_description,
    const AtomicString& family,    // ← 例如 "Roboto"
    AlternateFontName altername_font_name) {
    
  const FontPlatformData* platform_data = GetFontPlatformData(
      font_description,
      FontFaceCreationParams(AdjustFamilyNameToAvoidUnsupportedFonts(family)),
      // ↑ family = "Roboto"
  );
  
  if (platform_data) {
    return FontDataFromFontPlatformData(platform_data, ...);
    // ↑ 返回 SimpleFontData，包含字体文件、大小、样式等
  }
  
  return nullptr;
}
```

---

## 不同平台的行为对比

### Android
```
未指定 CSS 字体
    ↓
FontSelector::FamilyNameFromSettings()
    ↓
FontCache::GetGenericFamilyNameForScript()  ← 直接查询系统
    ↓
SkFontMgr_New_Android() 扫描 /system/etc/fonts.xml
    ↓
返回系统默认字体（如 "Roboto"）
    ↓
字体加载完成 ✓
```

### macOS / Windows / Linux (非 Android)
```
未指定 CSS 字体
    ↓
FontSelector::FamilyNameFromSettings()
    ↓
settings.Standard() 查询 GenericFontFamilySettings
    ↓
如果配置为空 → 返回 g_empty_atom（空字符串）
    ↓
FontFallbackList 继续降级至 fallback font
    ↓
GetLastResortFallbackFont() 返回最后的后备字体
```

---

## 修改建议

### 问题
非 Android 平台在 `GenericFontFamilySettings::Standard()` 为空时，不会使用系统字体，而是返回 `g_empty_atom`。

### 解决方案
修改 `GenericFontFamilyForScript()` 函数（`generic_font_family_settings.cc:107`）:

```cpp
// 原代码
return g_empty_atom;

// 新代码
return FontCache::SystemFontFamily();
```

这样：
1. 当用户没有配置通用字体族时，使用系统字体
2. 与 Android 行为一致
3. 网页默认字体自动跟随系统字体设置

---

## 关键数据结构

### FontDescription
```cpp
class FontDescription {
  FontFamily family_;           // 用户指定的字体列表，如 "Arial, sans-serif"
  GenericFamilyType generic_family_;  // 标记位：如 kStandardFamily
  // 其他属性：大小、加粗、斜体等
};
```

### FontFamily 链表
```cpp
class FontFamily {
  AtomicString family_name_;    // 单个字体族名，如 "Arial"
  FontFamily* next_;             // 下一个备用字体族
  bool family_is_generic_;       // 是否为通用字体族（serif, sans-serif等）
};
```

### GenericFontFamilySettings
```cpp
class GenericFontFamilySettings {
  ScriptFontFamilyMap standard_font_family_map_;  // 脚本 → 字体名映射
  // 其他：serif_font_family_map_, fixed_font_family_map_ 等
};
```

---

## 测试方法

### 验证 family 的值
在浏览器 DevTools 中：
```javascript
// 网页中添加
window.getComputedStyle(document.querySelector('p')).fontFamily
// 查看返回值，应该是系统字体名
```

### 追踪代码流
```bash
# 添加日志到 font_selector.cc:FamilyNameFromSettings()
DLOG(INFO) << "FamilyNameFromSettings returned: " << result;

# 添加日志到 font_cache_android.cc:GetGenericFamilyNameForScript()
DLOG(INFO) << "System font: " << skia_family_name.c_str();

# 重新编译
autoninja -C out/Default chrome
```

