# RenderingNG 架构深度指南：Layout 之后的每一步

> 详细讲解 Layout 完成后到屏幕显示的完整流程，包含详细的输入/输出、代码示例和真实场景

**参考**: https://developer.chrome.com/docs/chromium/renderingng-architecture

---

## 概述：RenderingNG 的完整渲染流程

```
┌─────────────────────────────────────────────────────────────────┐
│                        RenderingNG Pipeline                      │
└─────────────────────────────────────────────────────────────────┘

DOM 树 + 样式 + 约束
      ↓
┌─────────────────────────────────────────────────────────────────┐
│ Layout: 计算所有元素的大小和位置 (LayoutNG)                     │
│ 输入: DOM 树、ComputedStyle                                      │
│ 输出: LayoutObject 树                                            │
│ 模块: third_party/blink/renderer/core/layout/                  │
└─────────────────────────────────────────────────────────────────┘
      ↓
┌─────────────────────────────────────────────────────────────────┐
│ Paint: 生成绘制指令列表 (DisplayItems)                         │
│ 输入: LayoutObject 树、ComputedStyle                            │
│ 输出: DisplayItems (绘制指令)、PaintArtifact                   │
│ 模块: third_party/blink/renderer/core/paint/                  │
└─────────────────────────────────────────────────────────────────┘
      ↓
┌─────────────────────────────────────────────────────────────────┐
│ Composite: 决定分层和合成方案                                   │
│ 输入: DisplayItems、PaintArtifact                               │
│ 输出: cc::Layers (合成层)                                       │
│ 模块: third_party/blink/renderer/core/compositor/              │
└─────────────────────────────────────────────────────────────────┘
      ↓
┌─────────────────────────────────────────────────────────────────┐
│ Rasterization: 光栅化 → GPU 纹理                               │
│ 输入: cc::Layers、PaintOps                                      │
│ 输出: GPU 纹理、绘制指令                                         │
│ 模块: cc/ (Chromium Compositor)                                 │
└─────────────────────────────────────────────────────────────────┘
      ↓
┌─────────────────────────────────────────────────────────────────┐
│ Aggregation & Display: 最终合成显示                            │
│ 输入: GPU 纹理队列                                              │
│ 输出: 屏幕画面                                                  │
│ 模块: ui/gfx/, gpu/                                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 第 1 阶段：Layout 完成

### 1.1 Layout 的输出

**文件**: `third_party/blink/renderer/core/layout/layout_object.h`

```cpp
// Layout 完成后的数据结构
class LayoutObject : public GarbageCollected<LayoutObject> {
 public:
  // 位置信息（相对于父元素或包含块）
  LayoutUnit X() const { return x_; }
  LayoutUnit Y() const { return y_; }
  
  // 尺寸信息
  LayoutUnit Width() const { return width_; }
  LayoutUnit Height() const { return height_; }
  
  // 盒子模型
  LayoutBoxExtent GetBorder() const;
  LayoutBoxExtent GetPadding() const;
  LayoutBoxExtent GetMargin() const;
  
  // 屏幕坐标（绝对位置）
  IntRect AbsoluteBoundingBoxRect() const;
  
  // 计算后的样式
  const ComputedStyle& StyleRef() const;
};
```

### 1.2 真实例子：简单 div 的 Layout 结果

```html
<!DOCTYPE html>
<style>
  body { margin: 0; }
  .box {
    width: 200px;
    height: 100px;
    margin: 10px;
    padding: 5px;
    background: blue;
  }
</style>
<div class="box">Hello</div>
```

**Layout 完成后的状态**:

```
LayoutBlockFlow (representing body)
  x=0, y=0
  width=800, height=auto
  ├─ LayoutBlockFlow (representing .box)
  │   x=10, y=10               // margin-top + margin-left
  │   width=200, height=100    // 明确指定的大小
  │   padding: 5px all sides
  │   border: 0
  │   margin: 10px all sides
  │   
  │   └─ LayoutText ("Hello")
  │       x=5, y=5 (相对于 .box, 加上 padding)
  │       width=auto, height=20
