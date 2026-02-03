# Chromium 字体系统数据结构完整继承图

## 📊 1. 完整类继承关系

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ComputedStyle (最终计算的样式)                        │
├─────────────────────────────────────────────────────────────────────────────┤
│ private:                                                                      │
│   FontDescription font_description_;    ← 字体配置                         │
│   Member<Font> font_;                   ← 当前使用的 Font 对象             │
│   ... 200+ 其他 CSS 属性 ...                                               │
└─────────────────────────────────────────────────────────────────────────────┘
         ▲
         │ 包含
         │
         ├──────────────┬───────────────────────────────────────┐
         │              │                                       │
         ▼              ▼                                       ▼
    ┌──────────────┐ ┌──────────────┐              ┌──────────────────┐
    │ FontDesc.    │ │ Font         │              │ 其他 CSS 属性    │
    │ (字体配置)   │ │ (字体对象)   │              │ (color, size...) │
    └──────────────┘ └──────────────┘              └──────────────────┘
         │ 包含           │ 包含
         │                │
         ├─→ FontFamily* ─┤  ┌─ FontFallbackList*
         │   (链表)       │  │   (多字体管理)
         │                │  │
         ├─→ GenericType  │  ├─→ Vector<FontData*>
         │ (kNoFamily,..) │  │   ├─ SimpleFontData ("Georgia")
         │                │  │   ├─ SimpleFontData ("Serif系统")
         ├─→ float size   │  │   └─ SimpleFontData ("系统默认")
         │                │  │
         ├─→ int weight   │  └─→ HashMap<char, SimpleFontData*>
         │                │      (字符缓存)
         └─→ FontStyle    │
```

---

## 🔗 2. FontFamily 链表详解

```
CSS: font-family: Georgia, serif, monospace
                  ↓ [CSS 解析]
        CSSValueList([...])
                  ↓ [StyleBuilderConverter::ConvertFontFamily()]
        FontDescription::family 链表

┌─────────────────────────────────────────────────────────────────┐
│ FontFamily (链表节点 1)                                          │
├─────────────────────────────────────────────────────────────────┤
│ family_name: "Georgia"                                          │
│ type: kFamilyName               (← 不是通用族)                 │
│ next: ─┐                                                        │
└────────┼────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ FontFamily (链表节点 2)                                          │
├─────────────────────────────────────────────────────────────────┤
│ family_name: "Serif系统字体"  (由 GenericFontFamilySettings 提供)│
│ type: kGenericFamily            (← 这是通用族映射)             │
│ next: ─┐                                                        │
└────────┼────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ FontFamily (链表节点 3)                                          │
├─────────────────────────────────────────────────────────────────┤
│ family_name: "Monospace系统字体"                                │
│ type: kGenericFamily            (← 这也是通用族映射)            │
│ next: nullptr                   (← 链表结束)                    │
└─────────────────────────────────────────────────────────────────┘

字体查询顺序:
  1. "Georgia"         → 尝试加载
  2. "Serif系统字体"   → 如果 1 失败,尝试加载
  3. "Monospace系统"   → 如果 2 失败,尝试加载
  4. 系统 fallback     → 如果 3 失败,系统默认字体
