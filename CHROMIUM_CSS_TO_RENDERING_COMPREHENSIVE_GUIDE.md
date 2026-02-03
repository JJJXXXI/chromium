# Chromium/Blink 字体渲染完整指南：从 CSS 到像素

## 📋 目录

1. [核心数据结构](#核心数据结构)
2. [完整处理流程](#完整处理流程)
3. [CSS 值转换](#css-值转换)
4. [字体选择算法](#字体选择算法)
5. [生命周期管理](#生命周期管理)
6. [Android 特殊处理](#android-特殊处理)
7. [代码示例与位置](#代码示例与位置)

---

## 核心数据结构

### 1. 字体相关的主要类

```
ComputedStyle (最终计算的样式)
    ↓ 包含
    ├─ FontDescription (字体描述)
    │   ├─ family (FontFamily 链表)
    │   ├─ size (字体大小)
    │   ├─ weight (字体粗细)
    │   ├─ style (normal/italic)
    │   ├─ generic_family (serif/sans-serif/monospace 等)
    │   └─ script (书写系统)
    │
    └─ Font (当前字体对象)
        ├─ font_data (FontData* 指针)
        ├─ font_description (引用 ComputedStyle 的描述)
        └─ font_fallback_list (FontFallbackList*)

Font 对象 → FontFallbackList (多个字体的 fallback 链)
    ↓
    ├─ FontData (链表节点，抽象基类)
    │   ├─ SimpleFontData (单个具体字体) ⭐
    │   └─ SegmentedFontData (多个字体集合)
    │
    └─ SkTypeface (Skia 层的字体)
        ↓
        └─ 系统字体文件 (.ttf, .otf 等)
```

### 2. 详细数据结构定义

#### 2.1 ComputedStyle 中的字体部分

**文件**: `third_party/blink/renderer/core/css/computed_style.h`

```cpp
class ComputedStyle : public RefCounted<ComputedStyle> {
 private:
  // ★ 字体相关成员变量
  FontDescription font_description_;   // 字体描述
  Member<Font> font_;                  // 当前使用的 Font 对象
  
  // ... 其他 200+ 个 CSS 属性 ...
};
```

#### 2.2 FontDescription 类 - 字体配置容器

**文件**: `third_party/blink/renderer/platform/fonts/font_description.h:88-200`

```cpp
class PLATFORM_EXPORT FontDescription final {
 public:
  // ★ 通用字体族枚举
  enum GenericFamilyType {
    kNoFamily,                    // 未指定通用族
    kStandardFamily,              // serif (衬线)
    kSerifFamily,
    kSansSerifFamily,
    kMonospaceFamily,
    kCursiveFamily,
    kFantasyFamily,
    kSystemUiFamily,
  };

  // ★ 核心构造函数
  FontDescription() {
    fields_.generic_family_ = kNoFamily;  // ← 默认值!
    fields_.is_absolute_size_ = false;
    fields_.kerning_ = kAutoKerning;
    fields_.font_smoothing_ = kAutoSmoothing;
    // ... 默认值初始化 ...
  }

 private:
  struct FontDescriptionFields {
    // 字体族信息 (链表结构)
    Member<FontFamily> family_;        // ← FontFamily 对象 (可为 null)
    
    // 通用族映射 (用于 generic-family fallback)
    GenericFamilyType generic_family_; // ← 0=kNoFamily 或其他值
    
    // 字体大小 (像素或百分比)
    float size_;
    float letter_spacing_;
    
    // 字体变体
    int16_t weight_;  // 100-900
    FontStretch stretch_;  // normal, condensed, expanded
    FontStyle style_;  // normal, italic, oblique
    
    // 文本渲染特性
    FontSmoothing font_smoothing_;
    TextRendering text_rendering_;
    
    // 文本方向
    TextOrientation text_orientation_;
    WritingMode writing_mode_;
  } fields_;

 public:
  // ★ 关键访问器
  const FontFamily& Family() const { return *family_; }
  GenericFamilyType GenericFamily() const { return fields_.generic_family_; }
  float Size() const { return fields_.size_; }
  int Weight() const { return fields_.weight_; }
};
```

#### 2.3 FontFamily 链表 - 字体族列表

**文件**: `third_party/blink/renderer/platform/fonts/font_family.h`

```cpp
class FontFamily {
 public:
  enum Type {
    kFamilyName,      // "Georgia", "Roboto" 等具体字体名
    kGenericFamily,   // "serif", "sans-serif" 等通用族
  };

 private:
  AtomicString family_name_;      // 字体名 ("Georgia" 或 "serif")
  Type type_;                     // 是具体字体名还是通用族
  scoped_refptr<SharedFontFamily> next_;  // ← 链表指针!

 public:
  const AtomicString& FamilyName() const { return family_name_; }
  Type GetType() const { return type_; }
  
  // 遍历链表
  bool operator bool() const { return !family_name_.IsNull(); }
  FontFamily Next() const;
};

// 示例链表结构
// "font-family: Georgia, serif, sans-serif" 对应:
// 
// FontFamily {
//   family_name_: "Georgia",
//   type_: kFamilyName,
//   next_: → FontFamily {
//     family_name_: "serif",
//     type_: kGenericFamily,
//     next_: → FontFamily {
//       family_name_: "sans-serif",
//       type_: kGenericFamily,
//       next_: → nullptr
//     }
//   }
// }
```

#### 2.4 Font 类 - 运行时字体对象

**文件**: `third_party/blink/renderer/platform/fonts/font.h:80-150`

```cpp
class PLATFORM_EXPORT Font {
 private:
  // ★ 核心成员
  FontDescription font_description_;    // 字体配置
  Member<FontData> font_data_;          // 当前字体数据 (SimpleFontData*)
  Member<FontFallbackList> fallback_list_;  // Fallback 链表

 public:
  // ★ 字形查询接口
  const SimpleFontData* PrimaryFont() const;
  const SimpleFontData* FontDataForCharacter(UChar32 c) const;
  
  // 文本测量
  float Width(const TextRun& run) const;
  gfx::RectF BoundingBox(const TextRun& run) const;
};

// ★ 使用示例
Font font(font_description);
const SimpleFontData* font_data = font.FontDataForCharacter('A');
// 返回能渲染 'A' 的 SimpleFontData (可能是第一个字体, 也可能是 fallback)
```

#### 2.5 FontData 类层次 - 字体数据的多态关系

**文件**: `third_party/blink/renderer/platform/fonts/font_data.h`

```cpp
// ★ 抽象基类
class PLATFORM_EXPORT FontData : public GarbageCollected<FontData> {
 public:
  virtual ~FontData() = default;

  // ★ 核心虚函数 (子类必须实现)
  virtual const SimpleFontData* FontDataForCharacter(UChar32) const = 0;

  // ★ 状态查询
  virtual bool IsCustomFont() const = 0;      // @font-face 加载的字体?
  virtual bool IsLoading() const = 0;         // 正在加载中?
  virtual bool IsLoadingFallback() const = 0; // 加载 fallback 中?
  virtual bool IsSegmented() const = 0;       // 是分段字体?
  virtual bool ShouldSkipDrawing() const = 0; // 应该跳过绘制?
};

// ★ 实现 1: SimpleFontData - 单个字体文件
class PLATFORM_EXPORT SimpleFontData final : public FontData {
 private:
  Member<const FontPlatformData> platform_data_;  // ← SkTypeface 包装
  FontMetrics font_metrics_;
  float max_char_width_;
  float avg_char_width_;
  float space_width_;
  Glyph space_glyph_;
  Glyph zero_glyph_;
  Member<const CustomFontData> custom_font_data_;

 public:
  // ★ 虚函数实现
  const SimpleFontData* FontDataForCharacter(UChar32) const override {
    // SimpleFontData 就是终点 - 返回自己
    return this;
  }

  // ★ 字形查询
  Glyph GlyphForCharacter(UChar32) const;      // 返回 Glyph ID
  float WidthForGlyph(Glyph) const;            // 字形宽度
  gfx::RectF BoundsForGlyph(Glyph) const;      // 字形边界
};

// ★ 实现 2: SegmentedFontData - 多个字体集合
class PLATFORM_EXPORT SegmentedFontData final : public FontData {
 private:
  Vector<Member<SimpleFontData>> fonts_;  // ← 多个字体的数组
  Member<SimpleFontData> pages_[256];    // ← 字符范围索引

 public:
  // ★ 虚函数实现
  const SimpleFontData* FontDataForCharacter(UChar32 c) const override {
    // 使用字符代码点查询对应的 SimpleFontData
    if (pages_[c >> 8]) {
      return pages_[c >> 8];
    }
    // 未找到 → 返回第一个字体作为 fallback
    return fonts_[0];
  }
};
```

#### 2.6 FontFallbackList - 字体 Fallback 链

**文件**: `third_party/blink/renderer/platform/fonts/font_fallback_list.h`

```cpp
class FontFallbackList : public RefCounted<FontFallbackList> {
 private:
  // ★ Fallback 链表
  Vector<Member<FontData>> font_list_;  // [SimpleFontData, SimpleFontData, ...]
  
  // ★ 缓存 (性能优化)
  mutable HashMap<UChar32, const SimpleFontData*> cache_;

 public:
  // ★ 核心方法
  const SimpleFontData* PrimaryFont() const {
    return To<SimpleFontData>(font_list_[0].Get());
  }

  const SimpleFontData* FontDataForCharacter(UChar32 c) const {
    // 1. 查询缓存
    auto it = cache_.Find(c);
    if (it != cache_.end()) {
      return it->value;
    }

    // 2. 遍历 Fallback 链
    for (const auto& font_data : font_list_) {
      if (const SimpleFontData* result = 
          font_data->FontDataForCharacter(c)) {
        cache_.insert(c, result);
        return result;
      }
    }

    // 3. 返回最后的 fallback
    return RelationFallbackFont();
  }

  // ★ 使用示例
  // font-family: Georgia, Arial, sans-serif 构建的 FontFallbackList:
  // font_list_: [
  //   SimpleFontData("Georgia"),    ← primary
  //   SimpleFontData("Arial"),      ← fallback 1
  //   SimpleFontData("DejaVu Sans") ← fallback 2 (system sans-serif)
  // ]
};
```

---

## 完整处理流程

### 流程图

```
1️⃣ HTML 解析与 DOM 构建
    ↓ (HTMLDocumentParser → HTMLTreeBuilder)
    └─> Document::FinishedParsing()

2️⃣ 样式重计算触发
    ↓ (Element::RecalcStyle → StyleResolver::ResolveStyle)
    └─> CSS 规则匹配

3️⃣ CSS 属性应用
    ↓ (StyleCascade::Apply → StyleBuilder::ApplyProperty)
    └─> StyleResolverState::UpdateFont() ★ 字体处理入口

4️⃣ FontDescription 创建
    ↓ (FontBuilder::CreateFont)
    ├─> CSS 值转换为 FontFamily 链表
    ├─> 映射通用族到具体字体名
    └─> 创建 ComputedStyle::font_description_

5️⃣ Font 对象创建
    ↓ (Font 构造)
    └─> FontFallbackList 构建

6️⃣ 字体加载与缓存
    ↓ (FontFallbackList::FontDataForCharacter)
    ├─> FontSelector::GetFontData()
    ├─> FontCache::GetFontData()
    └─> SimpleFontData 创建

7️⃣ 文本 Shaping
    ↓ (HarfBuzzShaper::Shape)
    ├─> 字符序列 + Font → HarfBuzz 处理
    └─> 生成 Glyph ID + 位置信息

8️⃣ 绘制渲染
    └─> GraphicsContext::DrawText
        └─> Skia Canvas::DrawGlyphs
            └─> 屏幕像素
```

### 详细步骤分解

#### 步骤 1: HTML 解析与 DOM 构建

**文件**: `third_party/blink/renderer/core/html/parser/html_document_parser.cc`

```cpp
void HTMLDocumentParser::Append(const String& input) {
  // 将 HTML 文本流转为 Token
  while (kHTMLTokenizer.nextToken(input, token)) {
    // 构建 DOM 树
    GetHTMLTreeBuilder()->ConstructTree(token);
  }
}

// HTML 解析完成后
void Document::FinishedParsing() {
  // 进入样式计算阶段
  SetReadyState(kInteractive);
  BeginLifecycleUpdatesIfRenderingReady();
}
```

#### 步骤 2: 样式重计算触发

**文件**: `third_party/blink/renderer/core/dom/document.cc`

```cpp
void Document::UpdateStyleAndLayoutTree() {
  // 1. 遍历所有需要重计算样式的元素
  GetStyleEngine().UpdateStyle();
}

void StyleEngine::UpdateStyle() {
  // 2. 对每个元素调用 Element::RecalcStyle()
  for (Element* element : elements_needing_style_recalc_) {
    element->RecalcStyle(recalc_context);
  }
}
```

#### 步骤 3: StyleResolver 核心算法

**文件**: `third_party/blink/renderer/core/css/resolver/style_resolver.cc:120-200`

```cpp
const ComputedStyle* StyleResolver::ResolveStyle(
    Element* element,
    const StyleRecalcContext& style_recalc_context,
    const StyleRequest& style_request) {
  
  // 1. 创建状态对象 (保存计算过程)
  StyleResolverState state(GetDocument(), *element, 
                          &style_recalc_context, style_request);
  
  // 2. 创建 CSS 级联对象
  StyleCascade cascade(state);
  
  // ★ 3. 应用基础样式 (包括 font-family)
  ApplyBaseStyle(element, style_recalc_context, 
                 style_request, state, cascade);
  
  // 4. 应用动画样式
  ApplyAnimatedStyle(state, cascade, style_recalc_context);
  
  // 5. 返回最终计算样式
  return state.TakeStyle();
}
```

#### 步骤 4: CSS 规则匹配与属性应用

**文件**: `third_party/blink/renderer/core/css/resolver/style_cascade.cc:50-150`

```cpp
void StyleCascade::Apply() {
  // 1. 收集匹配的 CSS 规则
  // - UA stylesheet (html.css)
  // - User stylesheet
  // - Author stylesheet (网页 CSS)
  // - Inline styles (style 属性)
  
  const StylePropertySet* matched_properties = ...;
  
  // 2. 遍历所有 CSS 属性
  for (const CSSProperty& property : matched_properties) {
    
    // 对于 font-family 属性:
    if (property.Id() == CSSPropertyID::kFontFamily) {
      StyleBuilder::ApplyProperty(
          CSSPropertyID::kFontFamily,
          state_,
          property.Value());  // ← CSSValueList
    }
    
    // 对于其他属性 (size, weight 等):
    StyleBuilder::ApplyProperty(...);
  }
  
  // ★ 3. 应用完毕 → 触发字体创建
  state_.UpdateFont();  ★★★ 关键调用
}
```

---

## CSS 值转换

### 关键函数: StyleBuilderConverter::ConvertFontFamily()

**文件**: `third_party/blink/renderer/core/css/resolver/style_builder_converter.cc:510-590`

这是 **CSS 值转换为内部 FontDescription** 的核心!

```cpp
FontDescription::FamilyDescription 
StyleBuilderConverter::ConvertFontFamily(
    StyleResolverState& state,
    const CSSValue& value) {
  
  // value 是 CSS Parser 的输出
  // 例: CSSValueList[CSSFontFamilyValue("Georgia"), CSSIdentifierValue(serif)]
  
  return StyleBuilderConverterBase::ConvertFontFamily(
      value,
      &state.GetFontBuilder(),
      &state.GetDocument());
}

// ★★★ 核心转换逻辑
FontDescription::FamilyDescription 
StyleBuilderConverterBase::ConvertFontFamily(
    const CSSValue& value,
    FontBuilder* font_builder,
    const Document* document_for_count) {
  
  FontDescription::FamilyDescription desc(FontDescription::kNoFamily);
  
  bool has_value = false;
  AtomicString family_name;
  FontDescription::GenericFamilyType family_type = FontDescription::kNoFamily;
  scoped_refptr<SharedFontFamily> next;

  // 1️⃣ 反向遍历 font-family 列表
  // 反向处理是为了构建链表 (最后一个 fallback 是 next 指针)
  for (auto& family : base::Reversed(To<CSSValueList>(value))) {
    
    AtomicString next_family_name;
    FontDescription::GenericFamilyType generic_family = 
        FontDescription::kNoFamily;
    
    // 2️⃣ 处理单个值 (详见下面的子函数)
    if (!ConvertFontFamilyName(*family, generic_family, next_family_name,
                               font_builder, document_for_count)) {
      continue;
    }
    
    // 3️⃣ 构建链表节点
    if (has_value) {
      // 将之前处理的节点链上来
      next = SharedFontFamily::Create(family_name, family_type, std::move(next));
    }
    
    family_name = next_family_name;
    family_type = (generic_family != FontDescription::kNoFamily) 
                  ? FontFamily::Type::kGenericFamily
                  : FontFamily::Type::kFamilyName;
    has_value = true;
  }

  // 4️⃣ 返回最终的 FamilyDescription
  desc.family = FontFamily(family_name, family_type, std::move(next));
  return desc;
}

// ★ 子函数: 处理单个字体值
static bool ConvertFontFamilyName(
    const CSSValue& value,
    FontDescription::GenericFamilyType& out_generic_family,
    AtomicString& out_family_name,
    FontBuilder* font_builder,
    const Document* document_for_count) {
  
  // Case 1: "具体字体名" (如 "Georgia")
  if (auto* font_family_value = DynamicTo<CSSFontFamilyValue>(value)) {
    out_generic_family = FontDescription::kNoFamily;      // ← 不是通用族!
    out_family_name = font_family_value->Value();         // ← "Georgia"
    return true;
  }
  
  // Case 2: "CSS 关键字" (如 "serif")
  if (auto* identifier = DynamicTo<CSSIdentifierValue>(value)) {
    // 转换关键字为枚举值
    auto generic_family = ConvertGenericFamily(identifier->GetValueID());
    
    if (generic_family != FontDescription::kNoFamily) {
      out_generic_family = generic_family;  // ← kSerifFamily 等
      
      // 向 FontBuilder 请求该通用族对应的具体字体
      if (font_builder) {
        out_family_name = font_builder->GenericFontFamilyName(generic_family);
      }
      return true;
    }
  }
  
  return false;
}

// ★ 关键映射函数
static FontDescription::GenericFamilyType ConvertGenericFamily(
    CSSValueID value_id) {
  switch (value_id) {
    case CSSValueID::kSerif:
      return FontDescription::kSerifFamily;
    case CSSValueID::kSansSerif:
      return FontDescription::kSansSerifFamily;
    case CSSValueID::kMonospace:
      return FontDescription::kMonospaceFamily;
    case CSSValueID::kCursive:
      return FontDescription::kCursiveFamily;
    case CSSValueID::kFantasy:
      return FontDescription::kFantasyFamily;
    case CSSValueID::kSystemUi:
      return FontDescription::kSystemUiFamily;
    default:
      return FontDescription::kNoFamily;
  }
}
```

### 转换示例

#### 示例 1: 具体字体名

```
CSS: <p style="font-family: Georgia">

输入:
  CSSFontFamilyValue("Georgia")

输出:
  FontDescription {
    generic_family: kNoFamily
    family: FontFamily {
      family_name: "Georgia"
      type: kFamilyName
      next: nullptr
    }
  }
```

#### 示例 2: 混合通用族和具体字体

```
CSS: <p style="font-family: Georgia, serif">

输入:
  CSSValueList [
    CSSFontFamilyValue("Georgia"),
    CSSIdentifierValue(serif)
  ]

转换过程 (反向遍历):
  1. 处理 CSSIdentifierValue(serif)
     → generic_family = kSerifFamily
     → family_name = FontBuilder::GenericFontFamilyName(kSerifFamily)
                   = "Roboto" (Android) 或 "DejaVu Serif" (Linux)
     
  2. 处理 CSSFontFamilyValue("Georgia")
     → 将之前的 Serif 链接为 next
     → generic_family = kNoFamily
     → family_name = "Georgia"

输出:
  FontDescription {
    generic_family: kNoFamily
    family: FontFamily {
      family_name: "Georgia"
      type: kFamilyName,
      next: → FontFamily {
        family_name: "Roboto"
        type: kGenericFamily
        next: nullptr
      }
    }
  }

字体查询顺序:
  1. 尝试加载 "Georgia"
  2. 如果失败,使用 "Roboto" (serif fallback)
  3. 如果仍失败,系统 fallback
```

---

## 字体选择算法

### FontSelector 与 FontCache 交互

**文件**: `third_party/blink/renderer/platform/fonts/font_selector.cc`

```cpp
// ★ 核心接口: 根据字体描述和字体名获取字体数据
const SimpleFontData* FontSelector::GetFontData(
    const FontDescription& font_description,
    const AtomicString& family_name) {
  
  // 1️⃣ 查询是否是通用族 (serif, sans-serif 等)
  bool is_generic_family = IsGenericFamily(family_name);
  
  if (is_generic_family) {
    // 2️⃣ 如果是通用族,从设置中获取映射的具体字体
    family_name = FamilyNameFromSettings(
      font_description,
      family_name,
      content_locale_);
  }
  
  // 3️⃣ 调用 FontCache 加载字体
  return FontCache::GetFontData(font_description, family_name);
}

// ★ 通用族映射
AtomicString FontSelector::FamilyNameFromSettings(
    const FontDescription& font_description,
    const AtomicString& generic_family,
    const LayoutLocale* content_locale) {
  
  if (!generic_family_settings_) {
    return "";
  }
  
  // 根据脚本类型获取映射
  UScriptCode script = font_description.GetScript();
  
  // 查询映射表: generic_family_settings_->standard_font_family_map_[script]
  const AtomicString& specific_font = 
    generic_family_settings_->Standard(script);
  
  if (!specific_font.IsEmpty()) {
    return specific_font;  // ← 返回映射的具体字体
  }
  
  // 如果映射表为空,则根据平台查询系统字体
  #if BUILDFLAG(IS_ANDROID)
    return FontCache::GetGenericFamilyNameForScript(
        generic_family,
        font_description,
        content_locale);
  #endif
  
  return "";
}
```

### FontCache 字体加载算法

**文件**: `third_party/blink/renderer/platform/fonts/font_cache.cc`

```cpp
// ★ FontCache 是全局单例
FontCache* FontCache::GetFontCache() {
  DEFINE_THREAD_SAFE_STATIC_LOCAL(FontCache, cache, {});
  return &cache;
}

// ★ 核心方法: 根据描述和字体名加载字体
const SimpleFontData* FontCache::GetFontData(
    const FontDescription& font_description,
    const AtomicString& family_name) {
  
  // 1️⃣ 查询缓存 (性能优化)
  FontCacheKey cache_key(font_description, family_name);
  
  auto it = font_data_cache_.find(cache_key);
  if (it != font_data_cache_.end()) {
    return it->second;  // ← 命中缓存,直接返回
  }
  
  // 2️⃣ 缓存未命中,创建新的 FontPlatformData
  FontPlatformData* platform_data = GetFontPlatformData(
      font_description,
      FontFaceCreationParams(family_name));
  
  if (!platform_data) {
    return nullptr;  // ← 字体不存在
  }
  
  // 3️⃣ 基于 FontPlatformData 创建 SimpleFontData
  SimpleFontData* font_data = 
    FontDataFromFontPlatformData(platform_data);
  
  // 4️⃣ 存入缓存
  font_data_cache_[cache_key] = font_data;
  
  return font_data;
}

// ★ 获取平台字体对象
FontPlatformData* FontCache::GetFontPlatformData(
    const FontDescription& font_description,
    const FontFaceCreationParams& creation_params) {
  
  FontPlatformDataCacheKey key(font_description, creation_params);
  
  // 查询平台数据缓存
  auto it = typeface_cache_.find(key);
  if (it != typeface_cache_.end()) {
    return &it->second;
  }
  
  // 调用平台相关代码创建
  std::unique_ptr<FontPlatformData> platform_data =
    CreateFontPlatformData(font_description, creation_params);
  
  if (!platform_data) {
    return nullptr;
  }
  
  auto result = typeface_cache_.emplace(key, *platform_data);
  return &result.first->second;
}
```

---

## 生命周期管理

### ComputedStyle 与 Font 对象的关系

```
1. ComputedStyle 创建
   ├─ new ComputedStyle()
   └─ font_description_ = FontDescription()  ← 默认值

2. StyleBuilder::ApplyProperty() 修改属性
   ├─ font-family CSS 值 → FontFamily 链表
   ├─ font-size CSS 值 → float size
   ├─ font-weight CSS 值 → int weight
   └─ 其他 font-* 属性 → FontDescription 字段

3. StyleResolverState::UpdateFont() 创建 Font
   ├─ FontBuilder::CreateFont()
   │  ├─ 处理通用族映射
   │  ├─ 调用 GenericFontFamilySettings
   │  └─ 生成最终 FontDescription
   └─ new Font(font_description)
      ├─ FontFallbackList 构建 (第一次字体加载)
      └─ 调用 FontCache::GetFontData()

4. Font 对象生命周期
   ├─ 与 ComputedStyle 绑定
   ├─ 缓存在 ComputedStyle::font_
   └─ ComputedStyle 销毁 → Font 也销毁

5. FontData 缓存池
   ├─ FontCache 维护全局缓存
   ├─ 跨 ComputedStyle 共享
   └─ 应用运行期间一直存活
```

### 缓存分层

```
第1层: ComputedStyle 级别缓存
├─ 每个元素的样式包含一个 Font 对象
├─ 多个元素可共享相同的 ComputedStyle (性能优化)
└─ 元素删除 → ComputedStyle 释放 → Font 释放

第2层: FontFallbackList 级别缓存
├─ 针对特定 FontDescription 缓存 SimpleFontData 链表
├─ HashMap<UChar32, const SimpleFontData*> 缓存字符查询结果
└─ 生存期: 与 Font 对象相同

第3层: FontCache 全局缓存 ⭐⭐⭐
├─ HashMap<FontCacheKey, SimpleFontData*> font_data_cache_
├─ HashMap<FontPlatformDataCacheKey, FontPlatformData> typeface_cache_
├─ 跨应用生存期缓存 (最后一次使用后仍保留)
├─ 线程安全 (Mutex 保护)
└─ 可手动清理: FontCache::InvalidateAllFontData()

第4层: 平台层缓存
├─ Skia SkTypeface 缓存
├─ HarfBuzz hb_font_t 缓存
└─ 字体文件内存映射 (mmap)
```

### 具体例子: "Georgia" 字体加载过程

```
Step 1: CSS 解析
  font-family: Georgia;
  → CSSFontFamilyValue("Georgia")

Step 2: 样式计算
  StyleBuilder::ApplyProperty(font-family, "Georgia")
  → FontDescription::family = "Georgia"

Step 3: UpdateFont() 调用
  FontBuilder::CreateFont()
  → no mapping (not generic family)
  → FontDescription ready

Step 4: Font 对象创建
  new Font(font_description)
  → FontFallbackList 构建开始

Step 5: 首次字体查询
  FontFallbackList::GetFontData('A')  ← 第一个使用的字符
  → FontSelector::GetFontData(font_description, "Georgia")
  → FontCache::GetFontData(font_description, "Georgia")
  
  [FontCache 查询 typeface_cache_]
  ❌ 缓存未命中
  → CreateFontPlatformData()
    → SkFontMgr->matchFamilyStyle("Georgia", style)
    → SkTypeface* 返回
  → new FontPlatformData(typeface)
  → new SimpleFontData(platform_data)
  
  [存入缓存]
  font_data_cache_[key] = SimpleFontData*
  typeface_cache_[key] = FontPlatformData
  
  → 返回 SimpleFontData*

Step 6: 后续字符查询
  FontFallbackList::GetFontData('B')
  → 缓存命中!
  → 直接返回 SimpleFontData*

Step 7: 文本渲染
  HarfBuzzShaper::Shape()
  → font_data->GetHarfBuzzFace()
  → hb_shape(hb_font, buffer, ...)
  → ShapeResult 包含 glyph ids 和位置

Step 8: 屏幕绘制
  GraphicsContext::DrawText()
  → canvas_->drawGlyphs(glyph_ids, positions, ...)
  → Skia 光栅化
  → 屏幕缓冲区像素
```

---

## Android 特殊处理

### Android 字体查询路径

**文件**: `third_party/blink/renderer/platform/fonts/android/font_cache_android.cc`

```cpp
// ★ Android 特殊入口
FontPlatformData* FontCache::CreateFontPlatformData(
    const FontDescription& font_description,
    const FontFaceCreationParams& creation_params) {
  
  // 1️⃣ 获取 SkFontMgr 实例
  SkFontMgr* font_manager = GetFontManager();
  // ↓ 内部调用 SkFontMgr_New_Android()
  // ↓ 解析 /system/etc/fonts.xml
  
  // 2️⃣ 查询字体
  sk_sp<SkTypeface> typeface = font_manager->matchFamilyStyle(
      creation_params.Family().Utf8().c_str(),   // 字体名 "Georgia" 或 "sans-serif"
      SkFontStyle(
          weight,    // 100-900
          width,     // 100%
          slant));   // normal/italic
  
  if (!typeface) {
    // 3️⃣ Fallback: 使用系统默认字体
    typeface = font_manager->legacyMakeTypeface(nullptr, slant);
  }
  
  // 4️⃣ 包装为 FontPlatformData
  return std::make_unique<FontPlatformData>(
      typeface,
      font_description.ComputedSize());
}

// ★ Android 通用族映射
AtomicString FontCache::GetGenericFamilyNameForScript(
    const AtomicString& generic_family,
    const FontDescription& font_description,
    const LayoutLocale* content_locale) {
  
  // Android 没有预设的 generic_family_map_ (为空)
  // 所以直接调用此函数获取系统字体
  
  // 查询 fonts.xml 配置
  // generic_family = "sans-serif" → 返回 "Roboto"
  // generic_family = "serif" → 返回 "Noto Serif"
  // generic_family = "monospace" → 返回 "Roboto Mono"
  
  return MatchFamilyNameFromAndroidConfiguration(
      generic_family, 
      font_description.GetScript());
}
```

### fonts.xml 配置示例

**文件**: `/system/etc/fonts.xml` (Android 系统)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<familyset>
  <!-- 主字体族定义 -->
  <family name="sans-serif">
    <font weight="400" style="normal">Roboto-Regular.ttf</font>
    <font weight="400" style="italic">Roboto-Italic.ttf</font>
    <font weight="700" style="normal">Roboto-Bold.ttf</font>
    <font weight="700" style="italic">Roboto-BoldItalic.ttf</font>
  </family>

  <family name="serif">
    <font weight="400" style="normal">NotoSerifRegular.ttf</font>
    <font weight="700" style="normal">NotoSerifBold.ttf</font>
  </family>

  <family name="monospace">
    <font weight="400" style="normal">Roboto-Mono.ttf</font>
  </family>

  <!-- Fallback 链 -->
  <alias name="sans-serif-thin" to="sans-serif" />
  <fallback_chain>
    <font>Roboto-Regular.ttf</font>
    <font>Noto-Emoji-Regular.ttf</font>
  </fallback_chain>
</familyset>
```

### Chromium 中文字体支持 (Android)

```cpp
// 在 android/font_cache_android.cc 中添加

void FontCache::RegisterChineseFonts() {
  // Android 系统字体路径
  const char* chinese_fonts[] = {
    "/system/fonts/NotoSansCJK-Regular.ttf",        // 思源黑体
    "/system/fonts/Roboto-Regular.ttf",             // 备用字体
    "/data/fonts/custom_chinese_font.ttf",          // 用户自定义字体
  };
  
  // 注册到 SkFontMgr
  for (const char* font_path : chinese_fonts) {
    sk_sp<SkTypeface> typeface = SkTypeface::MakeFromFile(
        font_path, 0);
    if (typeface) {
      // 添加到字体管理器
      AddToFontRegistry("Noto Sans CJK", typeface);
    }
  }
}
```

---

## 代码示例与位置

### 1. ComputedStyle 中的字体使用

```cpp
// File: third_party/blink/renderer/core/css/computed_style.h

class ComputedStyle {
  // 获取字体
  const Font& GetFont() const {
    return font_;
  }

  // 获取字体描述
  const FontDescription& GetFontDescription() const {
    return font_description_;
  }

  // 修改字体描述
  void SetFontDescription(const FontDescription& font_description) {
    font_description_ = font_description;
  }
};

// ★ 使用示例
const ComputedStyle* style = element->GetComputedStyle();
const Font& font = style->GetFont();

// 查询字符 'A' 的 SimpleFontData
const SimpleFontData* font_data = font.FontDataForCharacter('A');
```

### 2. FontBuilder 使用示例

```cpp
// File: third_party/blink/renderer/core/css/resolver/font_builder.cc

void FontBuilder::CreateFont(
    ComputedStyleBuilder& builder,
    const ComputedStyle* parent_style) {
  
  FontDescription description = builder.GetFontDescription();
  
  // 处理继承
  if (description.Size() == 0) {
    description.SetSize(parent_style->GetFontDescription().Size());
  }
  
  // 处理通用族
  if (description.GenericFamily() != FontDescription::kNoFamily) {
    UpdateGenericFontFamilySettings(document_, builder, description);
  }
  
  // 更新样式
  builder.SetFontDescription(description);
  
  // ★ 创建 Font 对象 (第一次加载字体!)
  builder.SetFont(Font(description));
}
```

### 3. FontFallbackList 查询流程

```cpp
// File: third_party/blink/renderer/platform/fonts/font_fallback_list.cc

const SimpleFontData* FontFallbackList::GetFontData(
    UChar32 character) const {
  
  // 1️⃣ 查询缓存
  auto it = font_character_map_.find(character);
  if (it != font_character_map_.end()) {
    return it->value;
  }
  
  // 2️⃣ 遍历 fallback 链
  for (const auto& font_data : font_list_) {
    // 询问每个 FontData 是否能渲染该字符
    if (font_data->IsSegmented()) {
      // SegmentedFontData: 包含多个字体的数组
      if (const SimpleFontData* simple_font = 
          DynamicCast<SegmentedFontData>(font_data)->FontDataForCharacter(character)) {
        font_character_map_[character] = simple_font;
        return simple_font;
      }
    } else {
      // SimpleFontData: 单个字体
      if (const SimpleFontData* simple_font = 
          To<SimpleFontData>(font_data.Get())) {
        font_character_map_[character] = simple_font;
        return simple_font;
      }
    }
  }
  
  // 3️⃣ 系统 fallback
  const SimpleFontData* fallback_font = RelationFallbackFont();
  font_character_map_[character] = fallback_font;
  return fallback_font;
}
```

### 4. SimpleFontData 的字形查询

```cpp
// File: third_party/blink/renderer/platform/fonts/simple_font_data.cc

Glyph SimpleFontData::GlyphForCharacter(UChar32 codepoint) const {
  // 1️⃣ 查询缓存 (常用字符)
  if (codepoint == ' ') {
    return space_glyph_;
  }
  if (codepoint == '0') {
    return zero_glyph_;
  }
  
  // 2️⃣ 查询 SkTypeface
  SkTypeface* typeface = platform_data_->Typeface();
  
  // ★ 核心: 字符代码 → Glyph ID
  Glyph glyph = typeface->unicharToGlyph(codepoint);
  
  return glyph;
}

// ★ 使用示例
SimpleFontData* font_data = ...;
Glyph glyph_a = font_data->GlyphForCharacter('A');      // → Glyph ID 68
float width = font_data->WidthForGlyph(glyph_a);        // → 600 (设计单位)
gfx::RectF bounds = font_data->BoundsForGlyph(glyph_a); // → {0, -100, 600, 700}
```

### 5. 文本 Shaping (HarfBuzz)

```cpp
// File: third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc

ShapeResult* HarfBuzzShaper::Shape(const TextRun& run) {
  // 1️⃣ 从 Font 获取 SimpleFontData
  const SimpleFontData* primary_font = run.font->PrimaryFont();
  
  // 2️⃣ 获取 HarfBuzz 字体对象
  hb_font_t* hb_font = primary_font->GetHarfBuzzFace()->GetScaledFont(
      run.font->GetFontDescription().ComputedSize());
  
  // 3️⃣ 创建 HarfBuzz 缓冲区
  hb_buffer_t* buffer = hb_buffer_create();
  hb_buffer_add_utf16(buffer,
      (const uint16_t*)run.text,
      run.length,
      0,
      run.length);
  
  // 4️⃣ Shaping (关键算法)
  hb_shape(hb_font, buffer, nullptr, 0);
  
  // 5️⃣ 提取结果
  unsigned glyph_count;
  hb_glyph_info_t* glyph_infos = hb_buffer_get_glyph_infos(buffer, &glyph_count);
  hb_glyph_position_t* glyph_positions = 
      hb_buffer_get_glyph_positions(buffer, &glyph_count);
  
  // 6️⃣ 创建 ShapeResult 对象
  ShapeResult* result = CreateShapeResult(glyph_infos, glyph_positions, glyph_count);
  
  hb_buffer_destroy(buffer);
  return result;
}
```

### 6. 完整调用栈示例

从 CSS font-family 到渲染像素的完整流程:

```cpp
// ========================
// 第一层: CSS 解析
// ========================
CSSParser::ParseDeclaration("font-family: Georgia, serif")
  → CSSValueList([CSSFontFamilyValue("Georgia"), CSSIdentifierValue(serif)])

// ========================
// 第二层: 样式计算
// ========================
Element::RecalcStyle()
  → StyleResolver::ResolveStyle()
    → StyleCascade::Apply()
      → StyleBuilder::ApplyProperty(kFontFamily, CSSValueList)
        → StyleBuilderConverter::ConvertFontFamily()
          → [结果] FontDescription { family: "Georgia" → "serif" }
      → StyleResolverState::UpdateFont()
        → FontBuilder::CreateFont()
          → new Font(FontDescription)

// ========================
// 第三层: 字体加载 (第一次使用字符时)
// ========================
TextLayout::ComputeMetrics()
  → Font::FontDataForCharacter('H')
    → FontFallbackList::GetFontData('H')
      → FontSelector::GetFontData(FontDescription, "Georgia")
        → FontCache::GetFontData(FontDescription, "Georgia")
          → FontCache::GetFontPlatformData()
            → FontCache::CreateFontPlatformData()
              → SkFontMgr->matchFamilyStyle("Georgia", style)
                → SkTypeface* (从 fonts.xml 或系统字体库)
          → new SimpleFontData(FontPlatformData)
          → 存入缓存
          → return SimpleFontData*

// ========================
// 第四层: 文本 Shaping
// ========================
TextLayout::ShapeText("Hello")
  → HarfBuzzShaper::Shape()
    → hb_font_t* = SimpleFontData->GetHarfBuzzFace()
    → hb_buffer_add_utf16("Hello")
    → hb_shape(hb_font, buffer)  ← ★ 核心 Shaping 算法
    → ShapeResult([Glyph 68, Glyph 69, Glyph 76, ...], [positions...])

// ========================
// 第五层: 渲染绘制
// ========================
PaintText()
  → GraphicsContext::DrawText(font, shape_result)
    → canvas_->drawGlyphs(glyph_ids, positions, SkFont)
      → Skia: FreeType->RasterizeGlyph()
      → Paint to canvas
      → Display list sent to compositor
      → GPU rasterization
      → Screen pixels ★★★
```

---

## 关键 API 汇总表

| 功能 | 类/函数 | 文件 | 说明 |
|-----|--------|------|------|
| CSS 值转换 | `StyleBuilderConverter::ConvertFontFamily()` | style_builder_converter.cc:510 | CSS 值 → FontFamily 链表 |
| 字体描述 | `FontDescription` | font_description.h | 字体配置容器 |
| 字体对象 | `Font` | font.h | 运行时字体对象 |
| 单个字体 | `SimpleFontData` | simple_font_data.h | 单个字体文件数据 |
| 字体数组 | `SegmentedFontData` | segmented_font_data.h | 多个字体的集合 |
| Fallback 链 | `FontFallbackList` | font_fallback_list.h | 字体 fallback 管理 |
| 全局缓存 | `FontCache` | font_cache.h | 全局字体缓存单例 |
| 字体选择 | `FontSelector` | font_selector.h | 字体选择和映射 |
| 通用族映射 | `GenericFontFamilySettings` | generic_font_family_settings.h | 通用族 → 具体字体 |
| 文本 Shaping | `HarfBuzzShaper` | harfbuzz_shaper.cc | 字符 → 字形位置 |
| 平台层 | `FontPlatformData` | font_platform_data.h | 平台字体对象包装 |
| Skia 集成 | `SkFontMgr` | third_party/skia/src | 系统字体查询 |
| Android 特殊 | `FontCache::GetGenericFamilyNameForScript()` | font_cache_android.cc | Android fonts.xml 查询 |

---

## 性能优化点

### 1. 多层缓存

```cpp
// 第1层: ComputedStyle 级别
ComputedStyle {
  Font font_;  // ← 缓存 Font 对象
}

// 第2层: FontFallbackList 级别
FontFallbackList {
  HashMap<UChar32, SimpleFontData*> cache_;  // ← 缓存字符查询
}

// 第3层: FontCache 全局
FontCache {
  HashMap<FontCacheKey, SimpleFontData*> font_data_cache_;  // ← 全局缓存
  HashMap<FontPlatformDataCacheKey, FontPlatformData> typeface_cache_;
}

// 第4层: Skia/HarfBuzz 缓存
SkTypeface { ... }  // ← 字体文件
hb_font_t { ... }   // ← 预计算数据
```

### 2. 惰性加载

```cpp
// Font 对象创建时不加载任何字体
new Font(font_description)  // ← 仅创建描述

// 第一次使用字符时才加载
Font::FontDataForCharacter('A')  // ← 触发 FontCache 查询
```

### 3. 字符缓存

```cpp
// 常用字符缓存在 SimpleFontData
SimpleFontData {
  Glyph space_glyph_;     // 空格
  Glyph zero_glyph_;      // '0'
  float space_width_;
  // ↑ 避免反复查询
}
```

---

## 常见问题解答

**Q1: 字体加载何时发生?**
A: 两个阶段:
- 第1阶段: Style 计算时 (UpdateFont) - 构建 FontDescription
- 第2阶段: 文本使用时 (FontFallbackList::GetFontData) - 真实加载字体文件

**Q2: @font-face 自定义字体如何处理?**
A: CustomFontData 类标记:
```cpp
SimpleFontData {
  Member<const CustomFontData> custom_font_data_;
  bool IsCustomFont() { return custom_font_data_; }
  bool IsLoading() { return custom_font_data_->IsLoading(); }
}
```

**Q3: 字体回退 (fallback) 如何工作?**
A: FontFallbackList 链表:
```
Font {
  font_list_: [
    SimpleFontData("Georgia"),     ← 第1选择
    SimpleFontData("Serif系统字体"), ← 第2选择
    SimpleFontData("系统默认字体"),   ← 第3选择
  ]
}
```

**Q4: 如何禁用字体加载?**
A: 在 prefs_tab_helper.cc:82 添加条件:
```cpp
#if !BUILDFLAG(IS_ANDROID) || BUILDFLAG(ENABLE_FONT_PREFERENCES)
  RegisterFontFamilyPrefs(registry, ...);
#endif
```

**Q5: 中文字体如何支持?**
A: SkFontMgr 自动处理:
```
中文字符 'A' (U+4E2D)
  → HarfBuzzShaper
  → SkFontMgr::matchFamilyStyle("Noto Sans CJK")
  → SkTypeface 返回支持 CJK 的字体
  → 字形 shaping 和渲染
```

---

## 参考资源

- [Blink Fonts 架构](https://docs.google.com/document/d/1dHHfJ4FlWMrSEkV-m1FY-OJUGjVfB55ZYXCeaL9LBjk)
- [CSS Font Module Level 3](https://www.w3.org/TR/css-fonts-3/)
- [HarfBuzz 官方文档](https://harfbuzz.github.io/)
- [Skia 字体系统](https://skia.org/)
- [Chromium 字体设计文档](https://www.chromium.org/developers/design-documents/font-rendering)