```

### 1.3 关键信息已准备好

此时，以下信息已完全计算：

```
✓ 每个元素的最终尺寸（宽、高）
✓ 每个元素的最终位置（x、y）
✓ 文本的行号、文字位置
✓ 图片的显示尺寸
✓ 表格的单元格边界
✓ Flexbox/Grid 的子项位置
✓ 所有计算后的样式（colors、fonts、backgrounds 等）
✓ Transform、opacity 等转换属性
```

---

## 第 2 阶段：Paint（绘制）- 生成 DisplayItems

### 2.1 Paint 的目的

**问题**: Layout 只告诉我们"这个元素在哪里、多大"，但**没有说怎么画它**。

Paint 阶段的工作：
- 遍历 LayoutObject 树
- 为每个元素生成"绘制指令"
- 这些指令称为 **DisplayItems**
- 收集所有 DisplayItems 形成 **PaintArtifact**

### 2.2 DisplayItem 是什么？

**文件**: `third_party/blink/renderer/core/paint/display_item.h`

```cpp
class DisplayItem {
 public:
  // DisplayItem 的类型（绘制指令的类型）
  enum class Type : uint8_t {
    // 背景相关
    kBoxDecorationBackground,    // 背景色、背景图
    kBackgroundImage,             // <img> 背景
    kBorder,                       // 边框
    
    // 内容相关
    kText,                         // 文本
    kImage,                        // 图片
    kSVG,                          // SVG
    
    // 视觉效果
    kClipped,                      // 剪切
    kScrollbar,                    // 滚动条
    
    // 其他
    kLayerChunk,                   // 合成层
    kEndLayerChunk,
    // ... 40+ 种类型
  };
  
  // 每个 DisplayItem 包含的信息
  Type GetType() const { return type_; }
  const FloatRect& VisualRect() const { return visual_rect_; }
  ClientPaintingData GetPaintingData() const;
};
```

### 2.3 Paint 的完整流程

**文件**: `third_party/blink/renderer/core/paint/paint_controller.h`

```cpp
class PaintController {
 public:
  // Paint 的主入口
  void Paint(GraphicsContext& graphics_context);
  
  // 内部过程
  void BeginFrame();
  void PaintArtifact();  // ← 关键：生成所有 DisplayItems
  void CommitNewDisplayItems();
  void EndFrame();
};
```

### 2.4 真实例子：绘制上面的 div

```html
<div class="box">Hello</div>
```

**Paint 生成的 DisplayItems 序列**:

```
Frame 开始
├─ DisplayItem::kBoxDecorationBackground
│   ├─ 类型: 背景装饰
│   ├─ 视觉范围: (10, 10, 210, 110)  [x, y, x+width, y+height]
│   ├─ 绘制操作:
│   │   1. DrawRect(10, 10, 200, 100)  // 矩形边界
│   │   2. FillColor(blue)             // 填充蓝色
│   │   3. StrokeColor(black)          // 边界色
│   └─ 计算来自: ComputedStyle::background-color, border, ...
│
├─ DisplayItem::kBorder
│   ├─ 类型: 边框
│   ├─ 视觉范围: (10, 10, 210, 110)
│   └─ 绘制操作:
│       DrawBorder(10, 10, 200, 100, border_style, border_width)
│
├─ DisplayItem::kText
│   ├─ 类型: 文本
│   ├─ 视觉范围: (15, 15, 60, 35)  // 相对于 padding
│   ├─ 内容: "Hello"
│   ├─ 字体: ComputedStyle::font-family
│   ├─ 颜色: ComputedStyle::color
│   ├─ 位置: (15, 15)
│   └─ 绘制操作:
│       1. SelectFont(Arial, 16px)
│       2. DrawText("Hello", 15, 15)
│
└─ Frame 结束
```

### 2.5 Paint 的递归过程

**文件**: `third_party/blink/renderer/core/paint/object_painter.cc`

```cpp
void ObjectPainter::Paint(const PaintInfo& paint_info) {
  // 1. 检查是否需要 paint
  if (!ShouldPaint(paint_info))
    return;
  
  // 2. Paint 背景 - 生成 DisplayItem::kBoxDecorationBackground
  PaintBackgroundColor(paint_info);       // 背景色
  PaintBackgroundImages(paint_info);      // 背景图片
  
  // 3. Paint 边框 - 生成 DisplayItem::kBorder
  PaintBorder(paint_info);
  
  // 4. Paint 内容 - 生成 DisplayItem::kText/kImage/etc
  PaintChildren(paint_info);              // ← 递归调用
  
  // 5. Paint 轮廓
  PaintOutline(paint_info);               // 生成 DisplayItem::kOutline
}

