# Chromium 字体渲染 - 快速参考

## 关键概念速查表

### 1. 字体选择优先级（Cascading Order）

```
1. 优先级字体 (Fallback Priority Fonts)
   - Emoji 字体（kEmojiEmoji）
   - 东亚语言特殊字体
   
2. CSS font-family 列表中的字体
   "Arial, '微软雅黑', sans-serif"
   └─ 逐个尝试直到成功
   
3. @font-face 分段字体 (SegmentedFontData)
   @font-face { unicode-range: U+0100-01FF; }
   └─ 按 unicode-range 检查
   
4. 用户首选字体
   - 浏览器设置的偏好字体
   
5. 系统 Fallback 字体
   - 通过 FontCache::FallbackFontForCharacter()
   - 平台相关（Android/Linux/Windows/macOS）
   
6. 最后一个候选
   - 用于渲染 .notdef glyph
   
7. 无法找到
```

### 2. Run Splitting 因子

| 因子 | 优先级 | 说明 |
|------|--------|------|
| **Script** | 1（最高） | U+0041 (Latin) vs U+4E00 (Han) |
| **Direction** | 2 | LTR (左到右) vs RTL (右到左) |
| **Font Fallback Priority** | 3 | kText vs kEmojiEmoji |
| **Small-caps** | 4 | 小写字母需要不同字体 |
| **Orientation** | 5 | Upright vs Mixed vs Sideways |
| **Symbols** | 6 | Emoji 等特殊符号 |

### 3. 平台字体来源

| 平台 | 字体位置 | API | CJK 特殊处理 |
|------|---------|-----|-------------|
| **Android** | `/system/etc/fonts.xml` | SkFontMgr | NotoSansCJK |
| **Linux** | `/usr/share/fonts/` | fontconfig | Noto 字体集 |
| **Windows** | `C:\Windows\Fonts\` | DirectWrite | 系统字体 |
| **macOS** | `~/Library/Fonts/` | Core Text | 系统字体 |

### 4. 字体数据结构体系

```
FontData (抽象基类)
├─ SimpleFontData (单个字体)
│  └─ 实际的字体文件
├─ SegmentedFontData (多个 @font-face)
│  └─ FontDataForRangeSet[] (按 unicode-range)
│     └─ SimpleFontData + UnicodeRangeSet
```

### 5. 关键函数速查

| 场景 | 函数 | 位置 |
|------|------|------|
| 获取字符的字体 | `FontFallbackIterator::Next()` | font_fallback_iterator.h#L50 |
| 检查 glyph 可用性 | `SimpleFontData::GlyphForCharacter()` | simple_font_data.h#L164 |
| 检查 unicode-range | `UnicodeRangeSet::Contains()` | unicode_range_set.h#L48 |
| 系统 fallback | `FontCache::FallbackFontForCharacter()` | font_cache.h#L114 |
| 文本成形 | `HarfBuzzShaper::Shape()` | harfbuzz_shaper.h#L60 |
| 文本分割 | `RunSegmenter::Consume()` | run_segmenter.h#L48 |

---

## 常见问题排查

### Q1: 为什么 CJK 文本可以跟随系统字体，但英文不行？

**答**: Android 特殊处理
```cpp
// Android 中
if (script == USCRIPT_HAN || USCRIPT_HIRAGANA || USCRIPT_HANGUL) {
  return GetCJKFamilyNameForScript(generic_family);  // CJK 字体
}
// 返回 NotoSansCJK 等系统字体

// 对于英文
return "Times New Roman";  // 硬编码，不存在于 Android
```

**解决方案**: 在 AwSettings 中实现非 CJK 字体的系统配置读取

### Q2: @font-face 中 unicode-range 如何工作？

**答**: 
```cpp
@font-face {
  unicode-range: U+0100-01FF;  // 仅这些字符使用此字体
}

// 渲染时
char c = U+0150;
if (font_face->Ranges()->Contains(c)) {  // ✓ 在范围内
  use_this_font = true;
} else if (c == U+00E9) {  // ✗ 不在范围内
  try_next_font = true;
}
```

### Q3: Fallback 时如何检查字体是否有某字符？

**答**: 三层检查
```cpp
// 1. Unicode-range 检查（快）
if (!range_set->Contains(character)) return nullptr;

// 2. Glyph 检查（中）
Glyph glyph = font_data->GlyphForCharacter(character);
if (glyph == 0) return nullptr;  // .notdef glyph

// 3. HarfBuzz 成形检查（慢，最准确）
ShapeResult* result = shaper->Shape(font, direction, start, end);
if (result->NumGlyphs() > 0) return this_font;
```

### Q4: 为什么某些文本中途变色（字体变化）？

**答**: Run splitting 导致
```
输入: "Hello 世界 مرحبا"
分割: 
  - "Hello " → Latin script → Arial
  - "世界 " → Han script → SimSun
  - "مرحبا" → Arabic script → Arial (不支持)
                           → 系统 Arabic 字体

每个 run 可能使用不同字体，导致视觉差异
```

### Q5: local() 字体匹配何时失败？

**答**:
```cpp
@font-face {
  src: local("Helvetica Neue"),  // PostScript 名
       local("helv"),            // 简短名
       url("font.woff2");        // URL fallback
}

