# Chromium 字体渲染系统 - 详细调用链文档

## 1. 整体架构流程图

```
┌──────────────────────────────────────────────────────────────────┐
│                        HTML/DOM → 样式计算                        │
└────────────────────────┬─────────────────────────────────────────┘
                         │ StyleResolver::ApplyStyleToElement()
                         ▼
┌──────────────────────────────────────────────────────────────────┐
│                  FontDescription 生成                            │
│  - family: ["Arial", "微软雅黑", "sans-serif"]                   │
│  - size: 16px                                                    │
│  - weight/style/variant 等                                       │
└────────────────────────┬─────────────────────────────────────────┘
                         │ new Font(description, selector)
                         ▼
┌──────────────────────────────────────────────────────────────────┐
│              Font 对象创建 → FontFallbackList 初始化              │
│  Font::Font(FontDescription, FontSelector*)                      │
│    └─> new FontFallbackList(font_selector)                       │
└────────────────────────┬─────────────────────────────────────────┘
                         │ font.DrawText() / font.Measure()
                         ▼
┌──────────────────────────────────────────────────────────────────┐
│         [Lazy] 字体选择 - FontFallbackList 缓存                   │
│  PrimarySimpleFontDataWithSpace(description)                     │
│    └─> DeterminePrimarySimpleFontData()                          │
│        └─> FontFallbackIterator + FontSelector::GetFontData()    │
└────────────────────────┬─────────────────────────────────────────┘
                         │ 文本渲染 LayoutText()
                         ▼
┌──────────────────────────────────────────────────────────────────┐
│            文本成形 - HarfBuzzShaper::Shape()                     │
│  1. RunSegmenter 分割（script/direction/small-caps）             │
│  2. 对每个 segment:                                              │
│     - FontFallbackIterator cascading                             │
│     - 检查 unicode-range                                         │
│     - HarfBuzz 成形调用                                          │
│     - 生成 ShapeResultRun                                        │
│  3. 返回 ShapeResult（多个 run）                                  │
└────────────────────────┬─────────────────────────────────────────┘
                         │ 实际绘制 DrawText()
                         ▼
┌──────────────────────────────────────────────────────────────────┐
│              渲染 - 对每个 run 调用平台绘制函数                     │
│  使用对应字体的 glyph ID 和位置信息                               │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. 字体选择（FontFallbackList）详细流程

```
Font::Font(FontDescription fd, FontSelector* selector)
  │
  └─> FontFallbackList::FontFallbackList(selector)
      ├─ font_selector_ = selector
      ├─ is_invalid_ = false
      └─ [延迟初始化]

[第一次使用时]

font.PrimaryFont()
  │
  └─> font_fallback_list_->PrimarySimpleFontDataWithSpace(fd)
      │
      ├─ [缓存检查] cached_primary_simple_font_data_with_space_
      │   ├─ 如果有效，直接返回
      │   └─ 如果无效，继续
      │
      └─> DeterminePrimarySimpleFontData(fd, uchar::kSpace, false)
          │
          ├─ [准备 hint 列表]
          │  └─ HintCharList: [' '] (space character)
          │
          └─> DeterminePrimarySimpleFontDataCore(fd, ' ', false)
              │
              ├─ FontFallbackIterator iterator(
              │      fd,
              │      this,
              │      FontFallbackPriority::kText
              │  );
              │
              ├─ [Fallback 阶段]
              │  │
              │  ├─ kFallbackPriorityFonts
              │  │  └─> FallbackPriorityFont(' ')
              │  │      [如果有 emoji/符号优先字体]
              │  │
              │  ├─ kFontGroupFonts
              │  │  └─> 遍历 CSS font-family 列表
              │  │      │
              │  │      ├─ 尝试 "Arial"
              │  │      │  └─> FontSelector::GetFontData(fd, "Arial")
              │  │      │      │
              │  │      │      ├─ [检查 @font-face]
              │  │      │      │  └─> FontFaceCache 中查找 "Arial"
              │  │      │      │      └─> CSSSegmentedFontFace::GetFontData()
              │  │      │      │
              │  │      │      ├─ [检查系统字族名]
              │  │      │      │  └─> IsPlatformFamilyMatchAvailable()
              │  │      │      │
              │  │      │      └─> FontCache::GetFontData(fd, "Arial")
              │  │      │          └─> 返回 SimpleFontData 或 nullptr
              │  │      │
              │  │      ├─ 检查 ' ' 是否有 glyph
              │  │      │  └─> font_data->GlyphForCharacter(' ')
              │  │      │
              │  │      └─ 如果成功，返回此 font_data
              │  │
              │  │  或尝试 "微软雅黑"、"sans-serif"
              │  │
              │  ├─ kSegmentedFace
              │  │  └─> @font-face SegmentedFontData
              │  │      ├─ unicode-range 检查
              │  │      └─ glyph 可用性检查
              │  │
              │  ├─ kPreferencesFonts
              │  │  └─> 用户首选字体
              │  │
              │  ├─ kSystemFonts
              │  │  └─> FontCache::FallbackFontForCharacter(fd, ' ', ...)
              │  │      ├─ [Android] 
              │  │      │  └─> GetGenericFamilyNameForScript()
              │  │      │      └─> SkFontMgr::matchFamilyStyleCharacter()
              │  │      │
              │  │      ├─ [Linux]
              │  │      │  └─> fontconfig 查询
              │  │      │
              │  │      └─ [Windows]
              │  │         └─> DirectWrite 查询
              │  │
              │  └─ kFirstCandidateForNotdefGlyph
              │     └─> 最后一个候选（用于 .notdef glyph）
              │
              └─ 返回第一个成功的 SimpleFontData
                  └─> 缓存到 cached_primary_simple_font_data_with_space_