```

---

## 🎲 3. FontData 多态关系

```
┌──────────────────────────────────────────────────────────────────────┐
│                    FontData (抽象基类)                               │
│                   GarbageCollected<FontData>                         │
├──────────────────────────────────────────────────────────────────────┤
│ virtual const SimpleFontData* FontDataForCharacter(UChar32) = 0;    │
│ virtual bool IsCustomFont() const = 0;                              │
│ virtual bool IsLoading() const = 0;                                 │
│ virtual bool IsLoadingFallback() const = 0;                         │
│ virtual bool IsSegmented() const = 0;                               │
│ virtual bool ShouldSkipDrawing() const = 0;                         │
└──────────────────────────────────────────────────────────────────────┘
        ▲                                ▲
        │ 继承                           │ 继承
        │                                │
        ├────┬────────────────────────────┴──┐
        │    │                               │
        ▼    ▼                               ▼
    ┌───────────────────┐    ┌────────────────────────────────┐
    │ SimpleFontData    │    │ SegmentedFontData              │
    │ (单个字体)        │    │ (多字体集合)                   │
    ├───────────────────┤    ├────────────────────────────────┤
    │ final class       │    │ final class                    │
    │                   │    │                                │
    │ 成员:             │    │ 成员:                          │
    │ • platform_data_  │    │ • Vector<SimpleFontData*>      │
    │   (SkTypeface)    │    │   fonts_                       │
    │ • font_metrics_   │    │ • SimpleFontData*              │
    │   (高度、宽度等)  │    │   pages_[256]                  │
    │ • space_width_    │    │   (字符范围索引)               │
    │ • zero_glyph_     │    │                                │
    │                   │    │ 实现:                          │
    │ 虚函数实现:       │    │ FontDataForCharacter() {       │
    │ FontDataForChar() │    │   // 使用字符代码查询          │
    │   → return this   │    │   // pages_[c >> 8]            │
    │                   │    │ }                              │
    └───────────────────┘    └────────────────────────────────┘
            ▲
            │ 使用
            │
    ┌───────────────────────────────────────────┐
    │ 存储在 FontFallbackList::font_list_       │
    │ [SimpleFontData*, SimpleFontData*, ...]   │
    └───────────────────────────────────────────┘
```

---

## 📦 4. Font 对象内部结构

```
┌────────────────────────────────────────────────────────────────┐
│                    Font (运行时字体对象)                        │
├────────────────────────────────────────────────────────────────┤
│ private:                                                        │
│                                                                │
│  FontDescription font_description_                             │
│  ├─ family: "Georgia" → "serif" → nullptr (FontFamily 链表)   │
│  ├─ generic_family: kNoFamily                                  │
│  ├─ size: 16.0                                                │
│  ├─ weight: 400                                               │
│  └─ style: normal                                              │
│                                                                │
│  Member<FontData> font_data_ ← 当前使用的字体                  │
│                   ↓ 通常指向 SimpleFontData                     │
│                   └─ SimpleFontData* platform_data_            │
│                      ├─ SkTypeface*  ← Skia 字体对象          │
│                      └─ FontMetrics                            │
│                                                                │
│  Member<FontFallbackList> fallback_list_                       │
│  └─ 详见下面 ▼                                                 │
│                                                                │
│ public:                                                        │
│  const SimpleFontData* PrimaryFont() const;                    │
│  const SimpleFontData* FontDataForCharacter(UChar32) const;    │
│  float Width(const TextRun& run) const;                        │
│  gfx::RectF BoundingBox(const TextRun& run) const;            │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## 🔀 5. FontFallbackList 详细结构

