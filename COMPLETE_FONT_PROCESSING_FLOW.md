# Chromium 完整字体处理流程 - 从HTML加载到字体渲染

## 概述

本文档详细记录了 Chromium (Blink) 从加载 HTML 到最终字体渲染的完整调用链路,包含源码位置和函数调用关系。

## 核心流程图

```
HTML 加载 → DOM 构建 → 样式计算 → 字体解析 → 字体加载 → 字体渲染
    ↓           ↓           ↓           ↓           ↓           ↓
 HTMLParser  Document  StyleResolver FontBuilder  FontCache   Paint
```

---

## 第一阶段: HTML 解析与 DOM 构建

### 1.1 HTML 加载与解析

**入口点**: `third_party/blink/renderer/core/html/parser/html_document_parser.cc`

```cpp
// HTMLDocumentParser 处理 HTML 文本流
void HTMLDocumentParser::Append(const String& input)
  → void HTMLDocumentParser::ProcessTokenizedChunk()
    → void HTMLTreeBuilder::ConstructTree()
      → void HTMLConstructionSite::InsertHTMLElement()
```

**关键函数**:
- `HTMLDocumentParser::Append()` - 接收 HTML 文本
- `HTMLTreeBuilder::ConstructTree()` - 构建 DOM 树
- `HTMLConstructionSite::InsertHTMLElement()` - 插入元素节点

### 1.2 DOM 树构建完成

**文件**: `third_party/blink/renderer/core/dom/document.cc`

```cpp
Document::FinishedParsing()
  → Document::BeginLifecycleUpdatesIfRenderingReady()
    → Document::UpdateStyleAndLayoutTree()
```

---

## 第二阶段: 样式计算触发 (Style Resolution)

### 2.1 样式重计算入口

**文件**: `third_party/blink/renderer/core/dom/document.cc`

```cpp
void Document::UpdateStyleAndLayoutTree() {
  // 标记需要样式重计算的元素
  GetStyleEngine().UpdateStyle();
}
```

### 2.2 StyleEngine 驱动样式计算

**文件**: `third_party/blink/renderer/core/css/style_engine.cc`

```cpp
void StyleEngine::UpdateStyle() {
  // 遍历需要样式重计算的元素
  UpdateActiveStyle();
  RecalcStyle();
}

void StyleEngine::RecalcStyle() {
  // 对每个元素调用样式解析
  Element::RecalcStyle();
}
```

### 2.3 Element 触发样式解析

**文件**: `third_party/blink/renderer/core/dom/element.cc`

```cpp
void Element::RecalcStyle(const StyleRecalcContext& style_recalc_context) {
  // 调用 StyleResolver 计算样式
  const ComputedStyle* new_style = 
    GetDocument().GetStyleResolver().ResolveStyle(
      this, style_recalc_context, StyleRequest());
}
```

---

## 第三阶段: StyleResolver 核心样式计算

### 3.1 ResolveStyle 主入口

**文件**: `third_party/blink/renderer/core/css/resolver/style_resolver.cc`

```cpp
const ComputedStyle* StyleResolver::ResolveStyle(
    Element* element,
    const StyleRecalcContext& style_recalc_context,
    const StyleRequest& style_request) {
  
  // 创建状态对象,用于累积计算结果
  StyleResolverState state(GetDocument(), *element, 
                          &style_recalc_context, style_request);
  
  // 创建 CSS 级联对象
  StyleCascade cascade(state);
  
  // ★ 计算基础样式(包括字体属性)
  ApplyBaseStyle(element, style_recalc_context, style_request, state, cascade);
  
  // 应用动画样式
  ApplyAnimatedStyle(state, cascade, style_recalc_context);
  
  // 返回最终计算的样式
  return state.TakeStyle();
}
```

**关键点**: 
- `StyleResolverState` 保存整个计算过程的状态
- `StyleCascade` 处理 CSS 层叠
- `ApplyBaseStyle()` 是字体计算的核心

### 3.2 匹配 CSS 规则

**文件**: `third_party/blink/renderer/core/css/resolver/style_resolver.cc`