```

---

## 3. @font-face Unicode-range 处理流程

```
CSS 解析阶段
  │
  @font-face {
    font-family: "MyFont";
    unicode-range: U+0100-01FF, U+0250-0377;
    src: url("myfont.woff2") format("woff2");
  }
  │
  ├─ [解析 unicode-range 描述符]
  │  └─> ParseUnicodeRangeDescriptor()
  │      └─> 返回 HeapVector<UnicodeRange>
  │          └─ UnicodeRange { from: 0x0100, to: 0x01FF }
  │          └─ UnicodeRange { from: 0x0250, to: 0x0377 }
  │
  ├─ [创建 CSSFontFace]
  │  └─> new CSSFontFace(
  │        font_face,
  │        std::move(unicode_ranges)  // 上面的 vector
  │      )
  │      ├─ ranges_ = new UnicodeRangeSet(std::move(ranges))
  │      └─ [添加到 FontFaceCache]
  │
  └─ [创建 CSSSegmentedFontFace]
     └─> 包含多个 CSSFontFace（如果有多个 @font-face）
         └─> font_data_table_<FontCacheKey, SegmentedFontData>

[字体选择阶段]

FontFallbackIterator::Next(hint_list)
  │
  ├─ [在 kSegmentedFace 阶段]
  │  └─> CSSSegmentedFontFace::GetFontData(fd)
  │      │
  │      ├─ [生成或获取缓存的 SegmentedFontData]
  │      │  └─> 遍历所有 CSSFontFace
  │      │      │
  │      │      ├─ CSSFontFace #1 (U+0100-01FF)
  │      │      │  ├─ 获取或加载字体数据
  │      │      │  └─> new FontDataForRangeSet(
  │      │      │        font_data,
  │      │      │        U+0100-01FF range_set
  │      │      │      )
  │      │      │
  │      │      ├─ CSSFontFace #2 (U+0250-0377)
  │      │      │  └─> new FontDataForRangeSet(...)
  │      │      │
  │      │      └─ faces_.push_back(face1);
  │      │         faces_.push_back(face2);
  │      │
  │      └─> 返回 new SegmentedFontData(faces_)
  │
  └─> FontDataForRangeSet

[渲染阶段 - 字符查询]

SegmentedFontData::FontDataForCharacter(UChar32 c)
  │
  ├─ 遍历 faces_[]
  │  │
  │  ├─ face #1: { font_data, U+0100-01FF }
  │  │  ├─ face->Ranges()->Contains(c)
  │  │  │  └─ if (0x0100 <= c && c <= 0x01FF) ✓
  │  │  │     else ✗ → 下一个 face
  │  │  │
  │  │  └─ if (c == 0x0150)  ✓ 在范围内
  │  │     ├─ glyph = font_data->GlyphForCharacter(0x0150)
  │  │     ├─ if (glyph != 0) ✓ 有 glyph
  │  │     └─ return font_data  ✓ 使用此字体
  │  │
  │  ├─ face #2: { font_data, U+0250-0377 }
  │  │  └─ ...类似检查...
  │  │
  │  └─ [如果都不匹配]
  │     └─ return nullptr
  │        └─> 继续 fallback 查询
  │
  └─ [如果找到]
     └─> 返回对应的 SimpleFontData
