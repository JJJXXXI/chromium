# Chromium 字体渲染系统 - 深度架构分析

## 📊 执行总结

本文档对 Chromium/Blink 字体渲染系统进行深度分析，涵盖从 CSS 解析、字体选择、到 HarfBuzz shaping 的完整调用链。包含 15+ 关键文件、10+ 关键类/函数、完整的流程图和平台特异性分析。

---

## 1️⃣ 关键文件列表（按架构层次）

### 1.1 字体选择和缓存层（Platform Layer）

| # | 文件路径 | 功能 | 关键类 |
|---|---------|------|--------|
| 1 | [third_party/blink/renderer/platform/fonts/font_selector.h](third_party/blink/renderer/platform/fonts/font_selector.h) | 字体选择抽象接口 | `FontSelector` |
| 2 | [third_party/blink/renderer/platform/fonts/font_selector.cc](third_party/blink/renderer/platform/fonts/font_selector.cc) | FontSelector 实现（generic family 映射） | 多态实现 |
| 3 | [third_party/blink/renderer/platform/fonts/font_cache.h](third_party/blink/renderer/platform/fonts/font_cache.h) | 全局字体缓存管理 | `FontCache` |
| 4 | [third_party/blink/renderer/platform/fonts/font_fallback_list.h](third_party/blink/renderer/platform/fonts/font_fallback_list.h) | 字体 fallback 列表缓存 | `FontFallbackList` |
| 5 | [third_party/blink/renderer/platform/fonts/font_fallback_iterator.h](third_party/blink/renderer/platform/fonts/font_fallback_iterator.h) | Fallback 字体迭代器 | `FontFallbackIterator` |

### 1.2 字体数据结构层

| # | 文件路径 | 功能 | 关键类 |
|---|---------|------|--------|
| 6 | [third_party/blink/renderer/platform/fonts/font.h](third_party/blink/renderer/platform/fonts/font.h) | 字体对象（包含 FontFallbackList） | `Font` |
| 7 | [third_party/blink/renderer/platform/fonts/font_description.h](third_party/blink/renderer/platform/fonts/font_description.h) | 字体描述符（大小、样式、变体等） | `FontDescription` |
| 8 | [third_party/blink/renderer/platform/fonts/font_family.h](third_party/blink/renderer/platform/fonts/font_family.h) | 字体族列表结构 | `FontFamily`, `SharedFontFamily` |
| 9 | [third_party/blink/renderer/platform/fonts/simple_font_data.h](third_party/blink/renderer/platform/fonts/simple_font_data.h) | 单个字体数据（平台相关） | `SimpleFontData` |
| 10 | [third_party/blink/renderer/platform/fonts/font_data.h](third_party/blink/renderer/platform/fonts/font_data.h) | 字体数据抽象基类 | `FontData` |
| 11 | [third_party/blink/renderer/platform/fonts/font_data_for_range_set.h](third_party/blink/renderer/platform/fonts/font_data_for_range_set.h) | 带 unicode-range 的字体数据 | `FontDataForRangeSet` |
| 12 | [third_party/blink/renderer/platform/fonts/segmented_font_data.h](third_party/blink/renderer/platform/fonts/segmented_font_data.h) | 多段字体数据（支持 unicode-range）| `SegmentedFontData` |

### 1.3 Unicode Range 和 @font-face 层

| # | 文件路径 | 功能 | 关键类 |
|---|---------|------|--------|
| 13 | [third_party/blink/renderer/platform/fonts/unicode_range_set.h](third_party/blink/renderer/platform/fonts/unicode_range_set.h) | Unicode 范围管理 | `UnicodeRangeSet`, `UnicodeRange` |
| 14 | [third_party/blink/renderer/core/css/css_font_face.h](third_party/blink/renderer/core/css/css_font_face.h) | @font-face 规则表示 | `CSSFontFace` |
| 15 | [third_party/blink/renderer/core/css/css_font_face_rule.h](third_party/blink/renderer/core/css/css_font_face_rule.h) | CSS @font-face 规则 DOM | `CSSFontFaceRule` |
| 16 | [third_party/blink/renderer/core/css/css_segmented_font_face.h](third_party/blink/renderer/core/css/css_segmented_font_face.h) | 分段 @font-face 处理 | `CSSSegmentedFontFace` |
| 17 | [third_party/blink/renderer/core/css/font_face_cache.h](third_party/blink/renderer/core/css/font_face_cache.h) | @font-face 全局缓存 | `FontFaceCache` |

### 1.4 Shaping 和文本处理层

| # | 文件路径 | 功能 | 关键类 |
|---|---------|------|--------|
| 18 | [third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.h](third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.h) | HarfBuzz 文本成形 | `HarfBuzzShaper` |
| 19 | [third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc](third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc) | HarfBuzz shaping 实现（1216 行） | 核心算法 |
| 20 | [third_party/blink/renderer/platform/fonts/shaping/harfbuzz_face.h](third_party/blink/renderer/platform/fonts/shaping/harfbuzz_face.h) | HarfBuzz 字体接口 | `HarfBuzzFace` |
| 21 | [third_party/blink/renderer/platform/fonts/shaping/shape_result.h](third_party/blink/renderer/platform/fonts/shaping/shape_result.h) | 文本成形结果 | `ShapeResult` |
| 22 | [third_party/blink/renderer/platform/fonts/shaping/shape_result_run.h](third_party/blink/renderer/platform/fonts/shaping/shape_result_run.h) | 单个成形运行（run）| `ShapeResultRun` |
| 23 | [third_party/blink/renderer/platform/fonts/shaping/run_segmenter.h](third_party/blink/renderer/platform/fonts/shaping/run_segmenter.h) | 文本运行分割 | `RunSegmenter` |

### 1.5 平台特异性实现

| # | 文件路径 | 平台 | 关键实现 |
|---|---------|------|---------|
| 24 | `third_party/blink/renderer/platform/fonts/android/font_cache_android.cc` | Android | `GetGenericFamilyNameForScript`, `PlatformFallbackFontForCharacter` |
| 25 | `third_party/blink/renderer/platform/fonts/skia/font_cache_skia.cc` | Linux/Skia | `GetLastResortFallbackFont` |
| 26 | `third_party/blink/renderer/platform/fonts/win/font_cache_skia_win.cc` | Windows | `PlatformFallbackFontForCharacter` |

---

## 2️⃣ 关键类和函数签名

### 2.1 FontSelector - 字体选择入口