```cpp
void StyleResolver::MatchAllRules(
    StyleResolverState& state,
    ElementRuleCollector& collector,
    bool include_smil_properties) {
  
  Element& element = state.GetElement();
  
  // 1. 匹配 UA (浏览器默认) 样式
  MatchUARules(element, collector);
  
  // 2. 匹配用户样式
  MatchUserRules(collector);
  
  // 3. 匹配 HTML presentation 属性
  MatchPresentationalHints(state, collector);
  
  // 4. 匹配作者样式(网页CSS)
  MatchAuthorRules(element, collector);
}
```

**UA 样式加载**:
```cpp
void StyleResolver::MatchUARules(
    const Element& element,
    ElementRuleCollector& collector) {
  
  // html.css, android.css 等默认样式表
  CSSDefaultStyleSheets& default_style_sheets =
      CSSDefaultStyleSheets::Instance();
  
  if (element.IsHTMLElement()) {
    // ★ 加载 html.css (不含 font-family 默认值!)
    collector.CollectMatchingRules(
      default_style_sheets.DefaultHtmlStyle());
  } else if (element.IsSVGElement()) {
    collector.CollectMatchingRules(
      default_style_sheets.DefaultSVGStyle());
  }
}
```

**重要发现**: 
- [third_party/blink/renderer/core/html/resources/html.css](third_party/blink/renderer/core/html/resources/html.css) 中没有设置 `html, body { font-family: ... }`
- 字体族的默认值不是通过 UA CSS 设置的

---

## 第四阶段: 字体属性处理 (Font Resolution)

### 4.1 StyleCascade 应用 CSS 属性

**文件**: `third_party/blink/renderer/core/css/resolver/style_cascade.cc`

```cpp
void StyleCascade::Apply() {
  // 应用所有 CSS 属性到 StyleBuilder
  ApplyMatchedProperties();
  
  // ★ 在属性应用后,创建 Font 对象
  state_.UpdateFont();
}
```

### 4.2 StyleResolverState::UpdateFont() 触发字体创建

**文件**: `third_party/blink/renderer/core/css/resolver/style_resolver_state.cc`

```cpp
void StyleResolverState::UpdateFont() {
  // 调用 FontBuilder 创建 Font 对象
  GetFontBuilder().CreateFont(StyleBuilder(), ParentStyle());
}
```

**关键对象**:
- `StyleResolverState` 包含 `FontBuilder font_builder_`
- `FontBuilder` 负责从 CSS 属性构建 `FontDescription`

### 4.3 FontBuilder 创建 FontDescription

**文件**: `third_party/blink/renderer/core/css/resolver/font_builder.cc`

```cpp
void FontBuilder::CreateFont(
    ComputedStyleBuilder& builder,
    const ComputedStyle* parent_style) {
  
  // 1. 创建 FontDescription (字体描述对象)
  FontDescription description = builder.GetFontDescription();
  
  // 2. ★ 更新 FontDescription 的字体族等属性
  UpdateFontDescription(description);
  
  // 3. 解析通用字体族 (如果需要)
  if (description.GenericFamily() != FontDescription::kNoFamily) {
    // 将 kStandardFamily 映射到具体字体
    UpdateGenericFontFamilySettings(document_, builder, description);
  }
  
  // 4. 设置回 ComputedStyle
  builder.SetFontDescription(description);
}
```

**FontDescription 初始化** (`font_description.cc:88-104`):
```cpp
FontDescription::FontDescription() {
  // ★ 默认值是 kNoFamily,不是具体字体!
  fields_.generic_family_ = kNoFamily;
  fields_.kerning_ = kAutoKerning;
  fields_.font_smoothing_ = kAutoSmoothing;
  // ... 其他默认值
}
```

### 4.4 字体族解析 (Generic Family → Concrete Font)

**场景 1: 如果 CSS 指定了 `font-family: sans-serif`**

**文件**: `third_party/blink/renderer/platform/fonts/font_selector.cc`

```cpp
AtomicString FontSelector::FamilyNameFromSettings(
    const FontDescription& font_description,
    const FontFamily& generic_family,
    const LayoutLocale* content_locale) {
  
  // 从 GenericFontFamilySettings 获取映射
  if (generic_family_settings_) {
    return generic_family_settings_->Standard(script);
  }
  
  // Android 特殊处理: 查询系统字体
  #if BUILDFLAG(IS_ANDROID)
    return FontCache::GetGenericFamilyNameForScript(
      generic_family_name, font_description, content_locale);
  #endif
}
```