```

---

## 4. HarfBuzz Shaping 详细流程

```
HarfBuzzShaper::Shape(
    const Font* font,
    TextDirection direction,
    unsigned start,
    unsigned end
)
  │
  ├─ 第 1 步：文本预分割
  │  │
  │  ├─ RunSegmenter segmenter(
  │  │      text_.Characters16(),
  │  │      font->GetFontDescription().Orientation()
  │  │  );
  │  │
  │  ├─ [分割因子]
  │  │  ├─ Script (通过 ScriptRunIterator)
  │  │  │  └─ 例：U+0041 (Latin) vs U+4E00 (Han)
  │  │  │
  │  │  ├─ Direction (通过 OrientationIterator)
  │  │  │  └─ 例：Upright vs Mixed vs Sideways
  │  │  │
  │  │  ├─ Small-caps (通过 SmallCapsIterator)
  │  │  │  └─ 大写字母 vs 小写字母
  │  │  │
  │  │  └─ Symbols (通过 SymbolsIterator)
  │  │     └─ 例：Emoji vs 普通文本
  │  │
  │  └─ 生成 segments: [Segment1, Segment2, ...]
  │     └─ 每个 segment 包含：
  │        ├─ start, end (位置)
  │        ├─ script (脚本)
  │        ├─ render_orientation (方向)
  │        └─ font_fallback_priority (优先级)
  │
  ├─ 第 2 步：创建 ShapeResult 容器
  │  │
  │  └─> new ShapeResult(start, end - start, direction)
  │      └─ runs_: HeapVector<Member<ShapeResultRun>>
  │
  ├─ 第 3 步：对每个 segment 进行 shaping
  │  │
  │  ├─ for (segment : segments)
  │  │  │
  │  │  └─> ShapeSegment(range_context, segment, result)
  │  │      │
  │  │      ├─ [获取字体列表]
  │  │      │  └─> FontFallbackIterator iterator(
  │  │      │        font_description,
  │  │      │        font->GetFontFallbackList(),
  │  │      │        segment.font_fallback_priority
  │  │      │     );
  │  │      │
  │  │      ├─ [cascading 循环]
  │  │      │  │
  │  │      │  └─> while (iterator.HasNext())
  │  │      │      │
  │  │      │      ├─ FontDataForRangeSet* candidate =
  │  │      │      │    iterator.Next(hint_chars);
  │  │      │      │
  │  │      │      ├─ [提取字体信息]
  │  │      │      │  ├─ const SimpleFontData* font_data =
  │  │      │      │  │    candidate->FontData();
  │  │      │      │  │
  │  │      │      │  └─ const UnicodeRangeSet* range_set =
  │  │      │      │      candidate->Ranges();
  │  │      │      │
  │  │      │      ├─ [创建 HarfBuzz font]
  │  │      │      │  └─> hb_font_t* hb_font =
  │  │      │      │       font_data->GetHarfBuzzFace()
  │  │      │      │         ->GetScaledFont(
  │  │      │      │            range_set,      // ← unicode-range 限制
  │  │      │      │            callbacks,
  │  │      │      │            specified_size
  │  │      │      │         );
  │  │      │      │
  │  │      │      ├─ [HarfBuzz 成形调用]
  │  │      │      │  └─> ShapeHarfBuzz(
  │  │      │      │        segment,
  │  │      │      │        font_data,
  │  │      │      │        hb_font,
  │  │      │      │        result
  │  │      │      │      )
  │  │      │      │      │
  │  │      │      │      ├─ 1. 获取 HarfBuzz buffer
  │  │      │      │      │  └─> hb_buffer_t* buffer
  │  │      │      │      │
  │  │      │      │      ├─ 2. 填充缓冲区（文本+特性+语言）
  │  │      │      │      │  ├─ hb_buffer_add_utf16(buffer, ...)
  │  │      │      │      │  ├─ hb_buffer_set_language(buffer, ...)
  │  │      │      │      │  └─ 设置 OpenType 特性
  │  │      │      │      │
  │  │      │      │      ├─ 3. HarfBuzz 成形
  │  │      │      │      │  └─> hb_shape(hb_font, buffer, ...)
  │  │      │      │      │      └─ 返回 glyph 数组
  │  │      │      │      │
  │  │      │      │      └─ 4. 创建 ShapeResultRun
  │  │      │      │         └─> new ShapeResultRun(
  │  │      │      │              font_data,
  │  │      │      │              hb_direction,
  │  │      │      │              canvas_rotation,
  │  │      │      │              script,
  │  │      │      │              start_index,
  │  │      │      │              num_glyphs,
  │  │      │      │              num_characters
  │  │      │      │            )
  │  │      │      │            └─ 填充 glyph_data_
  │  │      │      │
  │  │      │      └─ [添加到结果]
  │  │      │         └─> result->InsertRun(run)
  │  │      │
  │  │      └─ [如果没有 fallback 字体成功]
  │  │         └─> 使用最后一个候选（.notdef）
  │  │
  │  └─ 返回部分填充的 result
  │
  ├─ 第 4 步：后处理
  │  │
  │  ├─ 合并相邻的相同字体 run（可选）
  │  │  └─> run1.MergeIfPossible(run2)
  │  │
  │  └─ 设置 Bidi 覆盖（如需要）
  │
  └─ 返回 ShapeResult*
     └─ 包含所有 ShapeResultRun 及其 glyph 信息