```cpp
class FontSelector : public FontCacheClient {
 public:
  // 核心字体选择方法
  // 参数：FontDescription（字体描述）, FontFamily（CSS font-family 列表）
  // 返回：FontData* - 选择的字体数据
  virtual const FontData* GetFontData(const FontDescription&,
                                      const FontFamily&) = 0;

  // 通知选择器将使用此字体和文本
  virtual void WillUseFontData(const FontDescription&,
                               const FontFamily& family,
                               const String& text) = 0;

  // 通知使用带 unicode-range 的字体数据
  virtual void WillUseRange(const FontDescription&,
                            const AtomicString& family_name,
                            const FontDataForRangeSet&) = 0;

  virtual ExecutionContext* GetExecutionContext() const = 0;
  virtual FontFaceCache* GetFontFaceCache() = 0;
};
```

**位置**: [third_party/blink/renderer/platform/fonts/font_selector.h](third_party/blink/renderer/platform/fonts/font_selector.h#L54-L64)

### 2.2 FontFallbackList - 字体 Fallback 缓存

```cpp
class FontFallbackList : public GarbageCollected<FontFallbackList> {
 public:
  const SimpleFontData* PrimarySimpleFontDataWithSpace(
      const FontDescription& font_description);
  const SimpleFontData* PrimarySimpleFontDataWithDigitZero(
      const FontDescription& font_description);
  
  const FontData* FontDataAt(const FontDescription&, unsigned index);
  
  // 内部：确定主字体数据
  const SimpleFontData* DeterminePrimarySimpleFontData(
      const FontDescription&,
      UChar32 lookup_character = uchar::kSpace,
      bool should_contain_glyph = false);
};
```

**位置**: [third_party/blink/renderer/platform/fonts/font_fallback_list.h](third_party/blink/renderer/platform/fonts/font_fallback_list.h#L41-L114)

### 2.3 SimpleFontData - 单个字体数据

```cpp
class SimpleFontData final : public FontData {
 public:
  // 获取字符对应的字体数据
  const SimpleFontData* FontDataForCharacter(UChar32) const override;

  // 获取字符的 glyph ID
  Glyph GlyphForCharacter(UChar32) const;
  
  // 获取字符的数学 glyph
  Glyph GlyphForMathCharacter(UChar32, TextDirection) const;

  // 获取特定字符的边界
  gfx::RectF BoundsForGlyph(Glyph) const;
  float WidthForGlyph(Glyph) const;

  // 字体度量
  const FontMetrics& GetFontMetrics() const;
};
```

**位置**: [third_party/blink/renderer/platform/fonts/simple_font_data.h](third_party/blink/renderer/platform/fonts/simple_font_data.h#L83-L164)

### 2.4 FontCache - 全局字体缓存

```cpp
class FontCache final {
 public:
  static FontCache& Get();

  // 获取特定字族和字体描述的字体
  const SimpleFontData* GetFontData(
      const FontDescription&,
      const AtomicString& family,
      AlternateFontName = AlternateFontName::kAllowAlternate);

  // 获取字符的 fallback 字体
  const SimpleFontData* FallbackFontForCharacter(
      const FontDescription&,
      UChar32,
      const SimpleFontData* font_data_to_substitute,
      FontFallbackPriority = FontFallbackPriority::kText);

  // 最后的 fallback 字体（.notdef）
  const SimpleFontData* GetLastResortFallbackFont(const FontDescription&);

  // 平台特异性实现
  const SimpleFontData* PlatformFallbackFontForCharacter(
      const FontDescription&, UChar32, const SimpleFontData*,
      FontFallbackPriority);

  // Generic family 到具体字族的映射（Android）
  static AtomicString GetGenericFamilyNameForScript(
      const AtomicString& generic_family,
      const AtomicString& script_family,
      const FontDescription&);

  // 检查字族是否可用
  bool IsPlatformFamilyMatchAvailable(const FontDescription&,
                                      const AtomicString& family);
  bool IsPlatformFontUniqueNameMatchAvailable(
      const FontDescription&,
      const AtomicString& unique_font_name);  // local() 匹配
};
```

**位置**: [third_party/blink/renderer/platform/fonts/font_cache.h](third_party/blink/renderer/platform/fonts/font_cache.h#L94-L230)

### 2.5 HarfBuzzShaper - 文本成形

```cpp
class HarfBuzzShaper final {
 public:
  explicit HarfBuzzShaper(String text);

  // 主要 shaping 方法
  ShapeResult* Shape(const Font*, TextDirection, unsigned start, unsigned end) const;
  
  // 带预分割的 shaping
  ShapeResult* Shape(const Font*, TextDirection, unsigned start, unsigned end,
                     const Vector<RunSegmenter::RunSegmenterRange>&,
                     ShapeOptions = ShapeOptions()) const;
  
  // 单个 segmenter range 的 shaping
  ShapeResult* Shape(const Font*, TextDirection, unsigned start, unsigned end,
                     const RunSegmenter::RunSegmenterRange) const;

  // 获取 HarfBuzz glyph 数据（不涉及 cascading）
  struct GlyphData {
    unsigned cluster;
    Glyph glyph;
    gfx::PointF advance;
    gfx::PointF offset;
  };
  using GlyphDataList = Vector<GlyphData, 16>;
  void GetGlyphData(const SimpleFontData& font_data,
                    const LayoutLocale& locale,
                    UScriptCode script,
                    bool is_horizontal,
                    TextDirection direction,
                    GlyphDataList& glyphs);
};
```

**位置**: [third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.h](third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.h#L50-L110)

### 2.6 ShapeResult 和 ShapeResultRun

```cpp
// 整个文本的成形结果
class ShapeResult : public GarbageCollected<ShapeResult> {
 public:
  ShapeResult(unsigned start_index, unsigned num_characters, TextDirection);
  
  // 创建空结果
  static ShapeResult* CreateEmpty(const ShapeResult& other);
  
  // 创建制表符和空格结果
  static const ShapeResult* CreateForTabulationCharacters(...);
  static const ShapeResult* CreateForSpaces(...);
};

// 单个成形运行（使用同一字体）
struct ShapeResultRun : public GarbageCollected<ShapeResultRun> {
 public:
  ShapeResultRun(const SimpleFontData* font,
                 hb_direction_t dir,
                 CanvasRotationInVertical canvas_rotation,
                 hb_script_t script,
                 unsigned start_index,
                 unsigned num_glyphs,
                 unsigned num_characters);

  // 查找子 run
  ShapeResultRun* CreateSubRun(unsigned start, unsigned end);
  
  // 尝试合并相邻 run
  ShapeResultRun* MergeIfPossible(const ShapeResultRun& other) const;
  
  unsigned NumCharacters() const { return num_characters_; }
  unsigned NumGlyphs() const { return glyph_data_.size(); }
  bool HasLigatures() const { return NumGlyphs() < num_characters_; }
};
```

**位置**: 
- ShapeResult: [third_party/blink/renderer/platform/fonts/shaping/shape_result.h](third_party/blink/renderer/platform/fonts/shaping/shape_result.h#L100-L170)
- ShapeResultRun: [third_party/blink/renderer/platform/fonts/shaping/shape_result_run.h](third_party/blink/renderer/platform/fonts/shaping/shape_result_run.h#L53-L200)

### 2.7 FontFallbackIterator - Fallback 迭代

```cpp
class FontFallbackIterator {
 public:
  using HintCharList = Vector<UChar32, 16>;

  FontFallbackIterator(const FontDescription&,
                       FontFallbackList*,
                       FontFallbackPriority);

  bool HasNext() const { return fallback_stage_ != kOutOfLuck; }
  bool NeedsHintList() const;  // 是否需要完整的提示字符列表

  // 获取下一个 fallback 字体
  FontDataForRangeSet* Next(const HintCharList& hint_list);

  void Reset();

 private:
  enum FallbackStage {
    kFallbackPriorityFonts,     // 优先级字体（emoji 等）
    kFontGroupFonts,            // CSS font-family 列表中的字体
    kSegmentedFace,             // @font-face 分段字体
    kPreferencesFonts,          // 用户首选字体
    kSystemFonts,               // 系统 fallback 字体
    kFirstCandidateForNotdefGlyph,  // 最后候选（.notdef glyph）
    kOutOfLuck                  // 无法找到
  };
};
```

**位置**: [third_party/blink/renderer/platform/fonts/font_fallback_iterator.h](third_party/blink/renderer/platform/fonts/font_fallback_iterator.h#L31-L92)

### 2.8 UnicodeRangeSet - Unicode 范围管理

```cpp
struct UnicodeRange final {
  UChar32 From() const { return from_; }
  UChar32 To() const { return to_; }
  bool Contains(UChar32 c) const { return from_ <= c && c <= to_; }
};

class UnicodeRangeSet : public GarbageCollected<UnicodeRangeSet> {
 public:
  explicit UnicodeRangeSet(HeapVector<UnicodeRange>&&);

  // 检查字符是否在范围内
  bool Contains(UChar32) const;
  
  // 检查字符串与范围是否有交集
  bool IntersectsWith(const String&) const;
  
  // 检查是否代表整个代码空间（空 vector）
  bool IsEntireRange() const { return ranges_.empty(); }
};
```

**位置**: [third_party/blink/renderer/platform/fonts/unicode_range_set.h](third_party/blink/renderer/platform/fonts/unicode_range_set.h#L36-L75)

### 2.9 RunSegmenter - 文本运行分割

```cpp
class RunSegmenter {
 public:
  struct RunSegmenterRange {
    unsigned start = 0;
    unsigned end = 0;
    UScriptCode script = USCRIPT_INVALID_CODE;
    OrientationIterator::RenderOrientation render_orientation =
        OrientationIterator::kOrientationKeep;
    FontFallbackPriority font_fallback_priority = FontFallbackPriority::kText;
  };

  RunSegmenter(base::span<const UChar> buffer, FontOrientation);

  // 获取下一个 segmenter range
  bool Consume(RunSegmenterRange*);
};
```

**位置**: [third_party/blink/renderer/platform/fonts/shaping/run_segmenter.h](third_party/blink/renderer/platform/fonts/shaping/run_segmenter.h#L27-L65)

---

## 3️⃣ 完整的调用链流程

### 3.1 高层次流程概览

```
HTML/DOM 元素
    ↓
CSS 样式计算 (StyleResolver)
    ↓
FontDescription 生成
    ↓
Font 对象创建 → FontFallbackList 初始化
    ↓
字体选择 (FontSelector::GetFontData)
    ↓
    ├─ @font-face 检查 (FontFaceCache)
    ├─ CSS font-family 列表处理
    └─ Generic family 映射 (sans-serif, serif 等)
    ↓
FontFallbackList 缓存管理
    ↓
文本测量/渲染
    ↓
HarfBuzzShaper::Shape() 调用
    ↓
RunSegmenter 分割文本
    ↓
字体 cascading + fallback
    ↓
HarfBuzz 成形
    ↓
ShapeResult 返回
```

### 3.2 详细调用链：从文本到 Glyph

#### 3.2.1 字体选择阶段

```cpp
// 1. 从 Style 创建 Font 对象
Font font(font_description, font_selector);
    ↓
// Font 构造函数
Font::Font(const FontDescription& fd, FontSelector* selector)
    : font_fallback_list_(
        MakeGarbageCollected<FontFallbackList>(selector)) {}

    ↓
// 2. 首次使用时，FontFallbackList 选择字体
const SimpleFontData* primary = 
    font_fallback_list_->PrimarySimpleFontDataWithSpace(font_description);
    ↓
// 3. 调用 DeterminePrimarySimpleFontData
const SimpleFontData* FontFallbackList::DeterminePrimarySimpleFontData(
    const FontDescription& font_description,
    UChar32 lookup_character = uchar::kSpace,
    bool should_contain_glyph = false);
    ↓
// 4. 调用 FontFallbackIterator 进行 cascading
FontFallbackIterator iterator(
    font_description,
    this,
    should_contain_glyph ? 
        FontFallbackPriority::kEmojiEmoji : 
        FontFallbackPriority::kText
);

while (iterator.HasNext()) {
  FontDataForRangeSet* candidate = 
      iterator.Next(hint_list);
  // 检查候选字体...
}
```

**关键位置**:
- 字体选择器获取: [font_selector.h#L54](third_party/blink/renderer/platform/fonts/font_selector.h#L54)
- Fallback 列表初始化: [font_fallback_list.h#L41-L50](third_party/blink/renderer/platform/fonts/font_fallback_list.h#L41-L50)
- 迭代器创建: [font_fallback_iterator.h#L31-L40](third_party/blink/renderer/platform/fonts/font_fallback_iterator.h#L31-L40)

#### 3.2.2 @font-face unicode-range 匹配

```cpp
// 1. CSS 解析时，@font-face 规则被转换为 CSSFontFace
CSSFontFace face(
    font_face,
    std::move(unicode_ranges)  // 从 unicode-range 描述符解析
);

// 2. unicode_range_set 被保存在 CSSFontFace 中
Member<const UnicodeRangeSet> ranges_;

// 3. 获取字体数据时，检查范围
const SimpleFontData* CSSFontFace::GetFontData(
    const FontDescription& font_description) {
  // 检查 unicode-range 是否包含字符
  if (!ranges_->Contains(character)) {
    return nullptr;  // 不适用此 @font-face
  }
}

// 4. FontDataForRangeSet 存储字体和对应的范围
class FontDataForRangeSet {
  Member<const SimpleFontData> font_data_;      // 字体数据
  Member<const UnicodeRangeSet> range_set_;     // unicode-range
};

// 5. SegmentedFontData 包含多个 FontDataForRangeSet
class SegmentedFontData : public FontData {
  HeapVector<Member<FontDataForRangeSet>, 1> faces_;
  
  const SimpleFontData* FontDataForCharacter(UChar32 c) const {
    for (auto& face : faces_) {
      if (face->Contains(c)) {
        return face->FontData();
      }
    }
    return nullptr;
  }
};
```

**关键位置**:
- CSS font-face: [css_font_face.h#L40-L50](third_party/blink/renderer/core/css/css_font_face.h#L40-L50)
- Unicode range set: [unicode_range_set.h#L36-L60](third_party/blink/renderer/platform/fonts/unicode_range_set.h#L36-L60)
- 分段字体: [segmented_font_data.h#L35-L60](third_party/blink/renderer/platform/fonts/segmented_font_data.h#L35-L60)

#### 3.2.3 HarfBuzz Shaping 阶段

```cpp
// 1. 调用 HarfBuzzShaper::Shape()
HarfBuzzShaper shaper(text);
ShapeResult* result = shaper.Shape(
    font,           // Font 对象（包含 FontFallbackList）
    direction,      // TextDirection (RTL/LTR)
    start_index,    // 文本起始位置
    end_index       // 文本结束位置
);

// 2. 内部：预分割文本（script, orientation, small-caps）
RunSegmenter segmenter(text, font_orientation);
Vector<RunSegmenter::RunSegmenterRange> ranges;
RunSegmenter::RunSegmenterRange range;
while (segmenter.Consume(&range)) {
  ranges.push_back(range);
}

// 3. 对每个 segment 进行 shaping
for (const auto& segment : ranges) {
  ShapeSegment(
      range_context,
      segment,
      shape_result  // 输出
  );
}

// 4. ShapeSegment 中的字体 cascading
FontFallbackIterator iterator(
    font_description,
    font_fallback_list,
    segment.font_fallback_priority  // 优先级（CJK vs others）
);

while (iterator.HasNext()) {
  FontDataForRangeSet* font_range_data = 
      iterator.Next(hint_chars);
  
  // 使用 HarfBuzz 成形此字体
  const SimpleFontData* font_data = 
      font_range_data->FontData();
  
  // 检查 unicode-range 限制
  const UnicodeRangeSet* range_set = 
      font_range_data->Ranges();
  
  // 获取 HarfBuzz font
  hb_font_t* hb_font = 
      harfbuzz_face->GetScaledFont(
          range_set,      // 应用 unicode-range 限制
          vertical_layout_callbacks,
          specified_size
      );
  
  // HarfBuzz 成形调用（返回 glyphs）
  ShapeHarfBuzz(
      segment,
      font_data,
      hb_font,
      shape_result
  );
}

// 5. 返回 ShapeResult（包含多个 ShapeResultRun）
return shape_result;
```

**关键位置**:
- HarfBuzz shaper: [harfbuzz_shaper.h#L50-L80](third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.h#L50-L80)
- Shaping 实现: [harfbuzz_shaper.cc#L1-L100](third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc#L1-L100)
- Run segmenter: [run_segmenter.h#L27-L65](third_party/blink/renderer/platform/fonts/shaping/run_segmenter.h#L27-L65)

### 3.3 字符覆盖检查

```cpp
// 方法 1: SimpleFontData::GlyphForCharacter（无 shaping）
Glyph SimpleFontData::GlyphForCharacter(UChar32 c) const {
  // 直接查询 HarfBuzz face 是否有此字符的 glyph
  return platform_data_->SkFont().getUnicharMetrics(c, nullptr);
}

// 方法 2: HarfBuzzFace::HbGlyphForCharacter（简单查询）
Glyph HarfBuzzFace::HbGlyphForCharacter(UChar32 character) {
  hb_font_t* hb_font = GetScaledFont();
  return hb_font_get_nominal_glyph(hb_font, character, &glyph);
}

// 方法 3: SegmentedFontData::FontDataForCharacter（范围检查）
const SimpleFontData* SegmentedFontData::FontDataForCharacter(
    UChar32 c) const {
  // 对于每个 @font-face 段，检查：
  // 1. unicode-range 是否包含此字符
  // 2. 字体是否有此字符的 glyph
  for (const auto& face : faces_) {
    if (face->Contains(c)) {  // unicode-range 检查
      const SimpleFontData* font = face->FontData();
      if (font) {
        Glyph glyph = font->GlyphForCharacter(c);
        if (glyph != 0) {  // 非 .notdef glyph
          return font;
        }
      }
    }
  }
  return nullptr;
}

// 方法 4: Font::SelectFallbackFont（完整 cascading）
const SimpleFontData* Font::SelectFallbackFont(
    const FontDescription& font_description,
    UChar32 character,
    FontFallbackPriority fallback_priority) {
  // 使用 FontFallbackIterator 遍历 fallback 字体
  // 对每个字体检查 GlyphForCharacter
}
```

**关键位置**:
- GlyphForCharacter: [simple_font_data.h#L164](third_party/blink/renderer/platform/fonts/simple_font_data.h#L164)
- SegmentedFontData: [segmented_font_data.h#L40-L55](third_party/blink/renderer/platform/fonts/segmented_font_data.h#L40-L55)
- HarfBuzz face: [harfbuzz_face.h#L84](third_party/blink/renderer/platform/fonts/shaping/harfbuzz_face.h#L84)

---

## 4️⃣ Fallback 字体选择逻辑

### 4.1 Fallback 阶段（按优先级）

```cpp
enum FallbackStage {
  // 1. 优先级字体（emoji 等特殊内容）
  kFallbackPriorityFonts,
  
  // 2. CSS font-family 列表中的字体
  kFontGroupFonts,
  
  // 3. @font-face 分段字体（带 unicode-range）
  kSegmentedFace,
  
  // 4. 用户首选字体
  kPreferencesFonts,
  
  // 5. 系统 fallback 字体（平台相关）
  kSystemFonts,
  
  // 6. 最后一个候选（用于 .notdef glyph）
  kFirstCandidateForNotdefGlyph,
  
  // 7. 无法找到
  kOutOfLuck
};
```

### 4.2 平台特异性的 System Font Fallback

#### 4.2.1 Android 特性

```cpp
// 位置: font_cache_android.cc#L231
AtomicString FontCache::GetGenericFamilyNameForScript(
    const AtomicString& generic_family,
    const AtomicString& script_family,
    const FontDescription& font_description) {
  
  // CJK 处理（中文、日文、韩文）
  if (font_description.GetScript() == USCRIPT_HAN ||
      font_description.GetScript() == USCRIPT_HIRAGANA ||
      font_description.GetScript() == USCRIPT_HANGUL) {
    // 返回 CJK 特定的字族（NotoSansCJK 等）
    return GetCJKFamilyNameForScript(generic_family);
  }
  
  // 非 CJK 脚本
  return script_family;  // 返回原始 script_family
}

// 位置: font_cache_android.cc#L125
const SimpleFontData* FontCache::PlatformFallbackFontForCharacter(
    const FontDescription& font_description,
    UChar32 character,
    const SimpleFontData* font_data_to_substitute,
    FontFallbackPriority fallback_priority) {
  
  // 使用 SkFontMgr 查找字符的字体
  sk_sp<SkTypeface> typeface =
      skia::DefaultFontMgr()->matchFamilyStyleCharacter(
          nullptr,  // family (nullptr 表示任何字族)
          font_description.SkiaFontStyle(),
          &bcp47,
          1,
          character);  // 查询字符
  
  return GetFontData(font_description, family_name);
}
```

**关键点**:
- Android 系统通过 SkFontMgr 和 `/system/etc/fonts.xml` 提供 fallback
- CJK 文本获得系统字体（NotoSansCJK）
- 非 CJK 文本回退到硬编码的 generic family names

#### 4.2.2 Linux/Skia 特性

```cpp
// 位置: font_cache_skia.cc#L147
const SimpleFontData* FontCache::GetLastResortFallbackFont(
    const FontDescription& font_description) {
  
  // Linux 使用 fontconfig 或 HarfBuzz 提供的 fallback
  // 通过 GDI 或其他系统 API 获取
}
```

#### 4.2.3 Windows 特性

```cpp
// 位置: font_cache_skia_win.cc#L284
const SimpleFontData* FontCache::PlatformFallbackFontForCharacter(
    const FontDescription& font_description,
    UChar32 character,
    const SimpleFontData* font_data_to_substitute,
    FontFallbackPriority fallback_priority) {
  
  // Windows 使用 DirectWrite 或 GDI 查找字体
}
```

**关键位置**:
- Android: [font_cache_android.cc#L125-L230](third_party/blink/renderer/platform/fonts/android/font_cache_android.cc#L125-L230)
- Linux: [font_cache_skia.cc#L147-L160](third_party/blink/renderer/platform/fonts/skia/font_cache_skia.cc#L147-L160)
- Windows: [font_cache_skia_win.cc#L284-L310](third_party/blink/renderer/platform/fonts/win/font_cache_skia_win.cc#L284-L310)

---

## 5️⃣ Run Splitting 逻辑

### 5.1 Run Splitting 发生的位置

Run splitting 发生在 **HarfBuzz 成形之前**（pre-HarfBuzz splitting）：

```cpp
// 位置: harfbuzz_shaper.cc
ShapeResult* HarfBuzzShaper::Shape(
    const Font* font,
    TextDirection direction,
    unsigned start,
    unsigned end) {
  
  // 1. 预分割阶段（在 HarfBuzz 前）
  RunSegmenter segmenter(text_.Characters16(), orientation);
  Vector<RunSegmenter::RunSegmenterRange> ranges;
  
  RunSegmenter::RunSegmenterRange range;
  while (segmenter.Consume(&range)) {
    ranges.push_back(range);
    // 按以下因素分割：
    // - Script (USCRIPT_HAN, USCRIPT_LATIN 等)
    // - Orientation (Upright/Mixed/Sideways)
    // - Small-caps 状态
    // - Symbols (Emoji 等)
  }
  
  // 2. 对每个 segment 进行 shaping
  ShapeResult* result = MakeGarbageCollected<ShapeResult>(...);
  for (const auto& range : ranges) {
    ShapeSegment(range_context, range, result);
  }
  
  return result;
}

// ShapeSegment 内部：字体 cascading（在 HarfBuzz 前）
void HarfBuzzShaper::ShapeSegment(
    RangeContext* range_context,
    const RunSegmenter::RunSegmenterRange& range,
    ShapeResult* result) {
  
  FontFallbackIterator iterator(...);
  
  while (iterator.HasNext()) {
    FontDataForRangeSet* font_range_data = iterator.Next(hints);
    
    // 3. 对每个 fallback 字体进行 HarfBuzz 成形
    // 这里可能会再次分割（如果字体不能形成某些 cluster）
    
    ShapeHarfBuzz(
        range,
        font_range_data,
        result
    );
  }
}
```

### 5.2 Run Splitting 的原因

```
1. 脚本变化（Script boundaries）
   Example: "Hello 中文 World"
            Latin  Han  Latin
            → 3 个 segment

2. 方向变化（Bidi boundaries）
   Example: "English עברית"
            LTR     RTL
            → 2 个 segment

3. 字体变化（Font boundaries）
   Example: CSS: font-family: Arial, SimSun, serif
            ASCII 字符 → Arial
            中文字符 → SimSun
            其他字符 → serif

4. 字体加载状态（Font loading state）
   Example: @font-face { font-family: "Custom"; }
            已加载时用 Custom
            未加载时用 fallback

5. Small-caps 变化
   Example: text-transform: small-caps
            大写字母 → small-caps 字体
            小写字母 → 普通字体

6. Emoji 变化（Symbols）
   Example: "Hello 😀 World"
            Text  Emoji  Text
            → 3 个 segment（emoji 可能用不同字体）
```

**关键位置**:
- Run segmenter: [run_segmenter.h#L27-L65](third_party/blink/renderer/platform/fonts/shaping/run_segmenter.h#L27-L65)
- HarfBuzz shaper: [harfbuzz_shaper.cc#L200-L350](third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc#L200-L350)
- Script iterator: 脚本检测基础类
- Orientation iterator: 方向检测基础类

### 5.3 Shape Splitting 示意图

```
输入文本: "Hello 中文 😀 World"

↓ RunSegmenter 分割

Segment 1: "Hello " (USCRIPT_LATIN, text priority)
Segment 2: "中文 " (USCRIPT_HAN, text priority)
Segment 3: "😀 " (USCRIPT_COMMON, emoji priority)
Segment 4: "World" (USCRIPT_LATIN, text priority)

↓ 对每个 segment 进行 HarfBuzz shaping + font cascading

Segment 1 + Font Arial   → ShapeResultRun #1
Segment 2 + Font SimSun  → ShapeResultRun #2
Segment 3 + Font NotoColorEmoji → ShapeResultRun #3
Segment 4 + Font Arial   → ShapeResultRun #4

↓ 合并相同字体的相邻 run（可选优化）

ShapeResultRun #1 (Arial)
ShapeResultRun #2 (SimSun)
ShapeResultRun #3 (NotoColorEmoji)
ShapeResultRun #4 (Arial)  ← 注：不能与 #1 合并，因为中间有其他 run
```

---

## 6️⃣ Generic Family 到具体字族的映射

### 6.1 全局映射（WebPreferences）

```cpp
// Android 示例：来自 AwSettings.java
struct WebPreferences {
  // Generic family → 具体字族名
  using WebFontFamily = std::u16string;
  
  std::map<uint32_t, WebFontFamily> standard_font_family_map = {
    {GenericFamilyType::kStandardFamily, u"sans-serif"},
    {GenericFamilyType::kSerifFamily, u"serif"},
    {GenericFamilyType::kFixedFamily, u"monospace"},
    {GenericFamilyType::kCursiveFamily, u"cursive"},
    {GenericFamilyType::kFantasyFamily, u"fantasy"},
  };
};

// 位置: web_preferences.h#L45
```

### 6.2 FontSelector 的映射

```cpp
// 位置: font_selector.cc#L18-L99
AtomicString FontSelector::FamilyNameFromSettings(
    const GenericFontFamilySettings& settings,
    const FontDescription& font_description,
    const FontFamily& generic_family,
    UseCounter* use_counter) {
  
  // 非 Android 平台
  #if !BUILDFLAG(IS_ANDROID)
    UScriptCode script = font_description.GetScript();
    if (generic_family_name == font_family_names::kSerif)
      return settings.Serif(script);      // 返回用户设置的 serif 字族
    if (generic_family_name == font_family_names::kSansSerif)
      return settings.SansSerif(script);  // 返回用户设置的 sans-serif 字族
    if (generic_family_name == font_family_names::kMonospace)
      return settings.Fixed(script);      // 返回用户设置的 monospace 字族
  #else
    // Android 平台
    return FontCache::GetGenericFamilyNameForScript(
        generic_family_name,
        generic_family_name,
        font_description);
  #endif
}
```

**关键位置**:
- WebPreferences: [web_preferences.h#L45](third_party/blink/public/common/web_preferences/web_preferences.h#L45)
- FontSelector: [font_selector.cc#L18-L99](third_party/blink/renderer/platform/fonts/font_selector.cc#L18-L99)
- GenericFontFamilySettings: 用户偏好设置

---

## 7️⃣ local() 字体匹配

### 7.1 @font-face 中的 local() 处理

```cpp
// CSS @font-face 规则
@font-face {
  font-family: "MyFont";
  src: local("Helvetica Neue"),      // 本地字族名
       local("helvetica-neue"),       // 或 PostScript 名
       url("myfont.woff2") format("woff2");
}

// 处理流程

// 1. CSS 解析器识别 local() 源
// 位置: css_parsing_utils.cc#L6260-L6340

// 2. FontFace 存储 local() 名称列表
class CSSFontFaceSource {
  Vector<AtomicString> local_names_;  // ["Helvetica Neue", "helvetica-neue"]
};

// 3. 字体查询时，检查本地可用性
// 位置: font_cache.h#L122
bool FontCache::IsPlatformFontUniqueNameMatchAvailable(
    const FontDescription& font_description,
    const AtomicString& unique_font_name) {
  // 检查系统中是否存在此 PostScript 名或全名
  // - Windows: 查询注册表或 DirectWrite API
  // - macOS: 查询 CTFont API
  // - Linux: 查询 fontconfig
  // - Android: 查询 SkFontMgr
}

// 4. 如果 local() 字体可用，则使用
if (font_cache->IsPlatformFontUniqueNameMatchAvailable(
        font_description, unique_font_name)) {
  return font_cache->GetFontData(
      font_description,
      unique_font_name);
}

// 5. 否则，继续尝试下一个 src: url() 或其他 local()
```

**关键位置**:
- CSS 解析: [css_parsing_utils.cc#L6260-L6340](third_party/blink/renderer/core/css/properties/css_parsing_utils.cc#L6260-L6340)
- FontFaceSource: [css_font_face_source.h](third_party/blink/renderer/core/css/css_font_face_source.h)
- Platform 检查: [font_cache.h#L122-L130](third_party/blink/renderer/platform/fonts/font_cache.h#L122-L130)

---

## 8️⃣ Unicode Range 的实现

### 8.1 Unicode Range 解析

```cpp
// CSS 示例
@font-face {
  font-family: "MyFont";
  unicode-range: U+0100-01FF, U+0250-0377;
  src: url("...") format("woff2");
}

// 8.1.1 CSS 解析阶段
// 位置: css_parsing_utils.cc (unicode-range descriptor 解析)
ParseUnicodeRangeDescriptor(CSSParserTokenRange& range)
  → 返回 Vector<UnicodeRange>
  
// 8.1.2 UnicodeRange 结构
struct UnicodeRange {
  UChar32 from_;   // 范围起点（如 0x0100）
  UChar32 to_;     // 范围终点（如 0x01FF）
  
  bool Contains(UChar32 c) const {
    return from_ <= c && c <= to_;
  }
};

// 8.1.3 UnicodeRangeSet 管理多个范围
class UnicodeRangeSet : public GarbageCollected<UnicodeRangeSet> {
  HeapVector<UnicodeRange> ranges_;  // 排序的范围列表
  
  bool Contains(UChar32 c) const {
    // 二分搜索
    auto it = std::lower_bound(
        ranges_.begin(), ranges_.end(), c);
    return it != ranges_.end() && it->Contains(c);
  }
};
```

### 8.2 Unicode Range 在字体选择中的应用

```cpp
// 8.2.1 SegmentedFontData - 多个 @font-face 段
class SegmentedFontData : public FontData {
  HeapVector<Member<FontDataForRangeSet>, 1> faces_;
  
  const SimpleFontData* FontDataForCharacter(UChar32 c) const {
    for (const auto& face : faces_) {
      // 1. 检查 unicode-range 是否包含字符
      if (!face->Ranges()->Contains(c)) {
        continue;
      }
      
      // 2. 检查字体是否有此字符的 glyph
      const SimpleFontData* font_data = face->FontData();
      if (font_data && font_data->GlyphForCharacter(c) != 0) {
        return font_data;
      }
    }
    return nullptr;
  }
};

// 8.2.2 FontDataForRangeSet - 字体 + 范围对
class FontDataForRangeSet : public GarbageCollected<FontDataForRangeSet> {
  Member<const SimpleFontData> font_data_;
  Member<const UnicodeRangeSet> range_set_;
  
  bool Contains(UChar32 c) const {
    // 如果没有 range_set，则整个代码空间都包含
    return !range_set_ || range_set_->Contains(c);
  }
};

// 8.2.3 HarfBuzz 应用 unicode-range 限制
// 位置: harfbuzz_face.h#L68
hb_font_t* HarfBuzzFace::GetScaledFont(
    const UnicodeRangeSet* range_set,  // unicode-range 限制
    VerticalLayoutCallbacks callbacks,
    float specified_size) const {
  
  // HarfBuzz 获取 glyph 时，会检查范围
  // 如果字符不在 range_set 中，返回 0 (missing glyph)
}
```

**关键位置**:
- Unicode range 解析: [css_parsing_utils.cc](third_party/blink/renderer/core/css/properties/css_parsing_utils.cc)
- UnicodeRangeSet: [unicode_range_set.h#L36-L75](third_party/blink/renderer/platform/fonts/unicode_range_set.h#L36-L75)
- SegmentedFontData: [segmented_font_data.h#L30-L55](third_party/blink/renderer/platform/fonts/segmented_font_data.h#L30-L55)
- HarfBuzz 应用: [harfbuzz_face.h#L68-L77](third_party/blink/renderer/platform/fonts/shaping/harfbuzz_face.h#L68-L77)

---

## 9️⃣ 平台差异分析

### 9.1 Android 特性

#### 字体来源
- 系统字体位置: `/system/etc/fonts.xml` 配置的字体目录
- 用户字体: WebView 应用可通过 Java API 设置
- 默认字体: "sans-serif"（从 fonts.xml 获取）

#### 特殊处理
```cpp
// CJK 特殊处理
if (script == USCRIPT_HAN || script == USCRIPT_HIRAGANA || 
    script == USCRIPT_HANGUL) {
  // 返回 CJK 专用字族（如 NotoSansCJK）
  return GetCJKFamilyNameForScript(generic_family);
}

// 系统字体检测
SkFontMgr::legacyMakeTypeface(family_name, style)
  → 通过 Skia 查询系统字体
```

#### 关键文件
- [font_cache_android.cc](third_party/blink/renderer/platform/fonts/android/font_cache_android.cc)
- [AwSettings.java](android_webview/java/src/org/chromium/android_webview/AwSettings.java)

### 9.2 Linux/Skia 特性

#### 字体来源
- fontconfig 数据库（`~/.fonts.conf`）
- 系统字体目录（`/usr/share/fonts/`）
- Noto 字体集

#### 特殊处理
```cpp
// 使用 fontconfig 查询 fallback
fc_pattern_t* pattern = FcPatternCreate();
FcPatternAddString(pattern, FC_FAMILY, "DejaVu Sans");
FcPatternAddInteger(pattern, FC_CHARSET, character);
```

#### 关键文件
- [font_cache_skia.cc](third_party/blink/renderer/platform/fonts/skia/font_cache_skia.cc)

### 9.3 Windows 特性

#### 字体来源
- 注册表 (HKLM\Software\Microsoft\Windows NT\CurrentVersion\Fonts)
- 系统目录 (C:\Windows\Fonts\)
- DirectWrite 字体引擎

#### 特殊处理
```cpp
// 使用 DirectWrite 查询字体
IDWriteFontCollection* font_collection = ...;
IDWriteFont* font = font_collection->FindFamilyName(...);
```

#### 关键文件
- [font_cache_skia_win.cc](third_party/blink/renderer/platform/fonts/win/font_cache_skia_win.cc)

### 9.4 macOS 特性

#### 字体来源
- ~/Library/Fonts/
- /Library/Fonts/
- /System/Library/Fonts/

#### 特殊处理
```cpp
// 使用 Core Text / CTFont API
CTFontRef font = CTFontCreateWithName(name, size, matrix);
```

---

## 🔟 完整的代码流程示例

### 示例：渲染 "Hello 世界"

```cpp
// 1. HTML 解析和样式计算
<div style="font-family: Arial, '微软雅黑', serif;">
  Hello 世界
</div>

// 2. 创建 FontDescription
FontDescription fd;
fd.SetFamily(FontFamily::Create("Arial", ...));
fd.SetFamily(FontFamily::Create("微软雅黑", ...));
fd.SetFamily(FontFamily::Create("serif", ...));
fd.SetSize(16);
fd.SetScript(USCRIPT_LATIN);  // 初始脚本（稍后会变化）

// 3. 创建 Font 对象
Font font(fd, font_selector);

// 4. 获取主字体（空格用于测量）
const SimpleFontData* primary = 
    font.PrimaryFont();  // Arial

// 5. 执行文本 shaping
HarfBuzzShaper shaper("Hello 世界");

ShapeResult* shape_result = 
    shaper.Shape(&font, TextDirection::kLtr, 0, 11);

// 内部执行流程：
// 5.1 RunSegmenter 分割
//     Segment 1: "Hello " (USCRIPT_LATIN)
//     Segment 2: "世界" (USCRIPT_HAN)

// 5.2 对 Segment 1 进行 shaping
//     Script: USCRIPT_LATIN
//     FontFallbackIterator 尝试字体：
//       - 尝试 Arial（可用，有 ASCII glyph）
//       - 返回 ShapeResultRun with font=Arial

// 5.3 对 Segment 2 进行 shaping
//     Script: USCRIPT_HAN
//     FontFallbackIterator 尝试字体：
//       - 尝试 Arial（不可用，无 CJK glyph）
//       - 尝试 微软雅黑（可用，有 CJK glyph）
//       - 返回 ShapeResultRun with font=微软雅黑

// 6. 返回 ShapeResult（包含 2 个 ShapeResultRun）
//    Run #1: "Hello " (Arial)
//    Run #2: "世界" (微软雅黑)

// 7. 渲染
// 对每个 run 调用绘制函数（使用对应的 glyph ID 和字体）
```

**代码位置参考**:
- 文本 shaping: [harfbuzz_shaper.cc#L1055-L1160](third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc#L1055-L1160)
- Segment 分割: [run_segmenter.h](third_party/blink/renderer/platform/fonts/shaping/run_segmenter.h)
- 字体选择: [font_selector.cc](third_party/blink/renderer/platform/fonts/font_selector.cc)

---

## 📌 关键概念总结

| 概念 | 定义 | 位置 |
|------|------|------|
| **FontDescription** | 字体属性（大小、样式、变体等） | font_description.h |
| **Font** | 包含 FontDescription + FontFallbackList 的字体对象 | font.h |
| **FontFallbackList** | 字体 cascade 列表缓存 | font_fallback_list.h |
| **SimpleFontData** | 单个具体字体的数据（平台相关） | simple_font_data.h |
| **FontDataForRangeSet** | 字体 + unicode-range 对 | font_data_for_range_set.h |
| **SegmentedFontData** | 多个 @font-face 段（支持 unicode-range） | segmented_font_data.h |
| **UnicodeRange** | Unicode 范围（如 U+0100-01FF） | unicode_range_set.h |
| **CSSFontFace** | @font-face CSS 规则表示 | css_font_face.h |
| **RunSegmenter** | 按 script/orientation/small-caps 分割文本 | run_segmenter.h |
| **HarfBuzzShaper** | 使用 HarfBuzz 库进行文本成形 | harfbuzz_shaper.h |
| **ShapeResult** | 文本成形结果（多个 run） | shape_result.h |
| **ShapeResultRun** | 单个成形 run（同一字体） | shape_result_run.h |
| **Glyph** | 字体中字符的视觉表示（glyph ID） | glyph.h |

---

## 📞 关键函数调用链总结

```
HTML/DOM
    ↓
StyleResolver (CSS 样式计算)
    ↓
Font::Font(FontDescription, FontSelector)
    ↓
FontFallbackList::FontFallbackList(FontSelector)
    ↓
[首次使用]
    ↓
Font::PrimaryFont()
    → FontFallbackList::PrimarySimpleFontDataWithSpace()
    → FontFallbackList::DeterminePrimarySimpleFontData()
    → FontFallbackIterator::Next()  [循环]
    → FontSelector::GetFontData()
    → CSSSegmentedFontFace::GetFontData()  [如果是 @font-face]
    → SegmentedFontData::FontDataForCharacter()  [unicode-range 检查]
    → FontCache::GetFontData()
    → SimpleFontData::GlyphForCharacter()  [glyph 检查]
    ↓
[文本渲染]
    ↓
HarfBuzzShaper::Shape(Font*, direction, start, end)
    ↓
RunSegmenter 分割
    ↓
ShapeSegment() [对每个 segment]
    ↓
FontFallbackIterator + HarfBuzz + unicode-range 检查
    ↓
ShapeResult 返回
    ↓
绘制
```

---

## 附录：文件清单

### 完整的 26 个关键文件

1. `third_party/blink/renderer/platform/fonts/font_selector.h`
2. `third_party/blink/renderer/platform/fonts/font_selector.cc`
3. `third_party/blink/renderer/platform/fonts/font_cache.h`
4. `third_party/blink/renderer/platform/fonts/font_fallback_list.h`
5. `third_party/blink/renderer/platform/fonts/font_fallback_iterator.h`
6. `third_party/blink/renderer/platform/fonts/font.h`
7. `third_party/blink/renderer/platform/fonts/font_description.h`
8. `third_party/blink/renderer/platform/fonts/font_family.h`
9. `third_party/blink/renderer/platform/fonts/simple_font_data.h`
10. `third_party/blink/renderer/platform/fonts/font_data.h`
11. `third_party/blink/renderer/platform/fonts/font_data_for_range_set.h`
12. `third_party/blink/renderer/platform/fonts/segmented_font_data.h`
13. `third_party/blink/renderer/platform/fonts/unicode_range_set.h`
14. `third_party/blink/renderer/core/css/css_font_face.h`
15. `third_party/blink/renderer/core/css/css_font_face_rule.h`
16. `third_party/blink/renderer/core/css/css_segmented_font_face.h`
17. `third_party/blink/renderer/core/css/font_face_cache.h`
18. `third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.h`
19. `third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc` (1216 行)
20. `third_party/blink/renderer/platform/fonts/shaping/harfbuzz_face.h`
21. `third_party/blink/renderer/platform/fonts/shaping/shape_result.h`
22. `third_party/blink/renderer/platform/fonts/shaping/shape_result_run.h`
23. `third_party/blink/renderer/platform/fonts/shaping/run_segmenter.h`
24. `third_party/blink/renderer/platform/fonts/android/font_cache_android.cc`
25. `third_party/blink/renderer/platform/fonts/skia/font_cache_skia.cc`
26. `third_party/blink/renderer/platform/fonts/win/font_cache_skia_win.cc`

### 主要类（13 个）

1. **FontSelector** - 字体选择抽象接口
2. **Font** - 字体对象容器
3. **FontDescription** - 字体属性描述
4. **FontFamily** - 字体族列表
5. **SimpleFontData** - 单个字体数据
6. **FontData** - 字体数据抽象基类
7. **FontDataForRangeSet** - 带范围的字体数据
8. **SegmentedFontData** - 分段字体（多 @font-face）
9. **FontFallbackList** - Fallback 列表缓存
10. **FontFallbackIterator** - Fallback 迭代器
11. **FontCache** - 全局字体缓存
12. **HarfBuzzShaper** - 文本成形器
13. **ShapeResult** + **ShapeResultRun** - 成形结果

### 主要函数（10+ 个）

1. `FontSelector::GetFontData()`
2. `FontFallbackList::PrimarySimpleFontDataWithSpace()`
3. `FontFallbackList::DeterminePrimarySimpleFontData()`
4. `FontFallbackIterator::Next()`
5. `FontCache::GetFontData()`
6. `FontCache::FallbackFontForCharacter()`
7. `FontCache::GetLastResortFallbackFont()`
8. `FontCache::GetGenericFamilyNameForScript()` (Android)
9. `SimpleFontData::GlyphForCharacter()`
10. `SegmentedFontData::FontDataForCharacter()`
11. `HarfBuzzShaper::Shape()`
12. `RunSegmenter::Consume()`
13. `UnicodeRangeSet::Contains()`

---

## 📖 进一步阅读

- [Chromium 多进程架构](https://www.chromium.org/developers/design-documents/multi-process-architecture)
- [Blink 渲染引擎架构](https://docs.google.com/document/d/1aitSOucL0ulzfxruKy1xjEDGLGUc1UD0GXNcSwCF5zc)
- [CSS Fonts 规范](https://drafts.csswg.org/css-fonts/)
- [HarfBuzz 成形引擎](https://harfbuzz.github.io/)
- [Unicode 标准](https://unicode.org/)

---

**文档生成日期**: 2026-01-29  
**Chromium 版本**: Latest (Master)  
**深度分析**: 完整的字体渲染系统
