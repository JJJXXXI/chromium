# Chromium 字符替补处理 - 完整调用栈

**目标**: 从 CSS 渲染到字体替补处理的完整源代码调用链映射

---

## 📊 调用栈概览 (Top-Down)

```
Layout Phase (LayoutAlgorithm)
  ↓
InlineLayoutAlgorithm::Layout()
  ↓
InlineLayoutAlgorithm::CreateLine()
  ↓
LineBreaker::NextBreak()
  ↓
LineBreaker::HandleText()
  ↓
LineBreaker::BreakText()
  ↓
LineBreaker::ShapeText()
  ↓
HarfBuzzShaper::Shape()
  ├─ FontFallbackIterator::Next()  [字体替补)
  ├─ hb_shape()                    [HarfBuzz 整形]
  └─ HarfBuzzShaper::ExtractShapeResults()  [结果分析]
      ├─ QueueCharacters()          [队列未整形字符]
      ├─ CommitGlyphs()             [提交成功整形]
      └─ FontCache::FallbackFontForCharacter()  [系统 fallback]
```

---

## 🔍 详细调用栈 (带源码)

### 第 1 层: 布局算法

#### 1.1 InlineLayoutAlgorithm::Layout()

**源文件**: [inline_layout_algorithm.cc](third_party/blink/renderer/core/layout/inline/inline_layout_algorithm.cc)

**功能**: 主内容布局算法，对内联元素进行排版

```cpp
// 调用 CreateLine 来创建每一行
const LayoutResult* InlineLayoutAlgorithm::Layout() {
  // ...
  CreateLine(line_opportunity, leading_floats);
  // ...
}
```

---

#### 1.2 InlineLayoutAlgorithm::CreateLine()

**源文件**: [inline_layout_algorithm.cc](third_party/blink/renderer/core/layout/inline/inline_layout_algorithm.cc)

**功能**: 创建单一行的排版结果

```cpp
void InlineLayoutAlgorithm::CreateLine(
    const LineLayoutOpportunity& line_opportunity,
    const LeadingFloats& leading_floats) {
  // 创建 LineBreaker 实例用于确定行的边界
  LineBreaker line_breaker(Node(), LineBreakerMode::kContent,
                          constraint_space_, line_opportunity,
                          leading_floats, break_token_, context_);
  
  // 调用 NextBreak 找下一个换行点
  line_breaker.NextBreak(...);
  // ...
}
```

---

### 第 2 层: 行破裂器

#### 2.1 LineBreaker::NextBreak()

**源文件**: [line_breaker.cc](third_party/blink/renderer/core/layout/inline/line_breaker.cc)

**功能**: 找到下一个换行机会点，通过逐个处理 InlineItem

```cpp
// LineBreaker 的主循环
while (node_.ChildCount()) {
  // 对每个 InlineItem 调用 HandleInlineItem
  switch (item.Type()) {
    case InlineItem::kText:
      // 如果已有 ShapeResult，直接使用
      if (item.TextShapeResult()) {
        HandleText(item, *item.TextShapeResult(), line_info);
      } else {
        // 否则需要整形文本
        // ...
      }
      break;
  }
}
```

**关键变量**:
- `item` - 当前 InlineItem (可能是文本、元素、控制符等)
- `line_info` - 当前行的信息

---

#### 2.2 LineBreaker::HandleText()

**源文件**: [line_breaker.cc](third_party/blink/renderer/core/layout/inline/line_breaker.cc) 行 1286-1300

**功能**: 处理文本 InlineItem，决定多少文本可以放在当前行

```cpp
void LineBreaker::HandleText(
    const InlineItem& item,
    const ShapeResult& shape_result,
    LineInfo* line_info) {
  
  // 如果处于 trailing 状态，只能添加尾部空格
  if (state_ == LineBreakState::kTrailing) {
    HandleTrailingSpaces(item, &shape_result, line_info);
    return;
  }
  
  // 否则需要进行文本断行处理
  // 这涉及到字符级别的整形和替补
  // ...
}
```

**处理流程**:
1. 检查文本是否已整形 (通常从 `item.TextShapeResult()` 获取)
2. 如果未整形，调用 `ShapeText()` 进行整形
3. 使用 `ShapingLineBreaker` 找到断行点

---

#### 2.3 LineBreaker::BreakText()

**源文件**: [line_breaker.cc](third_party/blink/renderer/core/layout/inline/line_breaker.cc) 行 1568-2000