void ObjectPainter::PaintChildren(const PaintInfo& paint_info) {
  for (LayoutObject* child = FirstChild(); 
       child; 
       child = child->NextSibling()) {
    ObjectPainter(child).Paint(paint_info);  // ← 递归
  }
}
```

### 2.6 Paint 的输入和输出

**输入**:
```
1. LayoutObject 树（包含所有元素的尺寸、位置）
2. ComputedStyle（每个元素的计算后的样式）
3. PaintInfo（当前绘制上下文、clipping 区域）
```

**输出**:
```
1. DisplayItems 列表（一个线性的绘制指令序列）
2. PaintArtifact（包含所有 DisplayItems + 元数据）

示例:
[
  DisplayItem::kBoxDecorationBackground  (visual_rect: 10,10,210,110)
  DisplayItem::kBorder                   (visual_rect: 10,10,210,110)
  DisplayItem::kText                     (visual_rect: 15,15,60,35)
]
```

### 2.7 Paint 性能优化：Invalidation（失效）

```cpp
// 只重新 paint 改变的部分
void PaintController::InvalidatePaintOfRect(const IntRect& rect) {
  // 1. 记录哪些 DisplayItems 受影响
  for (auto& item : display_items_) {
    if (item->VisualRect().Intersects(rect)) {
      item->Invalidate();  // ← 标记为需要重新绘制
    }
  }
  
  // 2. 只重新绘制这些 items
  RegenInvalidPaintItems(rect);
}
```

**真实场景**:
```javascript
// 改变颜色
element.style.color = 'red';  // 只 repaint 文本，不 repaint 背景

// 改变位置
element.style.transform = 'translateX(10px)';  
// 不需要 repaint！（合成层优化）

// 改变尺寸
element.style.width = '300px';  
// 需要 relayout + repaint（重新计算所有东西）
```

---

## 第 3 阶段：Composite（合成）- 创建合成层

### 3.1 为什么需要 Composite？

**问题**: DisplayItems 是一个线性列表，但我们需要更复杂的操作：
- Transform（2D/3D 变换）
- Opacity（透明度混合）
- Filter（滤镜效果）
- Clip（剪切）
- Scrolling（滚动）

**解决方案**: 将 DisplayItems 分组成 **cc::Layers**（合成层），每个层可以：
- 独立变换
- 独立光栅化
- 在 GPU 上合成

### 3.2 如何决定层的分界？

**文件**: `third_party/blink/renderer/core/compositor/compositing_reasons.h`

```cpp
// 触发创建合成层的原因
enum class CompositingReason {
  // 3D 效果
  kTransform3D,                        // transform: rotateX/Y/Z 等
  kPerspective,                        // 透视投影
  
  // 透明度和混合
  kOpacity,                            // opacity < 1.0
  kMixBlendMode,                       // mix-blend-mode != normal
  
  // 其他
  kWillChange,                         // will-change 属性
  kBackfaceVisibility,                 // backface-visibility
  kAnimation,                          // 激活的 animation
  kScroll,                             // 滚动
  kVideoOverlay,                       // video 标签
  kCanvasOverlay,                      // canvas 标签
  // ... 30+ 种原因
};
```

### 3.3 真实例子：3 层合成

```html
<style>
  .background {
    width: 200px;
    height: 100px;
    background: blue;
  }
  .transformed {
    transform: rotate(45deg);  /* ← 创建合成层 */
  }
  .text {
    opacity: 0.8;  /* ← 创建合成层 */
  }
</style>

<div class="background">
  <div class="transformed">Transformed</div>
  <div class="text">Text</div>
</div>
```

**Composite 的结果**:

```
cc::Layer 树:
├─ cc::Layer 0: Background Layer
│   ├─ DisplayItems 来自 .background
│   ├─ 无特殊属性
│   └─ 范围: (0, 0, 200, 100)
│
├─ cc::Layer 1: Transform Layer
│   ├─ DisplayItems 来自 .transformed div
│   ├─ transform: rotate(45deg)
│   ├─ 范围: 计算后的边界
│   └─ 光栅化为单独的纹理
│
└─ cc::Layer 2: Opacity Layer
    ├─ DisplayItems 来自 .text div
    ├─ opacity: 0.8
    └─ 光栅化为单独的纹理
