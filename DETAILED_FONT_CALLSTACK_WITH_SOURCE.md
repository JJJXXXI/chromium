# Android Chromium 字体选择完整调用栈（带源代码）

## 调用栈总览

```
DOM 节点创建及样式更新
  ↓
Element::SetComputedStyle()
  ↓
LayoutObject::SetStyle()
  ↓
FontBuilder::CreateFont()
  ↓
Font 对象创建（包含 FontDescription）
  ↓
文本渲染时触发 FontFallbackList::FontDataAt()
  ↓
FontFallbackList::GetFontData()
  ↓
[关键分岔] FontSelector::GetFontData() 或 FontCache::GetFontData()
  ↓
FontSelector::FamilyNameFromSettings()
  ↓
[Android 特殊] FontCache::GetGenericFamilyNameForScript()
  ↓
SkFontMgr->matchFamilyStyleCharacter()
  ↓
SkFontMgr_New_Android 解析 /system/etc/fonts.xml
  ↓
返回字体文件路径
  ↓
FontCache::GetFontData()
  ↓
FontCache::GetFontPlatformData()
  ↓
SkTypeface 创建
  ↓
SimpleFontData 创建
  ↓
文本渲染（HarfBuzz shaping + Skia rasterization）
```

---

## 详细调用栈（源代码级）

### 阶段 1：样式计算 → FontBuilder::CreateFont()

#### 1.1 Element 样式变化触发

**文件**：`third_party/blink/renderer/core/dom/element.cc`

```cpp
void Element::SetComputedStyle(scoped_refptr<ComputedStyle> new_style) {
  DCHECK(new_style);
  
  // ... 通知 LayoutObject 样式变化
  if (GetLayoutObject())
    GetLayoutObject()->SetStyle(new_style, absl::nullopt);
}
```

#### 1.2 LayoutObject::SetStyle()

**文件**：`third_party/blink/renderer/core/layout/layout_object.cc`

```cpp
void LayoutObject::SetStyle(scoped_refptr<const ComputedStyle> new_style) {
  // ... 
  // 触发 FontBuilder
  if (GetDocument().InStyleRecalc()) {
    // 样式计算过程中调用
  }
}
```

#### 1.3 FontBuilder::CreateFont() - 关键函数

**文件**：`third_party/blink/renderer/core/css/resolver/font_builder.cc:653-680`

```cpp
void FontBuilder::CreateFont(ComputedStyleBuilder& builder,
                             const ComputedStyle* parent_style) {
  DCHECK(document_);

  if (!flags_) {
    return;  // 没有字体属性变化，早期退出
  }

  // 获取父样式的字体描述
  const FontDescription& parent_description =
      parent_style ? parent_style->GetFontDescription()
                   : builder.GetFontDescription();

  FontDescription description = builder.GetFontDescription();
  // 关键：更新 FontDescription（处理继承等）
  if (!UpdateFontDescription(description, builder.ComputeFontOrientation())) {
    flags_ = 0;
    return;
  }
  
  UpdateSpecifiedSize(description, parent_description);
  UpdateComputedSize(description, builder);

  // ★ 创建 FontSelector（持有字体选择逻辑）
  FontSelector* font_selector = ComputeFontSelector(builder);
  UpdateAdjustedSize(description, font_selector);

  // ★ 创建 Font 对象（包含 FontDescription 和 FontSelector）
  // 这是关键：FontDescription 此时包含了 generic_family 和空的 family_
  builder.SetFont(MakeGarbageCollected<Font>(description, font_selector));
  flags_ = 0;
}
```

**关键状态此时**：
```
FontDescription::family_ = ""           // 没有指定字体族
FontDescription::generic_family_ = kStandardFamily  // 标记为"标准"族
FontSelector = 指向当前 document 的 FontSelector
```

---

### 阶段 2：文本布局/渲染时触发字体查询

#### 2.1 文本渲染 → FontFallbackList::FontDataAt()

**文件**：`third_party/blink/renderer/platform/fonts/font_fallback_list.cc:200-220`

当 Blink 需要渲染文本时（例如 HarfBuzz shaping），调用：