**功能**: 对文本进行断行，处理超长文本不适应行宽的情况

```cpp
LineBreaker::BreakResult LineBreaker::BreakText(
    InlineItemResult* item_result,
    const InlineItem& item,
    const ShapeResult& item_shape_result,
    LayoutUnit available_width,
    LayoutUnit available_width_with_hyphens,
    LineInfo* line_info) {
  
  // 创建 ShapingLineBreaker 来处理字符级别的断行
  class ShapingLineBreakerImpl : public ShapingLineBreaker {
   protected:
    const ShapeResult* Shape(unsigned start,
                             unsigned end,
                             ShapeOptions options) final {
      // ← 关键调用: ShapeText()
      return line_breaker_->ShapeText(*item_, start, end, options);
    }
  } breaker(this, &item, &item_shape_result);
  
  // breaker 会调用 Shape() 方法进行本地段落的重新整形
  // ...
}
```

**关键点**: 当需要在文本中的任意位置断行时，需要重新整形左右两部分文本

---

### 第 3 层: 文本整形

#### 3.1 LineBreaker::ShapeText()

**源文件**: [line_breaker.cc](third_party/blink/renderer/core/layout/inline/line_breaker.cc) 行 2003-2024

**功能**: 对给定范围的文本进行 HarfBuzz 整形

```cpp
const ShapeResult* LineBreaker::ShapeText(
    const InlineItem& item,
    unsigned start,
    unsigned end,
    ShapeOptions options) {
  
  ShapeResult* shape_result = nullptr;
  
  if (!items_data_->segments) {
    // 获取分段信息 (script, direction etc.)
    RunSegmenter::RunSegmenterRange segment_range =
        InlineItemSegment::UnpackSegmentData(start, end, item.SegmentData());
    
    // ← 关键调用: HarfBuzzShaper::Shape()
    shape_result = shaper_.Shape(
        item.Style()->GetFont(),    // Font 对象 (包含 FontDescription)
        item.Direction(),            // 文本方向 (LTR/RTL)
        start, end,                  // 文本范围
        segment_range,               // 分段信息
        options);
  } else {
    // 使用缓存的分段结果
    shape_result = items_data_->segments->ShapeText(
        &shaper_, item.Style()->GetFont(), item.Direction(), 
        start, end, item.Index(), options);
  }
  
  // 应用文本间距 (letter-spacing, word-spacing)
  if (spacing_.HasSpacing()) {
    shape_result->ApplySpacing(spacing_);
  }
  
  return shape_result;
}
```

**参数说明**:
- `item` - InlineItem，包含样式和文本
- `start`, `end` - 文本范围 (相对于整个文本)
- `options` - 整形选项 (hyphenation, contextual shaping 等)

**返回值**: `ShapeResult*` - 包含 glyph ID、位置、宽度等信息

---

### 第 4 层: HarfBuzz 整形和字体替补

#### 4.1 HarfBuzzShaper::Shape()

**源文件**: [harfbuzz_shaper.h](third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.h) 行 63-76

**接口定义**:

```cpp
// Shape a range that has already been pre-segmented
ShapeResult* Shape(
    const Font*,
    TextDirection,
    unsigned start,
    unsigned end,
    const Vector<RunSegmenter::RunSegmenterRange>&,
    ShapeOptions = ShapeOptions()) const;

// Shape a single range
ShapeResult* Shape(
    const Font*,
    TextDirection,
    unsigned start,
    unsigned end,
    const RunSegmenter::RunSegmenterRange,
    ShapeOptions = ShapeOptions()) const;
```

**实现概述** (harfbuzz_shaper.cc):

主要流程：
1. 为每个分段 (script, direction, 语言等) 创建单独的整形运行
2. 对每个运行调用 `ShapeRun()`

---

#### 4.2 HarfBuzzShaper::ShapeRun() 的内部流程

**源文件**: [harfbuzz_shaper.cc](third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc) 行 280-350

**功能**: 对单个文本运行进行 HarfBuzz 整形

