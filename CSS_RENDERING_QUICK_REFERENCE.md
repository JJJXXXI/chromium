# CSS 到字体渲染快速参考

## 🎯 5 分钟快速理解

### 完整数据流

```
HTML input
  ↓ [HTMLParser]
DOM 树
  ↓ [StyleResolver]
ComputedStyle (包含 FontDescription)
  ↓ [FontBuilder::CreateFont()]
Font 对象 + FontDescription
  ↓ [FontFallbackList::GetFontData()]
SimpleFontData (首次使用字符时加载)
  ↓ [HarfBuzzShaper::Shape()]
字形 (Glyph) 位置信息
  ↓ [GraphicsContext::DrawText()]
屏幕像素 🎉
```

---

## 核心类关系图

```
ComputedStyle (样式对象)
    ├─ FontDescription font_description_  ← 字体配置
    └─ Font font_                          ← 当前字体
         ├─ FontDescription font_description_
         └─ FontFallbackList fallback_list_
              ├─ Vector<FontData*> font_list_
              │   ├─ SimpleFontData ("Georgia")
              │   ├─ SimpleFontData ("Serif系统字体")
              │   └─ SimpleFontData ("系统默认")
              └─ HashMap<UChar32, SimpleFontData*> cache_

FontDescription (字体配置)
    ├─ FontFamily* family       ← 链表: "Georgia" → "serif" → nullptr
    ├─ GenericFamilyType       ← kNoFamily / kSerifFamily / ...
    ├─ float size
    ├─ int weight
    └─ FontStyle style

FontFamily (链表节点)
    ├─ AtomicString family_name_  ← "Georgia" 或 "serif"
    ├─ Type type_                 ← kFamilyName 或 kGenericFamily
    └─ FontFamily* next_          ← 下个 fallback
```

---

## 关键代码位置速查表

| 操作 | 文件 | 行数 | 关键函数 |
|-----|------|------|---------|
| CSS 解析 font-family | `css_parser.cc` | - | `CSSParser::ParseValue()` |
| CSS 值转换 | `style_builder_converter.cc` | 510-590 | `ConvertFontFamily()` ⭐ |
| 应用 CSS 属性 | `style_cascade.cc` | 50-150 | `StyleCascade::Apply()` |
| 创建 Font 对象 | `font_builder.cc` | - | `FontBuilder::CreateFont()` |
| 字体加载 | `font_cache.cc` | - | `FontCache::GetFontData()` |
| 平台查询 | `font_cache_skia.cc` | - | `CreateFontPlatformData()` |
| Fallback 查询 | `font_fallback_list.cc` | - | `GetFontData()` |
| 文本 Shaping | `harfbuzz_shaper.cc` | - | `HarfBuzzShaper::Shape()` |
| 绘制 | `graphics_context.cc` | - | `DrawText()` |
| Android 特殊 | `font_cache_android.cc` | - | `GetGenericFamilyNameForScript()` |

---

## CSS 值转换详解

### 示例: `font-family: Georgia, serif, sans-serif`

#### Step 1: CSS 解析输出

```
CSSValueList [
  CSSFontFamilyValue("Georgia"),      ← 具体字体
  CSSIdentifierValue(serif),          ← 通用族
  CSSIdentifierValue(sans-serif)      ← 通用族
]
```

#### Step 2: 转换过程 (反向!)

```
StyleBuilderConverter::ConvertFontFamily() {
  // 从末尾开始反向处理
  
  1. CSSIdentifierValue(sans-serif)
     → generic_family = kSansSerifFamily
     → family_name = GenericFontFamilyName(kSansSerifFamily)
     → family_name = "Roboto" (Android) 或 "DejaVu Sans" (Linux)
  
  2. CSSIdentifierValue(serif)
     → generic_family = kSerifFamily
     → family_name = "Noto Serif"
     → 链接: "Noto Serif" → next: "Roboto"
  
  3. CSSFontFamilyValue("Georgia")
     → generic_family = kNoFamily
     → family_name = "Georgia"
     → 链接: "Georgia" → next: "Noto Serif" → "Roboto"
}
```

#### Step 3: 最终 FontDescription