```
┌──────────────────────────────────────────────────────────────────┐
│              FontFallbackList (Fallback 链管理)                  │
│          : public RefCounted<FontFallbackList>                   │
├──────────────────────────────────────────────────────────────────┤
│ private:                                                          │
│                                                                  │
│  Vector<Member<FontData>> font_list_  ← Fallback 链表           │
│  ┌────────────┐    ┌─────────────┐    ┌──────────────┐         │
│  │[0]         │───▶│[1]          │───▶│[2]           │         │
│  │SimpleFD    │    │SimpleFD     │    │SimpleFD      │         │
│  │("Georgia") │    │("DejaVu     │    │("系统        │         │
│  │            │    │Serif")      │    │默认")        │         │
│  └────────────┘    └─────────────┘    └──────────────┘         │
│                                                                  │
│  HashMap<UChar32, SimpleFontData*> font_character_map_          │
│  ┌──────────────────────────────────────────────────────┐       │
│  │ 'A' (U+0041) → SimpleFontData*("Georgia")           │       │
│  │ 'B' (U+0042) → SimpleFontData*("Georgia")           │       │
│  │ '中' (U+4E2D) → SimpleFontData*("Noto Sans CJK")    │       │
│  │ ...                                                  │       │
│  └──────────────────────────────────────────────────────┘       │
│  (性能优化: 缓存字符查询结果)                                   │
│                                                                  │
│ public:                                                          │
│  const SimpleFontData* GetFontData(                              │
│      const FontDescription& font_description);                  │
│  const SimpleFontData* FontDataForCharacter(UChar32 c) const {  │
│    // 1. 查询缓存                                               │
│    auto it = font_character_map_.find(c);                       │
│    if (it != font_character_map_.end()) {                       │
│      return it->value;  // ← 快!                               │
│    }                                                            │
│                                                                  │
│    // 2. 遍历 fallback 链                                        │
│    for (const auto& font : font_list_) {                        │
│      if (font->FontDataForCharacter(c)) {                       │
│        return cache_and_return(font);                           │
│      }                                                          │
│    }                                                            │
│                                                                  │
│    // 3. 系统 fallback                                           │
│    return RelationFallbackFont();                               │
│  }                                                              │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🏪 6. FontCache 全局缓存体系

```
┌──────────────────────────────────────────────────────────────────┐
│              FontCache (全局单例缓存)                             │
│ static FontCache* GetFontCache() { ... }                         │
├──────────────────────────────────────────────────────────────────┤
│ private:                                                          │
│                                                                  │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ font_data_cache_: HashMap<FontCacheKey, SimpleFontData*>  │ │
│ ├─────────────────────────────────────────────────────────────┤ │
│ │ Key 结构:                                                   │ │
│ │  {                                                          │ │
│ │    FontDescription desc,   ← 字体大小、粗细、风格等       │ │
│ │    AtomicString family_name  ← "Georgia", "Roboto", ...  │ │
│ │  }                                                          │ │
│ │                                                             │ │
│ │ Value: SimpleFontData*                                     │ │
│ │  ├─ platform_data_ → SkTypeface*                          │ │
│ │  ├─ font_metrics_                                         │ │
│ │  └─ space_width_ 等                                       │ │
│ │                                                             │ │
│ │ 应用生存期缓存 (只增不减)                                 │ │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ typeface_cache_: HashMap<FontPlatformDataCacheKey,       │ │
│ │                            FontPlatformData>              │ │
│ ├─────────────────────────────────────────────────────────────┤ │
│ │ Key: {FontDescription, FontFaceCreationParams}            │ │
│ │                                                             │ │
│ │ Value: FontPlatformData                                   │ │
│ │  ├─ SkTypeface* typeface  ← Skia 层字体对象             │ │
│ │  ├─ float text_size                                       │ │
│ │  └─ ... 平台相关字段 ...                                 │ │
│ │                                                             │ │
│ │ 应用生存期缓存                                             │ │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ 其他缓存:                                                   │ │
│ │ • fallback_font_cache_: 系统 fallback 字体                 │ │
│ │ • system_font_cache_: 系统字体查询缓存                     │ │
│ │ • SkFontMgr 实例 (单例)                                   │ │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ public:                                                          │
│  const SimpleFontData* GetFontData(                              │
│      const FontDescription& desc,                               │
│      const AtomicString& family_name);                          │
│                                                                  │
│  void InvalidateAllFontData();  ← 手动清除缓存                   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🔄 7. 字体加载流程 - 完整交互图

