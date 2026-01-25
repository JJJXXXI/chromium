# Chromium 字体体系汇总（快速导航版）

> 目标：把仓库中所有“字体相关”未跟踪文档按主题串起来，给出入口、用途和适合读者。若要深入，请在对应文件中查看细节。

---

## 🎯 核心架构理解
## 1) 选择与流程总览
- [CHROMIUM_FONT_SELECTION_FLOW.md](CHROMIUM_FONT_SELECTION_FLOW.md) — Mermaid 流程图：CSS→FontDescription→Font→延迟匹配，含有/无 CSS、@font-face/系统字体、Android fonts.xml 分支。
- [CHROMIUM_FONT_SELECTION_QUICK_REFERENCE.md](CHROMIUM_FONT_SELECTION_QUICK_REFERENCE.md) — 快速卡片：步骤、GDB/LOG、断点位、Android 特殊路径。
- [CHROMIUM_FONT_TERMINOLOGY_GUIDE.md](CHROMIUM_FONT_TERMINOLOGY_GUIDE.md) — 名词释义：generic_family 与 family_name、Font vs FontDescription vs CSSFontSelector。
- [COMPLETE_FONT_PROCESSING_FLOW.md](COMPLETE_FONT_PROCESSING_FLOW.md) — 端到端 6 阶段：HTML→解析→Style→Layout/Paint→Shaping→Raster。
- [CSS_FONT_SELECTION_WITH_STYLE.md](CSS_FONT_SELECTION_WITH_STYLE.md) — 有 CSS font-family 时：ConvertFontFamily、链表构建、Paint 时查询顺序。
- [NO_CSS_FONT_SELECTION_DETAIL.md](NO_CSS_FONT_SELECTION_DETAIL.md) — 无 CSS 时：kNoFamily→kStandardFamily→fonts.xml/Roboto→shaping/paint。

**适合**：想快速搞清“字体如何被选出/匹配/渲染”的读者。

---

## 2) Android 字体加载 / 渲染
- [ANDROID_FONT_LOADING_ANALYSIS.md](ANDROID_FONT_LOADING_ANALYSIS.md) — 流程与耗时分析。
- [ANDROID_FONT_LOADING_CALL_STACK.md](ANDROID_FONT_LOADING_CALL_STACK.md) — 实际调用栈。
- [ANDROID_FONT_LOADING_README.md](ANDROID_FONT_LOADING_README.md) — 阅读顺序/导航。
- [ANDROID_FONT_NAVIGATION.md](ANDROID_FONT_NAVIGATION.md) — 文件/模块导航。
- [ANDROID_FONT_RENDERING_INTEGRATION.md](ANDROID_FONT_RENDERING_INTEGRATION.md) — FontCache、SkFontMgr、HarfBuzz 的衔接。
- [ANDROID_FONT_SKFONTMGR_CORE.md](ANDROID_FONT_SKFONTMGR_CORE.md) — SkFontMgr 解析 fonts.xml / alias 核心逻辑。
- [Android_Font_Rendering_Code_Paths.md](Android_Font_Rendering_Code_Paths.md) — 全路径清单；含 Web 字体、缓存、调试章节。
- [SYSTEM_FONTS_TO_FONTCACHE_FLOW.md](SYSTEM_FONTS_TO_FONTCACHE_FLOW.md) — 系统字体配置到 FontCache 的流转。

**适合**：Android/嵌入式、性能与缓存优化、fonts.xml/SkFontMgr 深挖。

---

## 3) Blink / Chromium 字体架构
- [Blink_Font_Rendering_Architecture.md](Blink_Font_Rendering_Architecture.md) — Blink 层字体对象模型、FontCache、回退、@font-face、度量获取。
- [FONTCACHE_VS_FONTFACECACHE_ANALYSIS.md](FONTCACHE_VS_FONTFACECACHE_ANALYSIS.md) — FontCache vs FontFaceCache 职责对比。
- [FONTDATA_CLASS_HIERARCHY.md](FONTDATA_CLASS_HIERARCHY.md) — FontDescription/Font/FontData/PlatformData 等类层次。
- [FONTFACE_FONTDATA_RELATIONSHIP.md](FONTFACE_FONTDATA_RELATIONSHIP.md) — @font-face 声明与运行时 FontData 的关系。
- [FONTATIONS_MAKE_ANALYSIS.md](FONTATIONS_MAKE_ANALYSIS.md) — Fontations 构建/切换相关。
- [SKRIFA_FORMATS_GLYF_VS_CFF.md](SKRIFA_FORMATS_GLYF_VS_CFF.md) — Glyf vs CFF 格式笔记。

