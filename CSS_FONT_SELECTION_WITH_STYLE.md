# 有 CSS 时的字体选择详解

## 🎯 有 CSS vs 无 CSS 完整对比

### 快速对比

| 方面 | 无 CSS | 有 CSS |
|------|-------|--------|
| HTML | `<p>文字</p>` | `<p style="font-family: Georgia">文字</p>` |
| CSS 值 | 无 | `font-family: Georgia` |
| generic_family | kNoFamily → kStandardFamily | kNoFamily (Georgia 是具体字体) |
| family_name | "" → "DejaVu Sans" 或 "Roboto" | "Georgia" |
| 初始值应用 | ✓ 需要 InitialGenericFamily() | ✗ 不需要 |
| 查询映射表 | ✓ 需要 | ✗ 不需要 |
| 最终字体 | 系统默认字体 | 指定的字体 |

---

## 📍 有 CSS 时的完整流程

### 步骤 1: HTML 解析

```html
<p style="font-family: Georgia, serif">Hello World</p>
```

**此时**:
- 浏览器识别到 `style` 属性中有 `font-family`
- 准备进入 Style Calculation

---

### 步骤 2: CSS 解析 (CSS Parser)

**文件**: `third_party/blink/renderer/core/css/parser/css_parser.cc`

```cpp
// CSS Parser 读取 "font-family: Georgia, serif"
// 输出是 CSSValue 对象列表

CSSValueList {
  [0] CSSFontFamilyValue("Georgia")    ← 具体字体名
  [1] CSSIdentifierValue(serif)        ← 通用族关键字
}
```

**关键**:
- `CSSFontFamilyValue` = 字符串形式的字体名 (可以是任意名字)
- `CSSIdentifierValue` = CSS 关键字 (serif, sans-serif, monospace 等)

---

### 步骤 3: CSS 应用 (StyleCascade::Apply)

**文件**: `third_party/blink/renderer/core/css/resolver/style_cascade.cc`

```cpp
void StyleCascade::Apply() {
  // 1. 遍历 CSS 规则 (User Agent, Author, Inline styles 等)
  
  // 2. 对于 font-family 属性:
  StyleBuilder::ApplyProperty(
      CSSPropertyID::kFontFamily,     // ← 属性 ID
      state,                          // ← 样式状态
      css_value);                     // ← 解析后的 CSSValueList
  
  // 3. CSS 应用完毕 → 调用 UpdateFont
  state.UpdateFont();
}
```

---

### 步骤 4: CSS 值转换为 FontDescription (★ 关键!)

**文件**: `third_party/blink/renderer/core/css/resolver/style_builder_converter.cc:510-590`

这是**有 CSS 和无 CSS 差异最大的地方**!

#### 转换函数

```cpp
FontDescription::FamilyDescription 
StyleBuilderConverter::ConvertFontFamily(
    StyleResolverState& state,
    const CSSValue& value) {
  
  // value 是 CSS Parser 输出的 CSSValueList
  // 例如: [CSSFontFamilyValue("Georgia"), CSSIdentifierValue(serif)]
  
  return StyleBuilderConverterBase::ConvertFontFamily(
      value,
      &state.GetFontBuilder(),
      &state.GetDocument());
}

// ★ 核心转换逻辑
FontDescription::FamilyDescription 
StyleBuilderConverterBase::ConvertFontFamily(
    const CSSValue& value,
    FontBuilder* font_builder,
    const Document* document_for_count) {
  
  FontDescription::FamilyDescription desc(FontDescription::kNoFamily);
  
  // 遍历 font-family 列表
  // 例: font-family: Georgia, serif, sans-serif
  for (auto& family : base::Reversed(To<CSSValueList>(value))) {
    
    AtomicString next_family_name;
    FontDescription::GenericFamilyType generic_family = FontDescription::kNoFamily;
    
    // ★ 处理单个值
    if (!ConvertFontFamilyName(*family, generic_family, next_family_name,
                               font_builder, document_for_count)) {
      continue;
    }
    
    // 构建链表
    if (has_value) {
      next = SharedFontFamily::Create(family_name, family_type, std::move(next));
    }
    
    family_name = next_family_name;
    family_type = is_generic ? FontFamily::Type::kGenericFamily
                             : FontFamily::Type::kFamilyName;
    has_value = true;
  }
  
  desc.family = FontFamily(family_name, family_type, std::move(next));
  return desc;
}
```

#### 单个值转换 (ConvertFontFamilyName)