**场景 2: 如果没有指定 font-family (kNoFamily)**

**Android 系统字体查询**:

**文件**: `third_party/blink/renderer/platform/fonts/android/font_cache_android.cc`

```cpp
AtomicString FontCache::GetGenericFamilyNameForScript(
    const AtomicString& family_name,
    const FontDescription& font_description,
    const LayoutLocale* content_locale) {
  
  // 1. 解析 Android fonts.xml 配置
  SkFontMgr* font_mgr = SkFontMgr_New_Android(...);
  
  // 2. 查询默认字体族
  // 对于 "standard"/"sans-serif",通常返回 "Roboto"
  return MatchFamilyNameFromAndroidConfiguration(family_name, script);
}
```

**Android fonts.xml 示例**:
```xml
<familyset>
  <family name="sans-serif">
    <font weight="400" style="normal">Roboto-Regular.ttf</font>
    <font weight="700" style="normal">Roboto-Bold.ttf</font>
  </family>
</familyset>
```

### 4.5 GenericFontFamilySettings 映射

**文件**: `third_party/blink/renderer/platform/fonts/generic_font_family_settings.cc`

```cpp
const AtomicString& GenericFontFamilySettings::Standard(
    UScriptCode script) const {
  
  // 查找 standard_font_family_map_
  // 如果为空(Android默认情况),返回空字符串
  // 触发 FontCache 查询系统字体
  return standard_font_family_map_.Get(script);
}
```

**重要**: 
- Android 上 `standard_font_family_map_` 默认为空
- 空映射 → 触发 `FontCache::GetGenericFamilyNameForScript()`
- 最终从 fonts.xml 获取 "Roboto"

---

## 第五阶段: 字体加载 (Font Loading)

### 5.1 FontFallbackList 查找字体

**文件**: `third_party/blink/renderer/platform/fonts/font_fallback_list.cc`

```cpp
const SimpleFontData* FontFallbackList::GetFontData(
    const FontDescription& font_description) const {
  
  // 1. 遍历 font-family 列表
  for (const FontFamily& family : font_description.Family()) {
    const FontData* font_data = 
      font_selector_->GetFontData(font_description, family.FamilyName());
    
    if (font_data) {
      return font_data;
    }
  }
  
  // 2. 如果所有指定字体都失败,使用 fallback
  return GetFallbackFont(font_description);
}
```

### 5.2 FontCache 加载字体文件

**文件**: `third_party/blink/renderer/platform/fonts/font_cache.h`

```cpp
const SimpleFontData* FontCache::GetFontData(
    const FontDescription& font_description,
    const AtomicString& family_name) {
  
  // 1. 查询缓存
  FontPlatformData* platform_data = GetFontPlatformData(
    font_description, 
    FontFaceCreationParams(family_name));
  
  if (!platform_data) {
    return nullptr;
  }
  
  // 2. 创建 SimpleFontData (包含字形数据)
  return FontDataFromFontPlatformData(
    platform_data, 
    kDoNotRetain);
}
```

### 5.3 Android 平台字体查询

**文件**: `third_party/blink/renderer/platform/fonts/skia/font_cache_skia.cc`

```cpp
std::unique_ptr<FontPlatformData> FontCache::CreateFontPlatformData(
    const FontDescription& font_description,
    const FontFaceCreationParams& creation_params) {
  
  // Android: 使用 SkFontMgr 查询字体
  sk_sp<SkTypeface> typeface = 
    font_manager_->matchFamilyStyle(
      creation_params.Family().Utf8().c_str(),
      SkFontStyle(weight, width, slant));
  
  if (!typeface) {
    // Fallback 到系统默认字体
    typeface = font_manager_->legacyMakeTypeface(nullptr, slant);
  }
  
  return std::make_unique<FontPlatformData>(
    typeface, 
    font_description.ComputedSize());
}
```

