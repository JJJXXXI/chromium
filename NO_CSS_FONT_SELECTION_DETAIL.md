# 无 CSS 样式时的字体选择详解

## 核心问题

网页元素没有任何 CSS 样式指定 `font-family`，但为什么仍然能显示字体？字体是如何从 Android 系统中选出的？

---

## 极简答案

```
FontDescription(kNoFamily)
  ↓ [使用初始值]
FontDescription(kStandardFamily)
  ↓ [系统查询 fonts.xml]
"Roboto" (字体名)
  ↓ [加载字体文件]
屏幕显示 Roboto 字体
```

---

## 完整代码执行链路 - CSS 解析到字体选择流程图

```
HTML 解析 (无 font-family CSS)
    ↓
StyleResolver::ResolveStyle()
    ↓
━━━━━ CSS 应用阶段 ━━━━━
    ↓
StyleCascade::Apply()  [应用 CSS 级联]
    ├─ 若存在 font-family CSS 值:
    │    ↓
    │    StyleBuilder::ApplyProperty(CSSPropertyID::kFontFamily, ...)
    │    ↓
    │    StyleBuilderConverter::ConvertFontFamily()  ← 详见步骤 1A
    │    ↓
    │    FontDescription 被设为指定的字体
    │
    └─ 若无 font-family CSS (初始值):
         FontDescription 保留初始值 (kNoFamily)
    ↓
━━━━━ UpdateFont 阶段 ━━━━━
    ↓
StyleResolverState::UpdateFont()  [触发字体对象创建]
    ↓
FontBuilder::CreateFont()  [创建 Font 对象]
    ├─ 若 FontDescription.kNoFamily:
    │    ↓
    │    InitialGenericFamily() → kStandardFamily  ← 详见步骤 2A
    │    ↓
    │    FontDescription 被设为初始值
    │
    └─ 若 FontDescription 已有值:
         直接使用 CSS 指定的值
    ↓
Font 对象创建,赋予 CSSFontSelector  ← 详见步骤 3A
    ↓
延迟字体匹配 (Paint/Layout 时触发)
```

---

### 步骤 1A️⃣: CSS font-family 值转换为 FontDescription

**当存在 CSS font-family 时的流程** (例: `<body style="font-family: serif">`)

**文件**: `third_party/blink/renderer/core/css/resolver/style_builder_converter.cc:510-590`

```cpp
// ★ 入口点 1: StyleBuilder::ApplyProperty() 调用转换器
StyleBuilderConverter::ConvertFontFamily(
    StyleResolverState& state,
    const CSSValue& value) {
  
  // value 是 CSS 解析器产生的 CSSValue 对象
  // 例: CSSFontFamilyValue("serif") 或 CSSIdentifierValue(serif)
  
  return StyleBuilderConverterBase::ConvertFontFamily(
      value,
      &state.GetFontBuilder(),  // ← FontBuilder 指针
      &state.GetDocument());    // ← Document 指针
}

// ★ 核心转换逻辑
FontDescription::FamilyDescription 
StyleBuilderConverterBase::ConvertFontFamily(
    const CSSValue& value,
    FontBuilder* font_builder,
    const Document* document_for_count) {
  
  FontDescription::FamilyDescription desc(FontDescription::kNoFamily);
  
  // 遍历 font-family 列表 (通常是 font-family: serif, sans-serif, ...)
  for (auto& family : base::Reversed(To<CSSValueList>(value))) {
    
    AtomicString next_family_name;
    FontDescription::GenericFamilyType generic_family = FontDescription::kNoFamily;
    
    // ★ 关键函数: 将单个 CSSValue 转换为通用或字体名
    if (!ConvertFontFamilyName(*family, generic_family, next_family_name,
                               font_builder, document_for_count)) {
      continue;
    }
    
    // 例如: "serif" → generic_family = kSerifFamily
    // 或   "Arial" → generic_family = kNoFamily, next_family_name = "Arial"
    
    // 保存到 FontFamily 链表结构
    if (has_value) {
      next = SharedFontFamily::Create(family_name, family_type, std::move(next));
    }
    
    family_name = next_family_name;
    family_type = is_generic ? FontFamily::Type::kGenericFamily
                             : FontFamily::Type::kFamilyName;
    has_value = true;
  }
  
  // 返回转换后的 FontDescription::FamilyDescription
  desc.family = FontFamily(family_name, family_type, std::move(next));
  return desc;
}

// ★ 单个值转换: ConvertFontFamilyName
static bool ConvertFontFamilyName(
    const CSSValue& value,
    FontDescription::GenericFamilyType& generic_family,
    AtomicString& family_name,
    FontBuilder* font_builder,
    const Document* document_for_count) {
  
  if (auto* font_family_value = DynamicTo<CSSFontFamilyValue>(value)) {
    // 具体字体名: "Arial", "Times New Roman" 等
    generic_family = FontDescription::kNoFamily;
    family_name = font_family_value->Value();
    
  } else if (font_builder) {
    // 通用字体名: "serif", "sans-serif", "monospace" 等
    auto cssValueID = To<CSSIdentifierValue>(value).GetValueID();
    generic_family = ConvertGenericFamily(cssValueID);
    
    if (generic_family != FontDescription::kNoFamily) {
      // 例: kSerifFamily → 向 FontBuilder 请求对应的字体名
      family_name = font_builder->GenericFontFamilyName(generic_family);
    }
  }
  
  return !family_name.IsNull();
}
```