```

**Compositor 的工作**:

```
1. 光栅化所有层（cc::Layer → GPU 纹理）
2. 在 GPU 上合成：
   layer2_texture with opacity=0.8
   + layer1_texture with transform=rotate(45deg)
   + layer0_texture
3. 输出最终屏幕画面
```

### 3.4 Composite 的输入和输出

**输入**:
```
1. DisplayItems 列表
2. CompositingReasons（为每个元素）
3. 堆栈上下文信息（z-index、blend mode 等）
```

**输出**:
```
1. cc::Layer 树（分组后的 DisplayItems）
2. 每个 Layer 的属性（transform、opacity、clip 等）
3. Layer 的绘制指令（PaintOps）

示例:
cc::Layers [
  {
    id: 1,
    display_items: [BackgroundItem],
    bounds: (0, 0, 200, 100),
    transform: identity,
    opacity: 1.0
  },
  {
    id: 2,
    display_items: [TextItem],
    bounds: (0, 0, 100, 50),
    transform: rotate(45deg),
    opacity: 0.8
  }
]
```

---

## 第 4 阶段：Rasterization（光栅化）

### 4.1 从矢量到像素

**问题**: DisplayItems 是矢量指令（"画一个蓝色矩形"），但 GPU 需要像素数据。

**解决方案**: 光栅化 = 将矢量指令转换为 GPU 纹理（像素阵列）

### 4.2 光栅化的过程

**文件**: `cc/paint/raster_source.cc`

```cpp
// 光栅化一个层
void RasterSource::Raster(SkCanvas* canvas, 
                          const gfx::Rect& rect,  // 要光栅化的区域
                          float scale) {           // 缩放因子
  
  // 1. 创建虚拟 canvas
  SkBitmap bitmap;
  bitmap.allocPixels(SkImageInfo::MakeN32Premul(
      rect.width() * scale, 
      rect.height() * scale));
  SkCanvas raster_canvas(bitmap);
  
  // 2. 回放 DisplayItems 中的 PaintOps
  for (const auto& paint_op : paint_ops_) {
    // 例如：
    // DrawRect → canvas->drawRect()
    // DrawText → canvas->drawText()
    // DrawImage → canvas->drawImage()
    paint_op->Raster(&raster_canvas);  // ← 关键
  }
  
  // 3. 上传到 GPU
  UploadToGPU(bitmap);  // GPU 纹理
}
```

### 4.3 真实例子：光栅化蓝色 div

```cpp
// 光栅化 .box 元素
vector<PaintOp> paint_ops = [
  DrawRect(Rect(0, 0, 200, 100)),     // 位置和大小
  SetColor(Color::kBlue),              // 蓝色
  FillRect(Rect(0, 0, 200, 100)),     // 填充
  DrawText("Hello", Point(15, 15)),   // 绘制文本
];

// Rasterize at scale 2x (retina display)
Raster(canvas, Rect(0, 0, 200, 100), 2.0);

// 结果：一个 400x200 像素的 GPU 纹理
// [蓝色像素数据] + [文本像素数据]
```

### 4.4 Rasterization 的输入和输出

**输入**:
```
1. cc::Layer（包含 PaintOps）
2. 要光栅化的矩形区域 (Rect)
3. 缩放因子 (DPI, 设备像素比)
```

**输出**:
```
1. GPU 纹理（SkImage / sk::sp<SkImage>）
   - RGBA 像素数据
   - 尺寸: width × height × scale
   
示例:
GPU Texture {
  size: (400, 200)        // 2x scale
  format: RGBA_8888
  data: [0xff0000ff, 0xff0000ff, ...]  // 像素阵列
}
```

### 4.5 光栅化策略

**文件**: `cc/tile_manager.h`

```cpp
class TileManager {
 public:
  // 光栅化策略
  enum class RasterMode {
    // 1. 同步光栅化（所有东西立即光栅化）
    kBlocking,
    
    // 2. 异步光栅化（后台线程光栅化）
    kAsync,
    
    // 3. 优先级光栅化（重要的先光栅化）
    kPriority,
  };
};
```

**真实场景**:

```
场景 1: 用户滚动网页
  ├─ 可见区域需要立即光栅化（同步）
  └─ 即将可见的区域后台光栅化（异步）

场景 2: 页面加载
  ├─ viewport 区域先光栅化（高优先级）
  └─ 下方区域后光栅化（低优先级）