**Skia SkFontMgr** (`third_party/skia/src/ports/SkFontMgr_android.cpp`):
- 读取 `/system/etc/fonts.xml`
- 解析字体路径和 fallback 链
- 加载 `.ttf` 字体文件到内存

---

## 第六阶段: 字体渲染 (Text Shaping & Painting)

### 6.1 Layout 阶段: 文本 Shaping

**文件**: `third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc`

```cpp
void HarfBuzzShaper::Shape(const TextRun& run) {
  // 1. 从 FontData 获取 hb_font_t
  hb_font_t* hb_font = font_data->GetHarfBuzzFace()->GetScaledFont();
  
  // 2. 使用 HarfBuzz 进行字形 shaping
  hb_buffer_t* buffer = hb_buffer_create();
  hb_buffer_add_utf16(buffer, text, length, 0, length);
  hb_shape(hb_font, buffer, nullptr, 0);
  
  // 3. 获取字形位置信息
  unsigned glyph_count;
  hb_glyph_info_t* glyph_infos = hb_buffer_get_glyph_infos(buffer, &glyph_count);
  hb_glyph_position_t* glyph_positions = hb_buffer_get_glyph_positions(buffer, &glyph_count);
  
  // 4. 创建 ShapeResult (包含字形ID和位置)
  return CreateShapeResult(glyph_infos, glyph_positions, glyph_count);
}
```

**HarfBuzz**: 开源文本 shaping 引擎
- 处理复杂文本布局(连字、kerning、阿拉伯文重排等)
- 输入: Unicode 文本 + 字体
- 输出: 字形ID (Glyph ID) + 位置信息

### 6.2 Paint 阶段: 绘制字形

**文件**: `third_party/blink/renderer/platform/graphics/graphics_context.cc`

```cpp
void GraphicsContext::DrawText(
    const Font& font,
    const TextRunPaintInfo& run_info,
    const gfx::PointF& point) {
  
  // 1. 从 ShapeResult 获取字形
  const ShapeResult* shape_result = run_info.shape_result;
  
  // 2. 遍历每个字形,使用 Skia 绘制
  shape_result->ForEachGlyph([&](const Glyph& glyph, const gfx::PointF& position) {
    // Skia Canvas 绘制
    canvas_->drawGlyphs(
      1,                      // 字形数量
      &glyph.glyph_id,       // 字形 ID
      &position,             // 位置
      point,                 // 基线位置
      font.GetSkFont(),      // SkFont 对象
      paint);                // 绘制样式
  });
}
```

**Skia 字形渲染**:
- 使用 FreeType 或系统渲染器光栅化字形
- 应用 hinting、anti-aliasing
- 输出像素位图到屏幕缓冲区

---

## 关键调用链总结

### 完整调用路径 (从 HTML 到渲染)

```
1. HTMLDocumentParser::Append()
   └─> HTMLTreeBuilder::ConstructTree()
       └─> Document::UpdateStyleAndLayoutTree()
           └─> StyleEngine::UpdateStyle()
               └─> Element::RecalcStyle()
                   └─> StyleResolver::ResolveStyle()  ★ 核心入口
                       ├─> MatchAllRules()            // 匹配 CSS 规则
                       │   ├─> MatchUARules()         // html.css (无 font-family)
                       │   ├─> MatchUserRules()
                       │   └─> MatchAuthorRules()
                       └─> StyleCascade::Apply()      // 应用 CSS 属性
                           └─> StyleResolverState::UpdateFont()  ★ 字体处理入口
                               └─> FontBuilder::CreateFont()
                                   ├─> UpdateFontDescription()  // 构建 FontDescription
                                   └─> FontSelector::FamilyNameFromSettings()
                                       └─> [Android] FontCache::GetGenericFamilyNameForScript()
                                           └─> SkFontMgr::matchFamilyStyle()  // 查询 fonts.xml
                                               └─> 返回 "Roboto"

2. Layout Phase (字形 Shaping)
   LayoutText::ComputeTextMetrics()
   └─> Font::GetFontData()
       └─> FontFallbackList::GetFontData()
           └─> FontCache::GetFontData()
               └─> FontCache::CreateFontPlatformData()  // 加载 .ttf 文件
                   └─> HarfBuzzShaper::Shape()          // 文本 shaping
                       └─> hb_shape()                   // HarfBuzz

3. Paint Phase (字形绘制)
   TextPainter::Paint()
   └─> GraphicsContext::DrawText()
       └─> SkCanvas::drawGlyphs()    // Skia 渲染
           └─> FreeType              // 字形光栅化
```

