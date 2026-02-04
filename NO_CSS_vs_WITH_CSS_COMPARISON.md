# 有 CSS vs 无 CSS font-family 快速对比

## 🎯 一张图理解区别

```
┌─────────────────────────────────────────────────────────────┐
│ 无 CSS font-family                                          │
├─────────────────────────────────────────────────────────────┤
│ <p>Hello</p>                    (HTML 中无 style)           │
│   ↓                                                         │
│ StyleResolver (仅 UA 样式)                                 │
│   ↓                                                         │
│ FontDescription { generic_family: kNoFamily }              │
│   ↓ (★ 关键!)                                              │
│ InitialGenericFamily()  ← 转为 kStandardFamily             │
│   ↓                                                         │
│ FontDescription { generic_family: kStandardFamily }        │
│   ↓ (★ 无 family_name!)                                    │
│ FontSelector::GetFontData("", kStandardFamily)             │
│   ↓                                                         │
│ GenericFontFamilySettings::Standard() ← 映射表查询         │
│   ├─ Linux: "DejaVu Serif"                                 │
│   ├─ Windows: "Times New Roman"                            │
│   ├─ macOS: "Times"                                        │
│   └─ Android: "" (空) → SkFontMgr                          │
│   ↓                                                         │
│ SimpleFontData("DejaVu Serif")                             │
│   ↓                                                         │
│ 屏幕显示 (serif 字体) ✅                                    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 有 CSS font-family                                          │
├─────────────────────────────────────────────────────────────┤
│ <p style="font-family: Georgia">Hello</p>                  │
│   ↓                                                         │
│ StyleResolver (UA + 用户 CSS)                             │
│   ↓                                                         │
│ StyleBuilder::ApplyProperty(font-family)                   │
│   ↓ (★ 关键!)                                              │
│ StyleBuilderConverter::ConvertFontFamily()                 │
│   ↓                                                         │
│ FontDescription {                                          │
│   generic_family: kNoFamily          ← "Georgia" 不是通用  │
│   family_name: "Georgia"             ← CSS 提供           │
│   next: kSerifFamily ("serif")       ← Fallback          │
│ }                                                          │
│   ↓                                                         │
│ Font 创建                                                  │
│   ↓                                                         │
│ 字体查询:                                                  │
│   1. 尝试加载 "Georgia"                                    │
│   2. 失败? 尝试 "serif"                                    │
│   3. 仍失败? 系统 fallback                                 │
│   ↓                                                         │
│ SimpleFontData("Georgia")                                  │
│   ↓                                                         │
│ 屏幕显示 (Georgia 字体) ✅                                  │
└─────────────────────────────────────────────────────────────┘
```

---

## 📊 详细对比表

| 项目 | 无 CSS | 有 CSS |
|------|-------|--------|
| **HTML** | `<p>文字</p>` | `<p style="font-family: Georgia">文字</p>` |
| **CSS 规则** | 仅 UA 样式 | UA + 用户 CSS |
| **初始 FontDescription** | `generic_family_: kNoFamily` | `generic_family_: kNoFamily` |
| **关键转换** | `InitialGenericFamily()` | `ConvertFontFamily()` |
| **转换后的 generic_family** | `kStandardFamily` (serif) | `kNoFamily` (不变) |
| **family_name** | "" (空) | "Georgia" |
| **Fallback 链** | 无 (仅系统映射) | "Georgia" → "serif" → 系统 |
| **字体查询方式** | 通用族映射表 | 具体字体+fallback |
| **字体来源** | 系统默认 | CSS 指定 |
| **最终字体** | serif 系统字体 | Georgia 或 serif 备选 |

---

## 🔑 核心代码行对比

### 无 CSS: InitialGenericFamily()

**文件**: `third_party/blink/renderer/core/css/resolver/font_builder.cc:450-480`

```cpp
void FontBuilder::InitialGenericFamily(FontDescription& description) {
  // ★ 无 CSS font-family 时调用
  description.SetGenericFamily(FontDescription::kStandardFamily);
  // → kStandardFamily = serif (衬线体)
}
```

### 有 CSS: ConvertFontFamily()

**文件**: `third_party/blink/renderer/core/css/resolver/style_builder_converter.cc:510-590`

```cpp
FontDescription::FamilyDescription 
StyleBuilderConverter::ConvertFontFamily(
    StyleResolverState& state,
    const CSSValue& value) {  // ← CSS 值
  
  // ★ 有 CSS font-family 时调用
  // 处理 CSS 值,构建 FontFamily 链表
  // 不使用 InitialGenericFamily()
}
```

---

## 🔀 流程图对比

### 无 CSS

```
     HTML 解析
          ↓
    样式计算 (UA)
          ↓
 FontDescription
  (kNoFamily)
          ↓
★ InitialGenericFamily()
          ↓
 FontDescription
  (kStandardFamily)
          ↓
 字体查询: 通用族
          ↓
系统映射表
  (serif)
          ↓
SimpleFontData
(DejaVu Serif)
```

### 有 CSS

```
     HTML 解析
          ↓
    样式计算 (UA+CSS)
          ↓
★ ApplyProperty
(font-family: Georgia)
          ↓
★ ConvertFontFamily()
          ↓
 FontDescription
  (family: Georgia)
          ↓
 字体查询: 具体
          ↓
CSS 指定
(Georgia)
          ↓
SimpleFontData
  (Georgia)
```