**此时状态**: 
- CSS 的 CSSValue 对象被转换为 `FontDescription::FamilyDescription` 对象
- `FontDescription::FamilyDescription` 包含:
  - `family`: 具体字体名或通用族名
  - `generic_family`: 如果是通用族,记录其类型 (kSerifFamily, kSansSerifFamily 等)

**例子**:
- CSS: `font-family: serif, Arial` 
  → `FamilyDescription { generic_family: kSerifFamily, family: "Serif", next: { family: "Arial" } }`

- CSS: 无 (使用初始值)
  → `FamilyDescription { generic_family: kNoFamily, family: "" }`

---

### 步骤 2️⃣: 样式计算触发 (ResolveStyle)

**文件**: `third_party/blink/renderer/core/css/resolver/font_builder.cc:653-700`

```cpp
void FontBuilder::CreateFont(ComputedStyleBuilder& builder,
                             const ComputedStyle* parent_style) {
  DCHECK(document_);
  
  if (!flags_) {
    return;  // 无改动,直接返回
  }
  
  // 1. 从 builder 取出当前 FontDescription
  FontDescription description = builder.GetFontDescription();
  
  // 2. 更新字体属性(包括家族等)
  if (!UpdateFontDescription(description, builder.ComputeFontOrientation())) {
    flags_ = 0;
    return;
  }
  
  // 3. 更新大小相关属性
  UpdateSpecifiedSize(description, parent_description);
  UpdateComputedSize(description, builder);
  
  // ★ 关键 1: 确定使用哪个 FontSelector (系统字体选择器或当前树作用域)
  FontSelector* font_selector = ComputeFontSelector(builder);
  
  // ★ 关键 2: 使用 FontDescription 和 FontSelector 创建 Font 对象
  builder.SetFont(
      MakeGarbageCollected<Font>(description, font_selector));  // ← Font 的构造
  
  flags_ = 0;
}

// ★ 计算应使用的 FontSelector
FontSelector* FontBuilder::ComputeFontSelector(
    const ComputedStyleBuilder& builder) {
  
  if (IsSet(PropertySetFlag::kFamily)) {
    // font-family 被明确设置过 → 使用该树作用域的 FontSelector
    return FontSelectorFromTreeScope(family_tree_scope_);
  } else {
    // font-family 未设置 → 继承父元素的 FontSelector
    return builder.GetFont()->GetFontSelector();
  }
}

// ★ 从树作用域获取 FontSelector
FontSelector* FontBuilder::FontSelectorFromTreeScope(
    const TreeScope* tree_scope) {
  // 获取文档的主 FontSelector
  // (负责访问 @font-face 声明、系统字体等)
  return document_->GetStyleEngine().GetFontSelector();
}
```

**Font 构造函数**:

**文件**: `third_party/blink/renderer/platform/fonts/font.h`