```

---

## 5. Glyph 查询详细流程

### 5.1 简单查询（GlyphForCharacter）

```
SimpleFontData::GlyphForCharacter(UChar32 character)
  │
  └─> platform_data_->SkFont().getUnicharMetrics(character, ...)
      │
      ├─ [对于 Skia 后端]
      │  └─> SkTypeface::getMetrics(character)
      │      └─ 返回 glyph ID 或 0（.notdef）
      │
      └─ 返回 Glyph (uint32)
         └─ 0 = .notdef (字符不存在)
         └─ 1+ = 有效 glyph ID
```

### 5.2 带 unicode-range 的查询

```
SegmentedFontData::FontDataForCharacter(UChar32 c)
  │
  ├─ for (const auto& face : faces_)
  │  │
  │  ├─ 步骤 1：检查 unicode-range
  │  │  │
  │  │  └─> if (!face->Ranges()->Contains(c))
  │  │      └─ continue;  // 跳过此 face
  │  │
  │  ├─ 步骤 2：获取字体
  │  │  │
  │  │  └─> const SimpleFontData* font_data = face->FontData();
  │  │      └─ if (!font_data) continue;
  │  │
  │  ├─ 步骤 3：检查 glyph
  │  │  │
  │  │  └─> Glyph glyph = font_data->GlyphForCharacter(c);
  │  │      └─ if (glyph == 0) continue;  // 无此 glyph
  │  │
  │  └─ 步骤 4：返回
  │     │
  │     └─> return font_data;  ✓ 找到！
  │
  └─> return nullptr;  ✗ 未找到
```

### 5.3 完整 cascading 查询

```
Font::SelectFallbackFont(
    const FontDescription& fd,
    UChar32 character,
    FontFallbackPriority priority
)
  │
  └─> FontFallbackIterator iterator(fd, fallback_list, priority);
      │
      ├─ while (iterator.HasNext())
      │  │
      │  ├─ FontDataForRangeSet* candidate = 
      │  │    iterator.Next(hint_list);
      │  │
      │  ├─ [检查字体]
      │  │  ├─ if (!candidate->Contains(character))
      │  │  │  └─ continue;  // unicode-range 不匹配
      │  │  │
      │  │  └─ const SimpleFontData* font_data = 
      │  │      candidate->FontData();
      │  │
      │  ├─ [检查 glyph（两层）]
      │  │  ├─ Glyph glyph = font_data->GlyphForCharacter(c);
      │  │  ├─ if (glyph != 0)
      │  │  │  └─> return font_data;  ✓ 找到！
      │  │  │
      │  │  └─ 如果是 SegmentedFontData
      │  │     └─> const SimpleFontData* seg_font = 
      │  │          font_data->FontDataForCharacter(c);
      │  │         if (seg_font)
      │  │           └─> return seg_font;
      │  │
      │  └─ [继续下一个 fallback]
      │
      └─> return nullptr;  // 完全 fallback
```

---

## 6. 平台特异性实现对比

### 6.1 Android: GetGenericFamilyNameForScript

```cpp
// 位置: font_cache_android.cc#L231
AtomicString FontCache::GetGenericFamilyNameForScript(
    const AtomicString& generic_family,
    const AtomicString& script_family,
    const FontDescription& font_description)

InputScript → Output Font
──────────────────────────
USCRIPT_LATIN       → "sans-serif" (hardcoded)
USCRIPT_HAN         → "NotoSansCJK" (system font)
USCRIPT_HIRAGANA    → "NotoSansCJK" (system font)
USCRIPT_HANGUL      → "NotoSansCJK" (system font)
其他                → script_family (原值)