```
时间 ─────────────────────────────────────────────────────────────────→

T0: CSS 解析阶段
    ┌─────────────────────────────────┐
    │ CSSParser                       │
    └─────────────────────────────────┘
            │ input: "font-family: Georgia, serif"
            ▼
    ┌─────────────────────────────────┐
    │ CSSValueList                    │
    │ [CSSFontFamilyValue("Georgia")  │
    │  CSSIdentifierValue(serif)]     │
    └─────────────────────────────────┘

T1: 样式计算阶段 (毫秒级)
    ┌─────────────────────────────────┐
    │ StyleCascade::Apply()           │
    │  → StyleBuilder::ApplyProperty()│
    │  → StyleBuilderConverter::      │
    │     ConvertFontFamily()         │
    └─────────────────────────────────┘
            │
            ▼
    ┌─────────────────────────────────┐
    │ FontDescription (创建)          │
    │ family: "Georgia" → "Serif系统" │
    │ size: 16.0, weight: 400         │
    └─────────────────────────────────┘
            │
            ▼
    ┌─────────────────────────────────┐
    │ StyleResolverState::UpdateFont()│
    │  → FontBuilder::CreateFont()    │
    │  → new Font(FontDescription)    │
    └─────────────────────────────────┘
            │
            ▼
    ┌─────────────────────────────────┐
    │ FontFallbackList (初始化)       │
    │ [待加载]                        │
    └─────────────────────────────────┘
    
T2: 文本使用阶段 (第一个字符时触发)
    ┌──────────────────────────────────┐
    │ TextLayout::ShapeText("Hello")   │
    │  → 字符 'H' 首次使用             │
    └──────────────────────────────────┘
            │
            ▼
    ┌──────────────────────────────────┐
    │ Font::FontDataForCharacter('H')  │
    │  → FontFallbackList::GetFontData │
    └──────────────────────────────────┘
            │
            ▼
    ┌──────────────────────────────────┐
    │ FontCache 查询                   │
    │ [缓存未命中]                     │
    └──────────────────────────────────┘
            │
            ▼
    ┌──────────────────────────────────┐
    │ SkFontMgr::matchFamilyStyle()    │
    │ (系统调用 - 可能涉及磁盘 IO)    │
    └──────────────────────────────────┘
            │
            ▼
    ┌──────────────────────────────────┐
    │ SkTypeface* 返回                 │
    │ 创建 SimpleFontData              │
    └──────────────────────────────────┘
            │
            ▼
    ┌──────────────────────────────────┐
    │ FontCache 存入缓存               │
    │ font_data_cache_[key] = ...      │
    │ typeface_cache_[key] = ...       │
    └──────────────────────────────────┘

T3: 后续字符使用 (快速路径)
    ┌──────────────────────────────────┐
    │ Font::FontDataForCharacter('e')  │
    │  → FontFallbackList 缓存命中 ✓   │
    │  → 直接返回 SimpleFontData*      │
    │  [纳秒级 - 无磁盘 IO]           │
    └──────────────────────────────────┘

T4: 屏幕绘制
    ┌──────────────────────────────────┐
    │ HarfBuzzShaper::Shape()          │
    │  → hb_shape(hb_font, buffer)     │
    │  → ShapeResult (字形信息)        │
    └──────────────────────────────────┘
            │
            ▼
    ┌──────────────────────────────────┐
    │ GraphicsContext::DrawText()      │
    │  → SkCanvas::drawGlyphs()        │
    │  → 屏幕像素                      │
    └──────────────────────────────────┘
```

---

## 🎯 8. GenericFontFamilySettings 映射体系