```cpp
const FontData* FontFallbackList::FontDataAt(
    const FontDescription& font_description,
    unsigned realized_font_index) {
  // 检查缓存
  if (realized_font_index < font_list_.size())
    return font_list_[realized_font_index].Get();

  DCHECK_EQ(realized_font_index, font_list_.size());

  if (family_index_ == kCAllFamiliesScanned)
    return nullptr;

  // ★ 关键调用：查询字体数据
  const FontData* result = GetFontData(font_description);
  if (result) {
    font_list_.push_back(result);
    if (result->IsLoadingFallback())
      has_loading_fallback_ = true;
    if (result->IsCustomFont())
      has_custom_font_ = true;
  }
  return result;
}
```

---

#### 2.2 FontFallbackList::GetFontData() - 核心字体查询

**文件**：`third_party/blink/renderer/platform/fonts/font_fallback_list.cc:149-197`

```cpp
const FontData* FontFallbackList::GetFontData(
    const FontDescription& font_description) {
  // Step 1: 遍历用户指定的字体列表
  const FontFamily* curr_family = &font_description.Family();
  for (int i = 0; curr_family && i < family_index_; i++)
    curr_family = curr_family->Next();

  for (; curr_family; curr_family = curr_family->Next()) {
    family_index_++;
    
    if (!font_selector_) {
      // 没有 FontSelector，直接查询系统字体
      if (!curr_family->FamilyName().empty()) {
        if (auto* result = FontCache::Get().GetFontData(
                font_description, curr_family->FamilyName())) {
          return result;
        }
      }
      continue;
    }

    // ★ 关键：通过 FontSelector 查询字体
    const FontData* result =
        font_selector_->GetFontData(font_description, *curr_family);
    
    // Step 2: 如果 FontSelector 失败，降级到 FontCache
    if (!result && !curr_family->FamilyName().empty()) {
      result = FontCache::Get().GetFontData(font_description,
                                            curr_family->FamilyName());
    }
    
    if (result) {
      return result;  // 找到 → 返回
    }
  }
  
  // Step 3: 所有用户指定字体都失败，尝试通用族
  family_index_ = kCAllFamiliesScanned;

  if (font_selector_) {
    // ★ 尝试标准通用族（当没有用户字体时触发）
    FontFamily font_family(font_family_names::kWebkitStandard,
                           FontFamily::Type::kGenericFamily);
    if (const FontData* data =
            font_selector_->GetFontData(font_description, font_family)) {
      return data;
    }
  }

  // Step 4: 最后的后备方案
  auto* last_resort =
      FontCache::Get().GetLastResortFallbackFont(font_description);
  return last_resort;
}
```

**执行流程图（无 CSS 字体时）**：
```
GetFontData(FontDescription { family_="", generic_family=kStandardFamily })
  ↓
Step 1: 遍历 font_description.Family() 列表
        → 列表中只有一个：FamilyName=""（空字符串）
        → 跳过！（因为 FamilyName.empty()）
  ↓
Step 2: 没有用户字体，继续
  ↓
Step 3: 尝试通用族 kWebkitStandard
        → 调用 font_selector_->GetFontData(font_description, generic_family)
        ★ 这是关键调用！
  ↓
Step 4: 如果上面失败，使用 GetLastResortFallbackFont()
```

---

### 阶段 3：FontSelector::GetFontData() - 字体选择的核心逻辑

#### 3.1 FontSelector::GetFontData()

**文件**：`third_party/blink/renderer/platform/fonts/font_selector.h:71`

```cpp
virtual const FontData* GetFontData(const FontDescription& font_description,
                                    const FontFamily& family) = 0;
```

这是虚函数，具体实现依赖子类。对于 Document 的 FontSelector：

**文件**：`third_party/blink/renderer/core/css/dom_font_selector.cc`

```cpp
const FontData* DOMFontSelector::GetFontData(
    const FontDescription& font_description,
    const FontFamily& family) {
  AtomicString family_name = GetFamilyName(font_description, family);
  // 关键：GetFamilyName() 调用 FamilyNameFromSettings()
  
  if (family_name.empty()) {
    return nullptr;
  }

  return FontCache::Get().GetFontData(font_description, family_name);
}
```

#### 3.2 FamilyNameFromSettings() - 获取实际族名（Android 分岔点）

**文件**：`third_party/blink/renderer/platform/fonts/font_selector.cc:20-102`

