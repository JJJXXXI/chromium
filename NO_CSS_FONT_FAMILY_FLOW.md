# 无 CSS font-family 时的字体处理流程

## 📌 核心概念

**场景**: HTML 中没有 `style="font-family: ..."` 属性

```html
<!-- 无 CSS font-family -->
<p>Hello World</p>

<!-- vs 有 CSS font-family -->
<p style="font-family: Georgia">Hello World</p>
```

---

## 🔄 完整处理流程

### 第 1 步: HTML 解析

```
<p>Hello World</p>
  ↓
HTMLTreeBuilder::ConstructTree()
  └─ 创建 HTMLParagraphElement 节点
  └─ 无 style 属性
```

### 第 2 步: 样式计算 (无 CSS 规则匹配)

**文件**: `third_party/blink/renderer/core/css/resolver/style_resolver.cc`

```cpp
const ComputedStyle* StyleResolver::ResolveStyle(
    Element* element,
    const StyleRecalcContext& style_recalc_context,
    const StyleRequest& style_request) {
  
  StyleResolverState state(GetDocument(), *element, ...);
  StyleCascade cascade(state);
  
  // 匹配 CSS 规则
  MatchAllRules(element, cascade, ...);
  
  // ★ 对于 <p> 无 style: 无匹配规则!
  // 只有 UA 样式表的规则
}
```

### 第 3 步: 应用 UA 样式

**文件**: `third_party/blink/renderer/core/html/resources/html.css`

```css
/* html.css 中的 UA 样式 */
body {
  display: block;
  margin: 8px;
}

p {
  display: block;
  margin: 1em 0;
  /* ← 注意: 没有 font-family! */
}

/* 没有默认的 font-family 设置! */
```

**关键点**: UA 样式表不包含 `font-family` 设置!

```cpp
// style_cascade.cc
void StyleCascade::Apply() {
  // 应用 UA 样式
  ApplyUserAgentRules();  // ← 来自 html.css
  
  // 此时 FontDescription 仍为初始状态
  FontDescription {
    generic_family_: kNoFamily,
    family_name: "",
    size: 16.0,
    weight: 400,
    style: normal
  }
}
```

### 第 4 步: 字体计算 (关键!)

**文件**: `third_party/blink/renderer/core/css/resolver/font_builder.cc`

```cpp
void FontBuilder::CreateFont(ComputedStyleBuilder& builder,
                             const ComputedStyle* parent_style) {
  
  FontDescription description = builder.GetFontDescription();
  
  // 此时:
  // description.generic_family_ = kNoFamily
  // description.family_name = ""
  
  // ★ 关键步骤: 处理未设置的情况
  UpdateFontDescription(description, ...);
  
  // ★★★ 最关键的函数!
  if (description.GenericFamily() == FontDescription::kNoFamily 
      && description.Family().FamilyName().IsEmpty()) {
    
    // 应用初始值
    InitialGenericFamily(description);
    // → description.generic_family_ = kStandardFamily ← 重要!
  }
  
  // 现在:
  // description.generic_family_ = kStandardFamily (serif)
  // description.family_name = "" (待填充)
  
  builder.SetFont(MakeGarbageCollected<Font>(description, font_selector));
}
```

#### InitialGenericFamily() 函数

**文件**: `third_party/blink/renderer/core/css/resolver/font_builder.cc:450-480`

```cpp
void FontBuilder::InitialGenericFamily(FontDescription& font_description) {
  // 没有明确指定 font-family 时
  // 默认使用标准字体族 (serif)
  
  font_description.SetGenericFamily(
      FontDescription::kStandardFamily);  // ← "serif" 的代表
  
  // 不设置 family_name,等待 FontSelector 填充
}
```

**重要**: 
- `kStandardFamily` 代表 serif (衬线字体)
- 后面由 GenericFontFamilySettings 或平台决定具体字体

### 第 5 步: Font 对象创建

```cpp
Font {
  font_description_: FontDescription {
    generic_family_: kStandardFamily  // ← "serif"
    family_name: ""                    // ← 空的!
    size: 16.0
    weight: 400
    style: normal
  }
  fallback_list_: FontFallbackList {
    font_list_: []  // ← 还为空
    font_character_map_: {}
  }
}
```

### 第 6 步: 字体加载 (第一次使用字符时)

**文件**: `third_party/blink/renderer/platform/fonts/font_selector.cc`

```cpp
const SimpleFontData* FontSelector::GetFontData(
    const FontDescription& font_description,
    const AtomicString& family_name) {
  
  // family_name 为空
  // generic_family = kStandardFamily
  
  // ★ 查询通用族映射
  AtomicString specific_font = FamilyNameFromSettings(
      font_description,
      font_description.GenericFamily());  // ← kStandardFamily
  
  // 结果:
  // - Linux: "DejaVu Serif" 或 "Times New Roman"
  // - Windows: "Times New Roman"
  // - macOS: "Times"
  // - Android: "" (空, 触发系统查询)
  
  return FontCache::GetFontData(font_description, specific_font);
}
```