```cpp
FontDescription {
  family: FontFamily {
    family_name: "Georgia"
    type: kFamilyName
    next: → FontFamily {
      family_name: "Noto Serif"
      type: kGenericFamily
      next: → FontFamily {
        family_name: "Roboto"
        type: kGenericFamily
        next: nullptr
      }
    }
  }
}
```

#### Step 4: 字体查询顺序

```
字符 'A' 需要字体时:
  1. 尝试加载 "Georgia" → SkFontMgr::matchFamilyStyle()
     ✓ 成功 → 返回 SimpleFontData("Georgia")
     ✗ 失败 → 继续

  2. 尝试加载 "Noto Serif"
     ✓ 成功 → 返回 SimpleFontData("Noto Serif")

  3. 尝试加载 "Roboto"
  
  4. 系统 fallback (总是成功)
```

---

## 数据结构字段详解

### FontDescription 关键字段

```cpp
class FontDescription {
 public:
  enum GenericFamilyType {
    kNoFamily = 0,           // 未指定通用族 (具体字体)
    kStandardFamily = 1,     // serif (衬线体)
    kSerifFamily = 2,
    kSansSerifFamily = 3,
    kMonospaceFamily = 4,
    kCursiveFamily = 5,
    kFantasyFamily = 6,
    kSystemUiFamily = 7,
  };

 private:
  FontFamily* family_;              // ← 字体名链表
  GenericFamilyType generic_family_; // ← 0 (具体) 或 1-7 (通用族)
  float size_;                       // ← 像素大小
  int weight_;                       // ← 100-900
  FontStyle style_;                  // ← normal/italic/oblique
};
```

**示例**:
```cpp
// "Georgia" 字体
FontDescription {
  family: "Georgia",
  generic_family: kNoFamily,           // ← 0 (不是通用族!)
  size: 16.0,
  weight: 400,
  style: normal
}

// "serif" (无 CSS 时的默认)
FontDescription {
  family: "",                          // ← 空!
  generic_family: kStandardFamily,     // ← 1 (衬线体)
  size: 16.0,
  weight: 400,
  style: normal
}
```

---

## 字体加载流程

### 完整调用链

```
Font::FontDataForCharacter('A')
  ↓
FontFallbackList::GetFontData('A')
  ├─ 查询缓存 HashMap<UChar32, SimpleFontData*>
  ├─ 缓存未命中
  └─ 遍历 font_list_ (可能多个字体)
      ↓
      FontSelector::GetFontData(FontDescription, "Georgia")
      ├─ 判断是否通用族
      ├─ 查询 GenericFontFamilySettings 映射 (如果是通用族)
      └─ 调用 FontCache
      ↓
      FontCache::GetFontData()
      ├─ 查询全局缓存 font_data_cache_
      ├─ 缓存未命中
      └─ 调用 CreateFontPlatformData()
      ↓
      FontCache::CreateFontPlatformData()
      ├─ 获取 SkFontMgr 实例
      └─ SkFontMgr::matchFamilyStyle("Georgia", style)
         ├─ 查询系统字体
         └─ 返回 SkTypeface*
      ↓
      new SimpleFontData(FontPlatformData)
      ├─ 包装 SkTypeface
      └─ 初始化字体度量
      ↓
      存入缓存
      └─ 返回 SimpleFontData*
```

### 性能优化

```cpp
// 第1次使用 'A': 加载字体文件
char 'A' → FontCache 查询 → SkFontMgr 系统调用 → 磁盘IO ⚠️

// 第2-Nth 次使用 'A': 直接缓存
char 'A' → FontFallbackList 缓存命中 → 返回 ✓ (纳秒级)

// 不同元素使用同一字体:
element1.font → FontCache 全局缓存 ✓
element2.font → 同一个 SimpleFontData* (共享内存)
```

---

## Android 特殊处理

### 流程对比

```
Linux/Windows:
  "Georgia" → SkFontMgr → 系统字体库 (/usr/share/fonts)
  
Android:
  "Georgia" → SkFontMgr_Android → /system/etc/fonts.xml
            → /system/fonts/ + /product/fonts/ + /odm/fonts/
```