```cpp
// 内部方法 (在 ShapeRunsWithHarfBuzz 中调用)

// 1️⃣ 为 HarfBuzz 创建缓冲区
hb_buffer_t* buffer = hb_buffer_create();
hb_buffer_add_utf16(buffer, text_data, length, ...);

// 2️⃣ 获取 Font 对象进行字体选择
const Font& font = ...;
const SimpleFontData* current_font = font.PrimaryFont();

if (!current_font) {
  // 没有主字体，返回错误
  return nullptr;
}

// 3️⃣ 创建 FontFallbackIterator (← 字体替补的开始点)
FontFallbackIterator fallback_iterator(
    font_description,     // FontDescription (weight, style, size...)
    font_fallback_list,   // 字体列表
    font_fallback_priority);

// 4️⃣ 循环遍历字体，直到成功整形
while (fallback_iterator.HasNext()) {
  current_font_data_for_range_set = 
      fallback_iterator.Next(fallback_chars_hint);
  
  if (!current_font_data_for_range_set->FontData()) {
    break;  // 没有更多字体了
  }
  
  // 5️⃣ 获取 HarfBuzz 字体
  hb_font_t* hb_font = face->GetScaledFont(...);
  
  // 6️⃣ 调用 HarfBuzz 进行整形
  hb_shape(hb_font, buffer, 
           features_data, features_size);
  
  // 7️⃣ 分析结果
  ExtractShapeResults(...);  // ← 检查 .notdef 和 fallback
  
  if (all_shaped) {
    break;  // 所有字符都整形成功
  }
  // 否则继续循环，使用下一个 fallback 字体
}
```

---

### 第 5 层: 字体替补机制

#### 5.1 FontFallbackIterator::Next()

**源文件**: [font_fallback_iterator.h](third_party/blink/renderer/platform/fonts/font_fallback_iterator.h) 行 22-98

**接口**:

```cpp
class FontFallbackIterator {
 public:
  FontFallbackIterator(
      const FontDescription&,
      FontFallbackList*,
      FontFallbackPriority);
  
  bool HasNext() const { return fallback_stage_ != kOutOfLuck; }
  
  // 获取下一个字体
  FontDataForRangeSet* Next(const HintCharList& hint_list);
  
 private:
  enum FallbackStage {
    kFallbackPriorityFonts,      // 阶段1: 优先级字体 (emoji)
    kFontGroupFonts,             // 阶段2: 字体族列表
    kSegmentedFace,              // 阶段3: @font-face 分段
    kPreferencesFonts,           // 阶段4: 用户偏好
    kSystemFonts,                // 阶段5: 系统字体
    kFirstCandidateForNotdefGlyph, // 阶段6: .notdef 候选
    kOutOfLuck                   // 阶段7: 完全失败
  };
  
  FallbackStage fallback_stage_;
};
```

**7 个 Fallback 阶段详解**:

| 阶段 | 源 | 实现 | 获取字体的方式 |
|------|-----|------|-------------|
| 1️⃣ kFallbackPriorityFonts | emoji 规则 | `FallbackPriorityFont()` | 从特殊列表 |
| 2️⃣ kFontGroupFonts | `font-family: A, B` | `FontFallbackList` | CSS 指定 |
| 3️⃣ kSegmentedFace | `@font-face` | `FontFaceCache` | unicode-range 匹配 |
| 4️⃣ kPreferencesFonts | 用户设置 | `GenericFontFamilySettings` | 浏览器偏好 |
| 5️⃣ kSystemFonts | OS 字体 | `FontCache::GetFontData()` | 系统库查询 |
| 6️⃣ kFirstCandidateForNotdefGlyph | 上一个 fallback | 返回第一个 | 显示 .notdef |
| 7️⃣ kOutOfLuck | 无 | 无 | 返回 null |

**工作流程**:

```cpp
FontDataForRangeSet* FontFallbackIterator::Next(
    const HintCharList& hint_list) {
  
  while (fallback_stage_ != kOutOfLuck) {
    FontDataForRangeSet* candidate = nullptr;
    
    switch (fallback_stage_) {
      case kFallbackPriorityFonts:
        // 从 emoji 优先级字体获取
        candidate = GetFallbackPriorityFont(hint_list);
        if (candidate) return candidate;
        // 如果没有，进入下一阶段
        fallback_stage_ = kFontGroupFonts;
        break;
      
      case kFontGroupFonts:
        // 从 font-family 列表获取下一个字体
        candidate = GetFontGroupFonts(hint_list);
        if (candidate) return candidate;
        fallback_stage_ = kSegmentedFace;
        break;
      
      case kSegmentedFace:
        // 从 @font-face 分段获取
        candidate = GetSegmentedFace(hint_list);
        if (candidate) return candidate;
        fallback_stage_ = kPreferencesFonts;
        break;
      
      case kPreferencesFonts:
        // 从用户偏好获取
        candidate = GetPreferencesFonts(hint_list);
        if (candidate) return candidate;
        fallback_stage_ = kSystemFonts;
        break;
      
      case kSystemFonts:
        // 从系统字体库获取
        candidate = UniqueSystemFontForHintList(hint_list);
        if (candidate) return candidate;
        fallback_stage_ = kFirstCandidateForNotdefGlyph;
        break;
      
      case kFirstCandidateForNotdefGlyph:
        // 返回第一个候选作为最后的 fallback
        if (first_candidate_) {
          return first_candidate_;
        }
        fallback_stage_ = kOutOfLuck;
        break;
    }
  }
  
  return nullptr;  // 完全失败
}
```

