# Chromium 字体系统 - 名词詞典

## 🎯 核心概念速速查

### generic_family 是什么?

**generic_family** = "通用字体族" (不是具体某个字体)

```
通用族 (generic_family)
  ├─ kNoFamily (未指定)
  ├─ kStandardFamily (标准/sans-serif)
  ├─ kSerifFamily (衬线)
  ├─ kSansSerifFamily (无衬线)
  ├─ kMonospaceFamily (等宽)
  ├─ kCursiveFamily (手写体)
  └─ kFantasyFamily (幻想体)

具体字体名 (family_name)
  ├─ "Arial"
  ├─ "Times New Roman"
  ├─ "Roboto"
  ├─ "微软雅黑"
  └─ "DejaVu Sans"
```

**关键区别**:
- `generic_family = kSerifFamily` → "某种衬线字体" (模糊)
- `family_name = "Times New Roman"` → 这就是 "Times New Roman"! (具体)

---

## 📚 完整名词表

### 第一层: CSS 级别

| 名词 | 含义 | 例子 | 代码类型 |
|------|------|------|---------|
| **font-family** | CSS 属性 | `font-family: serif, Arial` | CSS 文本 |
| **CSSValue** | 解析后的 CSS 值对象 | CSSIdentifierValue("serif") | C++ 对象 |
| **CSSFontFamilyValue** | 具体字体名的 CSS 值 | CSSFontFamilyValue("Arial") | C++ 对象 |

**流程**:
```
CSS 文本: font-family: serif
  ↓ [CSS 解析器]
CSSIdentifierValue("serif")  ← CSSValue 对象
  ↓ [转换器]
FontDescription
```

---

### 第二层: 字体描述

| 名词 | 含义 | 包含信息 | 类型 |
|------|------|---------|------|
| **FontDescription** | 字体样式的完整描述 | generic_family, size, weight, style... | C++ 类 |
| **FontDescription::FamilyDescription** | 仅字体家族信息 | generic_family + family_name 链表 | C++ struct |
| **generic_family** (字段) | "通用族"字段 | kNoFamily, kStandardFamily... | enum |
| **family_name** (字段) | "具体字体名" | "Arial", "Roboto"... | AtomicString |

**层级关系**:
```
FontDescription {
  generic_family_ = kSerifFamily     ← "衬线" (模糊)
  family_name = "Times New Roman"    ← 具体字体名
  size = 16px
  weight = bold
  style = italic
  ...
}
```

---

### 第三层: 字体家族链表

| 名词 | 含义 | 用途 | 例子 |
|------|------|------|------|
| **FontFamily** | 单个字体的节点 | font-family 列表的链表节点 | {"Arial", next→{...}} |
| **SharedFontFamily** | 可共享的字体节点 | 内存优化 | 多个地方指向同一个节点 |
| **family_list_** | 整个字体家族链表 | 按优先级尝试多个字体 | {"serif", "Arial", "fallback"} |

**为什么是链表?**
```
CSS: font-family: serif, Arial, sans-serif
     ↓
FontDescription.family_list = {
  node1: {
    family_name: "serif" (通用)
    next: node2
  },
  node2: {
    family_name: "Arial" (具体)
    next: node3
  },
  node3: {
    family_name: "sans-serif" (通用)
    next: null
  }
}

这样做的目的: 
  font-family 第一个不可用 → 用第二个
  第二个也不可用 → 用第三个
  ...以此类推 (fallback 链)
```

---

### 第四层: 字体选择器

| 名词 | 含义 | 职责 | 代码位置 |
|------|------|------|---------|
| **FontSelector** | 基础选择器接口 | 定义选择字体的通用接口 | `font_selector.h` |
| **CSSFontSelector** | CSS 专用选择器 | 查询 @font-face 和系统字体 | `css_font_selector.h` |
| **GenericFontFamilySettings** | 通用族预设表 | 将通用族映射到具体字体 | `generic_font_family_settings.h` |

**职责链**:
```
Font 对象需要具体字体
  ↓
问 CSSFontSelector: "我需要 serif 字体"
  ↓
CSSFontSelector 问 GenericFontFamilySettings: "serif 映射到什么?"
  ├─ Linux 回答: "DejaVu Serif"
  └─ Android 回答: "" (空,继续查)
  ↓
若预设表为空,问 FontCache: "系统有 serif 吗?"
  ↓
FontCache 问 SkFontMgr: "查询 fonts.xml 吧"
```

---

### 第五层: 字体数据