// 匹配顺序
1. 检查系统是否有 "Helvetica Neue" 
   → 如果有，使用系统字体
2. 检查系统是否有 "helv"
   → 如果有，使用系统字体
3. 下载 url("font.woff2")
   → 使用 web 字体

// 失败场景
- 系统字体名不匹配（可能是大小写）
- 尝试了所有 local() 但都不存在
- URL 字体下载超时
```

---

## 代码审查要点

### ✅ 添加新字体源时

1. 继承 `FontData` 类
2. 实现 `FontDataForCharacter(UChar32)` 方法
3. 检查 unicode-range（如适用）
4. 验证 glyph 可用性
5. 添加平台特异性实现（如需要）

### ❌ 避免常见错误

1. ❌ 直接修改 `FontCache` 中的硬编码字族名
   - ✅ 通过 `WebPreferences` 或 platform API

2. ❌ 在 HarfBuzz 后进行 run splitting
   - ✅ 在 HarfBuzz 前通过 `RunSegmenter`

3. ❌ 忽略 unicode-range 检查
   - ✅ 总是检查 `range_set->Contains(c)`

4. ❌ 假设 glyph ID == 0 表示字符不存在
   - ✅ 使用 `.notdef` glyph（0）但需结合其他检查

5. ❌ 跳过平台特异性实现
   - ✅ 为 Android/Linux/Windows/macOS 都实现

---

## 性能优化提示

### 缓存策略

| 什么被缓存 | 在哪 | 有效期 |
|-----------|------|--------|
| SimpleFontData | FontCache | 全局生命周期 |
| FontFallbackList | Font 对象 | 直到 font update |
| ShapeResult | NGShapeCache | 直到文本变化 |
| @font-face 数据 | FontFaceCache | 直到 CSS 更新 |

### 热路径优化

```cpp
// ❌ 慢：每次都迭代
for (const auto& font : fallback_list) {
  if (font->GlyphForCharacter(c) != 0) {
    return font;  // 线性搜索
  }
}

// ✅ 快：缓存常见字符
if (c == ' ') return cached_space_font;
if (c == '0') return cached_digit_font;
if (IsCJK(c)) return cjk_fallback_font;  // 提前返回
```

---

## 调试技巧

### 1. 追踪字体选择

```cpp
// 在 FontFallbackList::DeterminePrimarySimpleFontData 中添加
VLOG(1) << "Selecting font for character: " << c 
        << " script: " << script
        << " fallback_stage: " << stage;
```

### 2. 验证 unicode-range

```cpp
// 在 SegmentedFontData::FontDataForCharacter 中
DCHECK(range_set->Contains(c)) 
    << "Character " << c << " should be in range";
```

### 3. 检查 HarfBuzz 结果

```cpp
// 在 ShapeResult 中
VLOG(1) << "Shape result: " << shape_result->NumRuns() << " runs";
for (const auto& run : shape_result->Runs()) {
  VLOG(1) << "  Run: font=" << run->FontData()->GetFamilyName()
          << " glyphs=" << run->NumGlyphs();
}
```

---

## 相关源代码链接

### 核心选择逻辑
- [FontSelector::GetFontData()](third_party/blink/renderer/platform/fonts/font_selector.h#L54)
- [FontFallbackIterator::Next()](third_party/blink/renderer/platform/fonts/font_fallback_iterator.h#L50)
- [FontCache::FallbackFontForCharacter()](third_party/blink/renderer/platform/fonts/font_cache.h#L114)

### Unicode-range 处理
- [UnicodeRangeSet::Contains()](third_party/blink/renderer/platform/fonts/unicode_range_set.h#L48)
- [SegmentedFontData::FontDataForCharacter()](third_party/blink/renderer/platform/fonts/segmented_font_data.cc#L33)

### Shaping 流程
- [HarfBuzzShaper::Shape()](third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.h#L60)
- [RunSegmenter::Consume()](third_party/blink/renderer/platform/fonts/shaping/run_segmenter.h#L48)
- [ShapeResult 结构](third_party/blink/renderer/platform/fonts/shaping/shape_result.h#L100)

### 平台实现
- [Android](third_party/blink/renderer/platform/fonts/android/font_cache_android.cc#L231)
- [Linux/Skia](third_party/blink/renderer/platform/fonts/skia/font_cache_skia.cc#L147)
- [Windows](third_party/blink/renderer/platform/fonts/win/font_cache_skia_win.cc#L284)

---

## 快速参考：添加新字体源

```cpp
// Step 1: 继承 FontData
class MyCustomFont : public FontData {
  const SimpleFontData* FontDataForCharacter(UChar32 c) const override {
    // Step 2: 检查 unicode-range（如需要）
    if (range_set && !range_set->Contains(c)) {
      return nullptr;
    }
    
    // Step 3: 检查 glyph 可用性
    if (GlyphForCharacter(c) == 0) {
      return nullptr;
    }
    
    return this_font_data;
  }
  
  // Step 4: 实现其他必需方法
  bool IsCustomFont() const override { return true; }
  bool IsLoading() const override { return loading_; }
  // ...
};

// Step 5: 注册到 FontFallbackIterator
// 在 kSegmentedFace 或 kFontGroupFonts 阶段返回
```

---

**快速参考版本**: 1.0  
**最后更新**: 2026-01-29
