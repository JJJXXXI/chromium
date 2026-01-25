# 字体格式二进制结构 & Hinting 指令详解 & 实际示例

> 从字体文件二进制层面分析格式差异、hinting 机制、以及实际字体示例

## 📋 目录

1. [字体文件的整体结构](#字体文件的整体结构)
2. [每种格式在文件中的体现](#每种格式在文件中的体现)
3. [Hinting 指令详解](#hinting-指令详解)
4. [实际字体文件示例](#实际字体文件示例)
5. [Hex 对比分析](#hex-对比分析)

---

## 🏗️ 字体文件的整体结构

### OpenType 字体的通用架构

```
Arial-Bold.ttf / Arial-Bold.otf
│
├─ [字体头]
│  ├─ Offset Table（偏移表）
│  │   └─ scalerType, numTables, ...
│  └─ Table Directory（表目录）
│      ├─ 每条记录 12 bytes
│      ├─ tag: 表名（4 字节）
│      ├─ checksum
│      ├─ offset: 文件中的偏移
│      └─ length: 表的长度
│
├─ [必需表]
│  ├─ head (54 bytes)     - 字体头信息
│  ├─ hhea (36 bytes)     - 水平排版信息
│  ├─ maxp (32 bytes)     - 最大轮廓信息
│  ├─ OS/2 (96 bytes)     - 操作系统信息
│  ├─ name (可变)         - 字体名称表
│  ├─ cmap (可变)         - 码点→glyph ID 映射
│  ├─ hmtx (可变)         - 字形水平指标
│  └─ post (可变)         - PostScript 信息
│
├─ [字形轮廓表] ← 格式不同之处！
│  ├─ glyf (可变)         - TrueType 轮廓 ✅ Glyf 格式
│  │   ├─ 简单字形数据
│  │   └─ 复合字形引用
│  │
│  ├─ CFF (可变)          - PostScript 轮廓 ✅ CFF/CFF2 格式
│  │   ├─ CharStrings 指令
│  │   └─ Global Subrs
│  │
│  ├─ COLR (可变)         - 彩色分层 ✅ COLR/COLR v1 格式
│  │   ├─ Paint records
│  │   └─ BaseGlyph records
│  │
│  ├─ CBDT (可变)         - 彩色位图 ✅ CBDT 格式
│  │   └─ PNG 图像数据
│  │
│  ├─ sbix (可变)         - Apple 位图 ✅ sbix 格式
│  │   └─ 图像数据
│  │
│  └─ EBDT (可变)         - 嵌入位图 ✅ EBDT 格式
│      └─ 位图字形数据
│
├─ [可选表]
│  ├─ gvar (可变)         - 可变轴增量 ✅ CFF2 需要
│  ├─ fvar (可变)         - 可变轴定义
│  ├─ avar (可变)         - 轴归一化映射
│  ├─ HVAR (可变)         - 水平指标变量
│  ├─ VVAR (可变)         - 竖直指标变量
│  ├─ STAT (可变)         - 样式属性表
│  └─ MATH (可选)         - 数学排版
│
└─ [数据]
   └─ 具体的表数据
```

---

## 📄 每种格式在文件中的体现

### 1. **Glyf 格式** (TrueType 轮廓)

#### 文件结构

```
Table Directory 中有 'glyf' 表：
  tag: 'glyf'
  checksum: 0x1a2b3c4d
  offset: 0x1000 (文件中的位置)
  length: 0x50000 (50KB)

Glyf 表内容：
  [glyph 0 数据]  12 bytes header + 轮廓数据
  [glyph 1 数据]
  [glyph 2 数据]
  ...
  [glyph 1500 数据]

Loca 表（定位表，必需）：
  offset[0] = 0x0000        ← glyph 0 的偏移量
  offset[1] = 0x0042        ← glyph 1 的偏移量
  offset[2] = 0x0088        ← glyph 2 的偏移量
  ...
```

#### Glyf 字形的二进制结构

```
// 字符 "A" 的 glyf 数据（简化）
00 02              ← numberOfContours = 2（两个轮廓：外轮廓和中间洞）
00 00              ← xMin = 0
00 00              ← yMin = 0
02 58              ← xMax = 600 (0x258)
02 BC              ← yMax = 700 (0x2BC)

// 轮廓信息
00 04              ← endPtsOfContours[0] = 4（第一个轮廓的最后点索引）
00 08              ← endPtsOfContours[1] = 8（第二个轮廓的最后点索引）

// Hinting 指令（TrueType VM 程序）
00 0A              ← instructionLength = 10 bytes
62 B0 61 B1 ...    ← [Hinting 指令字节码]

// 标志（点类型）
01 01 01 01        ← 第 0-3 点：on curve
00 01 01           ← 第 4-6 点：off curve / on curve
01                 ← 第 7 点：on curve

// X 坐标（相对编码，节省空间）
00                 ← 点 0: x = 0
96                 ← 点 1: x = 0 + 150 = 150
96                 ← 点 2: x = 150 + 150 = 300
-96                ← 点 3: x = 300 - 150 = 150
...

// Y 坐标
00                 ← 点 0: y = 0
2C                 ← 点 1: y = 0 + 700 = 700
-2C                ← 点 2: y = 700 - 700 = 0
...

// 完成！这就是"A"字形的轮廓数据
```

#### Hinting 指令示例（Glyf）

```
// 简化的 TrueType 指令伪代码
62 B0 61 B1 62 B0 64 B0 61 B1

解释：
62       SLOOP    ← 重复循环设置
B0 61    ...      ← 特定操作
B1       ...      ← 对齐操作
```

---

### 2. **CFF 格式** (PostScript 轮廓)

#### 文件结构

```
Table Directory 中有 'CFF ' 表（注意最后有空格！）：
  tag: 'CFF '
  offset: 0x2000
  length: 0x30000 (48KB，比 Glyf 小 20%)

CFF 表内容：
  ├─ CFF Header (4 bytes)
  │   └─ major, minor, hdrSize, offSize
  │
  ├─ Name INDEX
  │   ├─ count: 1
  │   └─ "Arial Bold"
  │
  ├─ Top DICT INDEX
  │   ├─ Private dict offset
  │   ├─ CharStrings offset
  │   ├─ ROS (CID font info)
  │   └─ ...参数
  │
  ├─ String INDEX
  │   └─ (非标准字符串)
  │
  ├─ Global Subrs
  │   ├─ 常用曲线代码段
  │   ├─ "stem hint"
  │   ├─ "curve segment"
  │   └─ 节省空间
  │
  └─ CharStrings INDEX
      ├─ charstring[0] = "A" 的指令
      ├─ charstring[1] = "B" 的指令
      └─ charstring[1500] = "..." 的指令
```

#### CFF CharString 的二进制结构

```
// 字符 "A" 的 CFF CharString 指令
60 64 6D 72        ← rmoveto(0, 100)  相对移动
2C 01 72           ← rlineto(300, 700) 相对直线
2C 01 4E 72        ← rlineto(300, -700) 相对直线
76 64 6D 72        ← ...

与 Glyf 对比：
  Glyf：坐标 + 标志（144 bytes）
  CFF：命令序列（110 bytes）← 节省 24%
```

#### CFF 中的 Hinting

```
// CFF 的 Hinting 指令
hstem 50 20        ← 水平缩放区间：y=50, 高度=20
vstem 100 30       ← 竖直缩放区间：x=100, 宽度=30
rmoveto 100 50     ← 移动到 (100, 50)
rlineto 200 0      ← 直线
dotsection         ← 分割区间

目的：
  ├─ 告诉渲染器"这些区间重要"
  ├─ 小尺寸（12px）时对齐到像素
  └─ 防止字形在 ppem 变化时"抖动"
```

---

### 3. **CFF2 格式** (可变 PostScript)

#### 文件结构

```
Table Directory 中有 'CFF2' 表：
  tag: 'CFF2'
  offset: 0x2000
  length: 0x40000 (64KB)

与 CFF 的区别：
  ├─ 新增：VarStore（可变数据）
  ├─ 新增：ItemVariationStore 结构
  ├─ 可变轴信息
  ├─ CharString 指令中包含可变数据引用
  └─ 文件略大（支持多个设计空间维度）

CFF2 表内容：
  ├─ CFF2 Header (5 bytes，多一个字节)
  ├─ Top DICT INDEX
  ├─ Global Subrs
  ├─ CharStrings INDEX（可变）
  ├─ VarStore ← ⭐ 新增！
  │   └─ 每个字形的增量数据
  │       Light weight → Bold weight
  │       Condensed width → Extended width
  │       等等
  └─ ItemVariationStore
```

#### CFF2 可变数据示例

```
// 原始 CharString (Light)
A_light = {
  rmoveto(0, 100)
  rlineto(300, 700)
  rlineto(300, -700)
  endchar
}

// 增量数据 (Light → Bold)
A_delta = {
  x_delta[0] = +5     // Bold 时，第一个 rmoveto 的 x 偏移 +5
  y_delta[0] = +10    // 第一个 rmoveto 的 y 偏移 +10
  x_delta[1] = +10    // 第一个 rlineto 的 x 偏移 +10
  y_delta[1] = +50    // 第一个 rlineto 的 y 偏移 +50
}

// 用户请求 weight=0.7:
A_rendered = {
  rmoveto(0 + 0.7×5, 100 + 0.7×10)
  rlineto(300 + 0.7×10, 700 + 0.7×50)
  rlineto(300 + 0.7×10, -700 - 0.7×50)
  endchar
}
```

---

### 4. **COLRv0 格式** (固定彩色分层)

#### 文件结构

```
Table Directory 中有两个表：
  tag: 'COLR'   ← 彩色定义
  tag: 'CPAL'   ← 调色板

COLR 表内容：
  ├─ BaseGlyph records
  │   └─ glyph_id = 1000 (emoji 😀)
  │       └─ firstLayerIndex = 0
  │       └─ numLayers = 5
  │
  ├─ Layer records
  │   ├─ Layer 0: glyph_id=1001, palette_index=0 (红色)
  │   ├─ Layer 1: glyph_id=1002, palette_index=1 (蓝色)
  │   ├─ Layer 2: glyph_id=1003, palette_index=2 (黄色)
  │   └─ ...
  │
  └─ Version 0 (固定)

CPAL 表内容：
  ├─ Palette count = 2
  ├─ Palette 0 (Light mode)
  │   ├─ Color[0] = #FF0000 (红)
  │   ├─ Color[1] = #0000FF (蓝)
  │   ├─ Color[2] = #FFFF00 (黄)
  │   └─ ...
  │
  └─ Palette 1 (Dark mode)
      ├─ Color[0] = #CC0000
      ├─ Color[1] = #0000CC
      └─ ...
```

#### 二进制示例

```
// COLR 表中的 BaseGlyph record for emoji "😀"
00 03 E8        ← glyphID = 1000 (😀)
00 00           ← firstLayerIndex = 0
00 05           ← numLayers = 5

// Layer records
00 01 E9 00     ← Layer 0: glyphID=1001, paletteIndex=0
00 02 EA 01     ← Layer 1: glyphID=1002, paletteIndex=1
00 03 EB 02     ← Layer 2: glyphID=1003, paletteIndex=2
00 04 EC 03     ← Layer 3: glyphID=1004, paletteIndex=3
00 05 ED 04     ← Layer 4: glyphID=1005, paletteIndex=4

// CPAL 表：颜色数据
FF 00 00 FF     ← Color[0] = #FF0000 (红，BGRA 格式)
00 00 FF FF     ← Color[1] = #0000FF (蓝)
FF FF 00 FF     ← Color[2] = #FFFF00 (黄)
...

渲染流程：
  😀 = Layer[0] (glyph_1001, 红色)
     + Layer[1] (glyph_1002, 蓝色)
     + Layer[2] (glyph_1003, 黄色)
     + ...（叠加）
```

---

### 5. **COLRv1 格式** (高级彩色 + 可变)

#### 文件结构

```
Table Directory 中有：
  tag: 'COLR'   ← 增强为 v1
  tag: 'CPAL'
  tag: 'fvar'   ← 可变轴（新增）

COLRv1 Paint 记录：
  ├─ Paint 0: Solid Fill
  │   └─ paletteIndex = 0, alpha = 255
  │
  ├─ Paint 1: Linear Gradient
  │   ├─ x0, y0, x1, y1 (渐变方向)
  │   ├─ varStoreIndex (可变数据)
  │   └─ colorStop[] (颜色停止点)
  │
  ├─ Paint 2: Radial Gradient
  │   ├─ centerX, centerY, radius
  │   └─ ...
  │
  ├─ Paint 3: Composite
  │   ├─ sourceLayerIndex
  │   ├─ compositeMode (multiply, screen, etc.)
  │   └─ backdropLayerIndex
  │
  └─ Paint 4: Var Solid
      ├─ paletteIndex
      ├─ varStoreIndex
      └─ alpha (可变)

BaseGlyph records (v1):
  └─ glyphID = 1000
      └─ Paint 序列 (可能很复杂)
```

#### 二进制示例（渐变 Emoji）

```
// Google "G" logo（使用渐变）
BaseGlyph 1000:
  Paint[0] = Linear Gradient
    x0=0, y0=0, x1=100, y1=100
    colorStop[0] = (0%, 红色 #FF0000)
    colorStop[1] = (100%, 橙色 #FF8800)
    varStoreIndex = 42 (可变数据 offset)

Paint[1] = Solid Fill
  paletteIndex = 2 (蓝色)
  alpha = 200

Paint[2] = Composite
  sourceLayerIndex = Paint[0]
  compositeMode = multiply
  backdropLayerIndex = Paint[1]
```

---

### 6. **CBDT 格式** (Google 彩色位图)

#### 文件结构

```
Table Directory 中有两个表：
  tag: 'CBDT'   ← 位图数据
  tag: 'CBLC'   ← 位图定位

CBDT 表内容：
  ├─ Version (4 bytes)
  │
  └─ Bitmap data (按字形顺序)
      ├─ [PNG 压缩图像] glyph 0 (emoji 😀 at 64×64)
      ├─ [PNG 压缩图像] glyph 1 (emoji 😁 at 64×64)
      ├─ [PNG 压缩图像] glyph 0 (emoji 😀 at 128×128)  ← 同一字形的高分辨率
      ├─ [PNG 压缩图像] glyph 1 (emoji 😁 at 128×128)
      └─ ... (256×256, 512×512 等)

CBLC 表内容：
  ├─ Version (4 bytes)
  ├─ numSizes (4 bytes) = 4  ← 4 个不同尺寸
  │
  └─ BitmapSize 记录
      ├─ BitmapSize[0] (64×64 ppem)
      │   ├─ indexSubTableArrayOffset
      │   ├─ imageSize = 64 (ppem)
      │   ├─ bigMetrics { height, width, ... }
      │   └─ indexSubTable (每个字形的偏移)
      │
      ├─ BitmapSize[1] (128×128 ppem)
      ├─ BitmapSize[2] (256×256 ppem)
      └─ BitmapSize[3] (512×512 ppem)
```

#### 二进制示例

```
// CBLC 表
00 00 00 02     ← version = 2
00 00 00 04     ← numSizes = 4 (4 种尺寸)

// BitmapSize[0] @ 64×64
00 00 01 00     ← indexSubTableArrayOffset
00 40           ← imageSize = 64 ppem
00 40           ← height = 64
00 40           ← width = 64
...

// CBDT 表：PNG 数据
89 50 4E 47 0D 0A 1A 0A  ← PNG 文件签名
...
(PNG 压缩数据 4-5KB)

89 50 4E 47 0D 0A 1A 0A  ← 下一个 PNG
...
```

---

### 7. **sbix 格式** (Apple 位图)

#### 文件结构

```
Table Directory 中有：
  tag: 'sbix'

sbix 表内容：
  ├─ Version (4 bytes) = 1
  ├─ flags = 0
  ├─ numStrikes = 3  ← 3 种尺寸
  │
  └─ strikeOffset[]
      ├─ strikeOffset[0] = 0x100 (64×64)
      ├─ strikeOffset[1] = 0x5000 (128×128)
      └─ strikeOffset[2] = 0xA000 (256×256)

Strike 记录：
  ├─ ppem = 64  ← 像素大小
  ├─ resolution = 72 (DPI)
  │
  └─ glyph 数据
      ├─ originOffsetX = 0
      ├─ originOffsetY = 64
      ├─ graphicType = 'png ' 或 'jpg '
      ├─ data (PNG/JPEG 二进制)
      │
      ├─ glyphOffset[1500] ← 定位表
      └─ ...
```

#### 二进制示例

```
// sbix 表
00 00 00 01     ← version = 1
00 00 00 00     ← flags = 0
00 00 00 03     ← numStrikes = 3

// strikeOffsets
00 00 00 20     ← Strike 1 at offset 0x20
00 00 50 00     ← Strike 2 at offset 0x5000
00 00 A0 00     ← Strike 3 at offset 0xA000

// Strike 1 @ 64×64
00 40           ← ppem = 64
00 48           ← resolution = 72

// Glyph records for Strike 1
00 00           ← originOffsetX = 0
00 40           ← originOffsetY = 64
70 6E 67 20     ← graphicType = 'png '
89 50 4E 47 ... ← PNG 数据
```

---

### 8. **EBDT 格式** (嵌入位图，较旧)

#### 文件结构

```
Table Directory 中有两个表：
  tag: 'EBLC'   ← 位图定位（定义表）
  tag: 'EBDT'   ← 位图数据

EBLC 表内容：
  └─ BitmapSize 记录
      ├─ ppem = 48
      ├─ subtable format (4 种子格式)
      ├─ IndexSubTable
      │   └─ 每个字形的偏移和大小
      └─ ...

EBDT 表内容：
  ├─ 位图数据（LZ77 压缩）
  └─ 按 EBLC 中的偏移定位

特点：
  ├─ 压缩：LZ77（比 PNG 压缩比低）
  ├─ 仅黑白：无彩色支持
  ├─ 固定尺寸：不支持缩放
  └─ 已过时：基本不再使用
```

---

## 🎯 Hinting 指令详解

### Hinting 的本质

```
问题：
  在 12px 屏幕上，字形的某些部分会"掉像素"
  例：Arial "I" 在 12px 时可能 0 像素宽

解决方案：Hinting
  告诉渲染器如何微调，使字形在小尺寸下仍可读
```

### TrueType Hinting（Glyf）

#### 虚拟机指令集

```
TrueType VM 是一个栈机（Stack Machine）

关键指令：
  SETLOOP n          ← 设置循环次数
  MIRP[.5] p, zone   ← 对齐点 p 到参考线（zone）
  MDRP[...] p        ← 移动并设置 RP0 到 p
  ROLL               ← 栈顶 3 个元素旋转
  DUP                ← 复制栈顶
  POP                ← 弹出栈顶
  PUSH n             ← 压入数值
  ...（共 256 个指令）

执行环境：
  ├─ Twilight zone（中间点）
  ├─ Graphics state（大小、颜色、区间定义）
  ├─ 栈（最多 65535 个元素）
  └─ 循环计数器等
```

#### 具体例子：Arial "I" 的 Hinting

```
问题：
  Arial "I" 字形宽度设计为 250 units
  但 12px 渲染时，1 px 可能对应 20.8 units
  结果：250 ÷ 20.8 = 12 像素 → 舍入后变成 11 像素 → 看起来很细

TrueType Hinting 指令（伪代码）：
  
  fpgm (Font Program) 和 prep (Control Value Program):
    └─ 全局设置（所有字形通用）

  glyph "I" 的指令：
    
    SETLOOP 2
      ← 设置循环，处理左右两条竖边
    
    MIRP[...] p0, zone1
      ← 对齐左边的点 p0 到水平 zone 1
      ← zone 1 可能是 "字体的 stem 厚度位置"
    
    MIRP[...] p1, zone1
      ← 对齐右边的点 p1 到同一 zone
    
    MDAP[...] p2
      ← 设置顶部点 p2 到网格
    
    结果：
      左边 x = 像素网格的 x 坐标
      右边 x = 像素网格的 x + stem_width_pixels
      ← stem_width_pixels 是 hinting 指令保证的对齐宽度
      ← 确保 "I" 在 12px 时正好是 3 像素宽（明确可见）

二进制形式（真实字体数据）：
  62        SETLOOP
  B0 61     指令...
  B1        MIRP
  62        SETLOOP
  B0 61     指令...
  64        MDAP
  B0        参数
  61        ...
```

#### Hinting 的作用对比

```
12px "I" 字形，没有 Hinting vs 有 Hinting：

没有 Hinting：
  I_unshinted = {
    left_x = 2.1px ─→ 舍入到 2px
    right_x = 14.9px ─→ 舍入到 15px
    宽度 = 15 - 2 = 13px （不稳定，看起来很细）
  }

有 Hinting：
  I_hinted = {
    left_x = 2px ─────────────→ 对齐到像素网格
    right_x = 5px（左 + stem） → 强制对齐
    宽度 = 5 - 2 = 3px （稳定，清晰）
  }

视觉效果：
  未 hint：█░░░░░░░░░ ← 很细，难以识别
  已 hint：███░░░░░░░ ← 清晰，易识别
```

### CFF Hinting（PostScript）

```
CFF 的 hinting 更简单：

hstem 50 20     ← 定义水平 stem 区间
                   y = 50, height = 20
                   告诉渲染器："50-70 这个区间很重要"

vstem 100 30    ← 定义竖直 stem 区间
                   x = 100, width = 30

dotsection      ← "接下来的是点（Dot）区间"

hint 限制：
  ├─ 仅定义重要区间
  ├─ 不像 TrueType 那么精细
  └─ 现代屏幕高分辨率下效果不明显
```

---

## 🎨 实际字体文件示例

### 示例 1：Roboto Regular (Glyf 格式，Web 字体)

```
文件名：Roboto-Regular.ttf
文件大小：165 KB

文件结构：
  ├─ Offset Table (12 bytes)
  ├─ Table Directory (288 bytes，24 个表)
  │   ├─ head: 54 bytes
  │   ├─ hhea: 36 bytes
  │   ├─ maxp: 32 bytes
  │   ├─ OS/2: 96 bytes
  │   ├─ name: 3.2 KB
  │   ├─ cmap: 12 KB
  │   ├─ post: 32 bytes
  │   ├─ hmtx: 15 KB (1500 字形 × 4 bytes/字)
  │   ├─ glyf: 95 KB  ← 最大！
  │   ├─ loca: 6 KB
  │   ├─ head, hhea, ... (其他)
  │   └─ ...
  │
  └─ 数据部分 (165 KB)

字形信息：
  ├─ 字形数量：1500
  ├─ Hinting：YES（TrueType 指令）
  ├─ 可变轴：NO
  ├─ 彩色：NO
  └─ 平均字形大小：95 KB ÷ 1500 = 63 bytes/字形

实际字形大小分布：
  ├─ 空字形（glyph 0）：8 bytes
  ├─ 标点（","）：28 bytes
  ├─ 字母（"A"）：120 bytes （轮廓 + hinting）
  ├─ 复杂字形（"@"）：250 bytes
  └─ 最大字形（"ñ" 的某种变体）：1.2 KB

Web 使用：
  ├─ WOFF2 压缩：→ 45 KB （27% 原大小）
  ├─ 加载时间：⚡ <100ms（4G 网络）
  └─ 缓存：浏览器可缓存
```

### 示例 2：Adobe Garamond (CFF 格式，桌面字体)

```
文件名：AGaramond-Regular.otf
文件大小：132 KB （比 Roboto 小 20%）

文件结构：
  ├─ Offset Table
  ├─ Table Directory (288 bytes，26 个表)
  │   ├─ ... (head, name 等)
  │   ├─ CFF (Not 'CFF ')：85 KB  ← 比 glyf 小！
  │   └─ ...
  │
  └─ CFF 表详细：
      ├─ Header (4 bytes)
      ├─ Name INDEX (32 bytes)
      ├─ Top DICT INDEX (200 bytes)
      ├─ String INDEX (150 bytes)
      ├─ Global Subrs (5 KB) ← 可复用的代码段
      ├─ FD Array (Private dict)
      └─ CharStrings (75 KB)

字形信息：
  ├─ 字形数量：1500
  ├─ Hinting：YES（PostScript 风格）
  ├─ 可变轴：NO
  ├─ 彩色：NO
  └─ 平均字形大小：75 KB ÷ 1500 = 50 bytes/字形 ← Glyf 的 79%

字形大小对比（与 Roboto）：
  CFF   Glyf  字形
  24 B  28 B  "," (逗号)
  95 B  120B  "A" (字母 A)
  180B  250B  "@" (at 符号)
  
  → CFF 平均小 20% ✓

文件压缩：
  ├─ TTF (Glyf) WOFF2：45 KB
  ├─ OTF (CFF) WOFF2：38 KB （13% 更小）
  └─ Desktop 使用：通常不压缩，直接使用 132 KB
```

### 示例 3：Inter (可变字体，Glyf)

```
文件名：Inter.var.ttf 或 Inter[slnt,wght].ttf
文件大小：486 KB （比 Roboto 大 3 倍，但包含多个轴）

特殊表：
  ├─ glyf：120 KB （与非可变版本类似）
  ├─ gvar：240 KB ← ⭐ 新增！增量数据
  ├─ fvar (可变轴定义)：1.2 KB
  │   ├─ wght (weight): 100-900
  │   ├─ slnt (slant): -12° - 0°
  │   └─ GRAD (grade): -200 - 0
  │
  ├─ avar（轴归一化，可选）：~2 KB
  ├─ HVAR（水平指标变量）：~50 KB
  ├─ VVAR（竖直指标变量）：~50 KB
  └─ STAT（样式属性表）：~10 KB

可变空间：
  ├─ wght 轴：weight 400 → 700
  │   ├─ Light (400): 所有字形的"基础"版本
  │   ├─ Regular (500): 400 + 0.5 × (700 - 400)
  │   ├─ Semibold (600): 400 + 1.0 × (700 - 400) 的一部分
  │   └─ Bold (700): 完整的加粗版本
  │
  ├─ slnt 轴：-12° ← → 0°
  │   ├─ -12°: 意大利体（全斜）
  │   ├─ -6°: 半斜
  │   └─ 0°: 正常
  │
  └─ GRAD 轴：品级调整

单个字形的数据：
  glyf 中的 "A"：
    ├─ 基础版本（400 weight）：120 bytes
    ├─ gvar 中的增量：
    │   ├─ weight 增量：+45 bytes （700 weight 时的差异）
    │   ├─ slant 增量：+30 bytes （-12° 时的差异）
    │   └─ grade 增量：+15 bytes
    │
    └─ 总计：210 bytes （基础 + 增量引用）

插值示例：
  用户请求 weight=600, slant=-6°:
    A = A_base +
        (600-400)/(700-400) × A_delta_weight +
        (-6-0)/(-12-0) × A_delta_slant
  
  → 得到 weight=600, slant=-6° 的 "A" 轮廓

Web 使用：
  ├─ 单文件：486 KB → WOFF2: 135 KB
  ├─ 传统方法（4 个文件）：4 × 45 KB = 180 KB
  ├─ 节省：180 - 135 = 45 KB ✓ （25% 省流量）
  └─ 灵活性：⚡ 无限种风格组合
```

### 示例 4：Noto Color Emoji (COLRv1 + 可变)

```
文件名：NotoColorEmoji-Regular.ttf
文件大小：12.5 MB （非常大！）

特殊表：
  ├─ glyf：800 KB
  ├─ COLR：5.2 MB ← ⭐ 最大！
  ├─ CPAL：12 KB
  ├─ fvar：2.5 KB
  │   ├─ wght: 400-600
  │   └─ FILL: 0-100 (填充比例)
  │
  └─ ...

COLR 中的 Paint 样例（"😀" 字形）：

BaseGlyph 1 (😀):
  └─ Paint Composite {
      sourceLayerIndex = Paint[0],
      compositeMode = multiply,
      backdropLayerIndex = Paint[1]
    }
    
    Paint[0] = Radial Gradient {
      centerX = 200, centerY = 200,
      radius = 150,
      colorStop[0] = (0%, 黄色 #FFD700),
      colorStop[1] = (100%, 橙色 #FFA500),
      varStoreIndex = 42
    }
    
    Paint[1] = Solid {
      paletteIndex = 0 (眼睛黑色),
      alpha = 255
    }

渲染流程：
  😀 = {
    眼睛 (glyph_id_10, 黑色)
    + 嘴巴 (glyph_id_11, 黑色)
    + 脸 (glyph_id_12, 黄→橙渐变)
    + 脸部阴影 (glyph_id_13, multiply 模式)
    + ...
  }

文件大小分析：
  ├─ 字形总数：3600+ (emoji 字符)
  ├─ glyf 表：800 KB (简单轮廓)
  ├─ COLR 表：5.2 MB (颜色定义和渐变数据)
  │   └─ 平均每个 emoji：1.4 KB
  ├─ 浏览器缓存关键：必须缓存，不能每页加载
  └─ 加载方式：
      仅在需要时加载（而非全局加载）

Android 使用：
  ├─ 预装在系统
  ├─ 路径：/system/fonts/NotoColorEmoji.ttf
  ├─ 由 Fontations 读取元数据
  └─ 字形按需渲染
```

### 示例 5：Noto Sans CJK JP (Glyf + TTC)

```
文件名：NotoSansCJK-Regular.ttc
文件大小：110 MB （单个文件，包含中日韩）

TTC 文件结构：
  ├─ TTC Header
  │   ├─ version = 0x00010000 (TTC 格式版本)
  │   ├─ numFonts = 4
  │   │   └─ 包含 4 种语言变体
  │   │       ├─ 中文 (中 Simplified Chinese)
  │   │       ├─ 繁体 (香港 Traditional HK)
  │   │       ├─ 繁体 (台湾 Traditional Taiwan)
  │   │       └─ 日文 (Japanese)
  │   │
  │   └─ offsetTable[4] (指向 4 个字体)
  │       ├─ offset[0] = 0x1000
  │       ├─ offset[1] = 0x5000000
  │       ├─ offset[2] = 0xA000000
  │       └─ offset[3] = 0xF000000
  │
  ├─ Shared tables (所有字体共享)
  │   ├─ 'glyf'：105 MB ← 共享字形！
  │   ├─ 'loca'：250 KB
  │   ├─ 'hmtx'：5 MB
  │   └─ 'cmap'：2 MB
  │
  └─ Language-specific tables
      ├─ Font 0 'name' 表：中文字体名
      ├─ Font 1 'name' 表：繁体字体名
      ├─ Font 2 'name' 表：繁体台湾字体名
      └─ Font 3 'name' 表：日文字体名

字形数量：
  ├─ 字形总数：18,000+ (汉字 + 假名 + 拉丁)
  ├─ 中文字形：13,000+
  ├─ 日文假名：80
  ├─ 拉丁字母和标点：400
  └─ 平均字形大小：105 MB ÷ 18000 ≈ 6 KB/字（复杂！）

可变轴：
  ├─ wght: 400-700
  ├─ wdth: 75-100 (宽度)
  ├─ opsz: 8-144 (光学尺寸)
  └─ Grade: -200-0

使用方式（Android）：
  ├─ 系统预装一个 TTC 文件
  ├─ 不同 app 可索引不同字体变体
  │   └─ NotoSansCJKjp-Regular.ttf:0 → 日文
  │   └─ NotoSansCJKjp-Regular.ttf:1 → 繁体
  │
  ├─ Fontations 读取时：
  │   ├─ 解析 TTC 头
  │   ├─ 发现 4 个字体
  │   └─ 共享 glyf 表减少内存占用
  │
  └─ 按需加载：字形在需要时从内存映射文件读取

文件节省：
  ├─ 分开文件：4 × 110 MB = 440 MB
  ├─ TTC 合并：110 MB （共享 glyf 表）
  ├─ 节省：330 MB ✓ （75% 节省）
```

---

## 🔍 Hex 对比分析

### 同一字符的不同格式对比

#### 字符 "A" 在 Glyf 和 CFF 中的对比

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

【Glyf 格式】(Arial-Bold.ttf)

00 01              ← numberOfContours = 1
00 00 02 58        ← bounding box
00 04              ← endPtsOfContours[0] = 4

00 0A              ← instructionLength = 10 bytes
┌─────────────────────────────────────────┐
│ 62 B0 61 B1 62 B0 64 B0 61 B1          │ ← TrueType Hinting
│ (10 bytes)                              │
└─────────────────────────────────────────┘

01 01 01 01 00     ← flags (点类型)

00 96 96 F6 32     ← x coordinates (相对)
00 2C F6 2C 98     ← y coordinates (相对)

[总大小：约 42 bytes]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

【CFF 格式】(AGaramond-Regular.otf)

rmoveto 0 100      ← 移动到 (0, 100)
rlineto 300 700    ← 直线到 (300, 700)
rlineto 300 -700   ← 直线到 (600, 0)
rlineto -150 200   ← 直线到 (450, 200)
rlineto 300 0      ← 直线（横线）
rlineto -300 0     ← 返回
endchar

编码为：
60 64 6D 72        ← rmoveto 编码
2C 01 72           ← rlineto 编码
2C 01 4E 72        ← ...
...
0B                 ← endchar

[总大小：约 34 bytes]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

对比：
  Glyf：42 bytes
  CFF：  34 bytes
  节省：19% ✓
```

#### 字符 "😀" 在 COLRv0 和 CBDT 中的对比

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

【COLRv0 格式】

COLR 表中的记录：
00 03E8             ← glyphID = 1000 (emoji)
00 00               ← firstLayerIndex = 0
00 05               ← numLayers = 5

Layer records:
00 01 E9 00         ← Layer 0: glyph=1001, color=0
00 02 EA 01         ← Layer 1: glyph=1002, color=1
00 03 EB 02         ← Layer 2: glyph=1003, color=2
00 04 EC 03         ← Layer 3: glyph=1004, color=3
00 05 ED 04         ← Layer 4: glyph=1005, color=4

CPAL 调色板：
FF 00 00 FF         ← Color 0: Red
00 00 FF FF         ← Color 1: Blue
FF FF 00 FF         ← Color 2: Yellow
00 FF 00 FF         ← Color 3: Green
FF 00 FF FF         ← Color 4: Magenta

[COLR 部分：约 50 bytes]
[CPAL 部分：约 200 bytes]
[总计：约 250 bytes]

渲染结果：
  Layer 0: 脸（红色）
  Layer 1: 眼睛（蓝色）
  Layer 2: 嘴巴（黄色）
  + ...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

【CBDT 格式】

CBLC 表中的位置信息：
00 00 00 01         ← version
00 00 00 01         ← numSizes (仅 1 种尺寸：64×64)

BitmapSize @ 64×64:
00 00 01 00         ← indexSubTableArrayOffset
00 40               ← imageSize = 64 ppem

CBDT 表中的数据：
89 50 4E 47 0D 0A 1A 0A  ← PNG 文件签名
1D 3A 58 9C EC 9D ...     ← PNG 压缩数据

[PNG 数据大小：约 3.8 KB]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

对比：
  COLRv0：250 bytes（矢量）+ 无限可扩展
  CBDT：   3,800 bytes（位图）+ 仅 64×64 固定
  
  文件大小：CBDT 大 15 倍！
  但质量：CBDT 可能更好（真实渲染的图像）
  扩展性：COLRv0 更好（任意尺寸）
```

---

## 📊 实际文件大小对比表

```
┌─────────────────────────────────────────────────┐
│        不同格式的实际文件大小对比               │
├─────────────────────────────────────────────────┤
│                                                 │
│ 字体类型         格式      文件大小  WOFF2   │
│ ─────────────────────────────────────────────  │
│                                                 │
│ Roboto          Glyf      165 KB   45 KB   │
│ Inter           Glyf      486 KB   135 KB  │
│ Adobe Garamond  CFF       132 KB   38 KB   │
│ Noto CJK        Glyf(TTC) 110 MB   28 MB   │
│ NotoColorEmoji  COLRv1    12.5 MB  (不压缩)│
│ Twemoji (小)    COLRv1    2.8 MB   720 KB  │
│                                                 │
│ Noto Color      CBDT      15 MB    (不压缩)│
│ (旧 Google)                                   │
│                                                 │
│ Apple Emoji     sbix      20 MB    (不压缩)│
│                                                 │
└─────────────────────────────────────────────────┘
```

---

## 🎯 总结表

| 格式 | 在文件中 | 轮廓数据 | Hinting | 可变 | 文件大小 | 实例 |
|------|---------|---------|---------|------|----------|------|
| **Glyf** | glyf 表 | 坐标+标志 | TrueType VM | ✅ | 100% | Roboto |
| **CFF** | CFF 表 | 命令序列 | PostScript | ❌ | 70-80% | Adobe |
| **CFF2** | CFF2 表 | 命令+增量 | PostScript | ✅ | 85-95% | (稀有) |
| **COLRv0** | COLR+CPAL | 分层 glyf | 无 | ❌ | 90% | Google |
| **COLRv1** | COLR+CPAL | 分层+渐变 | 无 | ✅ | 95% | Noto |
| **CBDT** | CBDT+CBLC | PNG 位图 | 无 | ❌ | 150-200% | Emoji |
| **sbix** | sbix | PNG/JPG | 无 | ❌ | 200%+ | Apple |
| **EBDT** | EBDT+EBLC | LZ77 位图 | 无 | ❌ | 120% | (已弃用) |