| 名词 | 含义 | 包含什么 | 创建时机 |
|------|------|---------|---------|
| **Font** | 运行时字体对象 | FontDescription + CSSFontSelector + FontFallbackList | Style Calc |
| **FontFallbackList** | 字体降级链 | 尝试多个字体直到找到可用的 | Paint (延迟) |
| **SimpleFontData** | 实际字体文件数据 | 字形表、度量信息、SkTypeface | Paint (延迟) |
| **SkTypeface** | Skia 字体对象 | 字形光栅化、字体文件指针 | Paint (延迟) |

**对象生命周期**:
```
Style Calc:
  Font(description, selector) 创建
  ├─ FontDescription: 已有
  ├─ CSSFontSelector: 已有
  └─ FontFallbackList: ❌ 还没创建!

Paint 时:
  Font::PrimaryFont() 被调用
    ↓
  FontFallbackList 第一次初始化 ← 🟠 延迟在这里!
    ↓
  CSSFontSelector::GetFontData() 查询
    ↓
  SimpleFontData 对象创建 ← 字体文件加载!
    ↓
  SkTypeface 获得 ← Skia 准备好绘制
```

---

## 🔄 名词之间的关系

### 单一元素的完整对象模型

```
HTML: <span>hello</span>  (无 style)
  ↓ Style Calc 阶段
ComputedStyle {
  Font {
    FontDescription {
      generic_family_: kStandardFamily      ★ "标准族"
      family_name: ""                       ★ 待填充
      size: 16px
      weight: 400
      ...
    }
    CSSFontSelector* {                      ★ 指向字体数据库
      @font-face 缓存
      系统字体信息
    }
    FontFallbackList: null                  ★ 未初始化
  }
}
  ↓ Paint 阶段 (延迟触发)
Font::PrimaryFont() 被调用
  ↓
FontFallbackList 初始化
  ↓
CSSFontSelector::GetFontData(kStandardFamily)
  ↓
FontFallbackList::PrimaryFont()
  ↓
SimpleFontData {
  SkTypeface* → Roboto.ttf
  metrics: {
    ascent, descent, x-height...
  }
}
```

---

## 📊 名词对比表

### generic_family vs family_name

| 方面 | generic_family | family_name |
|------|----------------|------------|
| **定义** | 通用族类型 | 具体字体名 |
| **值** | enum (kNoFamily, kSerifFamily...) | AtomicString ("Arial", "Roboto"...) |
| **来源** | CSS 关键字 (serif, sans-serif) | CSS 字符串 ("Arial") 或预设表 |
| **精度** | 低 (模糊的族概念) | 高 (精确的字体) |
| **例子** | kSerifFamily | "Times New Roman" |
| **用途** | 分类,选择合适的映射 | 查询系统字体 |

---

### FontDescription vs Font

| 方面 | FontDescription | Font |
|------|-----------------|------|
| **时机** | Style Calc | Style Calc + Paint |
| **内容** | CSS 样式参数 (size, weight...) | 运行时字体对象 + 选择器 + 数据 |
| **可变** | 基本不变 | 动态初始化部分 (FontFallbackList) |
| **用途** | 存储 CSS 值 | 供 layout/paint 获取字形和度量 |
| **包含** | size, weight, style, generic_family... | FontDescription + CSSFontSelector + FontFallbackList |
| **创建者** | StyleBuilder / FontBuilder | FontBuilder (从 FontDescription) |

---

### CSSFontSelector vs FontSelector

| 方面 | FontSelector | CSSFontSelector |
|------|--------------|-----------------|
| **类型** | 基础接口 | 具体实现类 |
| **文件** | `font_selector.h` | `css_font_selector.h` |
| **职责** | 定义选择字体的接口 | 实现 CSS 特定的选择逻辑 |
| **功能** | 虚函数声明 | 查询 @font-face + 系统字体 |
| **用处** | 让不同选择器实现互换 | 被 Font 对象使用 |

---

## 💡 实用对比例子

### 有 CSS 时

```
CSS: <p style="font-family: Georgia">text</p>

ConvertFontFamily() 输出:
  FamilyDescription {
    generic_family_: kNoFamily           ← 不是通用族
    family_name: "Georgia"               ← 具体字体
  }

→ SimpleFontData 会查 Georgia 字体
```

### 无 CSS 时

```
CSS: <p>text</p>  (无 font-family)

FontBuilder::CreateFont() 处理:
  FontDescription {
    generic_family_: kNoFamily  ← 默认
      ↓ [InitialGenericFamily()]
    generic_family_: kStandardFamily  ← 改为 "标准族"
    family_name: ""  ← 等待映射表填充
  }

FontSelector::FamilyNameFromSettings() 输出:
  family_name: "DejaVu Sans"  (Linux)
  或
  family_name: "Roboto"  (Android 从 fonts.xml)

→ SimpleFontData 会查 DejaVu Sans 或 Roboto 字体
```