#### Android 特殊处理

```cpp
// Android 上 GenericFontFamilySettings 为空
// 所以调用 FontCache::GetGenericFamilyNameForScript()

AtomicString FontCache::GetGenericFamilyNameForScript(
    const AtomicString& generic_family,  // "sans-serif"
    const FontDescription& font_description,
    const LayoutLocale* content_locale) {
  
  // 从 fonts.xml 查询 "sans-serif" 族
  // <family name="sans-serif">
  //   <font weight="400">Roboto-Regular.ttf</font>
  // </family>
  
  return "Roboto-Regular.ttf";  // ← 返回具体字体
}
```

### 第 7 步: SimpleFontData 创建

```cpp
FontCache::GetFontData(font_description, "Roboto") {
  
  // 1. 查询缓存
  // 2. 缓存未命中
  
  // 3. 创建 FontPlatformData
  FontPlatformData* platform_data = 
    CreateFontPlatformData(font_description, "Roboto");
  
  // 4. SkFontMgr 查询
  SkTypeface* typeface = 
    font_manager_->matchFamilyStyle("Roboto", style);
  
  // 5. 创建 SimpleFontData
  SimpleFontData* font_data = 
    new SimpleFontData(platform_data);
  
  // 6. 存入缓存
  return font_data;
}
```

---

## 📊 无 CSS vs 有 CSS 对比

### 完整流程对比

| 阶段 | 无 CSS | 有 CSS |
|------|-------|--------|
| **HTML** | `<p>文字</p>` | `<p style="font-family: Georgia">文字</p>` |
| **CSS 规则匹配** | 仅 UA 样式 | UA + 用户 CSS |
| **StyleBuilder** | generic_family = kNoFamily | generic_family = kNoFamily |
| **关键转换** | InitialGenericFamily() | ConvertFontFamily() |
| **FontDescription** | generic_family = kStandardFamily | generic_family = kNoFamily, family_name = "Georgia" |
| **字体查询** | 通用族 → 系统映射 | 具体字体 → fallback |
| **最终字体** | 系统默认 serif 字体 | 指定的字体 |

### 数据流对比

#### 无 CSS

```
StyleResolverState::UpdateFont()
  └─ FontBuilder::CreateFont()
      └─ InitialGenericFamily()
          └─ description.SetGenericFamily(kStandardFamily)
      └─ Font {
          generic_family: kStandardFamily
          family_name: ""
        }
      ↓ [字符使用时]
      └─ FontSelector::FamilyNameFromSettings(kStandardFamily)
          ├─ Linux: "DejaVu Serif"
          └─ Android: "" → SkFontMgr → "Roboto"
      ↓
      └─ SimpleFontData("DejaVu Serif" 或 "Roboto")
```

#### 有 CSS

```
StyleResolverState::UpdateFont()
  └─ StyleCascade::Apply()
      └─ StyleBuilderConverter::ConvertFontFamily()
          └─ FontDescription {
              generic_family: kNoFamily
              family_name: "Georgia"
              next: "serif"
            }
      ↓ [字符使用时]
      └─ FontSelector::GetFontData("Georgia")
          ├─ 查 "Georgia" (具体字体)
          └─ 若失败,查 next = "serif"
      ↓
      └─ SimpleFontData("Georgia" 或 "serif系统字体")
```

---

## 🔑 关键区别

### 1. 初始化不同

```cpp
// 无 CSS
FontDescription::FontDescription() {
  fields_.generic_family_ = kNoFamily;    // 初始值
  // → 触发 InitialGenericFamily()
  // → 转为 kStandardFamily
}

// 有 CSS
FontDescription {
  generic_family_: kNoFamily              // 保持不变
  family_name: "Georgia"                  // 由 CSS 提供
}
```

### 2. 字体查询不同

```cpp
// 无 CSS: 通用族查询
FontSelector::GetFontData(desc, "")
  ├─ desc.GenericFamily() = kStandardFamily
  ├─ 查询 GenericFontFamilySettings
  └─ 获得系统映射 "DejaVu Serif"

// 有 CSS: 具体字体查询
FontSelector::GetFontData(desc, "Georgia")
  ├─ 直接查询 "Georgia"
  ├─ 若失败,查询 next
  └─ 最后 fallback
```

### 3. Fallback 链不同

```cpp
// 无 CSS
FontDescription {
  family_: FontFamily("DejaVu Serif")  // 系统字体
  next_: nullptr                        // 无后续
}
// 简单 fallback

// 有 CSS
FontDescription {
  family_: FontFamily("Georgia")
  next_: FontFamily("serif")            // 明确指定 fallback
  next_next_: nullptr
}
// 复杂 fallback
```

---

## 💡 初始值映射规则

### InitialGenericFamily() 的映射