```cpp
class Font {
 public:
  // 构造函数: 接收 FontDescription 和 FontSelector
  Font(const FontDescription&, FontSelector*);
  
  // ★ 获取 FontSelector
  FontSelector* GetFontSelector() const { return font_selector_; }
  
  // ★ 获取字体数据 (延迟加载)
  const SimpleFontData* PrimaryFont() const;
  
 private:
  scoped_refptr<FontDescription> description_;
  scoped_refptr<FontSelector> font_selector_;  // ← 知道哪些字体可用
  mutable scoped_refptr<FontFallbackList> font_list_;  // ← 延迟初始化
};
```

**此时状态**: 
```
Font 对象被创建,包含:
  ├─ FontDescription: CSS 样式信息(大小、粗细等)
  ├─ CSSFontSelector: 可用字体数据库
  └─ FontFallbackList: 尚未初始化(延迟加载)
```

---

### 步骤 3B️⃣: 延迟字体匹配触发

**关键认知**: Font Matching 不在 Style Resolution 期间进行,而是在需要**度量**或**绘制**时才进行!

```cpp
// 例: 当 Layout 代码需要行高
float line_height = font.LineHeight();  // ← 触发 PrimaryFont() 调用
  ↓
Font::PrimaryFont() {
  if (!font_list_) {
    // ★ 第一次调用 → 初始化 FontFallbackList
    font_list_ = FontFallbackList::Create(
        this, description_, font_selector_);
  }
  return font_list_->PrimaryFont();
}
```

**文件**: `third_party/blink/renderer/platform/fonts/font_fallback_list.cc`

```cpp
scoped_refptr<FontFallbackList> FontFallbackList::Create(
    Font* font,
    const FontDescription& description,
    FontSelector* selector) {
  
  auto result = base::MakeRefCounted<FontFallbackList>();
  result->font_ = font;
  result->font_selector_ = selector;  // ← 保存对 FontSelector 的引用
  result->font_description_ = description;
  
  return result;
}

// ★ 延迟执行: 当第一次需要字体时
const SimpleFontData* FontFallbackList::PrimaryFont() {
  if (!primary_font_) {
    // 触发字体匹配
    primary_font_ = GetFontData(family_list_.PrimaryFamily());
  }
  return primary_font_;
}

// ★ 核心匹配逻辑
const SimpleFontData* FontFallbackList::GetFontData(
    const FontFamily& family) {
  
  // 使用 CSSFontSelector 查询字体数据
  return font_selector_->GetFontData(
      font_description_,
      family.FamilyName(),
      family.IsFamilyName() ? nullptr : family.GenericFamily());
}
```

**CSSFontSelector 的查询逻辑**:

**文件**: `third_party/blink/renderer/core/css/css_font_selector.cc`

```cpp
const SimpleFontData* CSSFontSelector::GetFontData(
    const FontDescription& font_description,
    const AtomicString& family_name,
    FontDescription::GenericFamilyType generic_family) {
  
  // 第一步: 查询 @font-face 缓存
  if (FontFaceCache* cache = GetFontFaceCache()) {
    const SimpleFontData* data = cache->Get(font_description, family_name);
    if (data) {
      return data;  // ← @font-face 字体找到
    }
  }
  
  // 第二步: 查询系统/通用字体
  // 使用 FontCache 和 FontSelector 进行匹配
  return font_cache_->GetFontData(font_description, family_name);
}
```

**此时状态**: 
```
Font Matching 开始
  ↓
查 @font-face 缓存 (FontFaceCache)
  ├─ 找到 → 返回
  └─ 未找到
      ↓
查系统字体 (FontCache)
  ├─ 找到 → 加载 .ttf 文件
  └─ 未找到 → fallback 链
```

---

### 步骤 2️⃣: 样式计算触发 (ResolveStyle)

**文件**: `third_party/blink/renderer/core/css/resolver/style_resolver.cc:1750-1850`

```cpp
const ComputedStyle* StyleResolver::ResolveStyle(
    Element* element,
    const StyleRecalcContext& style_recalc_context,
    const StyleRequest& style_request) {
  
  // 创建状态对象(含字体构建器)
  StyleResolverState state(GetDocument(), *element, 
                          &style_recalc_context, style_request);
  
  // 创建级联处理
  StyleCascade cascade(state);
  
  // ★ 核心:应用所有 CSS 规则,若有 font-family 则在此应用
  ApplyBaseStyle(element, style_recalc_context, style_request, state, cascade);
  
  // ...
  return state.TakeStyle();
}
```