### Android fonts.xml 结构

```xml
<familyset>
  <family name="sans-serif">
    <font weight="400" style="normal">Roboto-Regular.ttf</font>
    <font weight="400" style="italic">Roboto-Italic.ttf</font>
    <font weight="700" style="normal">Roboto-Bold.ttf</font>
  </family>

  <family name="serif">
    <font weight="400" style="normal">NotoSerifRegular.ttf</font>
  </family>
</familyset>
```

### 映射查询

```cpp
// 在 Chromium 中查询通用族
FontCache::GetGenericFamilyNameForScript(
    "sans-serif",            // 通用族名
    font_description,
    content_locale)
  ↓
SkFontMgr_Android::GetFamiliesData()
  ↓
解析 /system/etc/fonts.xml
  ↓
查找 <family name="sans-serif">
  ↓
返回第一个字体: "Roboto-Regular.ttf"
  ↓
加载 /system/fonts/Roboto-Regular.ttf
```

---

## 常用查询代码

### 查询某字符的字体

```cpp
// 获取元素的计算样式
const ComputedStyle* style = element->GetComputedStyle();

// 获取 Font 对象
const Font& font = style->GetFont();

// 查询字符 'A' 的 SimpleFontData
const SimpleFontData* font_data = font.FontDataForCharacter('A');

// 获取该字体的 Glyph ID
Glyph glyph = font_data->GlyphForCharacter('A');

// 获取 Glyph 宽度
float width = font_data->WidthForGlyph(glyph);
```

### 访问 FontDescription

```cpp
const FontDescription& font_desc = style->GetFontDescription();

// 字体族链表
const FontFamily& family = font_desc.Family();

// 通用族 (如果是)
FontDescription::GenericFamilyType generic = font_desc.GenericFamily();

// 字体大小
float size = font_desc.Size();

// 字重
int weight = font_desc.Weight();

// 字体风格
FontStyle style = font_desc.GetStyle();
```

### 调试: 打印字体链表

```cpp
const Font& font = element->GetComputedStyle()->GetFont();
const FontFallbackList* fallback_list = font.GetFallbackList();

for (const auto& font_data : fallback_list->font_list_) {
  DLOG(INFO) << "FontData: " << font_data->GetFamily();
}
```

---

## 关键概念速记

| 概念 | 说明 | 示例 |
|-----|------|------|
| **FontDescription** | 字体配置描述 | {family: "Georgia", size: 16, weight: 400} |
| **FontFamily** | 字体族链表节点 | "Georgia" → "serif" → nullptr |
| **GenericFamily** | CSS 通用族枚举 | serif, sans-serif, monospace 等 |
| **Font** | 运行时字体对象 | 包含 FontDescription 和 FontFallbackList |
| **SimpleFontData** | 单个字体文件 | 包含 SkTypeface 和字体度量 |
| **FontFallbackList** | Fallback 链管理 | [SimpleFontData1, SimpleFontData2, ...] |
| **FontCache** | 全局字体缓存 | 跨应用生存期缓存 SimpleFontData |
| **FontSelector** | 字体选择和映射 | 处理通用族映射和平台查询 |
| **SkTypeface** | Skia 层字体对象 | 包装系统字体文件 |
| **Glyph** | 字形 ID | 字符 'A' → Glyph 68 |

---

## 常见错误和陷阱

### ❌ 错误 1: 混淆 FontDescription 和 Font

```cpp
// 错误
FontDescription desc = ...;
desc->FontDataForCharacter('A');  // ✗ FontDescription 没有该方法!

// 正确
Font font(desc);
font.FontDataForCharacter('A');   // ✓ 使用 Font 对象
```

### ❌ 错误 2: 忘记 generic_family 的含义

```cpp
// generic_family = 0 (kNoFamily)
// 表示: 这是具体字体名 ("Georgia", "Arial")
// 不是: 字体无效或未设置!

// generic_family = 2 (kSerifFamily)
// 表示: 这是通用族 ("serif")
// 需要: 映射到具体字体名 ("Noto Serif")
```

### ❌ 错误 3: 在 UpdateFont() 前访问 Font

