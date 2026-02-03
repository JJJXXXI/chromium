# Chromium/Blink 字体渲染完整分析报告

## 目录
1. [总体流程图](#总体流程图)
2. [字体匹配流程（网络字体 vs 非网络字体）](#字体匹配流程)
3. [fallback 和 run splitting](#fallback-和-run-splitting)
4. [平台差异](#平台差异)
5. [代码证据和关键实现](#代码证据)

---

## 总体流程图

```
CSS 文本 (font-family, font-weight, font-style, text)
   ↓
FontDescription 解析 (CSS 字体属性)
   ↓
Font::EnsureFontFallbackList()
   ├→ FontFallbackList 构建
   └→ FontSelector::GetFontData()
      ↓
   font-family 列表逐个匹配:
      ├─ @font-face (网络字体) with unicode-range
      ├─ local() 字体名 (系统字体)
      └─ generic family (sans-serif/serif/monospace → 系统字体映射)
      ↓
   FontFallbackIterator (逐字符回退)
      ├→ kFallbackPriorityFonts (优先字体)
      ├→ kFontGroupFonts (字体组)
      ├→ kSegmentedFace (分段字体)
      ├→ kPreferencesFonts (偏好字体)
      └→ kSystemFonts (系统回退)
         ↓
   HarfBuzzShaper::ShapeSegment()
      ├─ RunSegmenter 分割 (脚本/方向)
      │  ↓
      ├─ FontFallbackIterator::Next(hint_chars)
      │  ├→ 检查 unicode-range::Contains(char)
      │  └→ 获取 FontDataForRangeSet
      │     ↓
      ├─ 按字符覆盖检查 (glyph 存在性)
      │  └→ SimpleFontData::GlyphForCharacter(codepoint)
      │     ↓
      ├─ 切分 run:
      │  ├→ 字符覆盖变化 → 新 font → 新 run
      │  └→ 脚本变化 → 新 run
      │     ↓
      └─ HarfBuzz shaping (每个 run 一个 font)
         ↓
         ShapeResult + ShapeResultRun (font per run)
            ↓
         渲染 (Skia drawing with per-run font)
```

---

## 字体匹配流程

### 1. font-family 列表匹配流程

**关键文件**: 
- [third_party/blink/renderer/platform/fonts/font_fallback_list.cc](third_party/blink/renderer/platform/fonts/font_fallback_list.cc)
- [third_party/blink/renderer/core/css/css_segmented_font_face.cc](third_party/blink/renderer/core/css/css_segmented_font_face.cc)

**核心流程**：
```
FontFallbackList::GetFontData(const FontDescription& font_description)
  ↓
FontSelector::GetFontForGenericFamily()  // 对每个 font-family 逐个调用
  ├─ 如果是 @font-face: CSSSegmentedFontFace::GetFontData()
  ├─ 如果是 generic family (serif): GenericFontFamilySettings::Serif()
  └─ 如果是普通名字: FontCache::GetFontData()
```

**证据代码** - [font_fallback_list.cc:149-160](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/platform/fonts/font_fallback_list.cc):

```cpp
const FontData* FontFallbackList::GetFontData(
    const FontDescription& font_description) {
  DCHECK(font_selector_);
  // 对每个 font-family 逐个获取字体数据
  for (int i = font_description.GenericFamily();
       i != kCAllFamiliesScanned;
       i = font_description.NextFamily(i)) {
    const FontData* result = font_selector_->GetFontData(
        font_description, 
        font_description.FamilyAt(i));
    if (result)
      return result;
  }
  return nullptr;
}
```

**说明**：FontFallbackList 迭代 font-family 列表中的每个 family，调用 FontSelector 查询。FontSelector 根据 family 名称的类型进行不同的处理。

---

### 2. @font-face 规则与 unicode-range 参与匹配

**关键文件**：
- [third_party/blink/renderer/core/css/css_segmented_font_face.cc:100-145](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/core/css/css_segmented_font_face.cc)
- [third_party/blink/renderer/platform/fonts/font_data_for_range_set.h](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/platform/fonts/font_data_for_range_set.h)

**匹配流程**：

```cpp
// CSSSegmentedFontFace::GetFontData() - 返回 SegmentedFontData
const FontData* CSSSegmentedFontFace::GetFontData(
    const FontDescription& font_description) {
  // ... 省略缓存逻辑 ...
  
  SegmentedFontData* created_font_data = 
      MakeGarbageCollected<SegmentedFontData>();

  // 反向迭代 font-face 列表（后定义的覆盖先定义的）
  font_faces_->ForEachReverse([&requested_font_description, 
                                &created_font_data](
                                  const Member<FontFace>& font_face) {
    if (!font_face->CssFontFace()->IsValid()) {
      return;
    }
    if (const SimpleFontData* face_font_data =
            font_face->CssFontFace()->GetFontData(requested_font_description)) {
      // 关键：结合 unicode-range
      created_font_data->AppendFace(
          MakeGarbageCollected<FontDataForRangeSet>(
              std::move(face_font_data),  // 字体数据
              font_face->CssFontFace()->Ranges()));  // unicode-range
    }
  });

  return created_font_data;  // 返回 SegmentedFontData，包含多个 FontDataForRangeSet
}
```

**unicode-range 检查** - [font_data_for_range_set.h](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/platform/fonts/font_data_for_range_set.h):

```cpp
class FontDataForRangeSet : public GarbageCollected<FontDataForRangeSet> {
 public:
  bool Contains(UChar32 test_char) const {
    // 如果没有 unicode-range 限制，返回 true（包含所有字符）
    // 否则检查 test_char 是否在 range_set 中
    return !range_set_ || range_set_->Contains(test_char);
  }

 private:
  Member<const SimpleFontData> font_data_;
  Member<const UnicodeRangeSet> range_set_;  // 来自 @font-face 的 unicode-range
};
```

**说明**：
- 每个 @font-face 规则对应一个 `FontDataForRangeSet`
- `unicode-range` 被解析为 `UnicodeRangeSet`，限制该字体的适用范围
- 在字符覆盖检查时，使用 `Contains()` 判断字符是否在该字体的 unicode-range 内

---

### 3. local() 字体名解析

**关键文件**：
- [third_party/blink/renderer/core/css/local_font_face_source.cc](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/core/css/local_font_face_source.cc)

**流程**：

```cpp
class LocalFontFaceSource : public FontFaceSource {
  // @font-face { src: local("Font Name") }
  // 其中 font_name_ 为 "Font Name"
  
  const SimpleFontData* CreateFontData(
      const FontDescription& font_description,
      const FontSelectionCapabilities& font_selection_capabilities) {
    
    // 检查本地字体是否可用
    bool font_available = 
        FontCache::Get().IsPlatformFontUniqueNameMatchAvailable(
            font_description, font_name_);  // 平台特定实现
    
    if (!font_available) {
      return nullptr;  // local() 字体不存在，尝试下一个 src
    }

    // 查询 FontCache 获取本地字体
    const SimpleFontData* unique_lookup_result = 
        FontCache::Get().GetFontData(
            unstyled_description, 
            font_name_,
            AlternateFontName::kLocalUniqueFace);  // 本地字体查询标志

    // ... 应用 font-variation-settings 等 ...
    return font_data_variations_palette_applied;
  }
};
```

**说明**：
- `local()` 中的字体名在 `src` 属性中被解析
- 在 `LocalFontFaceSource::CreateFontData()` 中调用 `FontCache::Get().IsPlatformFontUniqueNameMatchAvailable()` 检查字体是否存在
- 最终通过 `FontCache::Get().GetFontData()` 获取本地字体数据（使用 `AlternateFontName::kLocalUniqueFace` 标志）

---

### 4. 系统字体（generic family）映射

**关键文件**：
- [third_party/blink/renderer/platform/fonts/generic_font_family_settings.h](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/platform/fonts/generic_font_family_settings.h)
- [third_party/blink/renderer/platform/fonts/generic_font_family_settings.cc](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/platform/fonts/generic_font_family_settings.cc)

**generic family 类型**：
```cpp
enum GenericFamilyType {
  kGenericFamilyNone,
  kGenericFamilyStandard,      // sans-serif (默认)
  kGenericFamilyFixed,          // monospace
  kGenericFamilyMonospace,       // monospace
  kGenericFamilySerif,          // serif
  kGenericFamilySansSerif,      // sans-serif
  kGenericFamilyCursive,        // cursive
  kGenericFamilyFantasy,        // fantasy
  kGenericFamilySystemUi,       // system-ui
  // ...其他类型...
};
```

**映射实现** - [generic_font_family_settings.cc:170-195](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/platform/fonts/generic_font_family_settings.cc):

```cpp
const AtomicString& GenericFontFamilySettings::Serif(
    UScriptCode script) const {
  // 根据脚本（如 USCRIPT_LATIN, USCRIPT_HAN 等）
  // 返回对应的 serif 字体名
  return GenericFontFamilyForScript(serif_font_family_map_, script);
}

const AtomicString& GenericFontFamilySettings::SansSerif(
    UScriptCode script) const {
  return GenericFontFamilyForScript(sans_serif_font_family_map_, script);
}

// 脚本相关的映射
// sans-serif → "Roboto" (Android)
// sans-serif → "Segoe UI" (Windows)
// sans-serif → "Helvetica" (macOS)
// serif → "Droid Serif" (Android)
// serif → "Georgia" (Windows)
// monospace → "Droid Sans Mono" (Android)
```

**说明**：
- generic family（如 `sans-serif`）通过 `GenericFontFamilySettings` 映射到具体的平台字体名
- 映射与脚本相关，不同脚本的 generic family 可能映射到不同的字体
- 这个映射是配置化的，可以通过 `GenericFontFamilySettings::UpdateSerif()` 等方法修改

---

## fallback 和 run splitting

### 1. 字符覆盖检查（Glyph 存在性检查）

**关键函数**：

#### a) `SimpleFontData::GlyphForCharacter()`
**文件**: [third_party/blink/renderer/platform/fonts/simple_font_data.cc:250-265](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/platform/fonts/simple_font_data.cc)

```cpp
Glyph SimpleFontData::GlyphForCharacter(UChar32 codepoint) const {
  // 核心：查询字体是否包含该字符的 glyph
  const HarfBuzzFace* harfbuzz_face = 
      PlatformData().GetHarfBuzzFace();
  if (!harfbuzz_face) {
    return 0;  // 无有效字体，返回 0（表示 .notdef）
  }
  
  // 通过 HarfBuzz 查询 glyph ID
  return harfbuzz_face->HbGlyphForCharacter(codepoint);
}
```

**说明**：
- 这是检查字体是否包含某字符的最基础接口
- 返回 glyph ID（0 表示 .notdef glyph，即字符不被该字体支持）
- 直接调用 HarfBuzz 的 glyph 查询

#### b) `FontDataForRangeSet::Contains()`
**文件**: [third_party/blink/renderer/platform/fonts/font_data_for_range_set.h](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/platform/fonts/font_data_for_range_set.h)

```cpp
class FontDataForRangeSet {
 public:
  bool Contains(UChar32 test_char) const {
    // 检查字符是否在 unicode-range 内
    // 对于没有 unicode-range 的字体，总是返回 true
    return !range_set_ || range_set_->Contains(test_char);
  }

  bool HasFontData() const { return font_data_; }
  const SimpleFontData* FontData() const { return font_data_.Get(); }
};
```

**说明**：
- 在 HarfBuzz Shaper 中，对每个 run 检查字符是否在 `FontDataForRangeSet` 的 unicode-range 内
- 这是在 fallback 迭代中进行的第一级过滤

---

### 2. fallback 机制（HarfBuzzShaper 中的多层次回退）

**关键文件**：
- [third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc:865-960](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc)

**fallback 阶段**：

```cpp
enum FallbackFontStage {
  kIntermediate,         // 第一轮，尝试指定的字体列表
  kLast,                 // 最后一个字体（.notdef 字形）
  kIntermediateWithVS,   // 重新尝试（支持 Variation Selector）
  kLastWithVS,           // 最后尝试（支持 Variation Selector）
  kIntermediateIgnoreVS, // 忽略 Variation Selector 重试
  kLastIgnoreVS,         // 最后尝试（不支持 Variation Selector）
};
```

**fallback 流程** - [harfbuzz_shaper.cc:902-935](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc):

```cpp
void HarfBuzzShaper::ShapeSegment(
    RangeContext* range_data,
    const RunSegmenter::RunSegmenterRange& segment,
    ShapeResult* result) const {
  // 1. 创建 FontFallbackIterator，用于逐字体尝试
  FontFallbackIterator fallback_iterator(
      font->CreateFontFallbackIterator(...));

  // 2. 初始化 reshape queue（重塑队列）
  range_data->reshape_queue.push_back(
      ReshapeQueueItem(kReshapeQueueNextFont, 0, 0));
  range_data->reshape_queue.push_back(ReshapeQueueItem(
      kReshapeQueueRange, segment.start, segment.end - segment.start));

  // 3. fallback 主循环
  while (!range_data->reshape_queue.empty()) {
    ReshapeQueueItem current_queue_item = 
        range_data->reshape_queue.TakeFirst();

    // 3.1 如果不是范围项，则获取下一个 fallback 字体
    if (current_queue_item.action_ != kReshapeQueueRange) {
      // 调用 FontFallbackIterator::Next() 获取下一个字体
      current_font_data_for_range_set = 
          fallback_iterator.Next(fallback_chars_hint);
      
      if (!current_font_data_for_range_set->FontData()) {
        // 已用尽所有 fallback 字体
        break;
      }
      continue;
    }

    // 3.2 检查是否还有更多 fallback 字体
    if (!fallback_iterator.HasNext()) {
      // 转到最后阶段（将使用 .notdef 字形）
      fallback_stage = ChangeStageToLast(fallback_stage);
    }

    // 3.3 尝试用当前字体 shape
    const SimpleFontData* font_data = 
        current_font_data_for_range_set->FontData();
    
    // 3.4 调用 HarfBuzz shaping
    if (!ShapeRange(range_data->buffer, /* ... */, font_data)) {
      // shaping 失败，继续 fallback
      QueueCharacters(range_data, font_data, font_cycle_queued,
                      slice, font_stage);
      continue;
    }

    // 3.5 处理 shaping 结果，检查是否有 .notdef 字形
    ExtractShapeResults(range_data, font_cycle_queued,
                        current_queue_item, ...);
  }
}
```

**说明**：
- fallback 通过 `FontFallbackIterator` 逐个尝试字体
- 每次尝试失败（有 .notdef 字形），就将该字符范围加入 reshape queue，准备用下一个字体重试
- `fallback_stage` 用来跟踪是否已用尽所有 fallback 字体

---

### 3. run splitting（按字符覆盖切分 run）

**发生位置**：**在 HarfBuzz shaping 之前**

**关键文件**：
- [third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc:500-600](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc)

**run splitting 的关键代码**：

```cpp
// 在 ExtractShapeResults() 中检查 .notdef 字形
void HarfBuzzShaper::ExtractShapeResults(
    RangeContext* range_data,
    bool& font_cycle_queued,
    const ReshapeQueueItem& current_queue_item,
    const hb_glyph_info_t* glyph_info,
    unsigned num_glyphs,
    ...) const {
  
  // 遍历 HarfBuzz 输出的每个字形
  for (unsigned i = 0; i < num_glyphs; ++i) {
    hb_glyph_info_t info = glyph_info[i];
    
    // 如果遇到 .notdef 字形（glyph = 0）
    if (info.codepoint == 0) {
      // 标记该字符范围需要用下一个 fallback 字体重新 shape
      // 这会导致：
      // 1. 在 reshape queue 中添加该字符范围和下一个字体
      // 2. 最终该字符将由不同的字体处理，形成新的 run
      QueueCharacters(range_data, current_font, font_cycle_queued, 
                      slice_with_notdef, font_stage);
    }
  }
}
```

**run 创建** - [harfbuzz_shaper.cc:505-535](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc):

```cpp
void HarfBuzzShaper::CommitGlyphs(
    RangeContext* range_data,
    const SimpleFontData* current_font,
    UScriptCode current_run_script,
    CanvasRotationInVertical canvas_rotation,
    FallbackFontStage fallback_stage,
    const BufferSlice& slice,
    ShapeResult* shape_result) const {
  
  // 为本次 shaping 结果创建 ShapeResultRun
  auto* run = MakeGarbageCollected<ShapeResultRun>(
      current_font,  // 这个 run 的字体
      direction,
      canvas_rotation,
      script,
      run_start_index,
      current_slice->num_glyphs,
      current_slice->num_characters);
  
  // 插入到 ShapeResult
  shape_result->InsertRun(run, ...);
}
```

**说明**：
- **run splitting 发生在 HarfBuzz shaping 之前和之后**
  - **之前**：按脚本/方向分割（由 `RunSegmenter` 完成）
  - **之后**：按字符覆盖检查分割（由 HarfBuzz 输出 .notdef 字形来驱动）
- **关键机制**：当 HarfBuzz 返回 .notdef 字形时，Chromium 检测到该字符无法用当前字体渲染，将其添加到 reshape queue，用下一个 fallback 字体重试
- **结果**：最终形成多个 `ShapeResultRun`，每个 run 对应一个字体

---

### 4. HarfBuzz 在整个流程中的角色

**HarfBuzz 的职责**：
- ✅ **负责的**：
  - 对给定的字体和字符序列进行 shaping
  - 输出 glyph sequence 和位置信息
  - 报告 .notdef 字形（glyph ID = 0）

- ❌ **不负责的**：
  - 字体选择（font fallback）
  - 按字符覆盖的 run splitting
  - unicode-range 检查

**HarfBuzz 调用**：

```cpp
// 在 ShapeRange() 中调用 HarfBuzz
inline bool ShapeRange(hb_buffer_t* buffer,
                       const FontFeatureRanges& font_features,
                       const SimpleFontData* current_font,
                       const UnicodeRangeSet* current_font_range_set,
                       UScriptCode current_run_script,
                       hb_direction_t direction,
                       hb_language_t language,
                       float specified_size) {
  
  const HarfBuzzFace* face = 
      current_font->PlatformData().GetHarfBuzzFace();
  hb_font_t* hb_font = face->GetScaledFont(...);
  
  // === 关键调用 ===
  hb_shape(hb_font, buffer,
           FontFeatureRange::ToHarfBuzzData(argument_features.data()),
           argument_features.size());
  // === 调用后处理 glyph 位置 ===
  
  return true;
}
```

---

## 平台差异

### Android 特定实现

**关键文件**：
- [third_party/blink/renderer/platform/fonts/linux/font_unique_name_lookup_android.cc](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/platform/fonts/linux/font_unique_name_lookup_android.cc)
- [third_party/blink/renderer/platform/fonts/android/font_unique_name_lookup_android.cc](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/platform/fonts/android/font_unique_name_lookup_android.cc)

**特点**：
- 使用 `IsPlatformFontUniqueNameMatchAvailable()` 检查 local() 字体
- 系统字体映射通过 `GenericFontFamilySettings` 进行
- fallback 链：@font-face → local() → generic family → 系统字体

### Linux 特定实现

**关键文件**：
- [third_party/blink/renderer/platform/fonts/linux/font_cache_linux.cc](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/platform/fonts/linux/font_cache_linux.cc)

**特点**：
- 使用 FontConfig 进行字体查询
- 系统字体映射更灵活（通过 FontConfig）

### Windows/macOS 特定实现

- Windows：使用 DirectWrite API
- macOS：使用 CoreText API

---

## 代码证据

### 关键类和函数总表

| 类/函数 | 文件 | 行号 | 职责 |
|--------|------|------|------|
| `Font` | `third_party/blink/renderer/platform/fonts/font.h` | 59-70 | 字体对象，包含 FontFallbackList |
| `FontFallbackList` | `third_party/blink/renderer/platform/fonts/font_fallback_list.h` | 30-45 | 缓存字体查询结果 |
| `FontFallbackIterator` | `third_party/blink/renderer/platform/fonts/font_fallback_iterator.h` | 20-60 | 逐字体 fallback 迭代 |
| `FontDataForRangeSet` | `third_party/blink/renderer/platform/fonts/font_data_for_range_set.h` | 35-60 | 字体 + unicode-range 包装 |
| `SimpleFontData` | `third_party/blink/renderer/platform/fonts/simple_font_data.h` | 75-120 | 单个字体实例，包含 glyph 查询 |
| `CSSSegmentedFontFace` | `third_party/blink/renderer/core/css/css_segmented_font_face.cc` | 85-140 | @font-face 集合管理 |
| `LocalFontFaceSource` | `third_party/blink/renderer/core/css/local_font_face_source.cc` | 25-100 | local() 字体源 |
| `GenericFontFamilySettings` | `third_party/blink/renderer/platform/fonts/generic_font_family_settings.cc` | 170-195 | generic family 映射 |
| `HarfBuzzShaper` | `third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.h` | 45-80 | 调用 HarfBuzz shaping |
| `ShapeResult` | `third_party/blink/renderer/platform/fonts/shaping/shape_result.h` | 60-100 | shaping 结果 |
| `ShapeResultRun` | `third_party/blink/renderer/platform/fonts/shaping/shape_result_run.h` | 55-70 | 单个 run（单字体字符序列） |

---

### 完整调用链证据

#### 1. CSS 文本到 FontFallbackList

```cpp
// 在 LayoutObject 渲染时调用
const Font& font = GetStyle().GetFont();
font.EnsureFontFallbackList();  // 触发字体列表构建

// Font::EnsureFontFallbackList() [font.cc:72-75]
FontFallbackList* Font::EnsureFontFallbackList() const {
  if (!font_fallback_list_) {
    font_fallback_list_ =
        GetOrCreateFontFallbackList(font_description_, GetFontSelector());
  }
  return font_fallback_list_;
}

// GetOrCreateFontFallbackList() [font.cc:50-60]
FontFallbackList* GetOrCreateFontFallbackList(
    const FontDescription& font_description,
    FontSelector* font_selector) {
  // 通过 FontFallbackMap 获取或创建
  return font_fallback_map->Get(font_description);
}
```

#### 2. font-family 列表逐个匹配

```cpp
// CSSFontSelector::GetFontData() [css_font_selector.cc]
const FontData* CSSFontSelector::GetFontData(
    const FontDescription& font_description,
    const AtomicString& family_name) {
  // 根据 family_name 的类型进行不同处理
  
  if (IsCSSFontFaceFamily(family_name)) {
    // @font-face 字体
    const CSSSegmentedFontFace* face = 
        FindCSSFontFace(family_name);
    if (face) {
      return face->GetFontData(font_description);
    }
  } else if (IsFamilyKeyword(family_name)) {
    // generic family: serif, sans-serif, monospace, etc.
    const AtomicString& resolved_family_name = 
        font_description.GenericFamilyName(family_name);
    // 调用 FontCache 获取对应的平台字体
    return FontCache::Get().GetFontData(font_description, 
                                        resolved_family_name);
  } else {
    // 普通字体名
    return FontCache::Get().GetFontData(font_description, family_name);
  }
}
```

#### 3. unicode-range 在 fallback 中的应用

```cpp
// FontFallbackIterator::Next() [font_fallback_iterator.cc:100-150]
FontDataForRangeSet* FontFallbackIterator::Next(
    const HintCharList& hint_list) {
  
  // 在循环中遍历字体数据
  while (HasNext()) {
    FontDataForRangeSet* current = GetNextFont();  // 获取下一个字体
    
    // 检查 unicode-range
    if (!RangeSetContributesForHint(hint_list, current)) {
      // 该字体的 unicode-range 不包含任何 hint 字符，跳过
      continue;
    }
    
    return current;  // 返回符合条件的 FontDataForRangeSet
  }
  
  return nullptr;  // 没有更多字体
}

// RangeSetContributesForHint() [font_fallback_iterator.cc]
bool FontFallbackIterator::RangeSetContributesForHint(
    const HintCharList& hint_list,
    const FontDataForRangeSet* font_data_for_range_set) {
  
  for (UChar32 hint_char : hint_list) {
    // 检查 hint 字符是否在这个字体的 unicode-range 内
    if (font_data_for_range_set->Contains(hint_char)) {
      return true;
    }
  }
  return false;
}
```

#### 4. HarfBuzz shaping 和 fallback

```cpp
// HarfBuzzShaper::ShapeSegment() [harfbuzz_shaper.cc:865-950]
void HarfBuzzShaper::ShapeSegment(
    RangeContext* range_data,
    const RunSegmenter::RunSegmenterRange& segment,
    ShapeResult* result) const {
  
  FontFallbackIterator fallback_iterator(
      font->CreateFontFallbackIterator(...));

  // 主 fallback 循环
  while (!range_data->reshape_queue.empty()) {
    ReshapeQueueItem item = range_data->reshape_queue.TakeFirst();

    // 获取下一个 fallback 字体
    current_font_data_for_range_set = 
        fallback_iterator.Next(fallback_chars_hint);

    const SimpleFontData* font_data = 
        current_font_data_for_range_set->FontData();

    // 调用 HarfBuzz shaping
    if (!ShapeRange(range_data->buffer, ..., font_data)) {
      // 检查结果中的 .notdef 字形
      ExtractShapeResults(range_data, ..., item, ...);
    }
  }
}

// ShapeRange() [harfbuzz_shaper.cc:299-345]
inline bool ShapeRange(hb_buffer_t* buffer,
                       ...,
                       const SimpleFontData* current_font,
                       ...) {
  HarfBuzzFace* face = current_font->PlatformData().GetHarfBuzzFace();
  hb_font_t* hb_font = face->GetScaledFont(...);
  
  // === HarfBuzz shaping 调用 ===
  hb_shape(hb_font, buffer,
           FontFeatureRange::ToHarfBuzzData(...),
           ...);
  
  return true;
}
```

#### 5. run splitting（.notdef 检测）

```cpp
// ExtractShapeResults() [harfbuzz_shaper.cc:550-650]
void HarfBuzzShaper::ExtractShapeResults(
    RangeContext* range_data,
    bool& font_cycle_queued,
    const ReshapeQueueItem& current_queue_item,
    const hb_glyph_info_t* glyph_info,
    unsigned num_glyphs,
    unsigned old_glyph_index,
    unsigned new_glyph_index) const {
  
  hb_glyph_info_t* glyphs = glyph_info;
  
  // 检查是否有 .notdef 字形（glyph ID = 0）
  for (unsigned i = old_glyph_index; i < new_glyph_index; ++i) {
    if (glyphs[i].codepoint == 0) {  // .notdef 字形
      // 该字符在当前字体中无法渲染
      // 计算该字符的范围
      BufferSlice notdef_slice = ComputeSlice(...);
      
      // 将该字符加入 reshape queue，用下一个 fallback 字体重试
      QueueCharacters(range_data, current_font, 
                      font_cycle_queued, notdef_slice, font_stage);
      
      // 这会导致最终形成新的 ShapeResultRun
    }
  }
  
  // 提交成功 shaped 的部分（非 .notdef）
  CommitGlyphs(range_data, current_font, ..., slice, result);
}

// CommitGlyphs() [harfbuzz_shaper.cc:505-545]
void HarfBuzzShaper::CommitGlyphs(
    RangeContext* range_data,
    const SimpleFontData* current_font,
    ...,
    const BufferSlice& slice,
    ShapeResult* shape_result) const {
  
  // 为本次 shaping 结果创建一个 ShapeResultRun
  auto* run = MakeGarbageCollected<ShapeResultRun>(
      current_font,  // 这个 run 的字体！
      direction, ..., num_glyphs, num_characters);
  
  // 插入到 ShapeResult
  shape_result->InsertRun(run, ...);
}
```

---

## 总结：网络字体缺字时为何会发生跨字体 run

### 问题描述
当网络字体不包含某些字符时，Chromium 会从该字符处"切分 run"，用 fallback 字体继续渲染。这会导致：
- **不同字体在同一个单词中**：例如 "Hello中文" 可能用 "Arial" 渲染 "Hello"，用 "Noto Sans CJK" 渲染 "中文"
- **Complex script 断连**：在 Arabic、Thai 等脚本中，这可能破坏字形连接

### 根本原因（代码证据）

#### 1. **HarfBuzz 不负责 fallback**
HarfBuzz 只是一个 shaping 引擎，给定字体和字符序列，它输出 glyph。如果字体不包含某个字符，HarfBuzz 返回 .notdef glyph（ID = 0），不会自动尝试其他字体。

```cpp
// HarfBuzz 的职责 - 仅 shaping，不 fallback
hb_shape(hb_font, buffer, ...);
// 如果 hb_font 中没有某个字符，返回 .notdef glyph
```

#### 2. **Chromium 检测 .notdef 并驱动 run splitting**
Chromium 在 `ExtractShapeResults()` 中检测 .notdef 字形，识别出某字符无法用当前字体渲染，于是：
- 将该字符范围加入 reshape queue
- 用下一个 fallback 字体重试
- 最终该字符由不同的字体渲染，形成新的 `ShapeResultRun`

```cpp
if (glyphs[i].codepoint == 0) {  // .notdef 检测
  QueueCharacters(...);  // 加入 reshape queue，准备用下一个字体重试
}
```

#### 3. **结果：按字符覆盖的 run splitting**
最终结果是多个 `ShapeResultRun`：
```cpp
// ShapeResultRun 定义
class ShapeResultRun {
  const SimpleFontData* font_data_;  // 每个 run 有一个字体！
  unsigned num_glyphs_;
  unsigned num_characters_;
};
```

如果：
- run 1：字符 0-4，使用 "Arial"
- run 2：字符 5-6，使用 "Fallback"
- run 3：字符 7-9，使用 "Arial"

则 run 1 和 run 3 无法连接（因为 run 2 的字体可能有不同的连接规则）。

#### 4. **为何 Complex script 特别受影响**

Complex script（Arabic、Thai、Myanmar 等）需要 **contextual shaping**：
- 同一个字符的形状取决于其上下文（前后字符）
- 字形可能连接、变形或分解

当 run 被分割：
```
خ ط ي         (Arabic, needs contextual shaping)
↓  ↓  ↓
Font A, Font B, Font A
```

Font B 的 contextual rules 与 Font A 不同，导致：
- 字形连接断裂
- 大小、位置不协调
- 音调符号可能错位（如 Thai）

### 代码链路完整证明

```
CSS: font-family: Arial, "custom-web-font", serif;
text: "Hello中文"
      ↓
FontFallbackList 构建：Arial → custom-web-font → serif
      ↓
HarfBuzzShaper::ShapeSegment()
  ├─ 尝试用 Arial: "Hello中" → shaped OK，但"中"可能是 .notdef
  │    (如果 Arial 不包含中文)
  │    ↓
  │    ExtractShapeResults() 检测到 .notdef
  │    → QueueCharacters() 添加"中"到 reshape queue
  │
  ├─ 尝试用 custom-web-font: "中" → shaped OK
  │    → CommitGlyphs() 创建新的 ShapeResultRun
  │
  └─ 尝试用 serif: ...
      ↓
ShapeResult: 
  - Run 1: "Hello" (Arial)
  - Run 2: "中" (custom-web-font)  ← 字体不同！
  - Run 3: "文" (custom-web-font)
```

### 避免的方法

1. **确保 @font-face 字体包含所有使用的字符**
   - 使用 `unicode-range` 限制应用范围
   - 确保 fallback 字体链中有通用字体

2. **使用 `font-variation-settings` 或 `@supports`**
   - 根据字体支持情况选择不同的字体栈

3. **多语言方案**
   - 对不同脚本使用专用字体
   - 例如：`font-family: "Noto Sans", "Noto Sans CJK", sans-serif;`

---

## 附录：关键代码位置速查表

| 任务 | 文件位置 | 关键函数/类 |
|-----|--------|----------|
| font-family 匹配 | font_fallback_list.cc:149 | `FontFallbackList::GetFontData()` |
| @font-face + unicode-range | css_segmented_font_face.cc:100 | `CSSSegmentedFontFace::GetFontData()` |
| local() 解析 | local_font_face_source.cc:70 | `LocalFontFaceSource::CreateFontData()` |
| generic family 映射 | generic_font_family_settings.cc:170 | `GenericFontFamilySettings::Serif()` |
| glyph 覆盖检查 | simple_font_data.cc:250 | `SimpleFontData::GlyphForCharacter()` |
| Fallback 迭代 | font_fallback_iterator.h:30 | `FontFallbackIterator::Next()` |
| HarfBuzz shaping 调用 | harfbuzz_shaper.cc:339 | `hb_shape()` 调用 |
| .notdef 检测 | harfbuzz_shaper.cc:580 | `ExtractShapeResults()` |
| Run 创建 | harfbuzz_shaper.cc:520 | `CommitGlyphs()` → `ShapeResultRun` |
| Run 数据结构 | shape_result_run.h:58 | `ShapeResultRun` |

---

**文档完成日期**: 2026年1月29日
**分析基于 Chromium 主分支**: main