├─ CJK 处理
│  ├─ GetCJKFamilyNameForScript()
│  │  └─> 返回 system CJK font name
│  │
│  └─ font_description.LocaleOrDefault()
│     └─> 用于区分 Han/Hiragana/Hangul
│
└─ 非 CJK
   └─> 回退到原始 script_family
       └─ 通常是硬编码的 generic family
       └─ "Times New Roman" (不存在于 Android)
```

### 6.2 Linux: fontconfig

```cpp
// 位置: 通常在 font_cache_skia.cc 中
FcPattern* pattern = FcPatternCreate();
FcPatternAddString(pattern, FC_FAMILY, "DejaVu Sans");
FcPatternAddInteger(pattern, FC_CHARSET, character);
FcPatternAddBool(pattern, FC_SCALABLE, FcTrue);

FcFont* match = FcFontMatch(config, pattern, nullptr);
FcPatternGetString(match, FC_FILE, 0, &filename);

Input Character → Output Font File
──────────────────────────────────
U+0041 (A)       → /usr/share/fonts/dejavu/DejaVuSans.ttf
U+4E00 (中)      → /usr/share/fonts/noto/NotoSansCJK-Regular.ttc
U+0627 (Arabic)  → /usr/share/fonts/noto/NotoSansArabic-Regular.ttf
不存在的字符      → 系统默认字体或 .notdef
```

### 6.3 Windows: DirectWrite / GDI

```cpp
// 位置: font_cache_skia_win.cc#L284
IDWriteFontCollection* collection = ...;
IDWriteFont* font = collection->FindFamilyName(family_name, ...);

Input Character → Output IDWriteFont
────────────────────────────────────
U+0041 (A)       → "Segoe UI" (Windows default)
U+4E00 (中)      → "SimSun" (Windows CJK)
U+0627 (Arabic)  → "Arabic Typesetting"
特殊符号          → system fallback font
```

### 6.4 macOS: Core Text

```cpp
// 使用 CTFont API
CTFontRef font = CTFontCreateWithName(
    CFSTR("Helvetica"),
    14.0,
    nullptr
);

CGGlyph glyph;
bool has_glyph = CTFontGetGlyphsForCharacters(
    font,
    &character,
    &glyph,
    1
);

Input Character → Output Glyph
───────────────────────────────
U+0041 (A)       → glyph ID 36 (Helvetica)
U+4E00 (中)      → 0 (.notdef, fallback needed)
不存在            → 0 (.notdef)
```

---

## 7. Error Cases 和 Fallback 路径

### 7.1 字体不可用的情况

```
场景 1: CSS 指定的字体不存在
┌─────────────────────┐
│ font-family: Foo    │ ✗ 系统无此字体
└─────────────────────┘
         │
         ▼
┌─────────────────────┐
│ 尝试下一个字体       │
│ font-family: Bar    │ ✓ 系统有此字体
└─────────────────────┘
         │
         ▼
┌─────────────────────┐
│ 返回 Bar 字体       │
└─────────────────────┘

场景 2: @font-face 字体未加载
┌──────────────────────────────┐
│ @font-face { src: url(...); }│ ⏳ 下载中
└──────────────────────────────┘
         │
         ▼
┌──────────────────────────────┐
│ 使用 fallback 字体           │ 同时继续加载
│ (FontFallbackPriority)       │
└──────────────────────────────┘
         │
         ▼ [字体加载完成]
┌──────────────────────────────┐
│ 更新缓存，重新 render        │
│ (FontFaceInvalidated)        │
└──────────────────────────────┘

场景 3: 字符完全无 glyph
┌──────────────────────────┐
│ 字符: U+E000 (private)   │ ✗ 无定义
└──────────────────────────┘
         │
         ▼
┌──────────────────────────┐
│ 尝试系统 fallback        │
│ FontCache::             │
│ FallbackFontForCharacter│
└──────────────────────────┘
         │
    ┌────┴────┐
    │ ✓ 找到   │ ✗ 未找到
    ▼        ▼
 [使用字体]  [.notdef glyph]
```

### 7.2 Unicode-range 不匹配的处理

```
@font-face {
  font-family: "Subset";
  unicode-range: U+0100-01FF;  // 只支持 Ñ 等
  src: url("subset.woff2");
}

场景：需要 U+00E9 (é)
┌─────────────────────────────────┐
│ 检查 "Subset" 字体             │
│ U+00E9 ∉ [U+0100-U+01FF]       │ ✗ 不在范围内
└─────────────────────────────────┘
         │
         ▼ [跳过此 @font-face]