```cpp
// 错误
ComputedStyle style;
// Font 还没创建!
const SimpleFontData* data = style.GetFont().PrimaryFont();

// 正确
// 需要等待 UpdateFont() 被调用 (在 StyleResolverState::UpdateFont 中)
```

### ❌ 错误 4: 忘记字体是按需加载的

```cpp
// Font 对象创建快速
Font font(description);  // ✓ 毫秒级

// 但第一个字符可能触发磁盘 IO
font.FontDataForCharacter('A');  // ⚠️ 可能慢 (第一次)
                                  // ✓ 快 (缓存命中)
```

---

## 调试技巧

### 1. 在 FontCache::GetFontData() 添加日志

```cpp
// File: third_party/blink/renderer/platform/fonts/font_cache.cc

const SimpleFontData* FontCache::GetFontData(
    const FontDescription& font_description,
    const AtomicString& family_name) {
  
  DLOG(INFO) << "FontCache::GetFontData("
             << family_name.Utf8().c_str() << ", "
             << "size=" << font_description.Size() << ", "
             << "weight=" << font_description.Weight() << ")";
  
  // ... 原代码 ...
}
```

### 2. 观察字体加载

```bash
# 在 Chromium 启动时启用日志
./out/Default/chrome \
  --enable-logging=stderr \
  --v=2 \
  --vmodule=font_cache=2
```

### 3. 检查缓存命中率

```cpp
// 添加到 FontCache 类
size_t cache_hits_ = 0;
size_t cache_misses_ = 0;

// 在 GetFontData 中:
if (cached) {
  cache_hits_++;
  DLOG(INFO) << "FontCache hit rate: " 
             << 100.0 * cache_hits_ / (cache_hits_ + cache_misses_) << "%";
}
```

---

## 完整调用栈示例

从 CSS `font-family: Georgia` 到屏幕像素:

```
1. StyleBuilder::ApplyProperty(kFontFamily, CSSValueList(...))
   └─ StyleResolverState::UpdateFont()
      └─ FontBuilder::CreateFont()
         └─ new Font(FontDescription)

2. Element::RecalcStyle() 完成

3. LayoutText::ComputeTextMetrics()
   └─ Font::FontDataForCharacter('G')
      └─ FontFallbackList::GetFontData('G')
         └─ FontSelector::GetFontData(FontDescription, "Georgia")
            └─ FontCache::GetFontData(FontDescription, "Georgia")
               └─ SkFontMgr::matchFamilyStyle("Georgia", style)

4. TextLayout::ShapeText("Georgia")
   └─ HarfBuzzShaper::Shape()
      └─ hb_shape(hb_font, buffer)

5. PaintContext::DrawText()
   └─ GraphicsContext::DrawText()
      └─ SkCanvas::drawGlyphs()
         └─ GPU: RasterizeGlyphs()
            └─ Screen: Pixels ✅
```

---

## 性能指标

| 操作 | 耗时 | 频率 | 注意 |
|-----|------|------|------|
| 样式计算 (UpdateFont) | 纳秒 | 元素重排时 | 仅构建 FontDescription |
| 字体加载 (第1次) | 毫秒 | 字符首次使用 | 磁盘 IO |
| 字体加载 (缓存) | 纳秒 | 字符再次使用 | FastPath |
| 文本 Shaping | 微秒 | 每段文本 | HarfBuzz 优化 |
| 屏幕绘制 | 毫秒 | 每帧 | GPU 加速 |

---

## 参考链接

- 完整指南: [CHROMIUM_CSS_TO_RENDERING_COMPREHENSIVE_GUIDE.md](CHROMIUM_CSS_TO_RENDERING_COMPREHENSIVE_GUIDE.md)
- 初始化流程: [SYSTEM_FONTS_TO_FONTCACHE_FLOW.md](SYSTEM_FONTS_TO_FONTCACHE_FLOW.md)
- Android 特殊: [ANDROID_FONT_SELECTION_COMPLETE_FLOW.md](ANDROID_FONT_SELECTION_COMPLETE_FLOW.md)
- 源代码: `third_party/blink/renderer/platform/fonts/`