---

## 关键源码文件索引

| 阶段 | 文件路径 | 关键函数 | 作用 |
|------|---------|---------|------|
| **HTML 解析** | `core/html/parser/html_document_parser.cc` | `Append()`, `ConstructTree()` | 构建 DOM 树 |
| **样式触发** | `core/dom/document.cc` | `UpdateStyleAndLayoutTree()` | 触发样式计算 |
| | `core/css/style_engine.cc` | `UpdateStyle()`, `RecalcStyle()` | 驱动样式重计算 |
| **样式计算** | `core/css/resolver/style_resolver.cc` | `ResolveStyle()`, `MatchAllRules()` | CSS 规则匹配 |
| | `core/css/resolver/style_cascade.cc` | `Apply()` | CSS 层叠计算 |
| **字体解析** | `core/css/resolver/style_resolver_state.cc` | `UpdateFont()` | 触发字体创建 |
| | `core/css/resolver/font_builder.cc` | `CreateFont()`, `UpdateFontDescription()` | 构建 FontDescription |
| | `platform/fonts/font_description.cc` | `FontDescription()` 构造函数 | **默认 kNoFamily** |
| | `platform/fonts/font_selector.cc` | `FamilyNameFromSettings()` | 通用字体族映射 |
| | `platform/fonts/generic_font_family_settings.cc` | `Standard()` | 查询字体设置 |
| **Android 字体** | `platform/fonts/android/font_cache_android.cc` | `GetGenericFamilyNameForScript()` | **查询 fonts.xml** |
| | `third_party/skia/src/ports/SkFontMgr_android.cpp` | `matchFamilyStyle()` | Skia 字体匹配 |
| **字体加载** | `platform/fonts/font_cache.cc` | `GetFontData()`, `CreateFontPlatformData()` | 加载字体文件 |
| | `platform/fonts/font_fallback_list.cc` | `GetFontData()` | 字体回退链 |
| | `platform/fonts/skia/font_cache_skia.cc` | `CreateFontPlatformData()` | Skia 平台字体 |
| **文本 Shaping** | `platform/fonts/shaping/harfbuzz_shaper.cc` | `Shape()` | HarfBuzz shaping |
| **渲染** | `platform/graphics/graphics_context.cc` | `DrawText()` | 绘制文本 |
| | Skia (`third_party/skia`) | `SkCanvas::drawGlyphs()` | 字形光栅化 |

---

## 字体默认值流程详解

### 问题: 为什么没有 font-family,还能显示字体?

#### 1. html.css 不设置默认字体

[third_party/blink/renderer/core/html/resources/html.css](third_party/blink/renderer/core/html/resources/html.css):
```css
html, body {
  /* ★ 没有 font-family 属性! */
  display: block;
}
```

#### 2. FontDescription 初始化为 kNoFamily

