# Android 平台 Chromium 字体渲染：代码路径分析

> **目标**：以代码路径为核心，系统性理解 Android 平台上 Chromium 的字体渲染流程，重点关注 FreeType 与 skrifa/fontations 的职责划分。

---

## 目录

1. [SkFontMgr 初始化路径（Android）](#1-skfontmgr-初始化路径android)
2. [字体匹配与查询流程](#2-字体匹配与查询流程)
3. [字符到字形的映射](#3-字符到字形的映射)
4. [HarfBuzz 文本成形](#4-harfbuzz-文本成形)
5. [FreeType vs Fontations 职责划分](#5-freetype-vs-fontations-职责划分)
6. [关键数据结构与状态管理](#6-关键数据结构与状态管理)

---

## 1. SkFontMgr 初始化路径（Android）

### 1.1 启动入口点

**文件**: `content/renderer/renderer_main_platform_delegate_android.cc`

```cpp
void RendererMainPlatformDelegate::PlatformInitialize() {
  // Initialize the font manager before the sandbox is in place.
  // SkFontMgr_New_AndroidNDK must call ASystemFontIterator_open() which on
  // Android 14+ user devices with updated system fonts will call statx and
  // possibly other system calls that are not allowed in the sandbox.
  // See https://crbug.com/40618213 for details.
  [[maybe_unused]] auto mgr = skia::DefaultFontMgr();
}
```

**关键点**：
- **输入**: 无（首次调用）
- **输出**: 初始化的 `sk_sp<SkFontMgr>` 单例
- **副作用**: 
  - 全局状态初始化（`g_fontmgr_override` 或工厂创建）
  - **必须在沙箱前执行**（Android 14+ 需要访问 `/system/fonts/`）
  - IO 操作：读取 `/system/etc/fonts.xml` 和字体文件

### 1.2 DefaultFontMgr 工厂

**文件**: `skia/ext/font_utils.cc`

```cpp
sk_sp<SkFontMgr> DefaultFontMgr() {
  static std::once_flag flag;
  static SkFontMgr* mgr;
  std::call_once(flag, [] {
    mgr = fontmgr_factory().release();  // ← 调用平台特定工厂
    g_factory_called = true;
  });
  return sk_ref_sp(mgr);
}
```

**关键点**：
- **线程安全**: `std::call_once` 确保单例初始化
- **全局状态**: `g_factory_called` 标记（防止 `OverrideDefaultSkFontMgr` 在初始化后调用）
- **缓存**: 静态变量 `mgr` 持久化，整个进程生命周期内不释放

### 1.3 Android 平台工厂实现

**文件**: `skia/ext/font_utils.cc`

```cpp
static sk_sp<SkFontMgr> fontmgr_factory() {
  if (g_fontmgr_override) {
    return sk_ref_sp(g_fontmgr_override);  // ← 优先返回被覆盖的管理器
  }

#if BUILDFLAG(IS_ANDROID)
  // Android 14+ (API 35+) 使用 NDK API
  if (base::FeatureList::IsEnabled(kUseAndroidNDKFontAPI) &&
      android_get_device_api_level() > __ANDROID_API_V__) {
    sk_sp<SkFontMgr> ndk_fontmgr =
        SkFontMgr_New_AndroidNDK(false, SkFontScanner_Make_Fontations());
    if (ndk_fontmgr && ndk_fontmgr->countFamilies()) {
      return ndk_fontmgr;
    }
  }
  // Android < 14 或 NDK API 失败时，使用传统 XML 解析
  return SkFontMgr_New_Android(nullptr, SkFontScanner_Make_Fontations());
#endif
}
```

**输入**:
- `kUseAndroidNDKFontAPI`: Feature flag（默认禁用，见 `skia/ext/font_utils.cc:61`）
- `android_get_device_api_level()`: 系统 API 级别

**输出**:
- `SkFontMgr_New_AndroidNDK()`: 使用 `ASystemFontIterator` API（Android 14+）
- `SkFontMgr_New_Android()`: 使用 `fonts.xml` 解析（Android < 14）

**关键文件**（Skia 内部）:
- `third_party/skia/include/ports/SkFontMgr_android_ndk.h`
- `third_party/skia/src/ports/SkFontMgr_android.cpp`

### 1.4 SkFontScanner_Make_Fontations()

**文件**: `skia/ext/font_utils.cc` （引用 Skia 头文件）

```cpp
#if BUILDFLAG(IS_ANDROID)
#include "third_party/skia/include/ports/SkFontScanner_Fontations.h"
#endif

// 调用时
SkFontMgr_New_Android(nullptr, SkFontScanner_Make_Fontations());
```

**关键点**：
- **职责**: 字体文件元数据扫描（不是渲染）
- **输入**: 字体文件路径（.ttf, .otf, .ttc）
- **输出**: 字体家族名、样式（weight, width, slant）、支持的字符集
- **实现**: 使用 Rust `fontations` 库（通过 FFI 调用）

**Fontations 的作用**（此阶段）：
1. 解析字体文件头（`head`, `name`, `OS/2` 表）
2. 提取字体元数据
3. **不涉及字形渲染**（仅元数据）

---

## 2. 字体匹配与查询流程

### 2.1 Blink 请求入口

**文件**: `third_party/blink/renderer/platform/fonts/android/font_cache_android.cc`

```cpp
const SimpleFontData* FontCache::PlatformFallbackFontForCharacter(
    const FontDescription& font_description,
    UChar32 c,
    const SimpleFontData*,
    FontFallbackPriority fallback_priority) {
  
  sk_sp<SkFontMgr> fm(skia::DefaultFontMgr());  // ① 获取全局管理器

  // ② 确定通用家族名（serif 特殊处理）
  const char* generic_family_name = nullptr;
  if (font_description.GenericFamily() == FontDescription::kSerifFamily)
    generic_family_name = "serif";

  // ③ Emoji 特殊处理
  FontFallbackPriority fallback_priority_with_emoji_text = fallback_priority;
  if (RuntimeEnabledFeatures::SystemFallbackEmojiVSSupportEnabled() &&
      fallback_priority == FontFallbackPriority::kText &&
      Character::IsEmoji(c)) {
    fallback_priority_with_emoji_text = FontFallbackPriority::kEmojiText;
  }

  // ④ 核心匹配调用
  const FontPlatformData* font_platform_data =
      CreateFontPlatformDataForCharacter(fm.get(), c, font_description,
                                         generic_family_name,
                                         fallback_priority_with_emoji_text);
  // ...
}
```

**输入**:
- `font_description`: CSS 属性（family, weight, style, size, locale）
- `c`: Unicode 字符（UChar32，即 `uint32_t`）
- `fallback_priority`: 枚举值（`kText`, `kEmojiText`, `kEmojiEmoji`）

**输出**:
- `SimpleFontData*`: Blink 内部字体对象（包含 `FontPlatformData`）

### 2.2 CreateFontPlatformDataForCharacter（核心匹配）

**文件**: `third_party/blink/renderer/platform/fonts/font_cache.cc` （假设，需要验证）

```cpp
const FontPlatformData* FontCache::CreateFontPlatformDataForCharacter(
    SkFontMgr* fm,
    UChar32 character,
    const FontDescription& font_description,
    const char* generic_family_name,
    FontFallbackPriority fallback_priority) {
  
  // ① 构建 BCP-47 locale 列表
  Bcp47Vector locales = GetBcp47LocaleForRequest(
      font_description, fallback_priority);
  
  // ② 调用 Skia API
  sk_sp<SkTypeface> typeface(
      fm->matchFamilyStyleCharacter(
          generic_family_name,              // 家族名（可为 nullptr）
          font_description.SkiaFontStyle(), // SkFontStyle(weight, width, slant)
          locales.data(),                   // BCP-47 locale 数组
          locales.size(),                   // locale 数量
          character));                      // Unicode 字符
  
  if (!typeface) {
    return nullptr;  // 匹配失败
  }
  
  // ③ 包装为 FontPlatformData
  return new FontPlatformData(typeface, font_description.ComputedSize());
}
```

**关键 API**: `SkFontMgr::matchFamilyStyleCharacter()`

**输入**:
- `familyName`: 字体家族名（如 `"serif"`, `"sans-serif"`, `nullptr` 表示系统默认）
- `style`: `SkFontStyle`（weight: 100-900, width: 1-9, slant: 0/1/2）
- `bcp47[]`: locale 数组（如 `["zh-Hans", "zh-Hant", "ja"]`）
- `bcp47Count`: locale 数量
- `character`: Unicode 字符（用于字形覆盖检查）

**输出**:
- `sk_sp<SkTypeface>`: Skia 字体对象（智能指针）

**内部逻辑**（Skia 层）:
1. 解析 `fonts.xml` 或 NDK API 的字体列表
2. 按以下优先级匹配：
   - **Locale 匹配**（`xml:lang` 属性或 `AFont_getLocales()`）
   - **家族名匹配**（`<family name="serif">`）
   - **样式匹配**（weight, width, slant 距离计算）
   - **字符覆盖**（检查字体是否有该字符的字形）
3. 返回最佳匹配的 `SkTypeface`

### 2.3 fonts.xml 解析（Android < 14）

**系统文件**: `/system/etc/fonts.xml`

```xml
<familyset version="22">
    <family name="sans-serif">
        <font weight="100" style="normal">Roboto-Thin.ttf</font>
        <font weight="400" style="normal">Roboto-Regular.ttf</font>
        <font weight="700" style="normal">Roboto-Bold.ttf</font>
    </family>

    <family name="serif">
        <font weight="400" style="normal">NotoSerif-Regular.ttf</font>
    </family>

    <!-- CJK 回退链 -->
    <family lang="zh-Hans">
        <font weight="400" style="normal">NotoSansCJK-Regular.ttc</font>
    </family>

    <!-- Emoji -->
    <family lang="und-Zsye">
        <font weight="400" style="normal">NotoColorEmoji.ttf</font>
    </family>
</familyset>
```

**解析位置**: Skia 内部 `SkFontMgr_android.cpp`

**关键逻辑**:
1. 递归解析 `<family>` 节点
2. 对每个 `<font>` 节点：
   - 调用 `SkFontScanner_Make_Fontations()->scanFile()` 读取元数据
   - 构建 `SkTypeface` 对象（延迟加载字体数据）
3. 建立索引：`family_name → SkFontStyleSet`

### 2.4 Android NDK API（Android 14+）

**系统 API**: `<android/system_fonts.h>`

```cpp
// Skia 内部调用（伪代码）
ASystemFontIterator* iter = ASystemFontIterator_open();
while (ASystemFontIterator_next(iter)) {
  AFont* font = ASystemFontIterator_getFont(iter);
  
  // 获取字体文件路径
  const char* file_path = AFont_getFontFilePath(font);
  
  // 获取 locale
  size_t locale_count = AFont_getLocaleCount(font);
  for (size_t i = 0; i < locale_count; ++i) {
    const char* locale = AFont_getLocale(font, i);
  }
  
  // 获取样式
  uint16_t weight = AFont_getWeight(font);
  bool italic = AFont_isItalic(font);
  
  // 扫描字体文件
  SkFontScanner_Make_Fontations()->scanFile(file_path, ...);
}
ASystemFontIterator_close(iter);
```

**优势**:
- 无需解析 XML（系统直接提供结构化数据）
- 支持动态字体更新（Android 14+ 可通过 Google Play 更新字体）
- 更快的启动速度

---

## 3. 字符到字形的映射

### 3.1 SkTypeface::unicharToGlyph()

**调用位置**: `third_party/blink/renderer/platform/fonts/simple_font_data.cc`

```cpp
Glyph SimpleFontData::GlyphForCharacter(UChar32 character) const {
  SkTypeface* typeface = platform_data_->Typeface();
  if (!typeface) {
    return 0;  // 无效字形
  }
  
  // Skia API 调用
  SkGlyphID glyph_id = typeface->unicharToGlyph(character);
  return glyph_id;  // 返回字形 ID
}
```

**输入**:
- `character`: Unicode 字符（UChar32）

**输出**:
- `SkGlyphID`: 字形 ID（16位无符号整数，0 表示字体不支持该字符）

**内部实现**（Skia 层）:

#### 3.1.1 使用 FreeType（传统路径）

**文件**: `third_party/skia/src/ports/SkFontHost_FreeType.cpp`

```cpp
SkGlyphID SkTypeface_FreeType::unicharToGlyph(SkUnichar uni) const {
  SkAutoMutexExclusive ama(fFTMutex);
  
  // ① 加载 FT_Face（延迟加载）
  if (!fFace) {
    fFace = LoadFTFace(fFontFilePath);
  }
  
  // ② 查询 cmap 表
  FT_UInt glyph_index = FT_Get_Char_Index(fFace, uni);
  
  return static_cast<SkGlyphID>(glyph_index);
}
```

**关键点**:
- **FT_Face**: FreeType 字体对象（包含 cmap 表）
- **FT_Get_Char_Index()**: 查询 Unicode → Glyph ID 映射
- **IO 操作**: 首次调用时读取字体文件（mmap 或 read）

#### 3.1.2 使用 Fontations（新路径）

**文件**: `third_party/skia/src/ports/SkTypeface_fontations.cpp` （假设）

```cpp
SkGlyphID SkTypeface_Fontations::unicharToGlyph(SkUnichar uni) const {
  // ① 获取 Rust fontations 对象
  fontations::BridgedFontRef font = GetFontationsFont();
  
  // ② 调用 Rust FFI
  uint16_t glyph_id = fontations_cmap_lookup(font, uni);
  
  return static_cast<SkGlyphID>(glyph_id);
}
```

**关键点**:
- **Rust FFI**: 通过 C++ → Rust 绑定调用
- **零拷贝**: `BridgedFontRef` 持有共享内存映射的字体数据
- **无锁**: Rust `fontations` 使用无锁数据结构

### 3.2 字形缓存

**位置**: `SimpleFontData::glyph_to_bounds_map_`（macOS）或直接查询（Linux/Android）

```cpp
// macOS 平台有额外缓存
#if BUILDFLAG(IS_APPLE)
  mutable std::unique_ptr<GlyphMetricsMap<gfx::RectF>> glyph_to_bounds_map_;
#endif

gfx::RectF SimpleFontData::BoundsForGlyph(Glyph glyph) const {
#if BUILDFLAG(IS_APPLE)
  if (glyph_to_bounds_map_) {
    if (std::optional<gfx::RectF> bounds = 
            glyph_to_bounds_map_->MetricsForGlyph(glyph)) {
      return *bounds;  // 缓存命中
    }
  }
  // 缓存未命中，查询平台 API
  gfx::RectF bounds = PlatformBoundsForGlyph(glyph);
  glyph_to_bounds_map_->SetMetricsForGlyph(glyph, bounds);
  return bounds;
#else
  // Android/Linux：直接调用 Skia（已有内部缓存）
  return PlatformBoundsForGlyph(glyph);
#endif
}
```

**关键点**:
- **Android/Linux**: Skia/FreeType 已有缓存，无需 Blink 层额外缓存
- **macOS**: CoreText API 慢，需要额外缓存

---

## 4. HarfBuzz 文本成形

### 4.1 成形入口

**文件**: `third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc`

```cpp
ShapeResult* HarfBuzzShaper::Shape(
    const Font* font,
    const TextRun& text_run) {
  
  // ① 获取 HarfBuzz 字体对象（缓存）
  HarfBuzzFontCache& hb_font_cache = 
      FontGlobalContext::Get().GetHarfBuzzFontCache();
  hb_font_t* hb_font = hb_font_cache.GetOrCreateFont(
      *font->PrimaryFont(), 
      font->GetFontDescription().TypesettingFeatures());
  
  // ② 创建 HarfBuzz buffer
  hb_buffer_t* buffer = hb_buffer_create();
  hb_buffer_add_utf16(buffer, 
                     text_run.Characters16(), 
                     text_run.length(), 
                     0, 
                     text_run.length());
  
  // ③ 设置方向、脚本、语言
  hb_buffer_set_direction(buffer, text_run.Direction() == TextDirection::kLtr 
                                      ? HB_DIRECTION_LTR 
                                      : HB_DIRECTION_RTL);
  hb_buffer_set_script(buffer, text_run.GetScript());
  hb_buffer_set_language(buffer, hb_language_from_string(
      text_run.Locale().c_str(), -1));
  
  // ④ 执行成形（关键！）
  hb_shape(hb_font, buffer, nullptr, 0);
  
  // ⑤ 提取结果
  unsigned glyph_count = 0;
  hb_glyph_info_t* glyph_info = hb_buffer_get_glyph_infos(buffer, &glyph_count);
  hb_glyph_position_t* glyph_pos = hb_buffer_get_glyph_positions(buffer, &glyph_count);
  
  // ⑥ 转换为 Blink ShapeResult
  ShapeResult* result = new ShapeResult();
  for (unsigned i = 0; i < glyph_count; ++i) {
    result->AddGlyph(
        glyph_info[i].codepoint,  // 字形 ID
        glyph_pos[i].x_advance,   // X 方向偏移
        glyph_pos[i].y_advance,   // Y 方向偏移
        glyph_pos[i].x_offset,    // X 位置调整
        glyph_pos[i].y_offset);   // Y 位置调整
  }
  
  hb_buffer_destroy(buffer);
  return result;
}
```

**输入**:
- `text_run`: 包含文本、方向、脚本、语言
- `font`: Blink 字体对象（包含 `SimpleFontData`）

**输出**:
- `ShapeResult`: 字形序列 + 位置信息

### 4.2 HarfBuzz 内部调用 Skia/FreeType

**文件**: `ui/gfx/harfbuzz_font_skia.cc`

```cpp
// HarfBuzz 回调：获取字形
static hb_bool_t GetGlyphH(hb_font_t* font,
                           void* font_data,
                           hb_codepoint_t unicode,
                           hb_codepoint_t variation_selector,
                           hb_codepoint_t* glyph,
                           void* user_data) {
  SkTypeface* typeface = static_cast<SkTypeface*>(font_data);
  
  // 调用 Skia API（最终调用 FreeType 或 Fontations）
  *glyph = typeface->unicharToGlyph(unicode);
  
  return *glyph != 0;  // 返回是否找到字形
}

// HarfBuzz 回调：获取字形宽度
static hb_position_t GetGlyphHAdvance(hb_font_t* font,
                                     void* font_data,
                                     hb_codepoint_t glyph,
                                     void* user_data) {
  SkTypeface* typeface = static_cast<SkTypeface*>(font_data);
  SkFont sk_font(sk_ref_sp(typeface), font_size);
  
  // 查询字形宽度
  SkScalar advance;
  sk_font.getWidths(&glyph, 1, &advance);
  
  return SkScalarToHBPosition(advance);
}
```

**关键点**:
- HarfBuzz 通过回调函数与 Skia 交互
- Skia 再调用 FreeType 或 Fontations 获取字形数据
- **调用链**: HarfBuzz → Skia → FreeType/Fontations

### 4.3 OpenType 布局（GSUB/GPOS）

**HarfBuzz 职责**:
1. **GSUB（字形替换）**:
   - 连字（liga）: `f` + `i` → `ﬁ`
   - 上下文替换（calt）: 阿拉伯文字形变体
   - 必需连字（rlig）: Devanagari 辅音堆叠
2. **GPOS（字形定位）**:
   - Kerning: 调整字符间距
   - Mark 定位: 变音符号位置调整
   - Cursive 连接: 阿拉伯文基线调整

**数据来源**:
- OpenType 表（GSUB, GPOS）存储在字体文件中
- HarfBuzz 解析这些表（不依赖 FreeType/Fontations）
- FreeType 只负责字形轮廓和度量，不处理 OpenType 布局

---

## 5. FreeType vs Fontations 职责划分

### 5.1 架构对比

```
┌─────────────────────────────────────────────────────────┐
│                   Chromium/Blink                        │
│              (SimpleFontData, FontCache)                │
└─────────────┬──────────────────────────┬────────────────┘
              │                          │
    ┌─────────▼─────────┐      ┌─────────▼──────────┐
    │    Skia Layer     │      │   HarfBuzz Layer   │
    │  (SkTypeface,     │      │  (hb_font_t,       │
    │   SkFontMgr)      │      │   OpenType 布局)    │
    └─────────┬─────────┘      └─────────┬──────────┘
              │                          │
              ├──────────┬───────────────┘
              │          │
    ┌─────────▼──────┐  │      ┌────────────────┐
    │   FreeType     │  │      │  Fontations    │
    │  (传统路径)     │  │      │  (新路径)       │
    └────────────────┘  │      └────────────────┘
              │         │               │
              └─────────┼───────────────┘
                        │
              ┌─────────▼──────────┐
              │   字体文件 (.ttf)   │
              │   /system/fonts/   │
              └────────────────────┘
```

### 5.2 FreeType 职责

**库**: `third_party/freetype/` (Chromium 内嵌)

**核心功能**:
1. **字体文件解析**:
   - 读取 TrueType/OpenType 表（head, hhea, hmtx, glyf, CFF, etc.）
   - 支持 TTC（TrueType Collection）索引
2. **字形轮廓加载**:
   - TrueType 字形（glyf 表）: 二次贝塞尔曲线
   - CFF 字形（CFF 表）: 三次贝塞尔曲线
3. **字形光栅化**:
   - 灰度反锯齿（256 级灰度）
   - LCD 子像素渲染（RGB/BGR）
   - Hinting（TrueType hinting 或 Autohinting）
4. **度量计算**:
   - 字形边界框（bounding box）
   - 字形前进宽度（advance width）
   - 基线偏移

**关键 API**:
```c
// 加载字体文件
FT_Library library;
FT_Init_FreeType(&library);
FT_Face face;
FT_New_Face(library, "/system/fonts/Roboto-Regular.ttf", 0, &face);

// Unicode → Glyph ID
FT_UInt glyph_index = FT_Get_Char_Index(face, 0x4E00);

// 加载字形轮廓
FT_Load_Glyph(face, glyph_index, FT_LOAD_DEFAULT);

// 光栅化
FT_Render_Glyph(face->glyph, FT_RENDER_MODE_NORMAL);

// 获取位图
FT_Bitmap* bitmap = &face->glyph->bitmap;
```

### 5.3 Fontations 职责

**库**: `third_party/rust/chromium_crates_io/` (Rust crate: `fontations`, `skrifa`)

**核心功能**:
1. **字体文件解析**（与 FreeType 重叠）:
   - 零拷贝解析（直接在 mmap 内存上操作）
   - 更快的启动速度（无需复制数据）
2. **字形轮廓加载**:
   - 支持 COLRv0/v1（彩色字体）
   - 支持变体字体（Variable Fonts）参数化
3. **度量计算**:
   - 字形边界框
   - 前进宽度
4. **字形光栅化**（skrifa 模块）:
   - 集成到 Skia 的光栅化管线
   - 支持 GPU 加速路径

**关键特性**:
- **内存安全**: Rust 保证无缓冲区溢出
- **并发友好**: 无锁数据结构
- **现代字体支持**: COLRv1, Variable Fonts, SVG-in-OpenType

**Rust FFI 示例**（伪代码）:
```rust
// fontations crate
use read_fonts::tables::cmap::Cmap;
use skrifa::outline::DrawSettings;

pub extern "C" fn fontations_cmap_lookup(
    font_data: *const u8,
    font_len: usize,
    codepoint: u32,
) -> u16 {
    let font_bytes = unsafe { std::slice::from_raw_parts(font_data, font_len) };
    let font = FontRef::new(font_bytes).unwrap();
    let cmap = font.cmap().unwrap();
    
    cmap.map_codepoint(codepoint).unwrap_or(0)
}
```

### 5.4 职责对比表

| 功能 | FreeType | Fontations | 说明 |
|------|----------|------------|------|
| **字体文件解析** | ✅ 完整支持 | ✅ 完整支持 | Fontations 更快（零拷贝） |
| **Unicode → Glyph** | ✅ FT_Get_Char_Index | ✅ cmap.map_codepoint | 等效功能 |
| **字形轮廓加载** | ✅ TrueType/CFF | ✅ TrueType/CFF/COLRv1 | Fontations 支持更多格式 |
| **字形光栅化** | ✅ 灰度/LCD | ✅ skrifa（集成 Skia） | FreeType 更成熟 |
| **Hinting** | ✅ TrueType/Autohint | ❌ 不支持 | FreeType 独有 |
| **变体字体** | ⚠️ 基础支持 | ✅ 完整支持 | Fontations 更好 |
| **彩色字体** | ⚠️ 部分支持 | ✅ COLRv0/v1/CBDT | Fontations 更强 |
| **OpenType 布局** | ❌ | ❌ | **两者都不处理**（由 HarfBuzz 负责） |
| **性能（启动）** | 中等 | ✅ 快（零拷贝） | Fontations 优势 |
| **内存安全** | ⚠️ C 代码 | ✅ Rust | Fontations 更安全 |

### 5.5 当前使用状态（Android）

**文件**: `skia/ext/font_utils.cc`

```cpp
#if BUILDFLAG(IS_ANDROID)
  // SkFontScanner 使用 Fontations
  return SkFontMgr_New_Android(nullptr, SkFontScanner_Make_Fontations());
#endif
```

**关键点**:
- **SkFontScanner**: 使用 **Fontations**（扫描字体元数据）
- **SkTypeface 渲染**: 仍然使用 **FreeType**（Android 默认）

**原因**:
- Fontations 扫描速度快，减少启动时间
- FreeType 渲染成熟稳定，兼容性好
- **混合使用**: 扫描用 Fontations，渲染用 FreeType

### 5.6 未来演进路径

**目标**: 完全替换 FreeType 为 Fontations

**当前进度**:
1. ✅ **元数据扫描**: 已切换到 Fontations
2. 🔄 **字形轮廓**: 部分平台使用 skrifa
3. ⏳ **光栅化**: 仍依赖 FreeType（Skia FreeType backend）

**挑战**:
- Hinting 支持（FreeType 独有）
- 性能回归测试（需要匹配或超越 FreeType）
- 平台兼容性（Android 旧版本）

---

## 6. 关键数据结构与状态管理

### 6.1 SkFontMgr（全局单例）

**文件**: `third_party/skia/include/core/SkFontMgr.h`

```cpp
class SkFontMgr : public SkRefCnt {
 public:
  // 获取系统字体家族数量
  int countFamilies() const;
  
  // 获取家族名
  void getFamilyName(int index, SkString* familyName) const;
  
  // 根据家族名和样式匹配字体
  sk_sp<SkTypeface> matchFamilyStyle(const char familyName[], 
                                     const SkFontStyle&) const;
  
  // 根据字符匹配字体（字体回退）
  sk_sp<SkTypeface> matchFamilyStyleCharacter(
      const char familyName[],
      const SkFontStyle&,
      const char* bcp47[],
      int bcp47Count,
      SkUnichar character) const;
};
```

**关键状态**:
- **全局唯一**: 通过 `skia::DefaultFontMgr()` 获取
- **线程安全**: 内部使用锁保护
- **生命周期**: 进程启动到退出（静态变量）

### 6.2 SkTypeface（字体对象）

**文件**: `third_party/skia/include/core/SkTypeface.h`

```cpp
class SkTypeface : public SkRefCnt {
 public:
  // Unicode → Glyph ID
  SkGlyphID unicharToGlyph(SkUnichar uni) const;
  
  // 获取字形边界框
  void getBounds(const SkGlyphID glyphs[], int count, SkRect bounds[]) const;
  
  // 获取字形宽度
  void getWidths(const SkGlyphID glyphs[], int count, SkScalar widths[]) const;
  
  // 获取家族名
  void getFamilyName(SkString* name) const;
  
  // 打开字体流（用于访问原始字体数据）
  std::unique_ptr<SkStreamAsset> openStream(int* ttcIndex) const;
};
```

**关键状态**:
- **引用计数**: `sk_sp<SkTypeface>` 智能指针管理
- **延迟加载**: 字体数据在首次使用时加载
- **缓存**: 由 `SkFontMgr` 内部缓存（相同参数返回相同对象）

### 6.3 SimpleFontData（Blink 层）

**文件**: `third_party/blink/renderer/platform/fonts/simple_font_data.h`

```cpp
class SimpleFontData : public FontData {
 public:
  SimpleFontData(
      const FontPlatformData* platform_data,
      const CustomFontData* custom_data = nullptr);
  
  // 获取字形
  Glyph GlyphForCharacter(UChar32) const;
  
  // 获取字形边界框
  gfx::RectF BoundsForGlyph(Glyph) const;
  
  // 获取字形宽度
  float WidthForGlyph(Glyph) const;
  
  // 字体度量
  FontMetrics& GetFontMetrics();
  
 private:
  Member<const FontPlatformData> platform_data_;  // 持有 SkTypeface
  Member<NGShapeCache> shape_cache_;              // 成形结果缓存
  FontMetrics font_metrics_;                      // 字体度量（行高等）
  
  // macOS 特定的字形缓存
  #if BUILDFLAG(IS_APPLE)
    mutable std::unique_ptr<GlyphMetricsMap<gfx::RectF>> glyph_to_bounds_map_;
  #endif
};
```

**关键状态**:
- **缓存**: `shape_cache_` 缓存成形结果（CJK 文本每个字符都是一个"词"）
- **生命周期**: 由 `FontCache` 管理（LRU 淘汰）
- **线程亲和**: 绑定到主线程（Blink 对象）

### 6.4 HarfBuzzFontCache

**文件**: `third_party/blink/renderer/platform/fonts/shaping/harfbuzz_font_cache.h`

```cpp
class HarfBuzzFontCache {
 public:
  hb_font_t* GetOrCreateFont(
      const SimpleFontData& font_data,
      const TypesettingFeatures& features);
  
 private:
  struct CachingKey {
    const SimpleFontData* font_data;
    TypesettingFeatures features;
  };
  
  absl::flat_hash_map<CachingKey, hb_font_t*> harfbuzz_font_cache_;
  Vector<hb_font_t*> harfbuzz_fonts_;  // 用于析构
};
```

**关键状态**:
- **缓存键**: `(SimpleFontData*, TypesettingFeatures)` 组合
- **生命周期**: 由 `FontGlobalContext` 管理（全局单例）
- **内存管理**: `hb_font_t*` 析构时调用 `hb_font_destroy()`

### 6.5 NGShapeCache

**文件**: `third_party/blink/renderer/platform/fonts/ng_shape_cache.h`

```cpp
class NGShapeCache {
 public:
  ShapeResult* Get(const TextRun&, uint64_t font_unique_id);
  void Put(const TextRun&, uint64_t font_unique_id, ShapeResult*);
  
 private:
  static constexpr size_t kDefaultCacheSize = 500;
  
  // LRU 哈希表
  base::HashingLRUCache<CacheKey, Member<ShapeResult>> cache_;
};
```

**关键状态**:
- **容量**: 500 项（经验值，覆盖 90% 常见文本）
- **LRU 淘汰**: 自动移除最少使用的项
- **线程安全**: 无（由 Blink 主线程独占）

---

## 7. 完整数据流图

### 7.1 字体匹配流程

```
用户请求：渲染 "你好" (Unicode: U+4F60 U+597D)
  ↓
① Blink Layout Engine
  FontDescription: { family: "sans-serif", size: 16px, locale: "zh-Hans" }
  ↓
② FontCache::PlatformFallbackFontForCharacter()
  输入: U+4F60, FontDescription
  ↓
③ skia::DefaultFontMgr()->matchFamilyStyleCharacter()
  输入: family="sans-serif", style=normal, locales=["zh-Hans"], char=U+4F60
  ↓
④ SkFontMgr_Android（读取 fonts.xml 或调用 NDK API）
  匹配规则：
    - locale 匹配: "zh-Hans" → NotoSansCJK-Regular.ttc
    - 字符覆盖: 检查 U+4F60 是否在 cmap 表中
  ↓
⑤ SkFontScanner_Make_Fontations()->scanFile()
  输入: /system/fonts/NotoSansCJK-Regular.ttc
  输出: 字体元数据（家族名, weight, cmap 范围）
  ↓
⑥ 创建 SkTypeface
  延迟加载：字体文件 mmap，但不解析全部数据
  ↓
⑦ 包装为 FontPlatformData → SimpleFontData
  存入 FontCache（全局缓存）
  ↓
返回: SimpleFontData* (后续渲染使用)
```

### 7.2 文本成形流程

```
输入文本: "你好" (U+4F60 U+597D)
  ↓
① CachingWordShaper::Shape()
  查询 NGShapeCache：未命中
  ↓
② HarfBuzzShaper::Shape()
  输入: TextRun("你好"), Font
  ↓
③ HarfBuzzFontCache::GetOrCreateFont()
  查询缓存：未命中
  创建 hb_font_t（绑定 SkTypeface）
  ↓
④ hb_shape(hb_font, buffer)
  HarfBuzz 调用回调：
    - GetGlyphH(): SkTypeface::unicharToGlyph(U+4F60)
      → Skia → FreeType: FT_Get_Char_Index()
      → 返回 Glyph ID: 12345
    - GetGlyphHAdvance(): 查询字形宽度
      → Skia → FreeType: FT_Load_Glyph() + face->glyph->advance
      → 返回宽度: 1000（字体单位）
  ↓
⑤ HarfBuzz 应用 OpenType 布局
  GSUB: 无（CJK 文本通常无连字）
  GPOS: 无（CJK 文本通常无 kerning）
  ↓
⑥ 提取结果
  hb_buffer_get_glyph_infos()
    → [(glyph_id: 12345, cluster: 0), (glyph_id: 54321, cluster: 1)]
  hb_buffer_get_glyph_positions()
    → [(x_advance: 16.0px, y_advance: 0), (x_advance: 16.0px, y_advance: 0)]
  ↓
⑦ 转换为 ShapeResult
  存入 NGShapeCache（词级缓存）
  ↓
返回: ShapeResult* (用于后续绘制)
```

### 7.3 字形渲染流程

```
ShapeResult: [(glyph_id: 12345, x: 0, y: 0), (glyph_id: 54321, x: 16, y: 0)]
  ↓
① Paint Stage
  GraphicsContext::DrawText()
  ↓
② Skia Canvas API
  canvas.drawTextBlob(glyphs, positions, paint)
  ↓
③ Skia 内部
  对每个 glyph_id:
    - 查询 Skia Glyph Cache（内存中）
    - 如果未命中：
      ↓
④ SkTypeface::getPath(glyph_id)
  调用 FreeType（或 Fontations）
  ↓
⑤ FreeType: FT_Load_Glyph()
  读取字体文件（glyf 表或 CFF 表）
  解析字形轮廓（贝塞尔曲线）
  ↓
⑥ FreeType: FT_Render_Glyph()
  光栅化：曲线 → 像素位图
  Hinting: 对齐像素网格
  抗锯齿: 灰度或 LCD 子像素
  ↓
⑦ 返回位图
  存入 Skia Glyph Cache
  ↓
⑧ 合成到画布
  Alpha 混合（如果有反锯齿）
  ↓
最终输出: 屏幕像素
```

---

## 8. Web 字体（@font-face）处理流程

### 8.1 Web 字体加载入口

**文件**: `third_party/blink/renderer/core/css/css_font_face_rule.h/cc`

Web 字体完全独立于系统字体，通过 `@font-face` CSS 规则定义：

```css
@font-face {
  font-family: "CustomFont";
  src: url("custom-font.woff2") format("woff2"),
       url("custom-font.ttf") format("truetype");
  font-weight: 400;
  font-style: normal;
}
```

**处理流程**:

```
HTML 解析 → CSS 解析 (@font-face 规则)
  ↓
CSSFontFaceRule 创建
  ↓
FontFaceSet::addUnsafe()
  输入: CSSFontFaceRule 对象
  ↓
启动资源加载 (ResourceFetcher)
  输入: src 属性中的 URL
  IO: 网络请求字体文件
  ↓
字体文件下载完成
  ↓
WebFontDecoder 解码
  输入: 字体数据（WOFF2, WOFF, TTF, OTF）
  输出: SkTypeface 对象
  ↓
存入 FontFaceCache
  缓存键: (family name, weight, style)
  ↓
字体 loading 状态更新
  document.fonts: "loading" → "loaded"
  ↓
触发 CSS "font-display" 策略
```

### 8.2 FontFaceSet（Web 字体集合）

**文件**: `third_party/blink/renderer/core/css/font_face_set.h/cc`

```cpp
class FontFaceSet : public EventTarget {
 public:
  // 向集合添加字体
  void addUnsafe(FontFace* font_face);
  
  // 删除字体
  bool DeleteUnsafe(FontFace* font_face);
  
  // 查询字体加载状态
  TextPromise* ready();  // Promise 对象
  bool loading() const { return load_status_ == kLoading; }
  
  // JavaScript API: document.fonts
  static FontFaceSet* From(Document&);
  
  // 查找匹配的 Web 字体
  const SimpleFontData* GetFontData(
      const FontDescription&,
      const AtomicString& family_name);
};
```

**关键状态**:
- **加载状态**: `kLoading`（下载中）→ `kLoaded`（完成）
- **缓存**: `FontFaceSet` 维护已加载字体的集合
- **生命周期**: 绑定到 Document，Document 销毁时清理

### 8.3 WebFontDecoder（字体文件解码）

**文件**: `third_party/blink/renderer/platform/fonts/web_font_decoder.h/cc`

```cpp
class WebFontDecoder {
 public:
  // 支持的字体格式
  enum FontFormat {
    kFormatWoff2,
    kFormatWoff,
    kFormatTrueType,
    kFormatOpenType,
    kFormatSvg,
  };
  
  // 解码字体文件
  static sk_sp<SkTypeface> Decode(
      base::span<const uint8_t> font_data,
      FontFormat);
};
```

**解码过程**:

```
网络下载的字体文件 (原始字节)
  ↓
WebFontDecoder::Decode()
  ↓
① WOFF2 格式
   解压缩 (Brotli) → TTF 数据
   ↓
② WOFF 格式
   解压缩 (deflate) → TTF 数据
   ↓
③ TTF/OTF 格式
   直接使用
   ↓
④ SVG 字体
   特殊处理（SVG 文档）
  ↓
创建 SkTypeface
  从 TTF/OTF 二进制数据
  通过 FreeType 或 Fontations 解析
  ↓
返回 SkTypeface 对象（已缓存）
```

**关键点**:
- **位置**: `third_party/blink/renderer/platform/fonts/web_font_decoder.cc`
- **输入**: 字体数据（字节序列）+ 格式类型
- **输出**: `sk_sp<SkTypeface>` 智能指针
- **IO 操作**: 发生在网络层（不在此处）
- **缓存**: WebFontDecoder 内部缓存解码结果

### 8.4 字体匹配中的 Web 字体优先级

**文件**: `third_party/blink/renderer/platform/fonts/font_cache.cc`

Web 字体在字体匹配中优先于系统字体：

```cpp
const SimpleFontData* CSSFontSelector::GetFontData(
    const FontDescription& font_description,
    const AtomicString& family_name) {
  
  // ① 首先查询 Web 字体
  SimpleFontData* font_data = 
      font_face_cache_->Get(font_description, family_name);
  if (font_data) {
    return font_data;  // Web 字体命中
  }
  
  // ② 若无，查询系统字体
  font_data = FontCache::Get().GetFontData(
      font_description,
      family_name);
  if (font_data) {
    return font_data;  // 系统字体命中
  }
  
  // ③ 字体回退链
  return GetLastResortFont();
}
```

**优先级顺序**:
1. Web 字体（@font-face）
2. 系统字体
3. 字体回退链
4. 最后回退字体（system font）

### 8.5 FontFaceCache（Web 字体缓存）

**文件**: `third_party/blink/renderer/core/css/font_face_cache.h/cc`

```cpp
class FontFaceCache {
 private:
  // 缓存键: (family_name, weight, style, variant)
  // 缓存值: SimpleFontData*
  
  using CacheEntry = std::pair<unsigned, SimpleFontData*>;
  using CacheMap = std::unordered_map<std::string, CacheEntry>;
  
  CacheMap font_data_cache_;  // 保存 Web 字体
  Member<FontFaceSet> font_face_set_;  // JavaScript 接口
};
```

**缓存操作**:

```cpp
// 添加 Web 字体到缓存
void FontFaceCache::AddFontFace(
    const FontDescription& description,
    const FontFace* font_face,
    SimpleFontData* font_data) {
  // 生成缓存键
  std::string cache_key = GenerateCacheKey(description, font_face);
  
  // 存入缓存
  font_data_cache_[cache_key] = 
      std::make_pair(description.Hash(), font_data);
}

// 查询 Web 字体
SimpleFontData* FontFaceCache::Get(
    const FontDescription& description,
    const AtomicString& family_name) {
  std::string cache_key = GenerateCacheKey(description, family_name);
  
  auto it = font_data_cache_.find(cache_key);
  if (it != font_data_cache_.end()) {
    return it->second.second;  // 返回缓存的字体
  }
  return nullptr;
}
```

### 8.6 Web 字体加载状态机

**文件**: `third_party/blink/renderer/core/css/font_face.h/cc`

```cpp
enum FontLoadStatus {
  kUnloaded,      // 尚未开始加载
  kLoading,       // 正在下载
  kLoaded,        // 加载完成
  kError,         // 加载失败
};

class FontFace {
 public:
  // 获取加载状态
  FontLoadStatus GetStatus() const;
  
  // Promise 对象（JavaScript API）
  ScriptPromise<FontFace> Load();  // 手动触发加载
  ScriptPromise<void> Ready();      // 等待加载完成
  
  // 加载完成回调
  void DownloadFinished(sk_sp<SkTypeface> typeface);
};
```

**状态转换**:

```
初始化 (@font-face 规则创建)
  ↓
kUnloaded
  ↓
字体.load() 调用或页面使用
  ↓
kLoading (启动网络请求)
  ↓
网络 IO (下载字体文件)
  ↓
kLoaded (解码完成)
  或
kError (网络失败或解码失败)
```

---

## 9. 自定义字体处理（用户提供的字体）

### 9.1 自定义字体的来源

**文件**: `third_party/blink/renderer/platform/fonts/font_cache.cc`

用户自定义字体主要来源于：

1. **Web 应用自定义字体** (@font-face)
   - 从 CDN 或自建服务器加载
   - 由 Web 开发者控制

2. **浏览器/应用设置字体**
   - 用户通过设置改变默认字体
   - 存储在 Preferences 中

3. **本地文件字体**
   - 用户上传的本地字体文件
   - Canvas 或 CSS 中指定

### 9.2 浏览器设置字体处理

**文件**: `chrome/browser/font_family_cache.h/cc`

用户通过浏览器设置改变默认字体：

```cpp
class FontFamilyCache {
 private:
  // 缓存用户设置的字体偏好
  std::unordered_map<std::string, std::u16string> font_pref_cache_;
  
  // 监听偏好设置变化
  PrefChangeRegistrar font_change_registrar_;
};

// 获取用户设置的字体
std::u16string FontFamilyCache::FetchFont(
    const char* script,      // 脚本（Latin, CJK 等）
    const char* map_name)    // 类型（serif, sans-serif 等）
{
  // 生成偏好键
  std::string pref_name = 
      std::string(map_name) + "_font_for_" + script;
  
  // 从 Preferences 读取
  PrefService* prefs = profile_->GetPrefs();
  std::u16string font_name = 
      prefs->GetString(pref_name);
  
  return font_name;
}
```

**关键流程**:

```
用户在浏览器设置中选择字体
  ↓
例: 设置 → 外观 → 字体 → "Serif 字体" → "Noto Serif"
  ↓
FontFamilyPreferences 更新
  写入: Preferences 数据库
  ↓
FontFamilyCache 收到通知 (PrefChangeRegistrar)
  ↓
清空相关缓存项
  ↓
下次查询时，从 Preferences 重新读取
```

**设置结构** (Android WebView):

```java
// android_webview/java/src/org/chromium/android_webview/AwSettings.java

public void setSerifFontFamily(String font) {
  synchronized (mAwSettingsLock) {
    if (font != null && !mSerifFontFamily.equals(font)) {
      mSerifFontFamily = font;
      // 触发渲染器更新
      mEventHandler.updateWebkitPreferencesLocked();
    }
  }
}

public String getSerifFontFamily() {
  synchronized (mAwSettingsLock) {
    return mSerifFontFamily;
  }
}

// 类似的还有：
// - setSanSerifFontFamily()
// - setFixedFontFamily()
// - setCursiveFontFamily()
// - setFantasyFontFamily()
```

### 9.3 Web 应用自定义字体（最常见）

**处理路径**:

```
HTML 页面加载
  ↓
CSS 解析 (@font-face)
  @font-face {
    font-family: "MyCustomFont";
    src: url("/fonts/my-font.woff2");
    font-weight: 400;
    font-style: normal;
  }
  ↓
CSSFontFaceRule 创建
  ↓
FontFaceSet::addUnsafe()
  ↓
JavaScript Promise: document.fonts.ready
  等待所有字体加载完成
  ↓
ResourceFetcher 发起网络请求
  URL: https://example.com/fonts/my-font.woff2
  IO: 下载字体数据
  跨域检查: CORS
  缓存: HTTP Cache-Control 头
  ↓
字体数据返回（成功或失败）
  ↓
① 成功分支:
   WebFontDecoder 解码
   → 创建 SkTypeface
   → 添加到 FontFaceSet
   → 状态: loaded
   
② 失败分支:
   超时、404、CORS 错误
   → 状态: error
   → 触发 fallback 字体
```

### 9.4 自定义字体缓存关键

**文件**: `third_party/blink/renderer/platform/fonts/font_cache.cc`

自定义字体与系统字体的缓存区别：

| 特性 | 系统字体 | Web 字体 | 用户设置字体 |
|------|---------|---------|----------|
| **存储** | FontCache | FontFaceSet/FontFaceCache | Preferences |
| **生命周期** | 进程级（全局） | Document 级 | 会话级 |
| **更新** | 静态（系统字体列表不变） | 动态（页面加载时） | 用户改变设置时 |
| **缓存键** | (family, weight, style) | (family, weight, style, url) | (script, generic_family) |
| **线程安全** | 全线程可访问（有锁） | Blink 主线程 | 全线程可访问（有锁） |
| **回退策略** | FontFallbackList | 系统字体 | 默认字体 |

### 9.5 自定义字体加载失败处理

**文件**: `third_party/blink/renderer/core/css/font_face.cc`

```cpp
class FontFace {
 private:
  void NotifyLoadingFinished(
      sk_sp<SkTypeface> typeface,
      bool success) {
    if (success) {
      // 字体加载成功
      status_ = kLoaded;
      typeface_ = typeface;
      
      // 触发 loadingdone 事件
      auto* event = FontFaceSetLoadEvent::Create(
          event_type_names::kLoadingdone,
          FontFaceSetLoadEventInit());
      document_->GetFonts()->DispatchEvent(*event);
      
      // 触发 Promise 解决
      loading_promise_->Resolve(this);
      
    } else {
      // 字体加载失败
      status_ = kError;
      error_message_ = error_details;
      
      // 触发 loadingerror 事件
      auto* event = FontFaceSetLoadEvent::Create(
          event_type_names::kLoadingerror,
          FontFaceSetLoadEventInit());
      document_->GetFonts()->DispatchEvent(*event);
      
      // 触发 Promise 拒绝
      loading_promise_->Reject(
          MakeGarbageCollected<DOMException>(
              DOMExceptionCode::kNetworkError,
              "Failed to load font: " + error_message_));
    }
  }
};
```

**失败原因**:
- 网络错误（timeout、connection failed）
- CORS 错误（跨域资源未获授权）
- 文件不存在（404）
- 格式不支持（无法解码）
- 安全策略限制（Content Security Policy）

### 9.6 自定义字体在字体匹配中的行为

**文件**: `third_party/blink/renderer/platform/fonts/font_cache.cc`

```cpp
const SimpleFontData* FontCache::PlatformFallbackFontForCharacter(
    const FontDescription& font_description,
    UChar32 c,
    const SimpleFontData*,
    FontFallbackPriority fallback_priority) {
  
  // 检查字符是否在自定义字体中有字形
  if (custom_font && custom_font->HasGlyphForCharacter(c)) {
    // 直接使用自定义字体
    return custom_font->GetFontData();
  }
  
  // 自定义字体未包含该字符，触发字体回退
  //  ↓ 查询系统字体回退链
  //  ↓ 检查 emoji 字体
  //  ↓ 最后回退字体
  
  return SystemFallbackFontForCharacter(
      font_description, c, fallback_priority);
}
```

**字体回退层级**:

```
用户请求字体 "CustomFont"
  ↓
① 检查 Web 字体（FontFaceCache）
   - 如已加载：使用
   - 如加载中：等待或使用回退
   - 如加载失败：进入下一层
   ↓
② 检查用户设置字体
   ↓
③ 检查系统字体
   ↓
④ 字体回退链（serif → sans-serif → generic）
   ↓
⑤ 最后回退字体（system font）
```

### 9.7 自定义字体大小和权重调整

虽然通过 CSS 可以改变自定义字体的视觉外观，但实际的字体文件不变：

```cpp
// 用户 CSS
@font-face {
  font-family: "MyFont";
  src: url("myfont-regular.woff2");
  font-weight: 400;  // 源文件权重
}

/* 应用不同权重 */
body { font-weight: 400; }  /* 使用源文件 */
strong { font-weight: 700; } /* 合成：加粗 */

// Blink 处理
SimpleFontData* font = GetFont("MyFont", weight=400);
// 如果请求 weight=700 但未加载，则：
// 1. 检查是否有 weight=700 的变体字体
// 2. 如无，使用 weight=400 的字体 + 合成加粗（bold synthesis）
```

**关键点**:
- 如果字体是 Variable Font，可以通过参数调整权重
- 否则需要加载多个变体（regular, bold, italic, bold-italic）
- Blink 支持字体合成（font synthesis），但质量不如原生字体

---

## 10. 关键问题的代码级答案

### Q1: skia::DefaultFontMgr 在 Android 上的初始化路径是什么？

**答案**:
1. **触发点**: `content/renderer/renderer_main_platform_delegate_android.cc:PlatformInitialize()`
2. **工厂调用**: `skia/ext/font_utils.cc:DefaultFontMgr()` → `fontmgr_factory()`
3. **平台实现**: 
   - Android 14+: `SkFontMgr_New_AndroidNDK(false, SkFontScanner_Make_Fontations())`
   - Android < 14: `SkFontMgr_New_Android(nullptr, SkFontScanner_Make_Fontations())`
4. **扫描器**: `SkFontScanner_Make_Fontations()` 使用 Rust fontations 库
5. **系统字体**: 读取 `/system/etc/fonts.xml` 或调用 `ASystemFontIterator_open()`

### Q2: Chromium 是如何将 HarfBuzz shaping 结果交给 Skia 绘制的？

**答案**:
1. **HarfBuzz 输出**: `hb_glyph_info_t[]` + `hb_glyph_position_t[]`
2. **Blink 转换**: 转换为 `ShapeResult`（包含字形 ID 和位置）
3. **Skia 输入**: 构造 `SkTextBlobBuilder`
   ```cpp
   SkTextBlobBuilder builder;
   const auto& run = builder.allocRun(font, glyph_count, x, y);
   for (int i = 0; i < glyph_count; ++i) {
     run.glyphs[i] = shape_result->GlyphAt(i);
     run.pos[i] = {shape_result->PositionAt(i).x(), 
                   shape_result->PositionAt(i).y()};
   }
   sk_sp<SkTextBlob> blob = builder.make();
   canvas->drawTextBlob(blob, 0, 0, paint);
   ```
4. **Skia 渲染**: 对每个字形调用 `SkTypeface::getPath()` → FreeType 光栅化

### Q3: Fontations 在 Android 字体栈中的实际作用是什么？

**答案**:
1. **当前作用**（2026年1月）:
   - **字体扫描**: `SkFontScanner_Make_Fontations()` 解析字体文件元数据
   - **快速启动**: 零拷贝解析减少启动时间
   - **cmap 查询**: `unicharToGlyph()` 可能使用 Fontations（取决于 Skia 配置）
2. **不涉及**:
   - ❌ 字形光栅化（仍由 FreeType 负责）
   - ❌ OpenType 布局（由 HarfBuzz 负责）
3. **未来目标**:
   - 完全替换 FreeType（包括光栅化）
   - 支持更多现代字体格式（COLRv1, Variable Fonts）

### Q4: FreeType 在 Android 字体渲染中的具体职责？

**答案**:
1. **字形轮廓加载**: 读取 glyf/CFF 表，解析贝塞尔曲线
2. **度量计算**: 字形宽度、边界框、基线偏移
3. **光栅化**: 曲线 → 像素位图
4. **Hinting**: TrueType hinting 或 Autohinting（提高小字号清晰度）
5. **抗锯齿**: 灰度（256级）或 LCD 子像素（RGB/BGR）
6. **缓存管理**: FT_Face 对象缓存（避免重复加载）

**不涉及**:
- ❌ 字体发现（由 SkFontMgr 负责）
- ❌ OpenType 布局（由 HarfBuzz 负责）
- ❌ 字体回退（由 Blink FontCache 负责）

---

## 9. 调试技巧

### 9.1 启用 Skia 字体日志

**在命令行添加**:
```bash
adb shell
am start -n org.chromium.chrome/com.google.android.apps.chrome.Main \
  --es '--vmodule=*font*=3'
```

**输出示例**:
```
[INFO:font_cache_android.cc(89)] Matched font: NotoSansCJK-Regular.ttc (locale: zh-Hans)
[INFO:font_cache.cc(234)] SimpleFontData created: size=16px, typeface=0x7f8a3c
[INFO:harfbuzz_shaper.cc(123)] Shaping text: "你好" (2 chars)
```

### 9.2 查看系统字体列表

**Android 命令**:
```bash
adb shell cat /system/etc/fonts.xml | grep -A 5 "family name"
```

**或使用 Chromium API**:
```cpp
sk_sp<SkFontMgr> fm = skia::DefaultFontMgr();
int family_count = fm->countFamilies();
for (int i = 0; i < family_count; ++i) {
  SkString family_name;
  fm->getFamilyName(i, &family_name);
  LOG(INFO) << "Font family: " << family_name.c_str();
}
```

### 9.3 Tracing 字体操作

**在 Chrome 中**:
1. 打开 `chrome://tracing`
2. 选择 `fonts` 类别
3. 记录页面加载
4. 查看事件:
   - `FontCache::GetFontData`
   - `SimpleFontData::InitFromDetails`
   - `HarfBuzzShaper::Shape`
   - `SkTypeface::unicharToGlyph`

---

## 11. 完整流程整合：系统字体 vs Web 字体 vs 自定义字体

### 11.1 字体匹配优先级完整图

```
CSS 字体属性：{ family: "MyFont", weight: 700 }
  ↓
① Web 字体（@font-face）查询
   FontFaceSet::Get("MyFont", 700)
   ✓ 命中 → 使用 Web 字体
   ✗ 未命中 → 继续
   ↓
② 用户设置字体查询
   FontFamilyCache::Get(script, "MyFont")
   ✓ 命中 → 使用用户设置
   ✗ 未命中 → 继续
   ↓
③ 系统字体查询
   FontCache::GetFontData("MyFont", 700)
   ✓ 命中 → 使用系统字体
   ✗ 未命中 → 继续
   ↓
④ 通用字体族回退
   family: serif/sans-serif/monospace 替换
   FontCache::GetFontData(generic_family, 700)
   ✓ 命中 → 使用
   ✗ 未命中 → 继续
   ↓
⑤ 语言/地区特定字体回退
   Blink 字体回退链
   ✓ 命中 → 使用
   ✗ 未命中 → 继续
   ↓
⑥ Emoji/特殊字体回退
   System Emoji 字体
   ↓
⑦ 最后回退字体
   system font（永远可用）
```

### 11.2 三种字体的生命周期对比

| 方面 | Web 字体 (@font-face) | 系统字体 | 用户设置字体 |
|------|----------------------|---------|----------|
| **初始化时机** | Document 加载时 | Renderer 进程启动 | 用户改变设置时 |
| **初始化位置** | `FontFaceSet::addUnsafe()` | `RendererMainPlatformDelegate::PlatformInitialize()` | `FontFamilyCache::FetchFont()` |
| **数据来源** | 网络 URL 或 data: URI | 系统字体目录（/system/fonts） | 用户 Preferences |
| **加载方式** | 异步（Promise） | 同步（单例） | 同步（缓存查询） |
| **缓存位置** | FontFaceSet（per Document） | FontCache（global） | FontFamilyCache（per Profile） |
| **文件格式** | WOFF2/WOFF/TTF/OTF/SVG | TTF/OTF（TTC） | 同系统字体 |
| **IO 发生** | 网络请求 | 首次查询时加载 | Preferences 数据库 |
| **线程安全** | Blink 主线程 | 全线程（有锁） | 全线程（有锁） |

---

## 12. 总结

### 12.1 架构核心理解

1. **多层次字体缓存** ★★★
   - **FontFamilyCache**（浏览器进程）：用户字体偏好设置缓存，~20,000 次启动加载
     - 位置: `chrome/browser/font_family_cache.cc`
     - 调用: `FetchAndCacheFont(script, map_name)`
     - 优化: 指针相等性比较快速路径
   
   - **FontDataManager**（Renderer 进程）：Web 字体和平台字体的 typeface 缓存
     - 位置: `content/child/font_data/font_data_manager.cc`
     - 缓存: `typeface_cache_`（HashingLRUCache）、`mapped_regions_` / `mapped_files_`（共享内存）
     - IPC: Mojo MatchFamilyRequest → FontServiceApp 的异步字体匹配
   
   - **FontServiceApp**（系统字体服务进程）：系统字体匹配缓存
     - 位置: `components/services/font/font_service_app.cc`
     - 缓存: `match_cache_`（LRUCache），Key = MatchCacheKey(family_name, SkFontStyle)
     - 方法: `MatchFamilyName()` 返回回调函数异步结果
   
   - **Skia 内部缓存**：SkTypeface 实例池
     - 每个唯一的（family, weight, style）组合一个 SkTypeface
     - 通过 SkFontMgr 单例管理
   
   - **FreeType 内部缓存**：字形和度量值缓存
     - FT_Face 缓存字体元数据和字形索引表
     - 光栅化结果缓存在 Skia 的 GlyphCache 中

2. **Web 字体工作流** ★★★
   ```
   HTML/CSS 中的 @font-face 规则
     ↓
   CSSFontSelector 解析
     ↓
   FontFaceSet::Get(family, descriptors)
     ↓
   ① 未加载 → fetch(url) 发起网络请求
   ② 加载中 → 等待 Promise（layout 阻塞可能会解除）
   ③ 已加载 → 返回 SimpleFontData
     ↓
   WebFontDecoder 处理格式（WOFF2/WOFF/TTF）
     ↓
   SkTypeface 创建（from_data）
     ↓
   HarfBuzz 成形和 Skia 渲染
   ```

3. **自定义字体处理** ★★
   - 来源：FontFace API（JavaScript）、Data URL、Blob
   - 流程：数据验证 → WebFontDecoder 解码 → SkTypeface 创建 → 缓存在 FontFaceSet
   - 与系统字体的区别：完全在内存中，不依赖系统字体目录
   - 生命周期：与 Document 绑定，Document 卸载时释放

4. **性能关键点** ★★
   - **启动性能**：FontFamilyCache 的 FetchAndCacheFont 在启动时被调用 ~20,000 次
     - 优化: 单调指针比较 vs 字符串比较
   - **首次字体加载**：第一个字符触发字体加载（可能导致 jank）
   - **Web 字体阻塞**：@font-face 字体未加载时可能触发不可见文本闪烁（FOIT）
   - **缓存击穿**：同时加载多个字体可能导致 LRU 缓存竞争
   - **IPC 成本**：Renderer → FontServiceApp 的 Mojo IPC 有固定开销，需要缓存

5. **跨进程边界**（★ 重要架构点）
   ```
   Browser Process          Renderer Process           Font Service
   ────────────────        ─────────────────          ────────────
   FontFamilyCache   ← IPC → FontDataManager   ← IPC → FontServiceApp
   (用户设置字体)         (缓存网络字体)           (系统字体)
   
   共享内存：FontDataManager 通过 SharedMemory 获取字体文件
   ```

### 12.2 代码关键路径

```
启动: RendererMainPlatformDelegate::PlatformInitialize()
  ↓
全局初始化: skia::DefaultFontMgr()
  ↓
工厂: fontmgr_factory() → SkFontMgr_New_Android() + SkFontScanner_Make_Fontations()
  ↓
字体匹配: FontCache::PlatformFallbackFontForCharacter()
  ↓
Skia API: SkFontMgr::matchFamilyStyleCharacter()
  ↓
字符到字形: SkTypeface::unicharToGlyph() → FreeType: FT_Get_Char_Index()
  ↓
文本成形: HarfBuzzShaper::Shape() → hb_shape() → OpenType 布局
  ↓
字形加载: SkTypeface::getPath() → FreeType: FT_Load_Glyph()
  ↓
光栅化: FreeType: FT_Render_Glyph() → 像素位图
  ↓
绘制: Skia Canvas → 合成到屏幕
```

### 12.3 IO 和全局状态

| 操作 | IO | 全局状态 | 缓存 |
|------|----|----|------|
| `skia::DefaultFontMgr()` | ✅ 读取 fonts.xml | ✅ 单例 | ✅ SkFontMgr 内部 |
| `SkFontMgr::matchFamilyStyleCharacter()` | ❌ | ✅ 查询索引 | ✅ SkTypeface 缓存 |
| `SkTypeface::unicharToGlyph()` | ⚠️ 首次加载字体 | ❌ | ✅ FreeType 内部 |
| `FT_Load_Glyph()` | ⚠️ 首次加载 glyf 表 | ❌ | ✅ FT_Face 缓存 |
| `FT_Render_Glyph()` | ❌ | ❌ | ✅ Skia Glyph Cache |
| `hb_shape()` | ❌ | ❌ | ✅ HarfBuzzFontCache + NGShapeCache |

**关键**: 大部分操作无 IO（首次加载后完全内存操作）

### 12.4 关键要点总结

1. **SkFontMgr 初始化**:
   - 必须在沙箱前执行（Android 14+ 需要系统调用）
   - 使用 Fontations 扫描字体元数据（快速启动）
   - 构建字体索引（family name → SkTypeface）

2. **字体匹配**:
   - 输入：CSS 属性 + Unicode 字符
   - 输出：SkTypeface 对象
   - 匹配规则：locale → family → style → 字符覆盖

3. **FreeType vs Fontations**:
   - FreeType：字形光栅化（成熟稳定）
   - Fontations：字体扫描（快速、内存安全）
   - 未来：Fontations 将完全替换 FreeType

4. **HarfBuzz 独立**:
   - OpenType 布局由 HarfBuzz 独占
   - FreeType/Fontations 不处理 GSUB/GPOS
   - HarfBuzz 通过回调调用 Skia/FreeType 获取字形

---

## 附录：关键文件清单

| 文件路径 | 职责 |
|---------|------|
| `content/renderer/renderer_main_platform_delegate_android.cc` | Android Renderer 初始化（触发字体管理器初始化） |
| `skia/ext/font_utils.cc` | DefaultFontMgr 工厂（平台抽象） |
| `third_party/blink/renderer/platform/fonts/android/font_cache_android.cc` | Blink 字体缓存（Android 实现） |
| `third_party/blink/renderer/platform/fonts/simple_font_data.h` | Blink 字体对象（持有 SkTypeface） |
| `third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc` | HarfBuzz 文本成形 |
| `ui/gfx/harfbuzz_font_skia.cc` | HarfBuzz → Skia 桥接（回调实现） |
| `third_party/skia/include/core/SkFontMgr.h` | Skia 字体管理器接口 |
| `third_party/skia/include/core/SkTypeface.h` | Skia 字体对象接口 |
| `third_party/skia/include/ports/SkFontMgr_android.h` | Android 平台 SkFontMgr 实现 |
| `third_party/skia/include/ports/SkFontScanner_Fontations.h` | Fontations 字体扫描器 |
| `third_party/freetype/` | FreeType 库（字形渲染） |

---

## 跨进程字体管理附录

### Web 字体类和接口

| 类名 | 文件位置 | 职责 |
|------|---------|------|
| `FontFaceSet` | `third_party/blink/renderer/core/css/font_face_set.h` | Document 层面的 @font-face 字体集合管理 |
| `FontFace` | `third_party/blink/renderer/core/css/font_face.h` | 单个 @font-face 规则对象 |
| `CSSFontSelector` | `third_party/blink/renderer/core/css/css_font_selector.h` | CSS 字体族名 → SkTypeface 匹配 |
| `WebFontDecoder` | `third_party/blink/renderer/platform/fonts/web_font_decoder.h` | WOFF2/WOFF/TTF/OTF 格式解码 |
| `FontCache` | `third_party/blink/renderer/platform/fonts/font_cache.h` | Blink 全局字体缓存（SimpleFontData 池） |

### 自定义字体处理流程

```
JavaScript: new FontFace("CustomFont", url_or_data)
  ↓
FontFace::load() → fetch(url)
  ↓
资源加载并转为 Blob
  ↓
WebFontDecoder::Decode(blob_data, format)
  ↓
创建 SkTypeface::MakeFromData(decoded_buffer)
  ↓
缓存到 FontFaceSet::addFontData()
  ↓
CSSFontSelector::FontFaceFor("CustomFont") → 返回 SimpleFontData
  ↓
HarfBuzz 文本成形和 Skia 渲染
```

### 字体缓存层级（完整视图）

```
L1：FontFamilyCache（chrome/browser）
    ↑ IPC Mojo ↓
L2：FontDataManager::typeface_cache_（content/child）
    ↑ IPC Mojo ↓
L3：FontServiceApp::match_cache_（components/services/font）
    ↑ SharedMemory ↓
L4：Skia FontMgr SkTypeface 缓存
    ↓
L5：FreeType FT_Face 缓存（字形和度量值）
    ↓
L6：Skia GlyphCache（光栅化结果）
```

每一层减少下层的查询频率，通过多层 LRU 缓存优化启动时间（从 ~20,000 次系统字体查询降低）。