```cpp
AtomicString FontSelector::FamilyNameFromSettings(
    const GenericFontFamilySettings& settings,
    const FontDescription& font_description,
    const FontFamily& generic_family,
    UseCounter* use_counter) {
  // 参数检查
  auto& generic_family_name = generic_family.FamilyName();
  if (font_description.GenericFamily() != FontDescription::kStandardFamily &&
      font_description.GenericFamily() != FontDescription::kWebkitBodyFamily &&
      !generic_family.FamilyIsGeneric() &&
      generic_family_name != font_family_names::kWebkitStandard)
    return g_empty_atom;

  // ... 其他检查 ...

  // ★★★ Android 特殊路径开始！★★★
#if BUILDFLAG(IS_ANDROID)
  if (font_description.GenericFamily() == FontDescription::kStandardFamily ||
      font_description.GenericFamily() == FontDescription::kWebkitBodyFamily ||
      generic_family_name == font_family_names::kWebkitStandard) {
    // ★ 关键调用：调用 Android 特殊函数
    return FontCache::GetGenericFamilyNameForScript(
        font_family_names::kWebkitStandard,
        GetFallbackFontFamily(font_description),  // 来自 WebPreferences！
        font_description);
  }

  if (generic_family_name == font_family_names::kSerif ||
      generic_family_name == font_family_names::kSansSerif ||
      generic_family_name == font_family_names::kCursive ||
      generic_family_name == font_family_names::kFantasy ||
      generic_family_name == font_family_names::kMonospace) {
    return FontCache::GetGenericFamilyNameForScript(
        generic_family_name, generic_family_name, font_description);
  }

#else   // BUILDFLAG(IS_ANDROID) - 其他平台走这个分支
  UScriptCode script = font_description.GetScript();
  if (font_description.GenericFamily() == FontDescription::kStandardFamily ||
      font_description.GenericFamily() == FontDescription::kWebkitBodyFamily)
    return settings.Standard(script);  // 查询 GenericFontFamilySettings
  // ... 其他族的查询 ...
#endif  // BUILDFLAG(IS_ANDROID)

  return g_empty_atom;
}
```

**关键对比**：
```
Android:
  → FontCache::GetGenericFamilyNameForScript(
      "webkit-standard",
      GetFallbackFontFamily(...)  // "Times New Roman"（来自 WebPreferences）
      font_description)
  → 返回系统查询结果或 fallback

其他平台:
  → settings.Standard(script)
  → 查询 GenericFontFamilySettings::standard_font_family_map_
  → 返回配置的字体名
```

---

#### 3.3 GetFallbackFontFamily() - 获取 WebPreferences 的默认值

**文件**：`third_party/blink/renderer/platform/fonts/font_selector.cc`

```cpp
static AtomicString GetFallbackFontFamily(
    const FontDescription& font_description) {
  // ... 根据脚本选择 fallback ...
  // 返回来自 WebPreferences 的默认字体族名
}
```

这个函数返回的值来自哪里？追踪到 `Document::GetSettings()` → `Settings::GetGenericFontFamilySettings()`。

这些设置最初来自 `WebPreferences`：

**文件**：`third_party/blink/common/web_preferences/web_preferences.cc:23-38`

```cpp
WebPreferences::WebPreferences() {
  // ★★★ 硬编码的默认值！★★★
  standard_font_family_map[web_pref::kCommonScript] = u"Times New Roman";
  
  fixed_font_family_map[web_pref::kCommonScript] = u"Courier New";
  serif_font_family_map[web_pref::kCommonScript] = u"Times New Roman";
  sans_serif_font_family_map[web_pref::kCommonScript] = u"Arial";
  cursive_font_family_map[web_pref::kCommonScript] = u"Script";
  fantasy_font_family_map[web_pref::kCommonScript] = u"Impact";
  math_font_family_map[web_pref::kCommonScript] = u"Latin Modern Math";
}
```

---

### 阶段 4：FontCache::GetGenericFamilyNameForScript() - Android 系统字体查询

**文件**：`third_party/blink/renderer/platform/fonts/android/font_cache_android.cc:231-287`

