# Blink 的文本栈 #

本 README 是 Blink 文本栈的文档入口。

它可以在格式化形式下[在此查看](https://chromium.googlesource.com/chromium/src/+/main/third_party/blink/renderer/platform/fonts/README.md)。

## 概览 ##

Blink 的字体与文本栈负责在布局引擎中，对带 CSS 样式的 HTML 文本分段做测量、几何操作与绘制。

`font.h` 定义布局代码与字体代码的接口，主要支撑三类请求：

- 文本测量
- 文本选择相关的几何操作：
  - 计算边界框
  - 坐标到字符索引的映射
  - 字符索引到坐标的映射
- 文本绘制

从 HTML 到最终呈现大致经历：

- 从 CSS 样式生成 `Font` 对象
- 用该定义与可用的网页/系统字体进行匹配
- 在 shaping 前准备所需前置条件
- 将文本切分为适合 shaping 的片段
- 在词缓存中查找已 shaping 的结果
- 以匹配到的字体完成字符到字形的映射（shaping）
- 字体回退

## 从 CSS 样式到 `Font` 对象

在样式解析阶段，为每个 DOM 元素计算 `ComputedStyle`。每个 `ComputedStyle` 拥有一个 `Font` 和一个 `FontDescription`。CSS 被解析为 `CSSValue` 派生类型后，`style_builder_converter.cc` 的 `ConvertFont…` 把 `CSSValue` 转成 `FontDescription`；反向由 `computed_style_css_value_mapping.cc` 的 `ValueForFont…` 完成。

样式解析时调用 `FontBuilder::CreateFont`，依据 `FontDescription` 更新 `Font`，并通过 `CSSFontSelector` 把两者关联起来，`CSSFontSelector` 记录文档作用域可用的网页字体和系统字体。

此时 `Font` 还不对应具体字型，`FontDescription` 也未指向单一字型，因为它持有来自 `font-family` 的家族列表。

`Font` 为布局/绘制提供获取几何信息（行高、x-height、测量文本等）的 API，绘制阶段也用它画文本。只有在请求这些操作时，`Font` 才会真正去做字体匹配。

## 字体匹配

当 `Font` 需要工作时，必须解析 `FontDescription`，在网页/系统字体中找到具体字型，算法参考 CSS Fonts “Matching Font Styles”。

`Font` 通过 `Update(CSSFontSelector*)` 获知可用字体更新，并将 `CSSFontSelector` 传给 `FontFallbackList` 以解析 `font-family`。

`FontFallbackList` 调用 `CSSFontSelector::GetFontData`，先向 `FontFaceCache` 查找网页字体的 `FontData`。若未命中，借助 `FontSelectionAlgorithm::IsBetterMatchForRequest` 依 CSS 规范选出最佳匹配。

若仍无结果，则查询 `FontCache` 进行系统字体查找，平台实现分别在 `font_cache_skia.cc`、`font_cache_mac.mm`、`font_cache_linux.cc`、`font_cache_android.cc` 中调用系统 API。

## 说明：准备文本 shaping

Shaping 是把 Unicode 字符串映射到字体字形 ID 序列及其精确位置的过程，依赖字体中的 OpenType 布局。拉丁文字多是字形 ID 加水平进距，复杂脚本还涉及垂直定位、重排和字素簇归并。

单个可 shaping 的 run 需要以下输入保持不变：

- 字体
- 字号
- 文本方向（LTR/RTL）
- 书写方向（水平/垂直）
- 请求的 OpenType 特性
- Unicode 脚本
- Unicode 语言
- 文本内容
- 上下文（前后文）

因此在 shaping 前，需要把文本与 `FontDescription` 拆成这些输入恒定的子 run。例如多脚本文本要按脚本拆分；垂直排版夹杂拉丁文字也需拆分。

### Emoji

Emoji 有默认呈现样式（文本/表情），见 UTS #51 “Presentation Style”。为选择正确字体（彩色或轮廓），也需按 emoji 属性拆分。

## 词缓存

Shaping 和回退代价高，而布局中会反复执行几何操作，因此使用词缓存提速。

### 可缓存单元

基本单元是以空格分隔的词；CJK 无空格时，每个 CJK 字符视为一个词。

### 缓存键

Shaping 依赖字体、字号、脚本等多变量，且可用字体集合变化会使 `FontFallbackList` 结果变化。缓存键需包含词、`FontDescription`，以及 shaping 时的可用字体集合，通过 `FontFallbackList::CompositeKey` 生成。

### 访问缓存

`caching_word_shaper.h` 是入口，`caching_word_shape_iterator.h` 负责按词/空格或 CJK 分段，并响应 `TextRun::Width()` 或返回 `ShapeResultBuffer`。未命中时由 shaping 阶段生成 `ShapeResult`。`CachingWordShaper` 是 `Font` 与 `HarfBuzzShaper` 之间的加速层。

## Run 分段

`RunSegmenter` 顶层拆分 API，基于 `ScriptRunIterator`（脚本）、`OrientationIterator`（方向/书写方式）、`SymbolsIterator`（emoji 呈现）。

以 UTF-16 `UChar` 构造后，作为迭代器返回脚本、emoji 呈现、书写方向均恒定的子 run，适合用于 shaping。

## 文本 shaping

实现位于 `shaping/harfbuzz_shaper.h` 与 `.cc`。

流程：run 分段；用主字体先 shaping 初始分段；识别已/未 shaping 序列；对未 shaping 的子 run 依次尝试剩余字体与回退字体，直到最后兜底字体。

若请求 small/petite caps，还需大小写分段并匹配字体 OpenType 特性；若缺失则按 CSS Fonts Level 3 合成 small-caps。

示例（无 small-caps）：一段垂直文本被分成四段，标注脚本、方向和回退优先级。开头日文 Hiragana 无需旋转，回退优先级 `FontFallbackPriority::kText`。

```
0 い
1 ろ
2 は USCRIPT_HIRAGANA,
    OrientationIterator::OrientationKeep,
    FontFallbackPriority::kText

3 a
4 ̄ (Combining Macron)
5 a
6 A USCRIPT_LATIN,
    OrientationIterator::OrientationRotateSideways,
    FontFallbackPriority::kText

7 い
8 ろ
9 は USCRIPT_HIRAGANA,
     OrientationIterator::OrientationKeep,
     FontFallbackPriority::kText
```

假设 CSS：`font-family: "Heiti SC", Tinos, sans-serif;`，其中 *Tinos* 是复合网页字体：一条 `@font-face` 子集到 `U+00-U+FF`，另一条不限范围。

`FontFallbackIterator` 依次提供 *Heiti SC*、受限版 *Tinos*、全集 *Tinos*，最后系统 *sans-serif*。

段 0–2 用 Heiti SC 成功 shaping：

```
Glyphpos: 0  1  2
Cluster:  0  1  2
Glyph:    い ろ は
```

段 3–6 用 Heiti SC 失败（缺少结合符），结果类似：

```
Glyphpos: 3 4 5 6
Cluster:  3 3 5 6
Glyph:    a ☐ a A (☐ 为 .notdef)
```

`extractShapeResults()` 将含 a+̄ 的簇放入 `HolesQueue`，切换回退字体再处理。

切到受限 *Tinos* 仍缺字，再切到全集 *Tinos*，shaping 成功：

```
Glyphpos: 3 4 5 6
Cluster:  3 3 5 6
Glyph:    a ◌̄ a A （◌̄ 定位在首个 a 上方）
```

该子 run 插入 `ShapeResult`，`ShapeResult::insertRun` 负责合并到 `RunInfo` 向量。余下 Hiragana 子 run 同理处理并插入。

## 字体回退

Shaping 中字体选择是迭代的：先用主字体覆盖尽可能多的字形，再用次级字体填补 `.notdef`，直到无缺字。

`FontFallbackIterator` 以迭代器 API 提供下一字体。`FontFallbackList` 按 `font-family` 解析，优先网页字体，再到系统字体，符合 CSS Fonts 规范；细节见 `LocaleInFonts.md` 的“Installed Font Fallback”。

若 `HarfBuzzShaper` 在用尽 `FontFallbackList` 后仍缺字，说明 `font-family` 覆盖不足，需要系统回退。此时调用 `FontCache::FallbackFontForCharacter()`，按缺失字符找替代字体，引入额外系统字体。

总结：`FontFallbackIterator` 向 `HarfBuzzShaper` 依次提供 `font-family` 中的字体及系统回退字体，迭代填补缺字；若仍需渲染 .notdef，则使用主字体呈现。
