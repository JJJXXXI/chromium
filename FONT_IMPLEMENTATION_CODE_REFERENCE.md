# Chromium 字体系统实战代码参考

## 📖 目录

1. [核心 API 使用示例](#核心-api-使用示例)
2. [实战场景解决方案](#实战场景解决方案)
3. [调试技巧](#调试技巧)
4. [性能优化](#性能优化)
5. [常见错误修复](#常见错误修复)
6. [完整工作流示例](#完整工作流示例)

---

## 核心 API 使用示例

### 1. 获取元素的字体

```cpp
// 场景: 需要查询某个 DOM 元素使用的字体

// 方法 1: 直接从 ComputedStyle
Element* element = ...;
const ComputedStyle* style = element->GetComputedStyle();
if (!style) {
  return;  // 元素未布局
}

const Font& font = style->GetFont();
DLOG(INFO) << "Font size: " << font.GetFontDescription().Size();

// 方法 2: 通过 LayoutObject
LayoutObject* layout_obj = element->GetLayoutObject();
if (layout_obj) {
  const Font& font = layout_obj->StyleRef().GetFont();
  // ...
}
```

### 2. 查询字体支持的字符

```cpp
// 场景: 判断字体是否支持某个 Unicode 字符

Element* element = ...;
const Font& font = element->GetComputedStyle()->GetFont();

UChar32 char_code = 'A';  // 或任何 Unicode 代码点, 如 0x4E2D (中)

// 查询能渲染该字符的 SimpleFontData
const SimpleFontData* font_data = font.FontDataForCharacter(char_code);

if (!font_data) {
  DLOG(WARNING) << "No font found for character: " << char_code;
  return;
}

// 获取该字体的名称 (如果可用)
// 注: SimpleFont 没有直接的 GetName(), 需要从 platform_data_ 获取

// 获取该字符的 Glyph ID
Glyph glyph_id = font_data->GlyphForCharacter(char_code);

// 获取 Glyph 的度量信息
float glyph_width = font_data->WidthForGlyph(glyph_id);
gfx::RectF glyph_bounds = font_data->BoundsForGlyph(glyph_id);

DLOG(INFO) << "Glyph width: " << glyph_width
           << ", bounds: " << glyph_bounds.ToString();
```

### 3. 获取字体描述信息

```cpp
// 场景: 需要了解当前字体的详细配置

const ComputedStyle* style = element->GetComputedStyle();
const FontDescription& font_desc = style->GetFontDescription();

// 字体大小
float size = font_desc.Size();
DLOG(INFO) << "Font size: " << size << "px";

// 字重 (100-900)
int weight = font_desc.Weight();
DLOG(INFO) << "Font weight: " << weight;

// 字体风格 (normal, italic, oblique)
FontStyle style_enum = font_desc.GetStyle();
DLOG(INFO) << "Font style: " 
           << (style_enum == kNormalStyle ? "normal" : "italic/oblique");

// 字体族列表
const FontFamily& family = font_desc.Family();
DLOG(INFO) << "First family name: " << family.FamilyName().Utf8();

// 遍历整个字体族链表
for (const FontFamily* fam = &family; fam && *fam; 
     fam = &fam->Next()) {
  DLOG(INFO) << "  Fallback: " << fam->FamilyName().Utf8()
             << " (type=" << (int)fam->GetType() << ")";
}

// 通用族类型
FontDescription::GenericFamilyType generic = 
    font_desc.GenericFamily();
DLOG(INFO) << "Generic family: " << (int)generic
           << " (0=none, 1=serif, 3=sans-serif, 4=monospace)";

// 脚本类型 (用于选择合适的字体)
UScriptCode script = font_desc.GetScript();
DLOG(INFO) << "Script: " << uscript_getName(script);
```

### 4. 创建自定义字体描述

```cpp
// 场景: 需要以编程方式创建 Font 对象

FontDescription custom_desc;

// 设置字体大小
custom_desc.SetSize(14.0f);

// 设置字重
custom_desc.SetWeight(700);  // Bold

// 设置风格
custom_desc.SetStyle(kItalicStyle);

// 设置字体族
// 方式 1: 具体字体
custom_desc.SetFamily(
    FontFamily(AtomicString("Georgia")));

// 方式 2: 通用族 (需要 GenericFontFamilySettings 映射)
custom_desc.SetGenericFamily(FontDescription::kSerifFamily);

// 创建 Font 对象
Font custom_font(custom_desc);

// 使用 Font
const SimpleFontData* font_data = 
    custom_font.FontDataForCharacter('A');
```

### 5. 访问 FontFallbackList

```cpp
// 场景: 需要查看字体的完整 fallback 链

const Font& font = style->GetFont();
const FontFallbackList* fallback_list = font.GetFallbackList();

if (!fallback_list) {
  return;
}

// 获取主字体 (优先级最高)
const SimpleFontData* primary = fallback_list->GetPrimaryFont();
DLOG(INFO) << "Primary font loaded";

// 遍历完整的 fallback 链
// 注: FontFallbackList::font_list_ 是 private,需要用公共接口
for (UChar32 test_char = 'A'; test_char <= 'z'; test_char++) {
  const SimpleFontData* font_for_char = 
      fallback_list->FontDataForCharacter(test_char);
  // 不同字符可能使用不同的 fallback
}

// 查询是否有自定义字体 (@font-face)
const SimpleFontData* font_data = 
    fallback_list->FontDataForCharacter('A');
if (font_data && font_data->IsCustomFont()) {
  DLOG(INFO) << "Using custom font from @font-face";
}

// 查询是否正在加载字体
if (font_data && font_data->IsLoading()) {
  DLOG(INFO) << "Font is still loading...";
}
```

### 6. 查询系统字体

```cpp
// 场景: Android 特定 - 查询系统字体

#if BUILDFLAG(IS_ANDROID)

FontCache* font_cache = FontCache::GetFontCache();

FontDescription android_desc;
android_desc.SetSize(16.0f);
android_desc.SetFamily(
    FontFamily(AtomicString("sans-serif")));

// 查询 Android 系统字体
const SimpleFontData* system_font = 
    font_cache->GetFontData(android_desc, 
                           AtomicString("sans-serif"));

if (system_font) {
  DLOG(INFO) << "Android system font loaded";
}

// 或者查询通用族映射
AtomicString generic_family = 
    FontCache::GetGenericFamilyNameForScript(
        "sans-serif",
        android_desc,
        content_locale);

DLOG(INFO) << "Mapped to: " << generic_family.Utf8();

#endif
```

---

## 实战场景解决方案

### 场景 1: 实现字体选择器 UI

**问题**: 需要显示当前元素使用的字体列表

```cpp
// chrome/browser/ui/views/font_selector.cc

Vector<String> GetAvailableFonts() {
  Vector<String> fonts;
  
  FontCache* cache = FontCache::GetFontCache();
  
  // Android: 从 fonts.xml 读取
  #if BUILDFLAG(IS_ANDROID)
    fonts.push_back("sans-serif");
    fonts.push_back("serif");
    fonts.push_back("monospace");
    fonts.push_back("cursive");
    fonts.push_back("fantasy");
  #else
    // 其他平台: 需要调用系统 API
    // Linux: 解析 /etc/fonts/fonts.conf
    // Windows: 查询注册表
    // macOS: 调用 CTFontManager
  #endif
  
  return fonts;
}

String GetCurrentFont(Element* element) {
  if (!element) {
    return "";
  }
  
  const ComputedStyle* style = element->GetComputedStyle();
  if (!style) {
    return "";
  }
  
  const FontDescription& font_desc = 
      style->GetFontDescription();
  const FontFamily& family = font_desc.Family();
  
  return family.FamilyName().String();
}

void SetFontFamily(Element* element, const String& font_name) {
  if (!element) {
    return;
  }
  
  // 通过修改样式属性来应用字体
  auto* style_attr = 
      element->GetAttribute(html_names::kStyleAttr);
  
  String new_style;
  if (style_attr) {
    new_style = style_attr->String();
  }
  
  // 添加或更新 font-family
  // 简化实现 (生产环境需要更完善的 CSS 解析)
  new_style = "font-family: " + font_name + ";";
  
  element->SetAttribute(html_names::kStyleAttr, 
                       AtomicString(new_style));
  
  // 触发样式重计算
  element->GetDocument().GetStyleEngine().UpdateStyle();
}
```

### 场景 2: 字体回退链调试

**问题**: 文本显示异常,需要追踪使用了哪个 fallback 字体

```cpp
// third_party/blink/renderer/platform/fonts/font.cc

void Font::DumpFallbackChain(const TextRun& run) {
  DLOG(INFO) << "=== Font Fallback Chain Debug ===";
  DLOG(INFO) << "Text: " << run.text << " (length: " << run.length << ")";
  
  const FontDescription& desc = GetFontDescription();
  DLOG(INFO) << "Font size: " << desc.Size();
  DLOG(INFO) << "Font weight: " << desc.Weight();
  
  // 遍历每个字符,查看使用的字体
  for (unsigned i = 0; i < run.length; i++) {
    UChar32 char_code = run.text[i];
    const SimpleFontData* font_data = 
        FontDataForCharacter(char_code);
    
    if (font_data) {
      Glyph glyph = font_data->GlyphForCharacter(char_code);
      float width = font_data->WidthForGlyph(glyph);
      
      DLOG(INFO) << "Char[" << i << "]: U+" 
                 << std::hex << char_code << std::dec
                 << " (Glyph: " << glyph << ", Width: " << width << ")";
    } else {
      DLOG(WARNING) << "Char[" << i << "]: U+" 
                    << std::hex << char_code 
                    << std::dec << " - NO FONT FOUND!";
    }
  }
  
  DLOG(INFO) << "=== End Chain Debug ===";
}
```

### 场景 3: 性能分析 - 字体缓存命中率

**问题**: 字体加载性能差,需要分析缓存是否有效

```cpp
// third_party/blink/renderer/platform/fonts/font_cache.cc

class FontCacheMetrics {
 private:
  size_t total_lookups_ = 0;
  size_t cache_hits_ = 0;
  size_t cache_misses_ = 0;
  size_t font_loads_ = 0;
  
 public:
  const SimpleFontData* GetFontData(
      const FontDescription& desc,
      const AtomicString& family_name) {
    total_lookups_++;
    
    FontCacheKey key(desc, family_name);
    
    // 查询缓存
    auto it = font_data_cache_.find(key);
    if (it != font_data_cache_.end()) {
      cache_hits_++;
      return it->second;
    }
    
    cache_misses_++;
    font_loads_++;
    
    // 创建新字体...
    // ...
  }
  
  void DumpMetrics() {
    double hit_rate = 100.0 * cache_hits_ / total_lookups_;
    DLOG(INFO) << "FontCache Metrics:";
    DLOG(INFO) << "  Total lookups: " << total_lookups_;
    DLOG(INFO) << "  Cache hits: " << cache_hits_;
    DLOG(INFO) << "  Cache misses: " << cache_misses_;
    DLOG(INFO) << "  Font loads: " << font_loads_;
    DLOG(INFO) << "  Hit rate: " << std::fixed 
               << std::setprecision(1) << hit_rate << "%";
    
    if (hit_rate < 80.0) {
      DLOG(WARNING) << "Low cache hit rate! Consider optimization.";
    }
  }
};
```

### 场景 4: Android 自定义字体加载

**问题**: 需要在 Android Chromium 中加载用户自定义字体

```cpp
// chrome/android/java/src/org/chromium/chrome/browser/CustomFontManager.java

public class CustomFontManager {
    private static final String CUSTOM_FONTS_DIR = 
        "/data/data/com.android.chrome/custom_fonts/";
    
    public static void LoadCustomFont(String fontName, String fontPath) {
        // 1. 复制字体文件到 Chrome 私有目录
        copyFontFile(fontPath, CUSTOM_FONTS_DIR + fontName + ".ttf");
        
        // 2. 通知 C++ 端字体已添加
        nativeOnCustomFontAdded(fontName);
    }
    
    private static native void nativeOnCustomFontAdded(String fontName);
}

// chrome/android/java/jni/custom_font_manager_jni.cc

JNI_EXPORT void JNI_CustomFontManager_OnCustomFontAdded(
    JNIEnv* env,
    const JavaParamRef<jstring>& font_name) {
  std::string cpp_font_name = ConvertJavaStringToUTF8(env, font_name);
  
  // C++ 端: 向 SkFontMgr 注册字体
  #if BUILDFLAG(IS_ANDROID)
    FontCache* cache = FontCache::GetFontCache();
    
    std::string font_path = 
        "/data/data/com.android.chrome/custom_fonts/" + 
        cpp_font_name + ".ttf";
    
    sk_sp<SkTypeface> typeface = 
        SkTypeface::MakeFromFile(font_path.c_str(), 0);
    
    if (typeface) {
      // 注册到全局字体缓存
      // (需要扩展 FontCache 支持自定义字体注册)
      DLOG(INFO) << "Custom font loaded: " << cpp_font_name;
    }
  #endif
}
```

---

## 调试技巧

### 1. 启用详细日志

```bash
# 启动 Chromium 并启用字体系统日志
./out/Default/chrome \
  --enable-logging=stderr \
  --v=2 \
  --vmodule=font_cache=2,font_selector=2,harfbuzz_shaper=2 \
  --log-level=0

# 输出示例
# [1234:1234:1125/121530.456:VERBOSE1(font_cache.cc:123)] 
#   FontCache::GetFontData(Georgia, size=16, weight=400)
```

### 2. 在 DevTools 中检查字体

```javascript
// Chrome DevTools Console

// 获取元素计算样式中的字体
element = document.querySelector('p');
style = window.getComputedStyle(element);
console.log('font-family:', style.fontFamily);
console.log('font-size:', style.fontSize);
console.log('font-weight:', style.fontWeight);

// 检查字体是否加载
document.fonts.ready.then(() => {
  console.log('All fonts loaded!');
});

// 列出已加载的字体
document.fonts.forEach(font => {
  console.log(font.family, font.status);
});
```

### 3. 添加断点调试

```cpp
// third_party/blink/renderer/platform/fonts/font_cache.cc

const SimpleFontData* FontCache::GetFontData(
    const FontDescription& font_description,
    const AtomicString& family_name) {
  
  // 调试: 在特定字体处添加断点
  if (family_name == "Georgia") {
    // ← 在这里设置条件断点
    NOTREACHED();  // 或用 debugger 语句
  }
  
  // ... 正常流程 ...
}
```

### 4. 追踪字体加载时间

```cpp
// 在 font_cache.cc 中添加性能监测

base::TimeTicks start = base::TimeTicks::Now();

SimpleFontData* font_data = CreateSimpleFontData(platform_data);

base::TimeDelta elapsed = base::TimeTicks::Now() - start;

UMA_HISTOGRAM_TIMES("FontCache.LoadTime", elapsed);
DLOG(INFO) << "Font load time: " << elapsed.InMilliseconds() << "ms";

if (elapsed.InMilliseconds() > 100) {
  DLOG(WARNING) << "Slow font load detected!";
}
```

---

## 性能优化

### 1. 预加载常用字体

```cpp
// chrome/browser/font_preload.cc

void PreloadCommonFonts() {
  FontCache* cache = FontCache::GetFontCache();
  
  // 常见字体列表
  const char* common_fonts[] = {
    "sans-serif",
    "serif",
    "monospace",
  };
  
  for (const char* font : common_fonts) {
    FontDescription desc;
    desc.SetSize(14.0f);
    desc.SetFamily(FontFamily(AtomicString(font)));
    
    // 预加载 (不进行任何操作,仅触发缓存)
    cache->GetFontData(desc, AtomicString(font));
  }
}

// 在浏览器启动时调用
// Browser::OnStartup() {
//   PreloadCommonFonts();
// }
```

### 2. 批量字体查询优化

```cpp
// 场景: 需要查询多个字符的字体

// ❌ 低效方式 (多次缓存查询开销)
for (UChar32 c = 'A'; c <= 'z'; c++) {
  const SimpleFontData* font = font_obj.FontDataForCharacter(c);
  // 每个字符都要经过 FontFallbackList::GetFontData()
}

// ✓ 高效方式 (缓存利用)
const SimpleFontData* font_for_range = nullptr;
for (UChar32 c = 'A'; c <= 'z'; c++) {
  if (!font_for_range || 
      !font_for_range->GlyphForCharacter(c)) {
    // 仅在字体改变时查询
    font_for_range = font_obj.FontDataForCharacter(c);
  }
  // 使用 font_for_range ...
}
```

### 3. 减少 FontDescription 复制

```cpp
// ❌ 低效: 多次复制
FontDescription desc = style->GetFontDescription();
FontDescription desc2 = desc;  // 复制
desc2.SetSize(20);
Font font1(desc);
Font font2(desc2);  // 再复制

// ✓ 高效: 使用引用
const FontDescription& desc = style->GetFontDescription();
// 直接使用引用,无复制

// 或使用 move 语义
Font font(std::move(desc));
```

---

## 常见错误修复

### 错误 1: 字体加载失败导致空指针

```cpp
// ❌ 错误代码
const SimpleFontData* font_data = 
    font.FontDataForCharacter('A');
float width = font_data->WidthForGlyph(123);  // ← 如果 font_data 为 null 则崩溃!

// ✓ 正确代码
const SimpleFontData* font_data = 
    font.FontDataForCharacter('A');
if (!font_data) {
  // 使用 fallback
  font_data = font.GetFallbackList()->GetFallbackFont();
}
DCHECK(font_data);  // 确保不为 null
float width = font_data->WidthForGlyph(123);
```

### 错误 2: 在无样式计算时访问字体

```cpp
// ❌ 错误代码 (可能在 Element 构造时调用)
Element::Element() {
  Font font = GetComputedStyle()->GetFont();  // ← 此时无 ComputedStyle!
}

// ✓ 正确代码
void Element::OnLayoutObjectCreated() {
  DCHECK(GetLayoutObject());
  Font font = GetComputedStyle()->GetFont();
}
```

### 错误 3: 修改 FontDescription 后忘记重新计算

```cpp
// ❌ 错误代码
FontDescription desc = style->GetFontDescription();
desc.SetSize(20.0f);  // 修改大小
// font 仍然使用旧大小!
const Font& font = style->GetFont();

// ✓ 正确代码
FontDescription desc = style->GetFontDescription();
desc.SetSize(20.0f);
style->SetFontDescription(desc);  // 更新 style
// 需要触发重新布局
element->SetNeedsStyleRecalc(StyleRecalcChange::kRecalc);
```

### 错误 4: Android 字体查询返回空

```cpp
// ❌ 错误代码 (Android 特定)
FontCache* cache = FontCache::GetFontCache();
const SimpleFontData* font = 
    cache->GetFontData(desc, "CustomFont");
DCHECK(font);  // 可能在 Android 上为 null!

// ✓ 正确代码
FontCache* cache = FontCache::GetFontCache();
const SimpleFontData* font = 
    cache->GetFontData(desc, "CustomFont");

if (!font) {
  // 字体不存在,使用通用族 fallback
  #if BUILDFLAG(IS_ANDROID)
    font = cache->GetFontData(desc, "sans-serif");
  #endif
}

DCHECK(font);
```

### 错误 5: 混淆 GenericFamily 的含义

```cpp
// ❌ 错误理解
if (font_desc.GenericFamily() == 0) {
  // 错误: 认为 0 表示"未设置"
}

// ✓ 正确理解
FontDescription::GenericFamilyType generic = 
    font_desc.GenericFamily();

if (generic == FontDescription::kNoFamily) {
  // 正确: 这是具体字体名 (如 "Georgia")
  // 不是通用族
} else {
  // 这是通用族 (serif, sans-serif 等)
  // 需要映射到具体字体
}
```

---

## 完整工作流示例

### 完整场景: 从 HTML 到像素的所有步骤

```cpp
// 假设有 HTML: <p style="font-family: Georgia; font-size: 16px">Hello</p>

// ============ Step 1: DOM 解析 ============
// HTMLParser 读取 HTML 文本
// HTMLTreeBuilder 创建 DOM 树
// 创建 <p> 元素节点
HTMLElement* p_element = document->createElement("p");

// ============ Step 2: 样式计算 ============
// Element::RecalcStyle() 触发
const ComputedStyle* style = 
    document->GetStyleResolver().ResolveStyle(
        p_element, style_recalc_context, StyleRequest());

// CSS 解析过程 (在 StyleBuilder 中)
// "font-family: Georgia" → CSSFontFamilyValue("Georgia")
// CSSFontFamilyValue → FontFamily("Georgia")
// FontDescription 创建完毕

// ============ Step 3: Font 对象创建 ============
// StyleResolverState::UpdateFont() 调用
const Font& font = style->GetFont();

// ============ Step 4: 文本测量 ============
// LayoutText 需要计算文本度量
// 第一次使用字符 'H'
const SimpleFontData* font_data = 
    font.FontDataForCharacter('H');

// FontCache 缓存查询
// 缓存未命中 → SkFontMgr::matchFamilyStyle("Georgia")
// 加载字体文件 (可能磁盘 IO)
// 创建 SimpleFontData
// 存入缓存

// ============ Step 5: 字形信息 ============
// HarfBuzzShaper::Shape()
Glyph glyph_h = font_data->GlyphForCharacter('H');     // 68
Glyph glyph_e = font_data->GlyphForCharacter('e');     // 69
Glyph glyph_l = font_data->GlyphForCharacter('l');     // 76
Glyph glyph_o = font_data->GlyphForCharacter('o');     // 79

// HarfBuzz 计算位置
hb_buffer_t* buffer = hb_buffer_create();
hb_buffer_add_utf16(buffer, u"Hello", 5, 0, 5);
hb_shape(hb_font, buffer, nullptr, 0);

// 获取位置信息
hb_glyph_info_t* glyph_infos = 
    hb_buffer_get_glyph_infos(buffer, &glyph_count);
hb_glyph_position_t* glyph_positions = 
    hb_buffer_get_glyph_positions(buffer, &glyph_count);

// ShapeResult 构建完毕
// [{glyph: 68, pos: 0}, {glyph: 69, pos: 600}, ...]

// ============ Step 6: 绘制 ============
// GraphicsContext::DrawText()
// → SkCanvas::drawGlyphs()

SkPaint paint;
paint.setColor(SK_ColorBLACK);

for (unsigned i = 0; i < glyph_count; i++) {
  SkScalar x = glyph_positions[i].x_advance;
  SkScalar y = baseline_y;
  
  // Skia 光栅化字形到画布
  canvas->drawGlyphs(
      1,
      &glyph_infos[i].codepoint,
      &SkPoint::Make(x, y),
      font_data->GetSkFont(),
      paint);
}

// ============ Step 7: 屏幕显示 ============
// Display list 发送给 compositor
// GPU rasterization
// 最终像素显示在屏幕上 ✅

hb_buffer_destroy(buffer);
```

### 性能分析

```
事件序列:
T=0ms:   DOM 解析完成
T=10ms:  样式计算 (UpdateFont) - <1ms
         FontDescription 创建
         Font 对象创建
         FontFallbackList 初始化 (为空)
T=11ms:  布局计算
T=15ms:  第一次字符使用 'H'
         FontCache 查询 - 缓存未命中
         SkFontMgr::matchFamilyStyle() - 磁盘 IO (可能 5-10ms)
T=20ms:  SimpleFontData 创建
         存入 FontCache
         Glyph 信息获取 - <1ms
T=21ms:  HarfBuzz Shaping - <1ms
T=22ms:  屏幕绘制 - GPU 加速
T=25ms:  最终渲染完成

性能指标:
- 样式计算: 1ms (无磁盘 IO)
- 字体加载: 5-10ms (可优化: 预加载)
- Shaping: <1ms (HarfBuzz 高效)
- 绘制: GPU 加速
- 总耗时: 15-20ms 目标: 16ms (60fps)
```

---

## 集成检查清单

### 新增字体特性的集成步骤

```cpp
// 1. 修改 FontDescription (如果需要新字段)
// File: font_description.h
// - 添加新字段
// - 生成 getter/setter

// 2. 修改 CSS 转换
// File: style_builder_converter.cc
// - 在 ConvertFontXxx() 中处理新 CSS 属性

// 3. 修改 FontBuilder
// File: font_builder.cc
// - 在 CreateFont() 中应用新 FontDescription 字段

// 4. 修改 FontCache (如果涉及字体查询)
// File: font_cache.cc
// - 更新 CacheKey (包含新字段)
// - 更新字体加载逻辑

// 5. 修改平台层
// File: font_cache_skia.cc (或 font_cache_android.cc)
// - 传递新参数到 SkFontMgr

// 6. 添加单元测试
// File: font_description_test.cc
// - 测试字段设置/获取
// - 测试 CSS 转换

// 7. 添加浏览器测试
// File: web_font_browsertest.cc
// - 测试端到端渲染

// 8. 性能测试
// File: font_rendering_perf_test.cc
// - 确保无回归
```

---

参考: [CHROMIUM_CSS_TO_RENDERING_COMPREHENSIVE_GUIDE.md](CHROMIUM_CSS_TO_RENDERING_COMPREHENSIVE_GUIDE.md)