```cpp
void FontBuilder::InitialGenericFamily(FontDescription& desc) {
  desc.SetGenericFamily(FontDescription::kStandardFamily);
  // kStandardFamily 是什么?
}

// font_description.h
enum GenericFamilyType {
  kNoFamily = 0,             // 无指定 (具体字体)
  kStandardFamily = 1,       // ← 默认值 (serif)
  kSerifFamily = 2,
  kSansSerifFamily = 3,
  kMonospaceFamily = 4,
  kCursiveFamily = 5,
  kFantasyFamily = 6,
  kSystemUiFamily = 7,
};
```

**重要**: 无 CSS 时默认 = `kStandardFamily` (serif, 衬线体)

### 系统映射

```
Linux:
  kStandardFamily → "DejaVu Serif"
  kSansSerifFamily → "DejaVu Sans"
  kMonospaceFamily → "DejaVu Mono"

Windows:
  kStandardFamily → "Times New Roman"
  kSansSerifFamily → "Arial"
  kMonospaceFamily → "Courier New"

macOS:
  kStandardFamily → "Times"
  kSansSerifFamily → "Helvetica"
  kMonospaceFamily → "Courier"

Android:
  kStandardFamily → "" (空, 查 fonts.xml)
    → SkFontMgr → "Noto Serif"
  kSansSerifFamily → ""
    → "Roboto"
```

---

## 🔍 关键源代码位置

| 功能 | 文件 | 行数 |
|------|------|------|
| 初始值应用 | `font_builder.cc` | 450-480 |
| Font 创建 | `font_builder.cc` | 653-700 |
| 通用族查询 | `font_selector.cc` | 100-150 |
| Android 查询 | `font_cache_android.cc` | - |
| UA 样式 | `html/resources/html.css` | - |

---

## 📋 完整示例

### 无 CSS 的完整流程

```
1️⃣ HTML 解析
   <p>Hello</p>
   └─ HTMLParagraphElement (无 style)

2️⃣ 样式计算
   StyleResolver::ResolveStyle()
   ├─ MatchAllRules() → 仅 UA 样式
   ├─ FontDescription { generic_family: kNoFamily }
   └─ StyleResolverState::UpdateFont()

3️⃣ 初始化
   FontBuilder::CreateFont()
   ├─ InitialGenericFamily()
   └─ FontDescription { generic_family: kStandardFamily }

4️⃣ Font 对象
   Font {
     font_description_: { generic_family: kStandardFamily }
     fallback_list_: []
   }

5️⃣ 文本布局
   TextLayout::ComputeTextMetrics("Hello")
   ├─ 字符 'H' 首次使用
   └─ Font::FontDataForCharacter('H')

6️⃣ 字体查询
   FontSelector::GetFontData(desc, "")
   ├─ GenericFamily = kStandardFamily
   ├─ FamilyNameFromSettings(kStandardFamily)
   └─ Linux: "DejaVu Serif"
      Android: "Roboto" (via fonts.xml)

7️⃣ 字体加载
   FontCache::GetFontData("DejaVu Serif")
   ├─ SkFontMgr::matchFamilyStyle("DejaVu Serif")
   └─ SimpleFontData 创建

8️⃣ 文本 Shaping
   HarfBuzzShaper::Shape("Hello")
   └─ 字形信息生成

9️⃣ 屏幕渲染
   GraphicsContext::DrawText()
   └─ 显示为 serif 字体
```

### 有 CSS 的对比

```
1-2️⃣ 相同

3️⃣ CSS 转换 (关键不同!)
   StyleBuilder::ApplyProperty(font-family, CSS)
   ├─ StyleBuilderConverter::ConvertFontFamily()
   └─ FontDescription { 
        generic_family: kNoFamily
        family_name: "Georgia"
      }

4-9️⃣ 使用指定的字体 "Georgia"
```

---

## 🎯 要点总结

**无 CSS font-family 时**:

1. ✅ UA 样式中没有 font-family 设置
2. ✅ FontDescription 初始化为 `kNoFamily`
3. ✅ **关键**: `InitialGenericFamily()` 将其转为 `kStandardFamily`
4. ✅ `kStandardFamily` = serif (衬线)
5. ✅ 查询系统的 serif 字体映射
6. ✅ Linux/macOS: Times New Roman 等
7. ✅ Android: 从 fonts.xml 查询 "serif" → "Noto Serif"
8. ✅ 最终使用系统默认的 serif 字体

**与有 CSS 的主要区别**:

| 项 | 无 CSS | 有 CSS |
|----|-------|--------|
| 触发的转换 | InitialGenericFamily() | ConvertFontFamily() |
| 最终 generic_family | kStandardFamily | kNoFamily (通常) |
| 字体来源 | 系统映射 | CSS 指定 |
| 查询策略 | 通用族映射 | 具体字体+fallback |

---

## 参考

- [CHROMIUM_CSS_TO_RENDERING_COMPREHENSIVE_GUIDE.md](CHROMIUM_CSS_TO_RENDERING_COMPREHENSIVE_GUIDE.md) - 第四阶段
- [FONT_IMPLEMENTATION_CODE_REFERENCE.md](FONT_IMPLEMENTATION_CODE_REFERENCE.md) - 调试技巧
- `third_party/blink/renderer/core/css/resolver/font_builder.cc` - 核心实现