```cpp
// static
AtomicString FontCache::GetGenericFamilyNameForScript(
    const AtomicString& family_name,                     // "webkit-standard"
    const AtomicString& generic_family_name_fallback,    // "Times New Roman"
    const FontDescription& font_description) {
  
  // Step 1: 检查是否是 locale 特定的族名
  if (GetLocaleSpecificFamilyName(family_name))
    return family_name;

  // Step 2: Monospace 不应用 CJK hack
  if (family_name == font_family_names::kMonospace)
    return family_name;

  // Step 3: 检查 locale 信息
  const LayoutLocale* content_locale = font_description.Locale();
  if (!content_locale)
    return generic_family_name_fallback;  // ← 无 locale → 返回 "Times New Roman"

  // Step 4: 检测脚本
  UChar32 exampler_char;
  switch (content_locale->GetScript()) {
    case USCRIPT_SIMPLIFIED_HAN:          // 中文简体
    case USCRIPT_TRADITIONAL_HAN:         // 中文繁体
    case USCRIPT_KATAKANA_OR_HIRAGANA:    // 日文
      exampler_char = 0x4E00;  // "中" 字，用来查询支持 CJK 的字体
      break;
    case USCRIPT_HANGUL:                  // 韩文
      exampler_char = 0xAC00;  // "가" 字
      break;
    default:
      // 非 CJK 脚本 → 直接返回 fallback
      return generic_family_name_fallback;  // ← 返回 "Times New Roman"
  }

  // Step 5: 只有 CJK 才会执行到这里
  // 获取 locale 信息用于查询
  Bcp47Vector locales =
      GetBcp47LocaleForRequest(font_description, FontFallbackPriority::kText);
  
  // ★ 关键：调用 SkFontMgr 查询支持 exampler_char 的字体
  sk_sp<SkTypeface> typeface(
      skia::DefaultFontMgr()->matchFamilyStyleCharacter(
          nullptr,                    // 族名为 NULL（查询默认）
          SkFontStyle(),              // 默认样式
          locales.data(),
          locales.size(),
          exampler_char));            // 示例字符（0x4E00 或 0xAC00）
  
  // Step 6: 查询失败处理
  if (!typeface) {
    return generic_family_name_fallback;  // ← 返回 "Times New Roman"
  }

  // Step 7: 成功 → 提取字体名
  SkString skia_family_name;
  typeface->getFamilyName(&skia_family_name);
  return ToAtomicString(skia_family_name);  // ← 返回 "Noto Sans CJK SC" 等
}
```

**执行路径分析**：

| 情况 | 脚本 | 流程 | 返回 |
|------|------|------|------|
| 中文 | SIMPLIFIED_HAN | Step 4 → 0x4E00 → Step 5-7 | NotoSansCJKSC |
| 日文 | HIRAGANA/KATAKANA | Step 4 → 0x4E00 → Step 5-7 | NotoSansCJKJP |
| 韩文 | HANGUL | Step 4 → 0xAC00 → Step 5-7 | NotoSansCJKKR |
| 英文 | LATIN | Step 4 default → | **Times New Roman** |
| 无locale | - | Step 3 if | **Times New Roman** |
| 查询失败 | - | Step 6 | **Times New Roman** |

---

### 阶段 5：SkFontMgr 系统字体查询

#### 5.1 skia::DefaultFontMgr() 初始化

**文件**：`skia/ext/font_utils.cc:67-83`

```cpp
static sk_sp<SkFontMgr> fontmgr_factory() {
  if (g_fontmgr_override) {
    return sk_ref_sp(g_fontmgr_override);
  }

#if BUILDFLAG(IS_ANDROID)
  // Step 1: 尝试 NDK API（Android 14+）
  if (base::FeatureList::IsEnabled(kUseAndroidNDKFontAPI) &&
      android_get_device_api_level() > __ANDROID_API_V__) {
    sk_sp<SkFontMgr> ndk_fontmgr =
        SkFontMgr_New_AndroidNDK(false, SkFontScanner_Make_Fontations());
    if (ndk_fontmgr && ndk_fontmgr->countFamilies()) {
      return ndk_fontmgr;  // ← 成功 → 返回 NDK fontmgr
    }
  }
  
  // Step 2: 回退到传统 API
  return SkFontMgr_New_Android(nullptr, SkFontScanner_Make_Fontations());
  // ★ 这会扫描 /system/etc/fonts.xml
#elif ...
#endif
}
```

#### 5.2 SkFontMgr_New_Android 做什么

**文件**：Skia（外部库） - `third_party/skia/src/ports/SkFontMgr_android.cpp`