---

## 🗺️ 名词导航地图

```
HTML CSS 文本
  ↓ [CSS Parser]
CSSValue (CSSIdentifierValue, CSSFontFamilyValue)
  ↓ [ConvertFontFamily()]
FontDescription {
  generic_family_ (通用族类型)
  family_name (具体字体名)
  + 其他属性 (size, weight...)
}
  ↓ [FontBuilder::CreateFont()]
Font {
  description_: FontDescription
  font_selector_: CSSFontSelector ★
  font_list_: FontFallbackList (null)
}
  ↓ [Paint 时]
FontFallbackList {
  CSSFontSelector::GetFontData()
    ├─ 查 GenericFontFamilySettings
    ├─ 查 FontFaceCache (@font-face)
    └─ 查 FontCache (系统字体)
}
  ↓
SimpleFontData {
  SkTypeface* (字形数据)
}
```

---

## ❓ 常见疑惑解答

### Q1: generic_family 和 family_name 一直在变?

**A**: 不,他们的含义是:
- `generic_family`: **类型** (kSerifFamily 就是 kSerifFamily)
- `family_name`: **名字** (会通过映射表或查询改变)

```
generic_family: kSerifFamily  ← 这个不变
family_name: ""  ← 初始空
  ↓ [查询映射表]
family_name: "Times New Roman"  ← 现在有值了
```

### Q2: 为什么有 FontSelector 和 CSSFontSelector?

**A**: 多态设计。可能有其他类型的选择器:
- `CSSFontSelector` - CSS 样式中的字体
- 未来可能有 `SVGFontSelector` - SVG 中的字体
- 等等

只要都继承 `FontSelector` 接口,Font 对象不用改。

### Q3: FontFallbackList 为什么延迟初始化?

**A**: 性能! 
```
假设页面有 10000 个元素,但只显示 100 个
  ↓
若立即初始化 FontFallbackList
  → 要加载 10000 个字体 (浪费!)
  
延迟初始化
  → 只加载显示的 100 个字体 (高效!)
```

### Q4: generic_family == kNoFamily 和 == kStandardFamily 有什么区别?

**A**:
```
kNoFamily (初始状态)
  含义: "未指定通用族"
  来源: FontDescription() 构造函数默认值
  状态: 临时的

kStandardFamily (初始值)
  含义: "标准族" (无衬线)
  来源: InitialGenericFamily() 设置
  状态: 正式使用
  
无 CSS 时:
  kNoFamily (构造)
    ↓
  kStandardFamily (InitialGenericFamily 覆盖)
    ↓ [映射表查询]
  "DejaVu Sans" 或 "Roboto"
```

---

## 🎓 学习路径

### 初级 - 理解基础概念
1. ✅ generic_family = "通用族"
2. ✅ family_name = "具体字体"
3. ✅ FontDescription = "字体参数容器"

### 中级 - 理解对象关系
1. FontDescription 存什么 (generic_family + family_name + size/weight...)
2. Font 包什么 (FontDescription + CSSFontSelector + FontFallbackList)
3. CSSFontSelector 的作用 (查询字体数据库)

### 高级 - 理解完整流程
1. CSS 文本 → CSSValue → FontDescription (ConvertFontFamily)
2. FontDescription → Font 对象 (FontBuilder::CreateFont)
3. Paint 时 Font → SimpleFontData (FontFallbackList 初始化)

---

## 快速查询表

```
╔════════════════════════════════════════════════════════════╗
║ 名词               英文               含义                  ║
╠════════════════════════════════════════════════════════════╣
║ 通用族             generic_family     CSS 关键字类型         ║
║ 具体字体名         family_name        真实的字体文件名       ║
║ 字体描述           FontDescription    CSS 样式参数          ║
║ 字体对象           Font               运行时字体           ║
║ 字体选择器         CSSFontSelector    字体数据库查询       ║
║ 字体数据           SimpleFontData     字体文件内容         ║
║ 字体家族链表       FontFallbackList   备用字体列表         ║
║ 预设映射表         GenericFontFamily  通用族→具体字体      ║
╚════════════════════════════════════════════════════════════╝
```

---

**记住**: 无 CSS 时的流程就是:
```
kNoFamily → kStandardFamily → (映射表查询) → "DejaVu Sans" 或 "Roboto"
```

这三个名词就覆盖了从"未指定"到"具体字体"的完整过程! 🎯