---

## 🌐 平台差异

### 无 CSS 时的系统映射

#### Linux (GTK)
```
kStandardFamily → "DejaVu Serif"
kSansSerifFamily → "DejaVu Sans"
kMonospaceFamily → "DejaVu Mono"
```

#### Windows
```
kStandardFamily → "Times New Roman"
kSansSerifFamily → "Arial"
kMonospaceFamily → "Courier New"
```

#### macOS
```
kStandardFamily → "Times"
kSansSerifFamily → "Helvetica" / "San Francisco"
kMonospaceFamily → "Courier" / "Monaco"
```

#### Android
```
kStandardFamily → "" (空)
                → SkFontMgr 查询 fonts.xml
                → <family name="serif">
                → "Noto Serif"

kSansSerifFamily → ""
                → "Roboto"

kMonospaceFamily → ""
                → "Roboto Mono"
```

### 有 CSS 时

无平台差异，直接使用 CSS 指定的字体名
- 如果系统有该字体，使用该字体
- 如果没有，使用 fallback

---

## 💡 关键观察

### 观察 1: 初始值的转换

```cpp
// 无 CSS
FontDescription {
  generic_family: kNoFamily (初始)
} 
  ↓
InitialGenericFamily()
  ↓
FontDescription {
  generic_family: kStandardFamily (serif)
}

// 有 CSS
FontDescription {
  generic_family: kNoFamily (来自 CSS)
  family_name: "Georgia"
}
  ↓
NO InitialGenericFamily()!
  ↓
FontDescription {
  generic_family: kNoFamily (不变)
  family_name: "Georgia"
}
```

### 观察 2: 查询策略不同

```cpp
// 无 CSS: 通用族查询
GetFontData(desc, "")
  → desc.GenericFamily() = kStandardFamily
  → GenericFontFamilySettings::Standard()
  → "DejaVu Serif"

// 有 CSS: 具体字体查询
GetFontData(desc, "Georgia")
  → 直接查询 "Georgia"
  → 失败? 查询 fallback "serif"
  → 再失败? 系统默认
```

### 观察 3: family_name 的含义不同

```cpp
// 无 CSS
family_name: ""
  ← 不是"未设置",而是"通过 generic_family 设置"

// 有 CSS
family_name: "Georgia"
  ← 这就是 CSS 指定的字体名
```

---

## 🎯 使用场景

### 何时触发"无 CSS"流程?

✅ **常见场景**:
```html
<!-- 1. 完全无样式 -->
<p>Hello</p>

<!-- 2. 样式但无 font-family -->
<p style="color: red">Hello</p>

<!-- 3. CSS 类但无 font-family -->
<p class="highlight">Hello</p>
<!-- .highlight { color: red; } -->

<!-- 4. 继承父元素但无明确设置 -->
<body>
  <p>Hello</p>  <!-- 继承 body 的字体 -->
</body>
```

❌ **不会触发**:
```html
<!-- 1. 明确指定 font-family -->
<p style="font-family: Georgia">Hello</p>

<!-- 2. CSS 中指定 -->
<style>p { font-family: Arial; }</style>
<p>Hello</p>

<!-- 3. 继承有 font-family 的父元素 -->
<body style="font-family: Verdana">
  <p>Hello</p>
</body>
```

---

## 📈 流程复杂性

### 无 CSS 的流程

```
HTML → 样式计算 → InitialGenericFamily() → 系统映射 → 字体加载
(快速)      (快)          (快)            (快)      (第一次慢)
```

**总体**: 相对简单，依赖系统配置

### 有 CSS 的流程

```
HTML → 样式计算 → ConvertFontFamily() → Fallback 链 → 字体加载
(快速)    (快)       (快)             (可能长)     (第一次慢)
```

**总体**: 更复杂，需要处理 fallback 链

---

## 🔗 源代码位置速查

| 流程 | 文件 | 函数 |
|------|------|------|
| 无 CSS 转换 | `font_builder.cc` | `InitialGenericFamily()` |
| 有 CSS 转换 | `style_builder_converter.cc` | `ConvertFontFamily()` |
| 字体查询 | `font_selector.cc` | `GetFontData()` |
| 系统映射 | `generic_font_family_settings.h` | `Standard()` |
| Android 查询 | `font_cache_android.cc` | `GetGenericFamilyNameForScript()` |
| Font 创建 | `font_builder.cc` | `CreateFont()` |

---

## ✨ 快速记忆

**无 CSS**:
- 关键函数: `InitialGenericFamily()`
- 结果: `generic_family = kStandardFamily` (serif)
- 字体来源: 系统映射表

**有 CSS**:
- 关键函数: `ConvertFontFamily()`
- 结果: `family_name = "Georgia"`, `generic_family = kNoFamily`
- 字体来源: CSS 指定 + fallback

**记住**: 无 CSS 时会自动添加 `kStandardFamily`,有 CSS 时不会!

---

参考完整文档: [NO_CSS_FONT_FAMILY_FLOW.md](NO_CSS_FONT_FAMILY_FLOW.md)