```cpp
sk_sp<SkFontMgr> SkFontMgr_New_Android(const SkFontConfigInterface* fc,
                                       SkFontScanner* scanner) {
  // Step 1: 读取 /system/etc/fonts.xml
  // Step 2: 解析 XML，构建族名 → 字体文件的映射
  // Step 3: 返回 SkFontMgr_Android 实例
}

// 当调用 matchFamilyStyleCharacter 时：
sk_sp<SkTypeface> SkFontMgr_Android::matchFamilyStyleCharacter(
    const char* familyName,      // nullptr 或 "sans-serif"
    const SkFontStyle& style,
    const char* bcp47[],
    int bcp47Count,
    SkUnichar character) {       // 0x4E00 或其他
  // Step 1: 如果 familyName 为 NULL，使用默认族（通常 "sans-serif"）
  // Step 2: 在 fonts.xml 的族映射中查找
  // Step 3: 检查族中的字体是否支持 character
  // Step 4: 如果支持 → 返回 SkTypeface
  // Step 5: 如果不支持 → 继续查找 fallback 族（如 CJK 族）
  // Step 6: 最终找到支持该字符的字体 → 返回 SkTypeface
}
```

**fonts.xml 中的相关部分**：
```xml
<familyset version="23">
  <!-- 第一个族：sans-serif（默认） -->
  <family name="sans-serif">
    <font weight="400" style="normal">Roboto-Regular.ttf</font>
    <!-- ... 其他权重 ... -->
  </family>
  
  <!-- ... 其他族 ... -->
  
  <!-- CJK 族（fallback） -->
  <family lang="zh-Hans">
    <font index="2">NotoSansCJK-Regular.ttc</font>
  </family>
  <family lang="zh-Hant,zh-Bopo">
    <font index="3">NotoSansCJK-Regular.ttc</font>
  </family>
  <family lang="ja">
    <font index="0">NotoSansCJK-Regular.ttc</font>
  </family>
  <family lang="ko">
    <font index="1">NotoSansCJK-Regular.ttc</font>
  </family>
</familyset>
```

**查询流程**：
```
matchFamilyStyleCharacter(nullptr, style, locales, locales.size(), 0x4E00)
  ↓
Step 1: familyName=nullptr → 使用默认 "sans-serif"
  ↓
Step 2: fonts.xml 中查找 "sans-serif" 族
        → 找到：Roboto-Regular.ttf
  ↓
Step 3: 检查 Roboto 是否支持 U+4E00
        → 不支持（Roboto 只有拉丁字符）
  ↓
Step 4: 查找 fallback 族
        根据 locale（zh-Hans）→ 查找 <family lang="zh-Hans">
        → 找到：NotoSansCJK-Regular.ttc index=2
  ↓
Step 5: 返回 SkTypeface（NotoSansCJK-Regular）
  ↓
回到 GetGenericFamilyNameForScript()
  ↓
typeface->getFamilyName() → "Noto Sans CJK SC"
```

---

### 阶段 6：FontCache::GetFontData() - 从族名到字体文件

**文件**：`third_party/blink/renderer/platform/fonts/font_cache.cc:152-175`

```cpp
const SimpleFontData* FontCache::GetFontData(
    const FontDescription& font_description,
    const AtomicString& family,  // 例如 "Noto Sans CJK SC" 或 "Times New Roman"
    AlternateFontName altername_font_name) {
  
  // Step 1: 获取或创建 FontPlatformData
  if (const FontPlatformData* platform_data = GetFontPlatformData(
          font_description,
          FontFaceCreationParams(
              AdjustFamilyNameToAvoidUnsupportedFonts(family)),  // 调整族名
          altername_font_name)) {
    
    // Step 2: 从 FontPlatformData 创建 SimpleFontData
    return FontDataFromFontPlatformData(
        platform_data, font_description.SubpixelAscentDescent());
  }

  return nullptr;  // 失败
}
```

#### 6.1 GetFontPlatformData() 创建平台相关的字体数据

**文件**：`third_party/blink/renderer/platform/fonts/font_cache.cc:103-147`