**此时状态**: 
- 若 HTML 有 `<body style="font-family: serif">` → CSS 规则已被 StyleCascade 应用
- 若 HTML 无 font-family CSS → StyleBuilder 中仍为**初始值**

---

### 步骤 3️⃣: CSS 层叠完成后调用 UpdateFont

**文件**: `third_party/blink/renderer/core/css/resolver/style_cascade.cc:307`

```cpp
void StyleCascade::Apply() {
  // 应用 CSS 属性...
  
  // ★ CSS 应用完毕后,立即触发字体创建
  state_.UpdateFont();
}
```

**文件**: `third_party/blink/renderer/core/css/resolver/style_resolver_state.cc:442-443`

```cpp
void StyleResolverState::UpdateFont() {
  // 调用 FontBuilder 创建 Font 对象
  GetFontBuilder().CreateFont(StyleBuilder(), ParentStyle());
}
```

**此时状态**: UpdateFont 被调用 → FontBuilder::CreateFont() 即将执行

---

### 步骤 4️⃣: FontBuilder 检查并应用初始值

**文件**: `third_party/blink/renderer/core/css/resolver/font_builder.cc:653-700`

```cpp
void FontBuilder::CreateFont(ComputedStyleBuilder& builder,
                             const ComputedStyle* parent_style) {
  
  DCHECK(document_);
  
  if (!flags_) {
    return;  // 无改动,直接返回
  }
  
  // 1. 从 builder 取出当前 FontDescription
  const FontDescription& parent_description =
      parent_style ? parent_style->GetFontDescription()
                   : builder.GetFontDescription();
  
  FontDescription description = builder.GetFontDescription();
  
  // 2. 更新字体属性
  if (!UpdateFontDescription(description, builder.ComputeFontOrientation())) {
    flags_ = 0;
    return;
  }
  
  // ★ 关键: 检查通用家族是否为 kNoFamily,若是则使用初始值
  // UpdateFontDescription() 内部逻辑:
  //   if (description.GenericFamily() == FontDescription::kNoFamily 
  //       && description.KeywordSize()) {
  //     // 使用初始值 → kStandardFamily
  //   }
```

**关键函数**: `InitialGenericFamily()` 返回默认值

**文件**: `third_party/blink/renderer/core/css/resolver/font_builder.h:138`

```cpp
static FontDescription::GenericFamilyType InitialGenericFamily() {
  return FontDescription::kStandardFamily;  // ← 总是 standard/sans-serif 路径
}
```

**此时状态**: 
```
FontDescription::generic_family_ = kStandardFamily
  (若原本为 kNoFamily,则被覆盖)
```

---

### 步骤 5️⃣: 将通用家族映射为具体字体名

**文件**: `third_party/blink/renderer/platform/fonts/font_selector.cc:28-95`

```cpp
AtomicString FontSelector::FamilyNameFromSettings(
    const FontDescription& font_description,
    const FontFamily& generic_family,
    const LayoutLocale* content_locale) {
  
  // generic_family.familyName = "standard" (来自 kStandardFamily)
  UScriptCode script = font_description.GetScript();
  
  // 1. 先查预设映射表
  if (generic_family_settings_) {
    AtomicString mapped_name = 
      generic_family_settings_->Standard(script);  // <- 按脚本查表
    
    if (!mapped_name.empty()) {
      return mapped_name;  // 例: Linux 返回 "DejaVu Sans"
    }
  }
  
  // 2. 表为空 -> 触发系统查询
  #if BUILDFLAG(IS_ANDROID)
    return FontCache::GetGenericFamilyNameForScript(
      "standard", font_description, content_locale);
  #endif
}
```

**平台对比**:
- **Linux**: `generic_family_settings_->Standard()` 返回 `"DejaVu Sans"` (预设值)
- **Android**: `generic_family_settings_->Standard()` 返回空字符串 → 进入 `FontCache::GetGenericFamilyNameForScript()`

---

### 步骤 6️⃣: Android 查询 fonts.xml

**文件**: `third_party/blink/renderer/platform/fonts/android/font_cache_android.cc`