```cpp
static bool ConvertFontFamilyName(
    const CSSValue& value,
    FontDescription::GenericFamilyType& generic_family,
    AtomicString& family_name,
    FontBuilder* font_builder,
    const Document* document_for_count) {
  
  if (auto* font_family_value = DynamicTo<CSSFontFamilyValue>(value)) {
    // ★ 情况 1: 具体字体名 (如 "Georgia")
    generic_family = FontDescription::kNoFamily;      // ← 不是通用族!
    family_name = font_family_value->Value();         // ← "Georgia"
    return true;
    
  } else if (font_builder) {
    // ★ 情况 2: 通用族关键字 (如 "serif")
    auto cssValueID = To<CSSIdentifierValue>(value).GetValueID();
    generic_family = ConvertGenericFamily(cssValueID);  // ← 转为 enum
    
    if (generic_family != FontDescription::kNoFamily) {
      // 对于通用族,向 FontBuilder 请求映射
      family_name = font_builder->GenericFontFamilyName(generic_family);
    }
  }
  
  return !family_name.IsNull();
}

// ConvertGenericFamily 将 CSS 关键字转为 enum
static FontDescription::GenericFamilyType ConvertGenericFamily(
    CSSValueID value_id) {
  switch (value_id) {
    case CSSValueID::kSerif:
      return FontDescription::kSerifFamily;
    case CSSValueID::kSansSerif:
      return FontDescription::kSansSerifFamily;
    case CSSValueID::kMonospace:
      return FontDescription::kMonospaceFamily;
    // ...
    default:
      return FontDescription::kNoFamily;
  }
}
```

#### 转换结果示例

```
CSS: font-family: Georgia, serif, sans-serif
     ↓ [ConvertFontFamily()]
FontDescription::FamilyDescription {
  family_list: [
    {
      family_name: "Georgia"
      generic_family: kNoFamily          ← 具体字体
      type: kFamilyName
      next: node2
    },
    {
      family_name: "serif"
      generic_family: kSerifFamily       ← 通用族
      type: kGenericFamily
      next: node3
    },
    {
      family_name: "sans-serif"
      generic_family: kSansSerifFamily   ← 通用族
      type: kGenericFamily
      next: null
    }
  ]
}
```

---

### 步骤 5: FontBuilder 创建 Font 对象

**文件**: `third_party/blink/renderer/core/css/resolver/font_builder.cc:653-700`

```cpp
void FontBuilder::CreateFont(ComputedStyleBuilder& builder,
                             const ComputedStyle* parent_style) {
  
  // 1. 从 builder 获取当前 FontDescription
  FontDescription description = builder.GetFontDescription();
  
  // 2. UpdateFontDescription() 应用转换后的族信息
  if (!UpdateFontDescription(description, builder.ComputeFontOrientation())) {
    flags_ = 0;
    return;
  }
  
  // ★ 关键区别: 有 CSS 时不需要 InitialGenericFamily()
  // 因为 description 已经从 ConvertFontFamily() 获得了正确的值
  
  // 3. 更新大小等其他属性
  UpdateSpecifiedSize(description, parent_description);
  UpdateComputedSize(description, builder);
  
  // 4. 确定 FontSelector
  FontSelector* font_selector = ComputeFontSelector(builder);
  
  // 5. 创建 Font 对象
  builder.SetFont(MakeGarbageCollected<Font>(description, font_selector));
  
  flags_ = 0;
}
```

**此时 FontDescription 的状态**:

```
有 CSS 时:
  generic_family_: kNoFamily 或 kSerifFamily (来自 CSS 关键字)
  family_name: "Georgia" (来自 CSS 字体名)
  
无 CSS 时:
  generic_family_: kNoFamily (初始)
    → InitialGenericFamily() 转为 kStandardFamily
  family_name: "" (待填充)
```

---

### 步骤 6: Paint 时的延迟字体匹配

两种情况下都是相同的流程!

**文件**: `third_party/blink/renderer/platform/fonts/font_fallback_list.cc`

```cpp
const SimpleFontData* FontFallbackList::GetFontData(
    const FontFamily& family) {
  
  // 使用 CSSFontSelector 查询
  return font_selector_->GetFontData(
      font_description_,
      family.FamilyName(),              // ← "Georgia" 或 "serif"
      family.GenericFamily());          // ← kNoFamily 或 kSerifFamily
}
```

#### 有 CSS (具体字体) 的查询路径

```
CSSFontSelector::GetFontData("Georgia", kNoFamily)
  ↓
1️⃣ 查 FontFaceCache (@font-face)
   └─ 有没有 @font-face { font-family: Georgia }?
   
2️⃣ 若无,查 FontCache (系统字体)
   └─ 系统有 Georgia 字体吗?
       ├─ Windows: C:\Windows\Fonts\georgia.ttf
       ├─ Linux: /usr/share/fonts/...
       └─ Android: /system/fonts/... (但通常没有 Georgia)
   
3️⃣ 若都无,进入 fallback 链
   └─ 尝试链中的下一个字体 (serif)
```