```cpp
const FontPlatformData* FontCache::GetFontPlatformData(
    const FontDescription& font_description,
    const FontFaceCreationParams& creation_params,
    AlternateFontName alternate_font_name) {
  
  DCHECK(!font_description.Family().FamilyName().empty() ||
         creation_params.CreationType() == FontFaceCreationParams::kCreateFontByFamily);

  if (font_description.IsTextCombine()) {
    ...
  }

  // ★ 尝试从缓存获取（对性能很重要）
  return font_platform_data_cache_.GetOrCreateFontPlatformData(
      this, font_description, creation_params, alternate_font_name);
      // ↓
      // 调用 FontCache::CreateFontPlatformData()
}
```

#### 6.2 CreateFontPlatformData() - 平台特定实现

**文件**（Android）：`third_party/blink/renderer/platform/fonts/android/font_cache_android.cc:98-130`

```cpp
const FontPlatformData* FontCache::CreateFontPlatformData(
    const FontDescription& font_description,
    const FontFaceCreationParams& creation_params,
    float device_scale_factor) {
  
  DCHECK(!font_description.Family().FamilyName().empty());

  // Step 1: 获取 SkFontMgr
  sk_sp<SkFontMgr> fm = skia::DefaultFontMgr();
  
  // Step 2: 根据创建类型处理
  if (creation_params.CreationType() == FontFaceCreationParams::kCreateFontByFamily) {
    // ★ 最常见的路径：按族名创建
    
    const char* family_name_c_str = creation_params.Family().c_str();
    
    // Step 3: 从 SkFontMgr 创建 SkTypeface
    sk_sp<SkTypeface> typeface =
        fm->legacyMakeTypeface(family_name_c_str, skia_style);
    // 这会在 /system/etc/fonts.xml 中查找该族名
    
    if (!typeface) {
      return nullptr;  // 族名无效 → 返回空
    }
    
    // Step 4: 创建 FontPlatformData
    return new FontPlatformData(
        typeface, font_description.ComputedFontSize(),
        font_description.IsSyntheticBold(),
        font_description.IsSyntheticItalic(),
        font_description.Orientation(),
        device_scale_factor);
  }
  
  // ... 其他创建方式（如 @font-face） ...
}
```

**关键调用**：
```
fm->legacyMakeTypeface("Noto Sans CJK SC", skia_style)
  ↓
SkFontMgr_Android::legacyMakeTypeface()
  ↓
在 fonts.xml 中查找族名
  ↓
匹配到 <family lang="zh-Hans">
  ↓
返回 SkTypeface（指向 NotoSansCJK-Regular.ttc）
```

---

## 完整数据流示例

### 场景：无 CSS 字体 + 中文内容

```
HTML:
<p>你好</p>

CSS:
（无 font-family 属性）
```

**执行流程**：