```cpp
// 当 GenericFontFamilySettings 为空时调用
AtomicString FontCache::GetGenericFamilyNameForScript(
    const AtomicString& family_name,           // "standard"
    const FontDescription& font_description,
    const LayoutLocale* content_locale) {
  
  // 调用 Skia 的 Android 字体管理器
  // 它在底层解析 /system/etc/fonts.xml
  // 
  // fonts.xml 中的映射:
  // <alias name="standard" to="sans-serif" />
  // <family name="sans-serif">
  //   <font weight="400" style="normal">Roboto-Regular.ttf</font>
  //   <font weight="700" style="normal">Roboto-Bold.ttf</font>
  // </family>
  
  // 返回: "Roboto"
  return AtomicString("Roboto");
}
```

**此时状态**: 
```
"standard" (通用家族名)
  ↓
"Roboto" (具体字体名)
```

---

### 步骤 7️⃣: FontCache 加载字体文件

**文件**: `third_party/blink/renderer/platform/fonts/font_cache.cc`

```cpp
const SimpleFontData* FontCache::GetFontData(
    const FontDescription& font_description,
    const AtomicString& family_name) {  // family_name = "Roboto"
  
  // 1. 查询 Roboto 字体文件
  FontPlatformData* platform_data = GetFontPlatformData(
    font_description, 
    FontFaceCreationParams(family_name));
  
  if (!platform_data) {
    return nullptr;  // 找不到 -> 走 fallback
  }
  
  // 2. 加载字体文件(通常 /system/fonts/Roboto-*.ttf)
  // 创建 SimpleFontData 对象(含字形指针)
  return FontDataFromFontPlatformData(platform_data, kDoNotRetain);
}
```

**底层**: Skia (`third_party/skia/src/ports/SkFontMgr_android.cpp`)

```cpp
sk_sp<SkTypeface> SkFontMgr_Android::matchFamilyStyle(
    const char family[],  // "Roboto"
    const SkFontStyle& style) {
  
  // 1. 查询 fonts.xml 中 Roboto 条目
  // 2. 加载对应 .ttf 文件到内存
  // 3. 返回 SkTypeface 对象(含字形数据指针)
  
  // 若 Roboto 不可用 -> 走 fallback 链
  return result_typeface;
}
```

**此时状态**: 
```
Roboto .ttf 文件已加载
SimpleFontData 包含字形数据指针
```

---

### 步骤 8️⃣: HarfBuzz 处理文本

**文件**: `third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc`

```cpp
ShapeResult* HarfBuzzShaper::Shape(const TextRun& run) {
  // run.text = "无样式的文字"
  // run.font = Font 对象(内含 SimpleFontData -> Roboto)
  
  // 1. 从 FontData 获取 HarfBuzz 字体对象
  hb_font_t* hb_font = font_data->GetHarfBuzzFace()->GetScaledFont();
  
  // 2. 使用 HarfBuzz 进行文本 shaping
  hb_buffer_t* buffer = hb_buffer_create();
  hb_buffer_add_utf16(buffer, text, length, 0, length);
  hb_shape(hb_font, buffer, nullptr, 0);
  
  // 3. 获取字形信息(字形ID、x y 位移)
  unsigned glyph_count;
  hb_glyph_info_t* infos = hb_buffer_get_glyph_infos(buffer, &glyph_count);
  hb_glyph_position_t* positions = hb_buffer_get_glyph_positions(buffer, &glyph_count);
  
  // 4. 返回 ShapeResult
  return CreateShapeResult(infos, positions, glyph_count);
}
```

**此时状态**: 
```
每个字符已映射到 Roboto 字体中的字形
位置信息已计算(kerning、ligature 等)
```

---

### 步骤 9️⃣: Skia 光栅化

**文件**: `third_party/blink/renderer/platform/graphics/graphics_context.cc`

```cpp
void GraphicsContext::DrawText(
    const Font& font,
    const TextRunPaintInfo& run_info,
    const gfx::PointF& point) {
  
  const ShapeResult* shape_result = run_info.shape_result;
  
  // 遍历每个字形
  shape_result->ForEachGlyph([&](const Glyph& glyph, const gfx::PointF& position) {
    // 调用 Skia 的 drawGlyphs
    canvas_->drawGlyphs(
      1,                      // 字形数量
      &glyph.glyph_id,       // Roboto 中的字形 ID
      &position,             // 位置
      point,                 // 基线
      font.GetSkFont(),      // Roboto SkFont 对象
      paint);                // 绘制样式
  });
}
```