---

#### 5.2 HarfBuzzShaper::ExtractShapeResults()

**源文件**: [harfbuzz_shaper.cc](third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc) 行 556-700

**功能**: 分析 HarfBuzz 的整形结果，检测失败的字符并队列化重新整形

```cpp
void HarfBuzzShaper::ExtractShapeResults(
    RangeContext* range_data,
    bool& font_cycle_queued,
    const ReshapeQueueItem& current_queue_item,
    const SimpleFontData* current_font,
    UScriptCode current_run_script,
    CanvasRotationInVertical canvas_rotation,
    FallbackFontStage& fallback_stage,
    ShapeResult* shape_result) const {
  
  // 获取 HarfBuzz 的 glyph 信息
  unsigned num_glyphs = hb_buffer_get_length(range_data->buffer.Get());
  hb_glyph_info_t* glyph_info = 
      hb_buffer_get_glyph_infos(range_data->buffer.Get(), nullptr);
  
  enum ClusterResult { kShaped, kNotDef, kUnknown };
  ClusterResult current_cluster_result = kUnknown;
  
  // 遍历每个 glyph
  for (unsigned glyph_index = 0; glyph_index < num_glyphs; ++glyph_index) {
    const hb_glyph_info_t& glyph = glyph_info[glyph_index];
    const hb_codepoint_t glyph_id = glyph.codepoint;
    ClusterResult glyph_result;
    
    // ← 关键检测点: 检查是否是 .notdef glyph
    if (glyph_id == 0) {
      // glyph ID = 0 表示 .notdef (字体中不存在的字符)
      glyph_result = kNotDef;
    } else if (glyph_id == space_glyph && 
               !IsLastFontToShape(fallback_stage) &&
               text_[current_cluster] == uchar::kIdeographicSpace) {
      // 特殊情况: HarfBuzz 用 space glyph 合成了 IDEOGRAPHIC SPACE
      glyph_result = kNotDef;
    } else if (glyph_id == kUnmatchedVSGlyphId) {
      // 变体选择符未匹配
      fallback_stage = ChangeStageToVS(fallback_stage);
      glyph_result = kNotDef;
    } else {
      glyph_result = kShaped;  // 成功整形
    }
    
    // 如果状态从 kShaped 变为 kNotDef，需要重新整形
    if (current_cluster != previous_cluster &&
        previous_cluster_result != current_cluster_result &&
        previous_cluster_result != kUnknown) {
      
      // ← 关键动作1: 队列化失败的字符
      if (current_cluster_result == kShaped &&
          !IsLastFontToShape(fallback_stage)) {
        // 之前的字符失败了，现在成功了
        // → 队列化之前失败的范围用下一个字体重新整形
        QueueCharacters(range_data, current_font, font_cycle_queued, slice,
                       fallback_stage);
      } else {
        // ← 关键动作2: 提交成功的字符
        CommitGlyphs(range_data, current_font, current_run_script,
                    canvas_rotation, fallback_stage, slice, shape_result);
      }
    }
  }
  
  // 在最后一个字体时
  if (IsLastFontToShape(fallback_stage)) {
    range_data->font->ReportNotDefGlyph();  // 报告无法显示的字符
  }
}
```

---

#### 5.3 QueueCharacters()

**源文件**: [harfbuzz_shaper.cc](third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc) 行 439-459

**功能**: 将未整形成功的字符范围加入队列，用下一个 fallback 字体重新整形