```
[1] HTML 解析 + DOM 创建
    └─ <p> 元素创建

[2] 样式计算
    └─ CSSPropertyParser 应用规则
    └─ 没有 font-family 规则 → FontDescription::family_ = ""
    └─ FontDescription::generic_family = kStandardFamily

[3] FontBuilder::CreateFont() [font_builder.cc:653]
    └─ UpdateFontDescription() 处理继承
    └─ ComputeFontSelector() 创建 FontSelector
    └─ builder.SetFont(MakeGarbageCollected<Font>(description, font_selector))
    └─ 此时 Font 对象已包含：
        ├─ FontDescription { family_="", generic_family=kStandardFamily }
        └─ FontSelector（指向 document 的 DOMFontSelector）

[4] 文本渲染时触发
    └─ 调用 Font::GetFontData() 或类似
    └─ FontFallbackList::FontDataAt(font_description, 0)

[5] FontFallbackList::GetFontData() [font_fallback_list.cc:149]
    └─ Step 1: 遍历 FontDescription::Family()
        └─ 只有一个：FamilyName=""（空）→ 跳过
    └─ Step 2: 没有用户字体
    └─ Step 3: 尝试通用族 kWebkitStandard
        └─ FontFamily { name="webkit-standard", type=kGenericFamily }
        └─ 调用：font_selector_->GetFontData(font_description, font_family)

[6] DOMFontSelector::GetFontData() [dom_font_selector.cc]
    └─ family_name = GetFamilyName(font_description, family)
        └─ 调用：FamilyNameFromSettings()

[7] FontSelector::FamilyNameFromSettings() [font_selector.cc:20]
    └─ 参数：
        ├─ generic_family.FamilyName() = "webkit-standard"
        ├─ font_description.GenericFamily() = kStandardFamily
        └─ font_description.Locale() = zh_CN
    
    └─ 检查是否是 Android
        └─ #if BUILDFLAG(IS_ANDROID) ✓
    
    └─ 调用：FontCache::GetGenericFamilyNameForScript(
            "webkit-standard",
            GetFallbackFontFamily(font_description),  // "Times New Roman"
            font_description)
        ★ 关键调用！

[8] FontCache::GetGenericFamilyNameForScript() [font_cache_android.cc:231]
    └─ content_locale = font_description.Locale()  // zh_CN
    
    └─ content_locale->GetScript() 
        └─ 返回 USCRIPT_SIMPLIFIED_HAN
        └─ exampler_char = 0x4E00 ("中" 字)
    
    └─ locales = GetBcp47LocaleForRequest(...)  // ["zh-Hans"]
    
    └─ typeface = SkFontMgr->matchFamilyStyleCharacter(
            nullptr,              // 族名为 NULL
            SkFontStyle(),        // 默认样式
            ["zh-Hans"], 1,
            0x4E00)               // 查询支持该字符的字体
        ★ 关键系统调用！

[9] SkFontMgr_New_Android::matchFamilyStyleCharacter()
    └─ familyName = nullptr → 使用默认 "sans-serif"
    
    └─ 查询 fonts.xml
        ├─ 找到 <family name="sans-serif">
        │   └─ Roboto-Regular.ttf
        │   └─ 检查是否支持 U+4E00 → 不支持
        │
        └─ 查找 fallback
            └─ 根据 locale "zh-Hans"
            └─ 找到 <family lang="zh-Hans">
                └─ NotoSansCJK-Regular.ttc (index=2)
                └─ 支持 U+4E00 ✓
    
    └─ 返回 SkTypeface（NotoSansCJK）

[10] GetGenericFamilyNameForScript() 继续 [font_cache_android.cc:283]
    └─ typeface->getFamilyName(&skia_family_name)
        └─ skia_family_name = "Noto Sans CJK SC"
    
    └─ return ToAtomicString("Noto Sans CJK SC")

[11] FamilyNameFromSettings() 返回 [font_selector.cc]
    └─ return "Noto Sans CJK SC"

[12] DOMFontSelector::GetFontData() 继续 [dom_font_selector.cc]
    └─ family_name = "Noto Sans CJK SC"
    
    └─ return FontCache::Get().GetFontData(
            font_description,
            "Noto Sans CJK SC")

[13] FontCache::GetFontData() [font_cache.cc:152]
    └─ platform_data = GetFontPlatformData(
            font_description,
            FontFaceCreationParams("Noto Sans CJK SC"))

[14] GetFontPlatformData() → FontCache::CreateFontPlatformData() [font_cache_android.cc:98]
    └─ fm = skia::DefaultFontMgr()
    
    └─ typeface = fm->legacyMakeTypeface("Noto Sans CJK SC", style)
        └─ 查询 fonts.xml 中的族映射
        └─ 返回指向 NotoSansCJK-Regular.ttc 的 SkTypeface
    
    └─ return new FontPlatformData(typeface, size, ...)

[15] FontCache::GetFontData() 继续 [font_cache.cc:165]
    └─ return FontDataFromFontPlatformData(platform_data, ...)
        └─ 返回 SimpleFontData（包含实际字体信息）

[16] FontFallbackList::GetFontData() 返回 [font_fallback_list.cc]
    └─ return SimpleFontData（NotoSansCJK）

[17] 文本渲染
    └─ HarfBuzz shaping：使用 NotoSansCJK 的 cmap/GSUB/GPOS 表
    
    └─ Skia 光栅化：调用 FreeType/Skrifa 渲染字形
    
    └─ 屏幕显示 ✓

结果：
  文字显示为 "Noto Sans CJK SC"，而不是 "Times New Roman"
  ✓ 因为 Roboto 不支持 CJK，系统 fallback 到 NotoSansCJK
```

---

## 如果改成英文会发生什么