```
┌──────────────────────────────────────────────────────────────────┐
│            GenericFontFamilySettings (通用族映射)                │
│ (在 RenderingContext 中创建, 全局访问)                           │
├──────────────────────────────────────────────────────────────────┤
│ private:                                                          │
│                                                                  │
│  HashMap<UScriptCode, AtomicString> standard_font_family_map_  │
│  HashMap<UScriptCode, AtomicString> serif_font_family_map_     │
│  HashMap<UScriptCode, AtomicString> sans_serif_font_family_map_│
│  HashMap<UScriptCode, AtomicString> monospace_font_family_map_ │
│  HashMap<UScriptCode, AtomicString> cursive_font_family_map_   │
│  HashMap<UScriptCode, AtomicString> fantasy_font_family_map_   │
│  HashMap<UScriptCode, AtomicString> system_ui_font_family_map_ │
│                                                                  │
│  // 示例: Desktop Linux                                          │
│  standard_font_family_map_[USCRIPT_LATIN] = "DejaVu Serif"    │
│  sans_serif_font_family_map_[USCRIPT_LATIN] = "DejaVu Sans"   │
│  monospace_font_family_map_[USCRIPT_LATIN] = "DejaVu Mono"    │
│                                                                  │
│  // 示例: Android                                               │
│  // (通常为空 - 由 FontCache::GetGenericFamilyNameForScript() │
│  //  从 fonts.xml 动态获取)                                    │
│                                                                  │
│ public:                                                          │
│  const AtomicString& Standard(UScriptCode script) const {       │
│    return standard_font_family_map_[script];                    │
│  }                                                              │
│  // 其他通用族的类似访问器...                                   │
│                                                                  │
│  void SetStandardFontFamily(const AtomicString& family,         │
│                             UScriptCode script) {               │
│    standard_font_family_map_[script] = family;                 │
│  }                                                              │
│  // 其他通用族的类似设置器...                                   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

查询流程:

CSS: font-family: serif
      ↓ [ConvertFontFamily()]
generic_family = kSerifFamily
      ↓ [FontSelector::FamilyNameFromSettings()]
query: GenericFontFamilySettings::Serif(USCRIPT_LATIN)
      ├─ [Desktop] → "DejaVu Serif"
      ├─ [Android] → "" (空,触发 FontCache::GetGenericFamilyNameForScript)
      │                  ↓ [Android fonts.xml]
      │                  → "Noto Serif"
      └─ [Windows] → "Times New Roman"
```

---

## 📝 9. CSS 值到内部表示的映射

```
┌────────────────────────────────────────────────────────────────┐
│                    CSS 层 (字符串)                              │
├────────────────────────────────────────────────────────────────┤
│ font-family: Georgia, serif, sans-serif                        │
│ font-size: 16px                                                │
│ font-weight: 700                                               │
│ font-style: italic                                             │
│ line-height: 1.5                                               │
└────────────────────────────────────────────────────────────────┘
         │ [CSS Parser]
         ▼
┌────────────────────────────────────────────────────────────────┐
│                    CSSValue 层 (AST)                           │
├────────────────────────────────────────────────────────────────┤
│ font-family: CSSValueList [                                    │
│               CSSFontFamilyValue("Georgia"),                   │
│               CSSIdentifierValue(serif),                       │
│               CSSIdentifierValue(sans-serif)                   │
│             ]                                                  │
│ font-size: CSSPrimitiveValue(16, UnitType::kPixels)           │
│ font-weight: CSSIdentifierValue(bold)  ← 或 CSSNumeric(700) │
│ font-style: CSSIdentifierValue(italic)                        │
│ line-height: CSSPrimitiveValue(1.5, UnitType::kNumber)        │
└────────────────────────────────────────────────────────────────┘
         │ [StyleBuilder / StyleBuilderConverter]
         ▼
┌────────────────────────────────────────────────────────────────┐
│                    ComputedStyle 层 (内部)                     │
├────────────────────────────────────────────────────────────────┤
│ FontDescription {                                              │
│   family: FontFamily {                                         │
│     family_name: "Georgia",                                    │
│     type: kFamilyName,                                         │
│     next: → FontFamily {                                       │
│       family_name: "Serif系统字体",                           │
│       type: kGenericFamily,                                    │
│       next: → FontFamily {                                     │
│         family_name: "SansSerif系统字体",                      │
│         type: kGenericFamily,                                  │
│         next: nullptr                                          │
│       }                                                        │
│     }                                                          │
│   },                                                           │
│   size: 16.0f,                                                │
│   weight: 700,                                                │
│   style: kItalicStyle,                                        │
│   generic_family: kNoFamily,   ← "Georgia" 不是通用族!        │
│   line_height: 24.0f   ← 16 * 1.5                            │
│ }                                                              │
└────────────────────────────────────────────────────────────────┘
         │ [FontBuilder::CreateFont()]
         ▼
┌────────────────────────────────────────────────────────────────┐
│                    Font 对象 (运行时)                          │
├────────────────────────────────────────────────────────────────┤
│ Font {                                                         │
│   font_description_: FontDescription (如上)                   │
│   font_data_: SimpleFontData*                                 │
│              (第一次字符使用时由 FontCache 创建)               │
│   fallback_list_: FontFallbackList* {                          │
│     font_list_: [SimpleFD("Georgia"), SimpleFD("Serif"), ...]│
│     font_character_map_: HashMap<char, SimpleFD*>            │
│   }                                                           │
│ }                                                              │
└────────────────────────────────────────────────────────────────┘
         │ [HarfBuzzShaper]
         ▼
┌────────────────────────────────────────────────────────────────┐
│                    ShapeResult (字形数据)                      │
├────────────────────────────────────────────────────────────────┤
│ ShapeResult {                                                  │
│   glyph_infos: [                                               │
│     { glyph_id: 68, cluster: 0 },   ← 'G'                    │
│     { glyph_id: 69, cluster: 1 },   ← 'e'                    │
│     { glyph_id: 76, cluster: 2 },   ← 'o'                    │
│     { glyph_id: 76, cluster: 3 },   ← 'r'                    │
│     { glyph_id: 74, cluster: 4 },   ← 'g'                    │
│     { glyph_id: 75, cluster: 5 },   ← 'i'                    │
│     { glyph_id: 86, cluster: 6 },   ← 'a'                    │
│   ],                                                          │
│   positions: [                                                │
│     { advance: 600, offset_x: 0, offset_y: 0 },              │
│     { advance: 580, offset_x: 0, offset_y: 0 },              │
│     ...                                                       │
│   ]                                                           │
│ }                                                              │
└────────────────────────────────────────────────────────────────┘
         │ [SkCanvas::drawGlyphs()]
         ▼
┌────────────────────────────────────────────────────────────────┐
│                    屏幕像素                                    │
└────────────────────────────────────────────────────────────────┘
```