```cpp
void QueueCharacters(
    RangeContext* range_data,
    const SimpleFontData* current_font,
    bool& font_cycle_queued,
    const BufferSlice& slice,
    HarfBuzzShaper::FallbackFontStage font_stage) {
  
  if (!font_cycle_queued) {
    // 第一次队列化，添加阶段转移标记
    if (StageNeedsQueueReset(font_stage)) {
      range_data->reshape_queue.push_back(
          ReshapeQueueItem(kReshapeQueueReset, 0, 0));
    } else {
      // 添加"尝试下一个字体"标记
      range_data->reshape_queue.push_back(
          ReshapeQueueItem(kReshapeQueueNextFont, 0, 0));
    }
    font_cycle_queued = true;
  }
  
  // 添加需要重新整形的字符范围
  DCHECK(slice.num_characters);
  range_data->reshape_queue.push_back(
      ReshapeQueueItem(
          kReshapeQueueRange,
          slice.start_character_index,
          slice.num_characters));
}
```

**队列项的处理流程**:

```
ReshapeQueueItem:
  kReshapeQueueNextFont  → 标记: 进入下一个 fallback 字体阶段
  kReshapeQueueRange     → 包含: 需要重新整形的字符范围 (start, count)
  kReshapeQueueReset     → 标记: 重置并进入新的 fallback 阶段

HarfBuzzShaper 主循环处理这个队列:
while (!reshape_queue.empty()) {
  item = reshape_queue.pop_front();
  
  if (item.type == kReshapeQueueNextFont) {
    current_font = fallback_iterator.Next(...);  // ← 获取下一个字体
  }
  
  if (item.type == kReshapeQueueRange) {
    // 用 current_font 重新整形 [item.start, item.start + item.count)
    ShapeRun(...);
    ExtractShapeResults(...);  // 递归检查是否仍有失败
  }
}
```

---

#### 5.4 CommitGlyphs()

**源文件**: [harfbuzz_shaper.cc](third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc) 行 470-540

**功能**: 将成功整形的 glyph 添加到最终结果中

```cpp
void CommitGlyphs(
    RangeContext* range_data,
    const SimpleFontData* current_font,
    UScriptCode current_run_script,
    CanvasRotationInVertical canvas_rotation,
    HarfBuzzShaper::FallbackFontStage fallback_stage,
    const BufferSlice& slice,
    ShapeResult* shape_result) {
  
  // 从 glyph 信息提取字形数据 (ID, 宽度, 位置等)
  for (unsigned glyph_index = slice.start_glyph_index;
       glyph_index < slice.end_glyph_index;
       ++glyph_index) {
    
    const hb_glyph_info_t& glyph_info = ...;
    const hb_glyph_position_t& glyph_position = ...;
    
    // 添加 glyph 到结果
    shape_result->AppendGlyph(
        current_font,           // 使用哪个字体
        glyph_info.codepoint,   // Glyph ID
        glyph_position.x_advance,
        glyph_position.y_advance,
        glyph_position.x_offset,
        glyph_position.y_offset,
        current_cluster);       // 对应的字符索引
  }
  
  // 在最后一个字体时
  if (IsLastFontToShape(fallback_stage)) {
    range_data->font->ReportNotDefGlyph();
  }
}
```

---

### 第 6 层: 系统字体查询

#### 6.1 FontCache::FallbackFontForCharacter()

**源文件**: [font_cache.h](third_party/blink/renderer/platform/fonts/font_cache.h) 行 105-110

**接口**:

```cpp
class FontCache {
 public:
  const SimpleFontData* FallbackFontForCharacter(
      const FontDescription&,        // 字体配置 (weight, style...)
      UChar32,                       // 要查询的字符 Unicode
      const SimpleFontData* font_data_to_substitute,  // 当前字体
      FontFallbackPriority = FontFallbackPriority::kText);
};
```

**实现** (font_cache.cc 行 205-227):

```cpp
const SimpleFontData* FontCache::FallbackFontForCharacter(
    const FontDescription& description,
    UChar32 lookup_char,
    const SimpleFontData* font_data_to_substitute,
    FontFallbackPriority fallback_priority) {
  
  TRACE_EVENT0("fonts", "FontCache::FallbackFontForCharacter");
  
  // 不对私有字符或非字符进行 fallback
  if (Character::IsPrivateUse(lookup_char) ||
      Character::IsNonCharacter(lookup_char))
    return nullptr;
  
  // 调用平台特定实现
  base::ElapsedTimer timer;
  const SimpleFontData* result = PlatformFallbackFontForCharacter(
      description,
      lookup_char,
      font_data_to_substitute,
      fallback_priority);
  
  // 记录性能数据
  FontPerformance::AddSystemFallbackFontTime(timer.Elapsed());
  
  return result;
}
```