```
场景：无 CSS 字体 + 英文内容

HTML:
<p>Hello World</p>

[1-7] 过程同上，直到 FontSelector::FamilyNameFromSettings()

[8] FontCache::GetGenericFamilyNameForScript() [font_cache_android.cc:231]
    └─ content_locale->GetScript() 
        └─ 返回 USCRIPT_LATIN（不是 CJK！）
    
    └─ switch 语句的 default 分支
        └─ return generic_family_name_fallback
        └─ return "Times New Roman"  ✗ 问题在这里！

[9] DOMFontSelector::GetFontData() 继续
    └─ family_name = "Times New Roman"
    
    └─ return FontCache::Get().GetFontData(
            font_description,
            "Times New Roman")

[10] FontCache::GetFontData() [font_cache.cc:152]
    └─ platform_data = GetFontPlatformData(..., "Times New Roman")

[11] FontCache::CreateFontPlatformData() [font_cache_android.cc:98]
    └─ typeface = fm->legacyMakeTypeface("Times New Roman", style)
        └─ 在 /system/etc/fonts.xml 中查找 "Times New Roman"
        └─ ❌ 找不到！（fonts.xml 中没有这个族）
        └─ typeface = nullptr
    
    └─ 返回 nullptr

[12] FontCache::GetFontData() 返回 nullptr
    └─ return nullptr

[13] FontFallbackList::GetFontData() 继续 [font_fallback_list.cc]
    └─ result = nullptr（失败）
    
    └─ 继续尝试通用族的其他选项
    
    └─ 最终降级到 GetLastResortFallbackFont()
        └─ 返回系统的最后后备字体（通常 Roboto）

[14] 文本渲染
    └─ 使用 Roboto 渲染

结果：
  本来想用 "Times New Roman"，但 Android 上不存在
  所以降级到 Roboto
  ✓ 实际显示为 Roboto（而不是我们期望的 Times New Roman）
```

---

## 修改建议与效果

### 当前问题
```
WebPreferences::WebPreferences() {
  standard_font_family_map[kCommonScript] = u"Times New Roman";  // ← 问题！
}
```

- 硬编码的默认值
- 对所有平台都适用
- Android 上不存在此字体 → 降级到 Roboto/Noto

### 方案 1：Android 特定默认值

**修改**：`third_party/blink/common/web_preferences/web_preferences.cc:25`

```cpp
WebPreferences::WebPreferences() {
#if BUILDFLAG(IS_ANDROID)
  standard_font_family_map[web_pref::kCommonScript] = u"sans-serif";
#else
  standard_font_family_map[web_pref::kCommonScript] = u"Times New Roman";
#endif
  // ... 其他族的设置 ...
}
```

**效果**：
- Android：`GetFallbackFontFamily()` 返回 "sans-serif"
- 传给 `FontCache::GetGenericFamilyNameForScript()`
- 最终使用 Roboto（fonts.xml 中的默认）
- ✓ 和 UI 字体一致

### 方案 2：改 Android 特殊路径

**修改**：`third_party/blink/renderer/platform/fonts/android/font_cache_android.cc:231`

```cpp
// 删掉整个 CJK hack，直接返回 fallback
return generic_family_name_fallback;
```

**效果**：
- 所有脚本都用 `generic_family_name_fallback`（"Times New Roman"）
- ❌ 问题：Times New Roman 仍然不存在
- 最终还是会降级

---

## 总结

| 层级 | 组件 | 源文件 | 关键行 | 作用 |
|------|------|--------|--------|------|
| **1** | HTML/DOM | `core/dom/element.cc` | - | 创建元素、样式变化 |
| **2** | CSS 样式应用 | `core/css/` | - | 解析 CSS，创建 FontDescription |
| **3** | FontBuilder | `core/css/resolver/font_builder.cc` | 653 | 创建 Font 对象，持有 FontSelector |
| **4** | 文本布局/渲染 | `platform/fonts/font_fallback_list.cc` | 149 | 按优先级查找字体 |
| **5** | 通用族查询 | `platform/fonts/font_selector.cc` | 20 | **分岔点**：Android vs 其他 |
| **6**★ | Android 特殊 | `platform/fonts/android/font_cache_android.cc` | 231 | **CJK Hack**：用 0x4E00 查询 |
| **7** | SkFontMgr | `skia/ext/font_utils.cc` | 83 | 扫描 /system/etc/fonts.xml |
| **8** | 字体文件查询 | `platform/fonts/font_cache.cc` | 98-130 | 从族名到字体文件 |
| **9** | 渲染 | `platform/fonts/shaping/` | - | HarfBuzz + Skia |

**关键问题在第 6 步**（Android 特殊路径）和 **WebPreferences 默认值**。