#### 有 CSS (通用族) 的查询路径

```
CSS: font-family: serif

CSSFontSelector::GetFontData("serif", kSerifFamily)
  ↓
1️⃣ 查 FontFaceCache (@font-face)
   └─ 有没有 @font-face { font-family: serif }?
   
2️⃣ 若无,查 GenericFontFamilySettings 映射表
   └─ kSerifFamily 映射到什么?
       ├─ Linux: "Times New Roman" 或 "DejaVu Serif"
       └─ Android: "" (空)
   
3️⃣ 若映射表为空 (Android),查 FontCache
   └─ SkFontMgr 从 fonts.xml 查询
       └─ <alias name="serif" to="serif" />
           └─ <family name="serif">
               └─ "Noto Serif"
```

---

## 📊 完整的代码路径对比

### 无 CSS 的路径

```
HTMLElement (无 style)
  ↓
StyleResolver::ResolveStyle()
  ├─ StyleCascade::Apply()  [无匹配规则]
  └─ StyleResolverState::UpdateFont()
      └─ FontBuilder::CreateFont()
          ├─ FontDescription 仍为 kNoFamily
          ├─ InitialGenericFamily() → kStandardFamily ★
          ├─ FontSelector::FamilyNameFromSettings()
          │   ├─ Linux: "DejaVu Sans"
          │   └─ Android: "" → SkFontMgr
          └─ Font 对象创建
  
  Paint 时:
  FontFallbackList::PrimaryFont()
    └─ SimpleFontData 加载 (Roboto 或 DejaVu Sans)
```

### 有 CSS 的路径

```
HTMLElement (style="font-family: Georgia, serif")
  ↓
StyleResolver::ResolveStyle()
  ├─ StyleCascade::Apply()  [匹配 inline style]
  │   ├─ CSSParser: "font-family: Georgia, serif"
  │   │   └─ CSSValueList [CSSFontFamilyValue("Georgia"), CSSIdentifierValue(serif)]
  │   │
  │   └─ StyleBuilder::ApplyProperty(kFontFamily, ...)
  │       └─ StyleBuilderConverter::ConvertFontFamily() ★
  │           └─ FontDescription::FamilyDescription {
  │               family_name: "Georgia"
  │               generic_family: kNoFamily
  │               next: {
  │                 family_name: "serif"
  │                 generic_family: kSerifFamily
  │               }
  │             }
  │
  └─ StyleResolverState::UpdateFont()
      └─ FontBuilder::CreateFont()
          ├─ FontDescription 已从 ConvertFontFamily 获值
          ├─ ✗ 不需要 InitialGenericFamily() ★
          └─ Font 对象创建
  
  Paint 时:
  FontFallbackList::PrimaryFont()
    └─ CSSFontSelector::GetFontData("Georgia", kNoFamily)
        ├─ 查 @font-face
        ├─ 查系统 Georgia 字体
        ├─ 若无 → 试 serif (fallback)
        └─ SimpleFontData 加载
```

---

## 🔍 具体例子

### 例 1: 只指定具体字体

```html
<p style="font-family: 'Times New Roman'">Hello</p>
```

**流程**:
```
CSSFontFamilyValue("Times New Roman")
  ↓ [ConvertFontFamily()]
FontDescription {
  generic_family: kNoFamily           ← 不是通用族!
  family_name: "Times New Roman"      ← 具体字体
}
  ↓ [Paint]
CSSFontSelector::GetFontData("Times New Roman", kNoFamily)
  ├─ @font-face 缓存: 无
  ├─ 系统字体查询: 
  │   ├─ Windows: 找到 → times.ttf
  │   ├─ Linux: 找到 → Times-Roman
  │   └─ Android: 通常找不到 → fallback
  └─ SimpleFontData 创建
```

---

### 例 2: 只指定通用族

```html
<p style="font-family: serif">Hello</p>
```

**流程**:
```
CSSIdentifierValue(serif)
  ↓ [ConvertFontFamily()]
FontDescription {
  generic_family: kSerifFamily        ← 通用族
  family_name: "serif"                ← 通用族名字
}
  ↓ [Paint]
CSSFontSelector::GetFontData("serif", kSerifFamily)
  ├─ GenericFontFamilySettings 查询
  │   ├─ Linux: "Times New Roman"
  │   └─ Android: "" (空) → 继续查
  ├─ SkFontMgr fonts.xml 查询 (Android)
  │   └─ "Noto Serif"
  └─ SimpleFontData 创建
```

---

### 例 3: 混合具体和通用