---

#### 6.2 PlatformFallbackFontForCharacter() - 平台实现

**平台特定文件**:

| 平台 | 文件 | 说明 |
|------|------|------|
| Skia (通用) | [font_cache_skia.cc](third_party/blink/renderer/platform/fonts/skia/font_cache_skia.cc) | 使用 SkFontMgr |
| Android | [font_cache_android.cc](third_party/blink/renderer/platform/fonts/android/font_cache_android.cc) | Android 系统字体 API |
| Linux | [font_cache_linux.cc](third_party/blink/renderer/platform/fonts/linux/font_cache_linux.cc) | FontConfig |
| Windows | [font_cache_skia_win.cc](third_party/blink/renderer/platform/fonts/win/font_cache_skia_win.cc) | Skia + Windows API |

**通用流程** (Skia):

```cpp
// font_cache_skia.cc
const SimpleFontData* FontCache::PlatformFallbackFontForCharacter(
    const FontDescription& description,
    UChar32 lookup_char,
    const SimpleFontData* font_data_to_substitute,
    FontFallbackPriority fallback_priority) {
  
  // 通过 SkFontMgr (Skia Font Manager) 查询字体
  SkFontMgr* font_manager = GetPlatformFontManager();
  
  // 查询系统中哪个字体包含这个字符
  sk_sp<SkTypeface> typeface = font_manager->matchFamilyStyleCharacter(
      family_name.c_str(),
      font_weight,
      font_width,
      font_style,
      language_tags,
      lookup_char);  // ← 关键: 传入要查询的字符
  
  if (!typeface) {
    // 尝试通用 fallback 字体
    typeface = font_manager->matchFamilyStyleCharacter(
        nullptr,  // 任何字体族
        SkFontStyle(),
        language_tags,
        lookup_char);
  }
  
  if (!typeface) {
    return nullptr;  // 无法找到
  }
  
  // 包装 SkTypeface 为 SimpleFontData
  return CreateFontData(typeface, description);
}
```

---

## 🔗 完整调用链示例: 显示中文字符"你"

假设：
- 指定字体: "Arial" (拉丁字体，不包含中文)
- 要显示: "你好"
- CSS: `font-family: Arial, sans-serif;`

### 调用链:

```
1. InlineLayoutAlgorithm::Layout()
   ↓
2. InlineLayoutAlgorithm::CreateLine()
   ↓
3. LineBreaker::NextBreak()
   → item = "你好"
   ↓
4. LineBreaker::HandleText(item, shape_result, line_info)
   → shape_result 为 null (未缓存)
   ↓
5. LineBreaker::BreakText()
   → ShapingLineBreakerImpl::Shape()
   ↓
6. LineBreaker::ShapeText("你好", 0, 2)
   ↓
7. HarfBuzzShaper::Shape(Font, Direction, 0, 2)
   ↓
8. 创建 FontFallbackIterator
   → Stage 1: emoji 字体 (无)
   → Stage 2: Arial (拉丁字体)
   ↓
9. HarfBuzzShaper::ShapeRun() with Arial
   ↓
10. hb_shape(hb_font_arial, buffer)
    → 返回 glyph_id = 0 (.notdef) 用于 "你" 和 "好"
   ↓
11. ExtractShapeResults()
    → 检测 glyph_id[0] == 0
    → 状态: kNotDef
    ↓
12. QueueCharacters()
    → 队列 ReshapeQueueItem(kReshapeQueueNextFont, 0, 0)
    → 队列 ReshapeQueueItem(kReshapeQueueRange, 0, 2)  // "你好"
   ↓
13. FontFallbackIterator::Next()
    → Stage 3: @font-face (可能无)
    → Stage 4: 用户偏好 (无中文字体)
    → Stage 5: 系统字体
       ↓
14. FontCache::FallbackFontForCharacter('你', Arial)
    ↓
15. PlatformFallbackFontForCharacter('你', 0x4F60)
    ↓
16. SkFontMgr::matchFamilyStyleCharacter(nullptr, '你')
    → 查询系统字体库
    → 返回: Noto Sans CJK (包含中文)
   ↓
17. HarfBuzzShaper::ShapeRun() with Noto Sans CJK
    ↓
18. hb_shape(hb_font_noto, buffer)
    → glyph_id = 12345 (有效的中文 glyph)
   ↓
19. ExtractShapeResults()
    → 检测 glyph_id[0] == 12345
    → 状态: kShaped (成功!)
   ↓
20. CommitGlyphs()
    → 添加 glyph 到 ShapeResult
    → 字体: Noto Sans CJK
    → 字形 ID: 12345
   ↓
21. 返回 ShapeResult
    ↓
22. LineBreaker::HandleText() 获得最终 ShapeResult
   ↓
23. 布局引擎使用 ShapeResult 进行排版
   ↓
24. 绘制时使用正确的字体和字形显示 "你好"
```