场景 3: 动画执行
  ├─ 每帧 60fps 都需要光栅化
  └─ 使用 GPU 加速（transform、opacity）
```

---

## 第 5 阶段：Aggregation & Display（最终合成显示）

### 5.1 什么是 Aggregation？

Aggregation = 将所有 GPU 纹理**合成（composite）**到屏幕上

**文件**: `cc/layer_tree_host.cc`

```cpp
class LayerTreeHost {
 public:
  // 合成所有层
  void CompositeFrame() {
    // 1. 收集所有光栅化的纹理
    vector<gpu::Mailbox> textures = GatherTextures();
    
    // 2. 在 GPU 上组合它们
    gpu::GpuCommandBuffer commands = BuildCompositingCommands(textures);
    
    // 3. 发送到显示服务
    SendToDisplay(commands);
  }
};
```

### 5.2 真实例子：3 层合成到屏幕

```
GPU 纹理队列:
├─ Texture 1 (Background): 400x200 蓝色像素
├─ Texture 2 (Transformed): 250x250 旋转45度的像素
└─ Texture 3 (Text): 200x100 半透明文本像素

Compositing Commands:
1. BindTexture(texture1)
2. DrawRect(0, 0, 400, 200)           // 绘制背景
3. 
4. BindTexture(texture2)
5. ApplyTransform(rotate(45deg))
6. DrawRect(calc_bounds)              // 绘制旋转内容
7.
8. BindTexture(texture3)
9. SetOpacity(0.8)
10. DrawRect(0, 0, 200, 100)          // 绘制半透明文本
11.
12. Flush()                            // 提交到显示缓冲
```

### 5.3 Aggregation 的输入和输出

**输入**:
```
1. cc::Layers 列表（已光栅化）
2. Layer 的变换和透明度信息
3. 视口尺寸
```

**输出**:
```
1. Display Buffer（屏幕缓冲）
   - 最终的像素数据
   - 准备发送到显示驱动

2. DisplayItems（用于下一帧）
   - 可能需要更新的元素列表
```

---

## 完整的数据流示例

### 场景：简单网页的完整渲染

```html
<!DOCTYPE html>
<style>
  body { margin: 0; background: white; }
  .container {
    width: 400px;
    margin: 20px auto;
  }
  .header {
    height: 50px;
    background: #2196F3;
    color: white;
  }
  .content {
    padding: 20px;
    background: #f5f5f5;
  }
  .button {
    background: green;
    padding: 10px;
    transform: scale(1.1);  /* ← 创建合成层 */
  }
</style>

<body>
  <div class="container">
    <div class="header">My Website</div>
    <div class="content">
      <p>Welcome!</p>
      <button class="button">Click me</button>
    </div>
  </div>
</body>
```

### 数据流追踪

```
1️⃣ LAYOUT 阶段
━━━━━━━━━━━━━━━━━
输入:
  DOM: <div class="container">...</div>
  ComputedStyle: width=400px, margin=20px auto, ...

Processing:
  ✓ body: 计算 viewport 宽度 = 800px
  ✓ .container: 计算 x=200, y=20, width=400
  ✓ .header: 计算 x=200, y=20, width=400, height=50
  ✓ .content: 计算 x=200, y=70, width=400, height=auto
  ✓ button: 计算 x=220, y=100, width=50, height=20

输出:
  LayoutObject 树:
  ├─ LayoutView (800x600 viewport)
  │  └─ LayoutBlockFlow (body)
  │     └─ LayoutBlockFlow (.container @ 200,20,400,x)
  │        ├─ LayoutBlockFlow (.header @ 200,20,400,50)
  │        │  └─ LayoutText ("My Website")
  │        └─ LayoutBlockFlow (.content @ 200,70,400,x)
  │           ├─ LayoutText ("Welcome!")
  │           └─ LayoutBlockFlow (.button @ 220,100,50,20)
  │              └─ LayoutText ("Click me")


2️⃣ PAINT 阶段
━━━━━━━━━━━━━━━━━
输入:
  LayoutObject 树 + ComputedStyle