```html
<p style="font-family: Georgia, 'Times New Roman', serif">Hello</p>
```

**流程**:
```
[CSSFontFamilyValue("Georgia"), 
 CSSFontFamilyValue("Times New Roman"),
 CSSIdentifierValue(serif)]
  ↓ [ConvertFontFamily()]
FontDescription {
  family_list: [
    { family_name: "Georgia", generic_family: kNoFamily },
    { family_name: "Times New Roman", generic_family: kNoFamily },
    { family_name: "serif", generic_family: kSerifFamily }
  ]
}
  ↓ [Paint - 依次尝试]
尝试 1: CSSFontSelector::GetFontData("Georgia", kNoFamily)
  → 假设找不到
    ↓
尝试 2: CSSFontSelector::GetFontData("Times New Roman", kNoFamily)
  → 假设找不到
    ↓
尝试 3: CSSFontSelector::GetFontData("serif", kSerifFamily)
  → 使用映射表或 fonts.xml
  → 找到 "Times New Roman" 或 "Noto Serif"
    ↓
SimpleFontData 创建
```

---

## 💡 关键区别总结

| 阶段 | 无 CSS | 有 CSS |
|------|-------|--------|
| **CSS 解析** | 无 | CSSParser 输出 CSSValueList |
| **转换** | 无 (默认值) | ConvertFontFamily() 转换 |
| **generic_family** | kNoFamily → kStandardFamily | 保持 CSS 中的值 |
| **family_name** | "" → 映射表查询 | 直接使用 CSS 值 |
| **初始值应用** | ✓ InitialGenericFamily() | ✗ 不需要 |
| **映射表查询** | ✓ 需要 | △ 仅通用族需要 |
| **Paint 时** | 相同 | 相同 |

---

## 🎓 ConvertFontFamily 的本质

**ConvertFontFamily 就是做一个转换**:

```cpp
// 输入: CSS 解析后的值
CSSValue (CSSFontFamilyValue 或 CSSIdentifierValue)

// 处理:
├─ 若是 CSSFontFamilyValue("Georgia")
│   → 输出 { family_name: "Georgia", generic_family: kNoFamily }
│
└─ 若是 CSSIdentifierValue(serif)
    → 输出 { family_name: "serif", generic_family: kSerifFamily }

// 输出: 标准化的 FontDescription::FamilyDescription
FamilyDescription
```

**这个转换的作用**:
- 统一处理不同类型的 CSS 值
- 将 CSS 关键字转为对应的 enum
- 建立 font-family 链表 (fallback 顺序)

---

## 🔗 代码文件导航

### CSS 解析和转换
- **css_parser.cc** - CSS 文本 → CSSValue
- **style_builder_converter.cc:510-590** - CSSValue → FontDescription
  - 行 470: `ConvertFontFamilyName()`
  - 行 510: `StyleBuilderConverterBase::ConvertFontFamily()`
  - 行 585: `StyleBuilderConverter::ConvertFontFamily()`

### Font 对象创建
- **font_builder.cc:653** - `FontBuilder::CreateFont()`
- **font_builder.h:138** - `InitialGenericFamily()`

### Paint 时的字体查询
- **font_fallback_list.cc** - 字体链表和查询
- **css_font_selector.cc** - CSSFontSelector 实现
- **font_selector.cc** - GenericFontFamilySettings 查询

---

## ❓ 常见问题

### Q: 为什么有 CSS 时不需要调用 InitialGenericFamily()?

**A**: 因为 ConvertFontFamily() 已经根据 CSS 值设置了 generic_family，不需要用初始值覆盖。

```
有 CSS:
  CSS: "Georgia"
    ↓
  ConvertFontFamily(): generic_family = kNoFamily ✓
    (明确表示这是具体字体,不是通用族)

无 CSS:
  初始值: generic_family = kNoFamily
    ↓
  InitialGenericFamily(): generic_family = kStandardFamily ✓
    (覆盖初始值为标准族)
```

### Q: generic_family 在有 CSS 时会变成 kNoFamily 吗?

**A**: 会,当具体字体时。

```
CSS: font-family: Georgia
  ↓
generic_family = kNoFamily  ← 表示 "不是通用族"
family_name = "Georgia"     ← 具体字体名

CSS: font-family: serif
  ↓
generic_family = kSerifFamily  ← 表示 "是通用族"
family_name = "serif"
```

### Q: 如果 CSS 中既有具体字体又有通用族呢?

**A**: 构建链表,按顺序尝试!

```
CSS: font-family: Georgia, serif

family_list: [
  { "Georgia", kNoFamily } → 先尝试这个
  { "serif", kSerifFamily } → 不行再尝试这个
]
```