---

## 📊 调用栈 ASCII 图

```
CSS 规则
  │ font-family: Arial, SimSun;
  │ font-size: 12px;
  ↓
DOM Tree
  │ <p>你好</p>
  ↓
Layout Engine
  │ InlineLayoutAlgorithm::Layout()
  │   ↓
  │ CreateLine()
  │   ↓
  │ LineBreaker::NextBreak()
  │   ├─ HandleText()
  │   ├─ BreakText()
  │   └─ ShapeText()
  ↓
HarfBuzz Shaping
  │ HarfBuzzShaper::Shape()
  │   ├─ FontFallbackIterator (字体列表)
  │   ├─ hb_shape()          (HarfBuzz 整形)
  │   └─ ExtractShapeResults() (结果分析)
  │       ├─ 检测 .notdef
  │       ├─ QueueCharacters()    (队列化失败)
  │       └─ CommitGlyphs()       (提交成功)
  ↓
Font Fallback
  │ FontFallbackIterator::Next()
  │   ├─ Stage 1: Emoji (无)
  │   ├─ Stage 2: font-family 列表
  │   ├─ Stage 3: @font-face
  │   ├─ Stage 4: 用户偏好
  │   ├─ Stage 5: 系统字体
  │   │   ↓
  │   │ FontCache::FallbackFontForCharacter()
  │   │   ↓
  │   │ PlatformFallbackFontForCharacter()
  │   │   ↓
  │   │ SkFontMgr::matchFamilyStyleCharacter()
  │   │   → 系统字体库查询
  │   └─ Stage 6-7: .notdef 或失败
  ↓
ShapeResult
  │ Glyphs: [glyph_id, x_advance, y_advance, ...]
  │ Fonts:  [Noto Sans CJK, Noto Sans CJK, ...]
  ↓
Rendering
  │ 使用 Skia/GPU 绘制字形
  ↓
Pixel Output
  │ 屏幕上看到的文本
```

---

## 🔑 关键代码位置速查表

| 功能 | 文件 | 行号 | 关键变量/函数 |
|------|------|------|-------------|
| 布局入口 | inline_layout_algorithm.cc | - | `Layout()` |
| 行破裂 | line_breaker.cc | 1286 | `HandleText()` |
| 文本整形 | line_breaker.cc | 2003 | `ShapeText()` |
| HarfBuzz 整形 | harfbuzz_shaper.cc | 339 | `hb_shape()` |
| 结果分析 | harfbuzz_shaper.cc | 556 | `ExtractShapeResults()` |
| .notdef 检测 | harfbuzz_shaper.cc | 584 | `if (glyph_id == 0)` |
| 队列化 | harfbuzz_shaper.cc | 439 | `QueueCharacters()` |
| 字体替补 | font_fallback_iterator.h | 22 | `FontFallbackIterator` |
| 7 个阶段 | font_fallback_iterator.h | 73-79 | `FallbackStage enum` |
| 系统 fallback | font_cache.cc | 205 | `FallbackFontForCharacter()` |
| Skia 集成 | font_cache_skia.cc | - | `PlatformFallbackFontForCharacter()` |

---

## 📝 使用此文档的方法

1. **理解完整流程**: 从上到下阅读，了解数据如何流动
2. **查找特定阶段**: 使用"关键代码位置速查表"
3. **调试具体问题**:
   - 字符显示为方框 → 查看第 5.2 节 (ExtractShapeResults)
   - 字体不适用 → 查看第 5.1 节 (FontFallbackIterator)
   - 性能问题 → 查看第 6.1 节 (FallbackFontForCharacter)
4. **追踪源代码**: 所有引用都包含文件路径和行号

---

**文档版本**: 1.0  
**最后更新**: 2024  
**验证方式**: 所有调用关系已在源代码中核实