Processing:
  ✓ Paint .container 背景 → DisplayItem::kBoxDecorationBackground
      visual_rect: (200, 20, 600, 200)
      color: white
      
  ✓ Paint .header 背景 → DisplayItem::kBoxDecorationBackground
      visual_rect: (200, 20, 600, 70)
      color: #2196F3 (蓝色)
      
  ✓ Paint header 文本 → DisplayItem::kText
      visual_rect: (200, 20, 400, 50)
      text: "My Website"
      color: white
      
  ✓ Paint .content 背景 → DisplayItem::kBoxDecorationBackground
      visual_rect: (200, 70, 600, 200)
      color: #f5f5f5 (灰色)
      
  ✓ Paint 内容文本 → DisplayItem::kText
      visual_rect: (220, 90, 320, 110)
      text: "Welcome!"
      
  ✓ Paint button 背景 → DisplayItem::kBoxDecorationBackground
      visual_rect: (220, 100, 270, 120)
      color: green
      
  ✓ Paint button 文本 → DisplayItem::kText
      visual_rect: (220, 100, 270, 120)
      text: "Click me"

输出:
  DisplayItems (线性序列):
  [
    kBoxDecorationBackground { rect: (200,20,400,180), color: white },
    kBoxDecorationBackground { rect: (200,20,400,50), color: #2196F3 },
    kText { rect: (200,20,400,50), "My Website" },
    kBoxDecorationBackground { rect: (200,70,400,130), color: #f5f5f5 },
    kText { rect: (220,90,100,20), "Welcome!" },
    kBoxDecorationBackground { rect: (220,100,50,20), color: green },
    kText { rect: (220,100,50,20), "Click me" }
  ]


3️⃣ COMPOSITE 阶段
━━━━━━━━━━━━━━━━━
输入:
  DisplayItems + CompositingReasons

Processing:
  分析每个元素的合成理由：
  ✓ body - 无特殊属性 → 保留在 Layer 0
  ✓ .container - 无特殊属性 → 保留在 Layer 0
  ✓ .header - 无特殊属性 → 保留在 Layer 0
  ✓ .content - 无特殊属性 → 保留在 Layer 0
  ✓ .button - transform: scale(1.1) ✓✓✓ → 创建 Layer 1

输出:
  cc::Layers:
  Layer 0 (Root):
    display_items: [bg, header_bg, header_text, content_bg, text]
    bounds: (200, 20, 400, 180)
    transform: identity
    opacity: 1.0
    
  Layer 1 (Button):
    display_items: [button_bg, button_text]
    bounds: (220, 100, 50, 20)
    transform: scale(1.1)
    opacity: 1.0


4️⃣ RASTERIZATION 阶段
━━━━━━━━━━━━━━━━━
输入:
  cc::Layers + 设备像素比 (DPR = 2.0 for Retina)

Processing:
  
  ✓ Rasterize Layer 0:
    - 创建 SkCanvas (800x360 @ 2x = 1600x720 像素)
    - 回放 DisplayItems:
      1. DrawRect(400,40,800,360) → FillColor(white)
      2. DrawRect(400,40,800,100) → FillColor(#2196F3)
      3. DrawText("My Website", 400,40) → white font
      ... 等等
    - 上传到 GPU → GPU Texture 1
    
  ✓ Rasterize Layer 1 (button):
    - 创建 SkCanvas (100x40 @ 2x = 200x80 像素)
    - 回放 DisplayItems:
      1. DrawRect(0,0,100,40) → FillColor(green)
      2. DrawText("Click me", 20,10) → black font
    - 上传到 GPU → GPU Texture 2

输出:
  GPU 纹理:
  ├─ Texture 1: 1600x720 像素, RGBA 格式
  │   内容: 白色背景 + 蓝色 header + 灰色 content
  │
  └─ Texture 2: 200x80 像素, RGBA 格式
      内容: 绿色 button + 黑色文字


5️⃣ AGGREGATION 阶段
━━━━━━━━━━━━━━━━━
输入:
  cc::Layers (已光栅化) + 显示参数

Processing:
  GPU 合成命令:
  1. BindFramebuffer(screen)
  2. SetViewport(0, 0, 800, 600)
  3. Clear(white)
  
  // 合成 Layer 0
  4. BindTexture(texture1)
  5. DrawQuad(layer0_bounds)  // 400x360 @ (200,20)
  
  // 合成 Layer 1 (带 transform)
  6. BindTexture(texture2)
  7. SetTransform(scale(1.1))
  8. DrawQuad(layer1_bounds)  // 50x20 @ (220,100)
  
  9. Flush()

输出:
  屏幕缓冲 (Display Buffer):
  最终像素数据 (1600x1200 @ 2x DPR)
  
  ✓ 全部准备好显示
  ✓ VSync 信号到达 → 显示到屏幕
```

---

## 关键模块的职责

| 模块 | 职责 | 关键类 | 输入 | 输出 |
|------|------|--------|------|------|
| **Layout** | 计算元素大小和位置 | `LayoutObject`, `LayoutNG` | DOM + Style | 位置、尺寸 |
| **Paint** | 生成绘制指令列表 | `ObjectPainter`, `PaintController`, `DisplayItem` | LayoutObject | DisplayItems |
| **Composite** | 决定分层方案 | `LayerTreeBuilder`, `CompositingReasons` | DisplayItems | cc::Layers |
| **Rasterize** | 矢量→像素 | `RasterSource`, `SkCanvas` | cc::Layers | GPU 纹理 |
| **Display** | 合成到屏幕 | `LayerTreeHost`, `Display` | GPU 纹理 | 屏幕画面 |

---

## 性能优化的关键点

### 1. Avoid Forced Reflow

```javascript
// ❌ 每次都触发 layout
for (let i = 0; i < 100; i++) {
  element.style.width = (i * 10) + 'px';  // 100 次 layout!
}

// ✅ 只 layout 一次
element.style.width = '1000px';
```

### 2. Use Transform Instead of Position

```javascript
// ❌ 触发 repaint + layout
element.style.left = '100px';

// ✅ 不触发 layout（只合成）
element.style.transform = 'translateX(100px)';
```

### 3. Batch DOM Updates

```javascript
// ❌ 3 次 repaint
element1.style.color = 'red';
element2.style.color = 'blue';
element3.style.color = 'green';

// ✅ 只需 1 次 repaint（requestAnimationFrame 批量）
requestAnimationFrame(() => {
  element1.style.color = 'red';
  element2.style.color = 'blue';
  element3.style.color = 'green';
});
```

### 4. Use will-change for Animations

```css
/* 告诉浏览器提前创建合成层 */
.animated {
  will-change: transform, opacity;
  animation: slide 2s infinite;
}
```

---

## DevTools 中看到的内容对应

### Chrome DevTools Performance Timeline

```
Timeline 中的事件              对应的渲染阶段

Parse                         HTML 解析 (前面讲过)
↓
Evaluate Script              JavaScript 执行
↓
Recalculate Style           CSS 样式计算 + Layout
↓
Layout                      布局计算 (Layout 阶段)
↓
Update Layer Tree           合成决策 (Composite 阶段)
↓
Paint                       生成 DisplayItems (Paint 阶段)
↓
Composite                   光栅化 + 合成 (Rasterize + Aggregation)
↓
Rendering                   一帧完成，等待 VSync
```

---

## 常见性能问题诊断

| 现象 | 原因 | 解决方案 |
|------|------|---------|
| 频繁出现 Layout | DOM 修改、样式改变 | 缓存读取、批量更新 |
| 频繁出现 Paint | 内容改变、颜色改变 | 使用 transform/opacity |
| Composite 时间长 | 合成层太多 | 减少 will-change 使用 |
| 帧率低（< 60 fps） | Rasterization 耗时 | 优化 DisplayItems 数量 |
| GPU 内存溢出 | 合成层太大/太多 | 简化场景、异步加载 |

---

## 总结

RenderingNG 的完整流程：

```
1. Layout
   输入: LayoutObject 树 + ComputedStyle
   输出: 每个元素的位置和尺寸
   
2. Paint
   输入: LayoutObject 树 + ComputedStyle
   输出: DisplayItems（绘制指令列表）
   
3. Composite
   输入: DisplayItems + CompositingReasons
   输出: cc::Layers（分组后的指令）
   
4. Rasterize
   输入: cc::Layers + DPI
   输出: GPU 纹理（像素数据）
   
5. Aggregation & Display
   输入: GPU 纹理
   输出: 屏幕画面
```

关键理解：
- ✅ DisplayItems 是**线性**的绘制指令序列
- ✅ cc::Layers 是**分组**后的指令（用于优化）
- ✅ Rasterization 是**矢量→像素**的转换
- ✅ Transform/Opacity 不需要 repaint（合成层优化）
- ✅ 每帧都重复这个流程（60 fps = 每秒 60 次）

