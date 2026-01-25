# Skrifa 支持的字体格式 & Glyf vs CFF 深度对比

> 分析 Fontations/Skrifa 支持的所有字体格式和字形描述方式的区别

## 📋 目录

1. [Skrifa 支持的格式总览](#skrifa-支持的格式总览)
2. [Glyf vs CFF 深度对比](#glyf-vs-cff-深度对比)
3. [其他字形格式](#其他字形格式)
4. [实际应用场景](#实际应用场景)
5. [Chromium 中的使用](#chromium-中的使用)

---

## 📦 Skrifa 支持的格式总览

根据官方文档，Skrifa 支持以下格式：

```
┌─────────────────────────────────────────────────────┐
│         Skrifa 支持的字形格式（Glyph Formats）     │
├─────────────────────────────────────────────────────┤
│                                                     │
│  源类型     解码    可变轴    Hinting             │
│  ─────────────────────────────────────────────────│
│                                                     │
│  glyf    ✔️      ✔️      ✔️                        │
│  CFF     ✔️      ✗       ✔️                        │
│  CFF2    ✔️      ✔️      ✗                        │
│  COLRv0  ✔️      ✗       ✗                        │
│  COLRv1  ✔️      ✔️      ✗                        │
│  EBDT*   ✔️      ✗       ✗                        │
│  CBDT*   ✔️      ✗       ✗                        │
│  sbix*   ✔️      ✗       ✗                        │
│                                                     │
│ * = 仅通过 read-fonts 原始支持（需自己解析数据）   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 简单分类

```
按字形描述方式分类：

1️⃣  矢量字形（Vector Outlines）
   ├─ glyf      : TrueType 轮廓（4 次 Bézier 曲线）
   ├─ CFF       : PostScript 轮廓（3 次 Bézier 曲线）
   └─ CFF2      : CFF 可变版本

2️⃣  彩色字形（Color Outlines）
   ├─ COLRv0    : 分层彩色（固定颜色）
   ├─ COLRv1    : 分层彩色（支持渐变、可变）
   ├─ sbix      : 位图表（Apple）
   ├─ CBDT      : 彩色位图表（Google）
   └─ EBDT      : 嵌入式位图表（较旧）

3️⃣  特殊目的格式
   └─ (SVG)     : 未来目标（规划中）
```

---

## ⚡ Glyf vs CFF 深度对比

### 1. 基础概念

#### Glyf（TrueType 轮廓）

```
名字由来：
  glyf = "glyph formats"
  源自 TrueType 字体标准（Apple & Microsoft 联合开发）

特点：
  ├─ 使用 4 次贝塞尔曲线（Quadratic Bézier curves）
  ├─ 由"点"和"标志"构成轮廓
  ├─ 每个点记录：x, y 坐标 + on/off curve 标志
  ├─ 支持轮廓指令（hinting）
  ├─ 原生支持可变轴（Variable Fonts）
  └─ 文件大小通常较大

例子：字母"A"的轮廓
  A = { 
    Point(0, 0),     on_curve     ← 左下角
    Point(10, 30),   on_curve     ← 顶点
    Point(20, 0),    on_curve     ← 右下角
    Point(8, 15),    off_curve    ← 中间横杆端点（曲线控制）
    Point(12, 15),   off_curve
  }
```

#### CFF（PostScript 轮廓）

```
名字由来：
  CFF = "Compact Font Format"
  来自 Adobe PostScript 技术

特点：
  ├─ 使用 3 次贝塞尔曲线（Cubic Bézier curves）
  ├─ 由"命令序列"构成轮廓
  ├─ 每条命令：rlineto（相对直线）、rcurveto（相对曲线）等
  ├─ 高度字形化（Charstring 指令集）
  ├─ 支持轮廓指令（hinting）
  ├─ CFF 标准版本 NOT 支持可变轴
  └─ 文件大小通常较小（10-30% 压缩）

例子：字母"A"的轮廓
  A = {
    rmoveto(0, 0),      ← 移动到 (0, 0)
    rlineto(10, 30),    ← 直线到 (10, 30)
    rlineto(10, -30),   ← 直线到 (20, 0)
    rlineto(-8, 15),    ← 直线到 (12, 15)
    rcurveto(-4, 0, -2, 0, 0, 0),  ← 曲线...
    ...
  }
```

### 2. 数据结构对比

#### Glyf 表结构

```cpp
// TrueType glyf 表
struct GlyfGlyph {
  int16_t  numberOfContours;      // < 0 = 复合字形，> 0 = 简单字形
  int16_t  xMin, yMin, xMax, yMax;  // Bounding box
  
  // 如果是简单字形：
  uint16_t  endPtsOfContours[];   // 每个轮廓的最后点索引
  uint16_t  instructionLength;    // Hinting 指令长度
  uint8_t   instructions[];       // 字体提示程序
  uint8_t   flags[];              // 点标志（on/off curve）
  int8_t    xCoordinates[];       // X 坐标（相对编码）
  int8_t    yCoordinates[];       // Y 坐标（相对编码）
  
  // 如果是复合字形（组合多个基础字形）：
  struct Component {
    uint16_t  flags;
    uint16_t  glyphIndex;        // 引用的字形 ID
    int16_t   argument1, argument2;  // 位移参数
  } components[];
};

示例：Arial "A"
  numberOfContours = 2          // 两个轮廓（外轮廓和中间洞）
  bounding box = (0, 0, 600, 700)
  points = 20
  instructions = 8 bytes        // Hinting 程序
  flags = [0x01, 0x01, ..., 0x00, 0x01, ...]  // 点类型
  xCoordinates = [0, 150, ..., 200]
  yCoordinates = [0, 700, ..., 350]
```

#### CFF 表结构

```
CFF 表是一个"字体程序"集合：
  ├─ Header (4 bytes)
  ├─ Name INDEX (字体名称表)
  │   └─ "Arial Bold"
  ├─ Top DICT (字体级参数)
  │   ├─ FamilyBlues = [-12, 0, ...]
  │   ├─ BlueValues = [...]
  │   └─ CharStrings 偏移
  ├─ String INDEX (非标准字符串)
  ├─ Global Subrs (全局子程序)
  │   └─ { 常用轮廓段代码 }
  └─ CharStrings (每个字形的程序)
      └─ A = { 
           rmoveto(0, 0)
           rlineto(300, 700)
           rlineto(300, -700)
           ...
           endchar
         }

示例：Arial "A" 的 CharString
  000 010 100 rmoveto        ← 移动到 (0, 100)
  300 700 rlineto            ← 直线
  300 -700 rlineto           ← 直线
  -150 200 rlineto           ← 水平线
  300 0 rlineto              ← 横线
  ...
  endchar
```

### 3. 精度和大小对比

```
┌──────────────────────────────────────────┐
│           数据精度与大小对比              │
├──────────────────────────────────────────┤
│                                          │
│  特性          Glyf         CFF         │
│  ────────────────────────────────────   │
│  曲线阶数       4 次      3 次          │
│  坐标精度       16 bit    16 bit        │
│  子像素精度     1/64      1/65536      │
│  文件大小       较大      较小         │
│  单个字形       ~50B      ~30B         │
│  字体总大小     350KB     250KB        │
│  压缩率         80-100%   70-80%       │
│                                          │
└──────────────────────────────────────────┘
```

### 4. 可变字体支持

#### Glyf（完全支持）

```
TrueType 可变轴机制：
  ├─ fvar 表：定义轴
  │   └─ Weight: 400 → 700
  │   └─ Width: 75% → 100%
  │   └─ Italic: 0° → 12°
  │
  ├─ gvar 表：glyf 增量数据
  │   └─ Light 版本增量
  │   └─ Bold 版本增量
  │   └─ 线性插值：Bold = Light + t × (增量)
  │
  └─ avar 表（可选）：非线性映射
      └─ Weight 0.5 → 0.3 （权重调整）
```

**例子：可变字体 "A"**
```
坐标规范化范围：[0, 1]

Light (weight=0.0):
  └─ A 轮廓 = { (0,0), (150,700), (300,0) }

Bold (weight=1.0):
  └─ A 轮廓 = { (0,0), (160,750), (320,0) }  ← 线条变粗

用户请求 weight=0.7:
  └─ A 轮廓 = Light + 0.7 × (Bold - Light)
              = { (0,0), (157, 735), (314, 0) }
```

#### CFF（标准版本不支持）

```
原因：CFF 使用"命令序列"，难以插值

CFF 字形：
  A = { rmoveto(0,0), rlineto(300,700), ... }

CFF2（CFF 可变版本）：
  ├─ 新增 CFF2 表格式
  ├─ 支持多轴插值
  ├─ 但仍不如 glyf 成熟
  ├─ 支持范围：⚠️ 有限
  └─ 采用率：⚠️ 很低（主要是 Google Fonts）
```

### 5. Hinting（屏幕最优化）

#### Glyf Hinting

```
Glyf 表中的 hinting 指令：
  ├─ 在 instructions[] 字段
  ├─ 虚拟机执行指令
  ├─ 用于小尺寸像素级调整
  │   └─ "确保横线对齐"
  │   └─ "防止字形折叠"
  │   └─ "调整字间距"
  │
  └─ 例子（伪代码）：
      SETLOOP   2
      MIRP[0]   rp0 = rp2 ← 对齐点到参考点
      ROLL            ← 栈操作
      ...

问题：
  ├─ 复杂且难以编写
  ├─ 需要特殊工具（FontLab、VTT）
  └─ 高分辨率屏幕上效果不明显
```

#### CFF Hinting

```
CFF 中的 hinting：
  ├─ "Charstring Hints"（PostScript 风格）
  ├─ 使用堆栈机操作
  ├─ 例子（伪代码）：
      hstem 50 20        ← 水平缩放区间
      vstem 100 30       ← 垂直缩放区间
      rmoveto 100 50
      rlineto 200 0
      hint
      ...
  │
  └─ 效果：
      ├─ 对齐基线
      ├─ 保护重要区间
      └─ 一般不如 TrueType 精细
```

---

## 🎨 其他字形格式

### 1. COLR/CPAL（彩色分层字形）

#### COLRv0（固定彩色）

```
结构：
  ├─ COLR 表：记录每个字形的分层
  │   └─ "A" = [
  │        { base_layer: glyf_id_10, color: red },
  │        { base_layer: glyf_id_11, color: blue },
  │        { base_layer: glyf_id_12, color: yellow },
  │      ]
  │
  └─ CPAL 表：调色板定义
      └─ palette_0 = [red=#FF0000, blue=#0000FF, ...]
      └─ palette_1 = [red=#CC0000, blue=#0000CC, ...]  ← 暗色调板

应用：
  ├─ Emoji（彩色表情符号）
  │   └─ 😀 = 多个字形分层
  ├─ 彩色图标字体
  └─ 徽章和装饰
```

#### COLRv1（高级彩色 + 可变 + 渐变）

```
新增功能：
  ├─ 渐变（Gradient）
  │   ├─ linear gradient
  │   ├─ radial gradient
  │   └─ sweep gradient
  │
  ├─ 位移和缩放变换
  ├─ 可变轴支持
  ├─ 合成操作（blending mode）
  │   ├─ multiply
  │   ├─ screen
  │   └─ overlay
  │
  └─ 例子（彩色 Google Logo）
      G = {
        red part: glyf_id_20, gradient=linear(red→orange)
        blue part: glyf_id_21, gradient=linear(blue→cyan)
        yellow part: glyf_id_22, solid=yellow
        ...
      }
```

### 2. Bitmap 格式（sbix、CBDT、EBDT）

#### sbix（Apple 位图）

```
结构：
  ├─ sbix 表（Scalable Bitmap）
  ├─ 包含：
  │   ├─ PNG 图像
  │   ├─ JPEG 图像
  │   └─ TIFF 图像
  │
  └─ 用途：
      ├─ Apple Emoji（old style）
      ├─ 高分辨率位图备份
      └─ 离线渲染加速

大小：
  ├─ 每个 emoji：~2-5KB
  ├─ 完整字体：50MB+
  └─ 不推荐用于桌面字体（太大）
```

#### CBDT/CBLC（Google 彩色位图）

```
结构：
  ├─ CBDT 表：位图数据
  │   └─ PNG 图像数据
  ├─ CBLC 表：位图定位
  │   └─ 每个字形的位图尺寸和位置
  │
  └─ 用途：
      ├─ Noto Color Emoji（Google）
      ├─ 谷歌彩色表情符号
      └─ 高分辨率渲染

特点：
  ├─ 支持多个尺寸（ppem）
  │   └─ 64px, 128px, 256px 等
  ├─ 彩色和 alpha 透明度
  └─ 文件大小大（但比 sbix 小）
```

#### EBDT（嵌入式位图，较旧）

```
用途：
  ├─ 较旧的位图字体标准
  ├─ 主要用于 Windows 和 DOS 时代
  ├─ 现已基本弃用
  └─ Skrifa 仅做原始支持

vs CBDT：
  ├─ EBDT：无损压缩（LZ77）
  ├─ CBDT：PNG 压缩（高效）
  └─ CBDT 更现代更高效
```

---

## 📊 实际应用场景

### 场景 1：英文正文字体（如 Arial）

```
选择：Glyf + TrueType
原因：
  ├─ 字形数量多（~1500 个）
  ├─ 需要优质屏幕显示
  ├─ 广泛支持（Windows、macOS、Web）
  ├─ Hinting 优化成熟
  └─ 文件大小可接受（300-400KB）

性能：
  ├─ 解析速度：⚡ 快
  ├─ 渲染速度：⚡ 快
  ├─ 屏幕质量：⭐⭐⭐⭐⭐
```

### 场景 2：中文/日文字体（如 Noto Sans CJK）

```
选择：Glyf + TrueType
原因：
  ├─ 字形数量超多（>20000 个）
  ├─ 需要支持可变轴（Weight）
  ├─ 屏幕显示质量重要
  ├─ Hinting 不太重要（大字体）
  └─ 使用 TrueType Collections（TTC）

性能：
  ├─ 字体文件：30-60MB
  ├─ 可变轴：Weight 400-700
  ├─ 缓存关键（不能全加载）
```

### 场景 3：Adobe 专业字体库

```
选择：CFF + PostScript
原因：
  ├─ 专业排版工具原生支持
  ├─ 文件大小较小
  ├─ 高精度要求
  ├─ Desktop 环境（不需网页兼容性）
  └─ 向后兼容旧系统

例子：
  ├─ Adobe Garamond
  ├─ Minion Pro
  ├─ Myriad Pro
  └─ 大多数 Adobe 字体库

性能：
  ├─ 文件大小：⭐ 较小
  ├─ 精度：⭐⭐⭐⭐⭐
  ├─ 支持度：⚠️ 仅 Desktop
```

### 场景 4：Emoji 和彩色字体

```
选择 1：COLR + glyf（推荐）
原因：
  ├─ 支持可变轴
  ├─ 现代标准
  ├─ Skrifa 完全支持
  └─ 性能最好
  
例子：
  ├─ Noto Color Emoji（新版本）
  ├─ Twemoji（Twitter）
  └─ Segoe Color Emoji（Windows 11）

选择 2：CBDT + PNG（Google）
原因：
  ├─ 彩色质量最高
  ├─ 高分辨率支持
  ├─ 但文件大（50MB+）
  └─ 移动应用不适用

例子：
  └─ Noto Color Emoji（Google Fonts）

性能对比：
  ├─ COLR：⚡ 快，可缩放，支持可变
  ├─ CBDT：🐌 慢，固定尺寸，无可变
```

### 场景 5：可变字体（Variable Fonts）

```
支持情况：
  ├─ Glyf + gvar：✅ 完全支持
  ├─ CFF2：⚠️ 有限支持
  ├─ COLR v1：✅ 完全支持
  └─ 其他：❌ 不支持

常见例子：
  ├─ Inter（字体设计师必备）
  ├─ Roboto Flex
  ├─ Fraunces
  ├─ Literata
  └─ IBM Plex（部分）

使用场景：
  ├─ 响应式网页设计
  ├─ 空间优化（一个文件，多种风格）
  ├─ 动画（重量在动画中变化）
  └─ 品牌灵活性（单一字体多场景）
```

---

## 🔧 Chromium 中的使用

### 1. 系统字体扫描

```cpp
// skia/ext/font_utils.cc
return SkFontMgr_New_Android(nullptr, SkFontScanner_Make_Fontations());

// Fontations 扫描支持：
// ├─ Glyf（TrueType）        → ✅ 完全支持
// ├─ CFF（PostScript）        → ✅ 完全支持
// ├─ CFF2                     → ✅ 支持
// ├─ COLR/COLRv1             → ✅ 支持
// ├─ sbix/CBDT/EBDT          → ⚠️ 仅原始支持
// └─ 目的：统一扫描所有格式
```

### 2. 字体加载流程

```
用户请求字体
  ↓
Skrifa 库（高阶 API）
  ├─ 读 'name' 表：格式识别
  ├─ 读 'glyf' 或 'CFF' 表：确定轮廓类型
  ├─ 读 'COLR' 表：检测彩色
  └─ 读 'fvar' 表：检测可变轴
  ↓
read-fonts 库（低阶解析）
  ├─ 原始二进制解析
  └─ 不做预处理
  ↓
返回 FontRef（句柄）
  ↓
HarfBuzz shaper（字形化）
  ├─ glyf：使用 SkPath 获取轮廓
  └─ CFF：使用 SkPath 获取轮廓
  ↓
Skia rasterizer（栅格化）
  ├─ TrueType hinting（如适用）
  ├─ 子像素渲染
  └─ GPU 缓存
```

### 3. 格式选择和性能

```
Chromium 中的实际使用：

Web 字体（@font-face）：
  ├─ WOFF2（基于 Brotli 压缩的 Glyf/CFF）
  ├─ Skrifa 解压后读取
  └─ 性能：⚡ 最好（已压缩）

系统字体：
  ├─ Android：Glyf（Roboto）+ CFF（某些字体）
  ├─ macOS：Glyf + COLR（Emoji）
  ├─ Linux：Glyf（多数）+ CFF（某些）
  └─ Windows：CFF（部分）+ Glyf（多数）

彩色表情符号（Emoji）：
  ├─ Android 10+：COLR v0 / v1（推荐）
  ├─ iOS：SBIX（苹果）
  ├─ Google Fonts：CBDT（备选）
  └─ Fallback：B&W 黑白方案
```

---

## 🎯 快速选择指南

### "我应该用什么格式？"

| 需求 | 推荐格式 | 原因 |
|------|--------|------|
| 标准网页正文 | WOFF2(Glyf) | 小、快、支持好 |
| 中日韩文字 | Glyf + TTC | 大字符集、质量好 |
| 可变字体 | Glyf + gvar | 完全支持、无限灵活 |
| 彩色 Emoji | COLR v1 | 可变、高效、现代 |
| Adobe 专业字体 | CFF | 原生支持、精度高 |
| 旧系统支持 | CFF | 向后兼容 |
| 超小文件 | CFF | 压缩效率好 |
| 桌面排版 | CFF + PostScript | 行业标准 |

### "Glyf 还是 CFF？"

```
选 Glyf 如果你需要：
  ✅ 可变轴支持
  ✅ 屏幕优化（hinting）
  ✅ 大字形集合
  ✅ Web 兼容性

选 CFF 如果你需要：
  ✅ 文件小（10-30% 压缩）
  ✅ 专业排版工具
  ✅ PostScript 兼容
  ✅ 高精度需求
```

---

## 📚 总结表

```
┌─────────────────────────────────────────────────────────────────┐
│                    Skrifa 字体格式总览                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ 格式     源头           用途              支持度    可变轴     │
│ ─────────────────────────────────────────────────────────────  │
│                                                                 │
│ Glyf     TrueType      通用字体           ⭐⭐⭐⭐⭐  ✅        │
│ CFF      PostScript    专业字体           ⭐⭐⭐⭐   ❌        │
│ CFF2     Adobe         可变 PostScript   ⭐⭐⭐⭐   ⚠️ 有限  │
│ COLRv0   OpenType      彩色分层          ⭐⭐⭐⭐   ❌        │
│ COLRv1   OpenType      高级彩色          ⭐⭐⭐⭐⭐  ✅        │
│ sbix     Apple         Apple Emoji      ⭐⭐⭐     ❌        │
│ CBDT     Google        Google Emoji     ⭐⭐⭐     ❌        │
│ EBDT     Old STD       嵌入位图          ⭐⭐       ❌        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔗 相关资源

### 文档和规范
- [OpenType 规范](https://docs.microsoft.com/en-us/typography/opentype/spec/)
- [Skrifa 官方文档](https://docs.rs/skrifa/)
- [read-fonts 源代码](https://github.com/google/fontations/tree/main/read-fonts)
- [Google Fonts 指南](https://fonts.google.com/metadata)

### 工具
- [FontTools](https://github.com/fonttools/fonttools) - Python 字体编辑
- [FontLab](https://www.fontlab.com/) - 专业字体编辑器
- [VTT（Visual TrueType）](https://www.monotype.com/products/vtt) - TrueType hinting
- [UFO 格式](http://unifiedfontobject.org/) - 字体源格式

### 示例字体
- **Glyf**：Roboto, Noto Sans, Inter, Roboto Flex
- **CFF**：Adobe Garamond, Minion Pro, Source Sans Pro
- **可变**：Inter, Fraunces, Literata, IBM Plex
- **彩色**：Noto Color Emoji, Twemoji, Segoe Color Emoji