┌─────────────────────────────────┐
│ 尝试下一个字体（CSS fallback）  │
└─────────────────────────────────┘
```

---

## 8. 缓存和无效化机制

```
┌──────────────────┐
│ FontFallbackList │
├──────────────────┤
│ cached_data:     │
│ - primary_font   │ ← 缓存主字体
│ - font_list      │ ← 缓存字体列表
│ - is_invalid_    │ ← 标记
└──────────────────┘

[无效化触发器]

@font-face 规则更新
  │
  └─> FontFaceInvalidated()
      │
      ├─ FontSelector::FontFaceInvalidated()
      │  └─> 遍历所有注册的 FontSelectorClient
      │
      └─> FontSelectorClient::FontsNeedUpdate()
         │
         ├─ FontFallbackList::MarkInvalid()
         │  └─ is_invalid_ = true
         │
         └─> [下次使用时重新计算]
            └─> font_fallback_list_ = new FontFallbackList(...)

字体加载完成
  │
  └─> CSSFontFace::FontLoaded()
      │
      ├─ 清除 SegmentedFontData 缓存
      │
      └─> FontFaceInvalidated()
         └─> [同上]

用户设置变化（WebPreferences）
  │
  └─> Document 更新字体设置
      │
      └─> FontFaceInvalidated()
         └─> [同上]
```

---

## 9. 性能关键路径

### 9.1 热路径（Hot Path）

```
绘制文本频率最高的操作：
1. ShapeResult 缓存命中率最高
   └─> NGShapeCache（per-font cache）
   └─> 95%+ 缓存命中

2. PrimaryFont 缓存（FontFallbackList）
   └─> 一次性计算，多次使用
   └─> 99%+ 缓存命中

3. GlyphForCharacter
   └─> 平台 glyph cache（Skia/DirectWrite）
   └─> 90%+ 缓存命中

4. RunSegmenter
   └─> 按需计算（每个文本 run）
   └─> 通常很快（ICU 库优化）
```

### 9.2 冷路径（Cold Path）

```
偶尔执行的操作：
1. @font-face 字体加载
   └─> 网络请求 + 解析
   └─> 可能需要 100ms+

2. System fallback 查询
   └─> 第一次调用较慢（系统 API 初始化）
   └─> 后续调用快速（缓存）

3. SegmentedFontData 创建
   └─> CSS 规则变化时执行
   └─> 通常 < 1ms

4. HarfBuzz shaping（新文本）
   └─> 复杂文本较慢（可能 10ms+）
   └─> 简单文本很快（1ms）
```

---

## 10. 常见调试技巧

### 10.1 跟踪字体选择

```cpp
// 在 FontFallbackList::DeterminePrimarySimpleFontData 中

VLOG(1) << "DeterminePrimarySimpleFontData:"
        << " character=" << (uint32_t)lookup_character
        << " script=" << font_description.GetScript();

FontFallbackIterator iterator(...);
int stage = 0;
while (iterator.HasNext()) {
  FontDataForRangeSet* candidate = iterator.Next(hint_list);
  VLOG(2) << "  Stage " << stage++
          << ": " << (candidate ? "found" : "skip");
}
```

### 10.2 验证 Unicode-range

```cpp
// 在 SegmentedFontData::FontDataForCharacter 中

const SimpleFontData* result = nullptr;
for (const auto& face : faces_) {
  bool in_range = face->Ranges()->Contains(c);
  VLOG(2) << "Face check: c=" << (uint32_t)c
          << " in_range=" << in_range;
  if (in_range) {
    // ... glyph 检查 ...
    result = face->FontData();
    break;
  }
}
```

### 10.3 追踪 HarfBuzz 成形

```cpp
// 在 ShapeHarfBuzz 中

hb_shape(hb_font, buffer, ...);

unsigned glyph_count = 0;
const hb_glyph_info_t* infos = hb_buffer_get_glyph_infos(buffer, &glyph_count);

VLOG(1) << "HarfBuzz shaped: " << glyph_count << " glyphs"
        << " from " << num_characters << " characters";

for (unsigned i = 0; i < glyph_count && i < 10; ++i) {
  VLOG(2) << "  Glyph[" << i << "]: codepoint=" << infos[i].codepoint
          << " cluster=" << infos[i].cluster;
}
```

---

**调用链详解版本**: 1.0  
**最后更新**: 2026-01-29  
**涵盖范围**: 完整的字体选择 → Shaping 流程