**适合**：Blink/排版引擎开发、架构梳理。

---

## 4) Web 字体 / 下载 / 格式
- [WEB_FONTS_DETAILED_ANALYSIS.md](WEB_FONTS_DETAILED_ANALYSIS.md) — Web 字体加载/匹配/渲染深 dive。
- [WEB_FONT_DOWNLOAD_PROCESSING.md](WEB_FONT_DOWNLOAD_PROCESSING.md) — 下载管线与缓存。
- [FONT_FORMATS_BINARY_HINTING_EXAMPLES.md](FONT_FORMATS_BINARY_HINTING_EXAMPLES.md) — 格式与 hinting 示例。

**适合**：前端性能、@font-face 调优、格式/兼容性。

---

## 5) Shaping / HarfBuzz
- [HARFBUZZ_HINTING_TIMING_ANALYSIS.md](HARFBUZZ_HINTING_TIMING_ANALYSIS.md) — HarfBuzz 成形与 hinting 时序、性能。

**适合**：文本 shaping、度量、性能分析。

---

## 6) 汇总 / 导航 / 其他
- [DOCUMENTATION_UPDATE_SUMMARY.md](DOCUMENTATION_UPDATE_SUMMARY.md) — 文档改动/亮点索引。
- [QUICK_REFERENCE.md](QUICK_REFERENCE.md) — 多主题速查。
- [LATEST_WORK_SUMMARY.md](LATEST_WORK_SUMMARY.md) — 近期工作总结与后续方向。
- [third_party/blink/renderer/platform/fonts/README.zh.md](third_party/blink/renderer/platform/fonts/README.zh.md) — Blink 字体平台中文 README（未跟踪版本）。

---

## 📌 快速选型指引
- 想看“无 CSS / 有 CSS 字体如何选”：CHROMIUM_FONT_SELECTION_FLOW + NO_CSS_FONT_SELECTION_DETAIL + CSS_FONT_SELECTION_WITH_STYLE
- 想查 Android fonts.xml / SkFontMgr：ANDROID_* 系列 + SYSTEM_FONTS_TO_FONTCACHE_FLOW
- 想搞清 FontCache vs FontFaceCache：FONTCACHE_VS_FONTFACECACHE_ANALYSIS
- 想看 Blink 对象模型：Blink_Font_Rendering_Architecture + FONTDATA_CLASS_HIERARCHY
- 想定位 @font-face 下载/解码：WEB_FONT_DOWNLOAD_PROCESSING + WEB_FONTS_DETAILED_ANALYSIS
- 想调试/下断点：CHROMIUM_FONT_SELECTION_QUICK_REFERENCE（含 GDB/LOG/断点）

---

## 📖 阅读顺序建议（按需求）
- **入门/总览**：CHROMIUM_FONT_SELECTION_FLOW → QUICK_REFERENCE → TERMINOLOGY_GUIDE
- **CSS/选择链**：NO_CSS_FONT_SELECTION_DETAIL → CSS_FONT_SELECTION_WITH_STYLE
- **Android 路径**：Android_Font_Rendering_Code_Paths → SKFontMgr 核心 → fonts.xml 流
- **Blink 架构**：Blink_Font_Rendering_Architecture → FONTCACHE_VS_FONTFACECACHE_ANALYSIS
- **Web 字体**：WEB_FONT_DOWNLOAD_PROCESSING → WEB_FONTS_DETAILED_ANALYSIS
- **性能/调试**：HARFBUZZ_HINTING_TIMING_ANALYSIS + CHROMIUM_FONT_SELECTION_QUICK_REFERENCE

---

## 🛠️ 常用调试入口
- ConvertFontFamily / ConvertFontFamilyName（CSS→FontDescription）
- FontBuilder::CreateFont / InitialGenericFamily（无 CSS 默认族）
- FontFallbackList::PrimaryFont / GetFontData（延迟匹配触发）
- CSSFontSelector::GetFontData（@font-face 优先，其次系统字体）
- SkFontMgr_android / fonts.xml 解析（Android）
- HarfBuzzShaper::Shape（成形时机与 OpenType 特性）

---

**最后更新**: 2026-01
**状态**: 汇总完成；如需深读请跳转对应文件。
---