---

## 🔍 10. Android 特殊处理流程

```
┌────────────────────────────────────────────────────────┐
│        Android Chromium 字体查询特殊流程                │
└────────────────────────────────────────────────────────┘

Case 1: 具体字体名 (如 "Georgia")
  ┌─ SkFontMgr::matchFamilyStyle("Georgia", style)
  │   ├─ 查询系统字体映射
  │   ├─ 不存在 → fallback
  │   └─ 返回接近的字体或默认
  └─ 

Case 2: 通用族 (如 "serif")
  ┌─ GenericFontFamilySettings::Serif(script)
  │   ├─ Android 上通常为空 (no font preferences)
  │   └─ 触发 FontCache::GetGenericFamilyNameForScript()
  │
  └─ FontCache::GetGenericFamilyNameForScript("serif", script)
     ┌─ 读取 /system/etc/fonts.xml 配置
     │
     ├─ 查找 <family name="serif">...</family>
     │
     ├─ 示例 fonts.xml:
     │  <familyset>
     │    <family name="serif">
     │      <font weight="400">NotoSerif-Regular.ttf</font>
     │      <font weight="700">NotoSerif-Bold.ttf</font>
     │    </family>
     │  </familyset>
     │
     └─ 返回: "NotoSerif-Regular.ttf" (for weight=400)
        或  "NotoSerif-Bold.ttf" (for weight=700)
        
             ↓
             
     SkFontMgr::matchFamilyStyle("NotoSerif-Regular", style)
     ├─ 加载 /system/fonts/NotoSerif-Regular.ttf
     └─ 返回 SkTypeface*

Case 3: 无字体指定 (generic_family = kNoFamily)
  ┌─ 使用 UA 默认值 (通常是 sans-serif)
  │  (具体取决于 Android 版本和 OEM 定制)
  │
  └─ → Case 2 (通用族处理)
```

---

## 📊 11. 内存生命周期