**最终**: 像素被绘制到屏幕 → 用户看到 Roboto 字体渲染的文字。

---

## 完整状态演变表

| 阶段 | FontDescription.generic_family | 具体字体 | 说明 |
|------|------|------|------|
| **初始** | `kNoFamily` | - | 构造函数默认 |
| **样式计算** | `kNoFamily`→`kStandardFamily` | - | 无CSS时使用初始值 |
| **映射阶段** | `kStandardFamily` | `"Roboto"` | 从 fonts.xml 查询 |
| **加载阶段** | `kStandardFamily` | `SimpleFontData` | 加载 .ttf 文件 |
| **Shaping** | - | `ShapeResult(字形ID)` | HarfBuzz 处理 |
| **绘制** | - | 像素 | Skia 光栅化 |

---

## FontFaceCache vs FontCache

### 无 CSS 时的角色

| 缓存类型 | 适用场景 | 无CSS参与? |
|---------|---------|--------|
| **FontFaceCache** | `@font-face` 声明时 | ❌ 否 |
| **FontCache** | 所有字体加载 | ✅ 是 |

**解释**:
- `FontFaceCache` (`core/css/font_face_cache.cc`) 仅用于缓存 `@font-face` 声明的字体
- 无 CSS 场景 → 没有 `@font-face` 声明 → `FontFaceCache` 不参与
- `FontCache` (`platform/fonts/font_cache.cc`) 用于加载系统字体、Web字体等所有字体
- 无 CSS 场景 → 必须经过 `FontCache::GetFontData()` 加载 Roboto

---

## 关键认知

### 1. kNoFamily 不是最终值

```cpp
FontDescription() {
  generic_family_ = kNoFamily;  // ← 裸初值
}
```

这是构造时的初值，**不**是最终进入选择器的值。

---

### 2. InitialGenericFamily() 是关键

```cpp
static FontDescription::GenericFamilyType InitialGenericFamily() {
  return FontDescription::kStandardFamily;  // ← 无CSS时使用这个
}
```

当 CSS 无 `font-family` 时，`FontBuilder` 会用这个初始值覆盖 `kNoFamily`。

---

### 3. 初始值 → 系统查询

```
kStandardFamily (通用)
  ↓
GenericFontFamilySettings::Standard() (查预设表)
  ├─ Linux: "DejaVu Sans"
  └─ Android: "" (空) → 触发系统查询
      ↓
FontCache::GetGenericFamilyNameForScript()
  ↓
SkFontMgr 解析 fonts.xml
  ↓
"Roboto" (具体字体名)
```

---

### 4. Android fonts.xml 别名

```xml
<alias name="standard" to="sans-serif" />
```

"standard" 通用族被别名为 "sans-serif"，再查询 "sans-serif" 对应的字体。

---

## 调试位置

| 调试点 | 文件 | 函数 | 用途 |
|------|------|------|------|
| 样式计算入口 | `style_resolver.cc` | `StyleResolver::ResolveStyle()` | 看元素是否进入样式计算 |
| 字体初值应用 | `font_builder.cc` | `FontBuilder::CreateFont()` | 看初始值是否被应用 |
| 映射表查询 | `font_selector.cc` | `FamilyNameFromSettings()` | 看映射表是否为空 |
| 系统查询 | `font_cache_android.cc` | `GetGenericFamilyNameForScript()` | 看是否触发 fonts.xml 查询 |
| 字体加载 | `font_cache.cc` | `GetFontData()` | 看字体是否成功加载 |
| HarfBuzz Shaping | `harfbuzz_shaper.cc` | `HarfBuzzShaper::Shape()` | 看字形是否正确映射 |

---

## 最短总结

```
无 CSS → kNoFamily → 使用初始值 kStandardFamily
→ 查 GenericFontFamilySettings (Android 空)
→ 查 fonts.xml → "Roboto"
→ 加载 Roboto.ttf
→ HarfBuzz shaping
→ Skia 光栅化
→ 屏幕显示
```