[third_party/blink/renderer/platform/fonts/font_description.cc:88-104](third_party/blink/renderer/platform/fonts/font_description.cc#L88-L104):
```cpp
FontDescription::FontDescription() {
  fields_.generic_family_ = kNoFamily;  // ★ 默认无字体族
}
```

#### 3. FontBuilder 检测 kNoFamily,触发通用字体族查询

[third_party/blink/renderer/core/css/resolver/font_builder.cc](third_party/blink/renderer/core/css/resolver/font_builder.cc):
```cpp
void FontBuilder::CreateFont(...) {
  FontDescription description = builder.GetFontDescription();
  
  if (description.GenericFamily() == FontDescription::kNoFamily) {
    // ★ 自动使用 kStandardFamily 作为 fallback
    description.SetGenericFamily(FontDescription::kStandardFamily);
  }
  
  UpdateGenericFontFamilySettings(document_, builder, description);
}
```

#### 4. Android: GenericFontFamilySettings 为空,触发系统查询

[third_party/blink/renderer/platform/fonts/generic_font_family_settings.cc:243-252](third_party/blink/renderer/platform/fonts/generic_font_family_settings.cc#L243-L252):
```cpp
void GenericFontFamilySettings::Reset() {
  standard_font_family_map_.clear();  // ★ Android 默认为空
  // ...
}
```

[third_party/blink/renderer/platform/fonts/font_selector.cc:28-95](third_party/blink/renderer/platform/fonts/font_selector.cc#L28-L95):
```cpp
AtomicString FontSelector::FamilyNameFromSettings(...) {
  // settings 为空,返回 null
  if (!generic_family_settings_) {
    return g_null_atom;
  }
  
  AtomicString family_name = generic_family_settings_->Standard(script);
  if (family_name.empty()) {
    // ★ Android 走这里: 查询系统字体
    #if BUILDFLAG(IS_ANDROID)
      return FontCache::GetGenericFamilyNameForScript(...);
    #endif
  }
}
```

#### 5. FontCache 读取 /system/etc/fonts.xml

[third_party/blink/renderer/platform/fonts/android/font_cache_android.cc](third_party/blink/renderer/platform/fonts/android/font_cache_android.cc):
```cpp
AtomicString FontCache::GetGenericFamilyNameForScript(...) {
  // Skia SkFontMgr 解析 fonts.xml
  SkFontMgr* font_mgr = SkFontMgr_New_Android(...);
  
  // 查询 "sans-serif" 或 "standard" 对应的字体
  // Android 默认返回 "Roboto"
  return AtomicString("Roboto");
}
```

#### 6. Android fonts.xml 示例

`/system/etc/fonts.xml`:
```xml
<familyset>
  <family name="sans-serif">
    <font weight="400" style="normal">Roboto-Regular.ttf</font>
    <font weight="700" style="normal">Roboto-Bold.ttf</font>
  </family>
  
  <alias name="arial" to="sans-serif" />
  <alias name="helvetica" to="sans-serif" />
</familyset>
```

---

## 无CSS样式字体选择更精细链路

当页面与UA样式都未显式指定 `font-family` 时,Blink 的默认选择并非依赖 UA CSS,而是由字体构建器的“初始值”驱动。以下是精确源码路径与状态演化:

- 初始值来源(非继承属性的初值提供者): [third_party/blink/renderer/core/css/resolver/font_builder.h](third_party/blink/renderer/core/css/resolver/font_builder.h)
  - `InitialFamilyDescription()` → 组合 `InitialGenericFamily()`
  - `InitialGenericFamily()` 返回 `FontDescription::kStandardFamily` (即通用的“标准”家族,通常等价于 sans-serif)

- 计算样式阶段构造 `ComputedStyleBuilder`:
  - [third_party/blink/renderer/core/css/resolver/style_resolver.cc](third_party/blink/renderer/core/css/resolver/style_resolver.cc)
    - `StyleResolver::ResolveStyle()` 创建 `StyleResolverState`
    - `StyleCascade::Apply()` 应用层叠属性后调用 `state_.UpdateFont()`

- 创建字体描述并填充初始家族:
  - [third_party/blink/renderer/core/css/resolver/style_resolver_state.cc](third_party/blink/renderer/core/css/resolver/style_resolver_state.cc)
    - `StyleResolverState::UpdateFont()` → 调用 `FontBuilder::CreateFont()`
  - [third_party/blink/renderer/core/css/resolver/font_builder.cc](third_party/blink/renderer/core/css/resolver/font_builder.cc)
    - `CreateFont(...)` 内部会在未看到作者/UA指定家族时,以 `InitialFamilyDescription()` 为基础将 `FontDescription` 的通用家族设为 `kStandardFamily`

- 将“通用家族”映射为具体家族名称:
  - [third_party/blink/renderer/platform/fonts/font_selector.cc](third_party/blink/renderer/platform/fonts/font_selector.cc)
    - `FamilyNameFromSettings(...)` 首先查 `GenericFontFamilySettings` 的映射(按脚本/语言)
    - Android 上 `GenericFontFamilySettings` 常为空 → 触发 `FontCache::GetGenericFamilyNameForScript(...)`
  - [third_party/blink/renderer/platform/fonts/android/font_cache_android.cc](third_party/blink/renderer/platform/fonts/android/font_cache_android.cc)
    - 查询系统 `/system/etc/fonts.xml` → 对 `sans-serif`/`standard` 返回 "Roboto"

- 加载字体与回退:
  - [third_party/blink/renderer/platform/fonts/font_cache.cc](third_party/blink/renderer/platform/fonts/font_cache.cc)
    - `GetFontData(...)` 通过 Skia / 系统接口加载具体字形数据
  - [third_party/blink/renderer/platform/fonts/font_fallback_list.cc](third_party/blink/renderer/platform/fonts/font_fallback_list.cc)
    - 若指定/映射家族均不可用 → 走 `GetFallbackFont(...)` 的平台回退链(含默认LastResort字体)

### 关于 `FontDescription` 的默认 `kNoFamily` 与实际生效的 `kStandardFamily`

- `FontDescription` 构造函数默认 `generic_family_ = kNoFamily` 出现在 [third_party/blink/renderer/platform/fonts/font_description.cc](third_party/blink/renderer/platform/fonts/font_description.cc) 中,这是一个“裸构造”的初始状态。
- 在实际样式计算时,`FontBuilder` 会用“初始家族”(standard)或“继承家族”(非根元素)覆盖该裸初值;因此最终进入 `FontSelector` 的 `FontDescription` 通常已带 `kStandardFamily` 或继承到的值,而非停留在 `kNoFamily`。

### `FontFaceCache` 在无CSS时不会参与

- [third_party/blink/renderer/core/css/font_face_cache.cc](third_party/blink/renderer/core/css/font_face_cache.cc) 仅在存在 `@font-face` 声明并与 `font-family` 匹配时参与缓存/匹配。
- 无CSS样式场景下没有 `@font-face` 参与,因此默认字体选择完全由“通用家族 → 系统映射 → 字体加载/回退”路径实现,与 `FontFaceCache` 无关。

### 最小闭环(无CSS的最终家族选择例子)

```
元素无 font-family → 继承链至根元素 → 根元素使用 InitialGenericFamily = kStandardFamily
→ FamilyNameFromSettings → [Android] FontCache::GetGenericFamilyNameForScript("standard")
→ fonts.xml 返回 "Roboto" → FontCache::GetFontData 加载 Roboto → HarfBuzz + Skia 渲染
```

这条链路解释了“没有CSS样式的文字”如何确定字体:初始家族为 standard,Android 通过系统配置将其解析到 Roboto,进而完成字体加载与渲染。

## 修改字体的三种方案

### 方案 1: 修改 UA CSS (不推荐)

**文件**: [third_party/blink/renderer/core/html/resources/html.css](third_party/blink/renderer/core/html/resources/html.css)

```css
html, body {
  font-family: "Noto Sans CJK", sans-serif;  /* 添加这一行 */
}
```

**缺点**: 
- 违反 CSS 规范(UA 样式不应强制字体族)
- 优先级过高,可能覆盖网页样式

### 方案 2: 修改 GenericFontFamilySettings (推荐)

**文件**: [third_party/blink/renderer/platform/fonts/generic_font_family_settings.cc](third_party/blink/renderer/platform/fonts/generic_font_family_settings.cc)

```cpp
void GenericFontFamilySettings::Reset() {
  // ★ 设置 Android 默认字体
  #if BUILDFLAG(IS_ANDROID)
    standard_font_family_map_.Set(
      USCRIPT_COMMON, 
      AtomicString("Noto Sans CJK"));
    
    sans_serif_font_family_map_.Set(
      USCRIPT_COMMON,
      AtomicString("Noto Sans CJK"));
  #endif
}
```

**优点**: 
- 遵循标准流程
- 只影响未指定字体的元素
- 可按脚本(Script)分别配置

### 方案 3: 修改系统 fonts.xml (系统级)

**文件**: `/system/etc/fonts.xml`

```xml
<familyset>
  <family name="sans-serif">
    <font weight="400" style="normal">NotoSansCJK-Regular.ttf</font>
    <font weight="700" style="normal">NotoSansCJK-Bold.ttf</font>
  </family>
</familyset>
```

**优点**: 
- 影响所有应用
- 无需重新编译 Chromium

**缺点**: 
- 需要系统权限
- 仅限 Android

---

## 调试技巧

### 1. 打印字体解析过程

在 [font_builder.cc](third_party/blink/renderer/core/css/resolver/font_builder.cc) 添加日志:

```cpp
void FontBuilder::CreateFont(...) {
  FontDescription description = builder.GetFontDescription();
  
  LOG(INFO) << "GenericFamily: " << description.GenericFamily();
  LOG(INFO) << "Family: " << description.Family().FamilyName();
  
  // 继续原有逻辑...
}
```

### 2. 断点位置

- **样式计算入口**: `StyleResolver::ResolveStyle()`
- **字体创建**: `FontBuilder::CreateFont()`
- **系统字体查询**: `FontCache::GetGenericFamilyNameForScript()` (Android)
- **字体加载**: `FontCache::GetFontData()`

### 3. Chrome DevTools

1. 打开 DevTools → Elements
2. 选择元素 → Computed 面板
3. 查看 `font-family` 的 Computed Value
4. 点击属性可追溯来源(UA / User / Author)

---

## 常见问题

### Q1: 为什么 html.css 没有 font-family 还能显示字体?

**A**: 
1. `FontDescription` 默认 `kNoFamily`
2. `FontBuilder` 自动补充为 `kStandardFamily`
3. `GenericFontFamilySettings` 为空时触发系统查询
4. Android 从 `fonts.xml` 返回 "Roboto"

### Q2: 修改 html.css 为什么不生效?

**A**: 可能原因:
1. 网页 CSS 使用了 `!important` 覆盖
2. 内联样式 (inline style) 优先级更高
3. 缓存问题: 清除 `~/.config/chromium/Default/Cache`

### Q3: Android 和桌面版字体解析有何不同?

| 平台 | GenericFontFamilySettings | 系统查询 | 默认字体 |
|------|--------------------------|---------|---------|
| **Android** | 默认为空 | `fonts.xml` | Roboto |
| **Linux** | 预设 "DejaVu Sans" | fontconfig | DejaVu Sans |
| **Windows** | 预设 "Segoe UI" | DirectWrite | Segoe UI |
| **macOS** | 预设 "Helvetica Neue" | CoreText | Helvetica Neue |

### Q4: 如何让字体跟随系统默认字体?

**A**: 在 Android 上已经是这样的逻辑(通过 `fonts.xml`),如果想手动设置:

```cpp
// font_cache_android.cc
AtomicString FontCache::GetGenericFamilyNameForScript(...) {
  // 读取系统设置
  String system_default = GetSystemProperty("ro.config.default_font");
  if (!system_default.empty()) {
    return AtomicString(system_default);
  }
  
  // fallback 到 fonts.xml
  return MatchFamilyNameFromAndroidConfiguration(...);
}
```

---

## 总结

Chromium 的字体处理是一个多层系统:

1. **CSS 层**: 解析 `font-family` 属性,支持通用字体族关键字
2. **映射层**: `GenericFontFamilySettings` 将通用族映射到具体字体
3. **系统层**: `FontCache` 查询操作系统字体配置
4. **加载层**: Skia `SkFontMgr` 加载字体文件
5. **Shaping 层**: HarfBuzz 处理复杂文本布局
6. **渲染层**: Skia/FreeType 光栅化字形

关键点:
- **UA 样式不设置字体族** (符合 CSS 规范)
- **默认值通过 GenericFontFamilySettings 和系统查询决定**
- **Android 使用 fonts.xml 配置字体**
- **修改 GenericFontFamilySettings 是最标准的定制方式**

## 参考资料

- [CSS Fonts Module Level 4](https://www.w3.org/TR/css-fonts-4/)
- [Blink Font Architecture](https://chromium.googlesource.com/chromium/src/+/main/third_party/blink/renderer/platform/fonts/README.md)
- [HarfBuzz Documentation](https://harfbuzz.github.io/)
- [Skia SkFontMgr](https://skia.org/docs/user/api/SkFontMgr_Reference/)