```
ComputedStyle 创建
  ↓ (on element recompute)
  ├─ new ComputedStyle()
  │  ├─ FontDescription 初始化 (小,栈上)
  │  └─ font_ = nullptr (还未创建)
  │
  └─ StyleResolverState::UpdateFont()
     ├─ new Font(FontDescription)
     │  ├─ 小对象 (~1KB) - 栈上分配
     │  └─ FontFallbackList::Create()
     │     ├─ new FontFallbackList()
     │     ├─ font_list_ = Vector (堆上,初始为空)
     │     └─ font_character_map_ = HashMap (堆上,初始为空)
     │
     └─ font_data_cache_[key] = SimpleFontData*
        ├─ SimpleFontData 创建 (第一次字符使用时)
        │  ├─ ~300-500 bytes (取决于平台)
        │  └─ platform_data_ → FontPlatformData*
        │     └─ SkTypeface* (系统级,大~MB)
        │
        └─ FontCache 缓存 (全局,应用生存期)

ComputedStyle 销毁 (element removed)
  ├─ font_ 销毁
  │  ├─ FontFallbackList 销毁
  │  │  ├─ font_list_ 销毁 (SimpleFontData* 列表)
  │  │  ├─ font_character_map_ 销毁 (HashMap 清空)
  │  │  └─ SimpleFontData 可能被销毁 (如果引用计数为 0)
  │  │     └─ FontPlatformData 销毁 (SkTypeface 释放)
  │  │
  │  └─ FontDescription 销毁
  │
  └─ FontCache 中的引用仍存活 (被其他元素共享)
     └─ 只有应用关闭或手动 InvalidateAllFontData() 才释放

跨文档复用:
  Document 1 Element        Document 2 Element
        ↓                          ↓
   ComputedStyle            ComputedStyle
        ↓                          ↓
      Font → FontFallbackList → SimpleFontData("Georgia")
                                      ↓
                            FontCache 全局缓存 ← 共享!
```

---

## 🎯 12. 关键对象创建时机总结

| 对象 | 创建时机 | 生命周期 | 备注 |
|-----|--------|--------|------|
| **FontDescription** | StyleResolver 计算样式时 | 绑定 ComputedStyle | 仅包含配置,不涉及磁盘 IO |
| **Font** | UpdateFont() 时 (样式计算末) | 绑定 ComputedStyle | 小对象,不加载字体 |
| **FontFallbackList** | Font 创建时 | 绑定 Font | 初始为空 |
| **SimpleFontData** | 第一个字符使用时 | 跨 ComputedStyle 共享 | ⚠️ 可能触发磁盘 IO |
| **SkTypeface** | 第一次字体加载时 | 应用生存期 | 系统资源,大对象 |
| **ShapeResult** | TextLayout 时 | 临时,使用后释放 | 每段文本创建 |

---

## 参考:代码路径

```
字体系统核心文件 (third_party/blink/renderer/platform/fonts/):
├── font.h / font.cc                    ← Font 类
├── font_data.h / font_data.cc          ← FontData 抽象基类
├── simple_font_data.h / .cc            ← SimpleFontData 实现
├── segmented_font_data.h / .cc         ← SegmentedFontData 实现
├── font_family.h / font_family.cc      ← FontFamily 链表
├── font_description.h / .cc            ← FontDescription 配置
├── font_fallback_list.h / .cc          ← FontFallbackList 管理
├── font_cache.h / font_cache.cc        ← 全局缓存
├── font_selector.h / .cc               ← 字体选择和映射
├── generic_font_family_settings.h      ← 通用族映射
├── android/
│   └── font_cache_android.cc           ← Android 特殊处理
├── skia/
│   └── font_cache_skia.cc              ← Skia 集成
└── shaping/
    └── harfbuzz_shaper.cc              ← HarfBuzz Shaping

样式计算相关 (third_party/blink/renderer/core/css/):
├── resolver/style_resolver.cc          ← 样式计算主函数
├── resolver/style_cascade.cc           ← CSS 应用
├── resolver/font_builder.cc            ← Font 创建
└── resolver/style_builder_converter.cc ← CSS 值转换 (关键!)
```

This comprehensive guide covers the complete data structure inheritance hierarchy, from CSS parsing through rendering to screen pixels, with specific focus on Android Chromium platform.
