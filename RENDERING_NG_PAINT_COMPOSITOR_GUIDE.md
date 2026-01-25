# RenderingNG Paint & Compositor 完整指南

## 目录
1. [Paint 阶段](#paint-阶段)
2. [DisplayItem 和 PaintArtifact](#displayitem-和-paintartifact)
3. [Compositor 处理](#compositor-处理)
4. [Rasterization 流程](#rasterization-流程)
5. [关键类和方法](#关键类和方法)
6. [数据流向](#数据流向)

---

## Paint 阶段

### Paint 入口和流程

**主要路径**：`LayoutObject::Paint()` → 递归遍历 LayoutTree 生成 DisplayItems

#### 文件位置
- **Paint 定义**: [third_party/blink/renderer/core/layout/layout_object.h](third_party/blink/renderer/core/layout/layout_object.h)
- **Paint 控制器**: [third_party/blink/renderer/platform/graphics/paint/paint_controller.h](third_party/blink/renderer/platform/graphics/paint/paint_controller.h)

#### 核心代码流程

```cpp
// Layout Object Paint 的核心逻辑
void LayoutObject::Paint(const PaintInfo& paint_info) {
  // 1. 检查 paint 的有效性
  if (!ShouldPaint()) {
    return;
  }
  
  // 2. 保存 canvas 状态
  paint_info.context->save();
  
  // 3. 应用 transform、clipping 等
  ApplyTransform(paint_info);
  
  // 4. 执行自身的 paint 逻辑
  PaintSelf(paint_info);
  
  // 5. Paint 子元素 (递归调用)
  for (LayoutObject* child = FirstChild(); child; 
       child = child->NextSibling()) {
    if (child->IsVisible()) {
      child->Paint(paint_info);  // ← 递归调用
    }
  }
  
  // 6. 恢复 canvas 状态
  paint_info.context->restore();
}
```

### Paint 的关键阶段

Paint 按多个阶段执行，每个阶段负责绘制不同的内容：

```
Paint(LayoutObject)
  ├─ PaintPhase::BackgroundPhase
  │   └─ 绘制背景色、背景图片
  │
  ├─ PaintPhase::BorderPhase  
  │   └─ 绘制边框
  │
  ├─ PaintPhase::ForegroundPhase
  │   ├─ 绘制文本
  │   ├─ 绘制替换元素（图片等）
  │   └─ PaintChildren()  ← 递归调用子元素
  │
  └─ PaintPhase::OutlinePhase
      └─ 绘制 outline 和其他装饰
```

### Paint 的输入：PaintInfo 结构

**文件**: `third_party/blink/renderer/core/paint/paint_info.h`

```cpp
struct PaintInfo {
  // 关键字段
  
  // 当前绘制阶段
  PaintPhase phase;
  
  // 绘制上下文（包含绘制目标）
  GraphicsContext* context;
  
  // 需要重新绘制的区域（裁剪矩形）
  gfx::Rect cull_rect;
  
  // 绘制时的变换
  const TransformPaintPropertyNode* fragment_paint_offset;
  
  // 当前的 LayoutObject
  DisplayItemClient* display_item_client;
  
  // 其他信息
  bool force_repaint;
  bool should_paint_background;
};
```

---

## DisplayItem 和 PaintArtifact

### DisplayItem 的结构

DisplayItem 是 Paint 输出的最小单位，表示一个绘制操作。

#### 文件位置
- **DisplayItem 定义**: [third_party/blink/renderer/platform/graphics/paint/display_item.h](third_party/blink/renderer/platform/graphics/paint/display_item.h)
- **DisplayItemList**: [third_party/blink/renderer/platform/graphics/paint/display_item_list.h](third_party/blink/renderer/platform/graphics/paint/display_item_list.h)

#### DisplayItem 的类型

```cpp
enum Type {
  kUninitializedType,
  
  // 绘制项 (从 kDrawingFirst 到 kDrawingLast)
  kDrawingFirst,
  kDrawingPaintPhaseFirst = kDrawingFirst,
  kDrawingPaintPhaseLast = kDrawingFirst + kPaintPhaseMax,
  
  // 具体的绘制类型
  kBoxDecorationBackground,      // 背景和边框
  kCaret,                         // 光标
  kSVGImage,                      // SVG 图像
  kImageAreaFocusRing,            // 图像焦点环
  kScrollCorner,                  // 滚动角
  
  // 外部层（foreign layers）
  kForeignLayerFirst,
  kForeignLayerCanvas,            // Canvas 画布
  kForeignLayerDevToolsOverlay,   // DevTools 覆盖层
  kForeignLayerPlugin,            // 插件
};
```

#### DisplayItem 的内容

每个 DisplayItem 包含：
- **绘制操作**: PaintOp 集合（如 DrawRectOp、DrawPathOp 等）
- **边界矩形**: visual_rect - 该项占据的区域
- **属性**: 绘制时的剪裁、变换等属性
- **来源信息**: 创建该项的 LayoutObject ID

### PaintArtifact 的结构

PaintArtifact 是 Paint 阶段的完整输出，包含所有 DisplayItems 和分组信息。

#### 文件位置
- **PaintArtifact**: [third_party/blink/renderer/platform/graphics/paint/paint_artifact.h](third_party/blink/renderer/platform/graphics/paint/paint_artifact.h)

#### PaintArtifact 的组成

```cpp
class PaintArtifact {
  // 1. DisplayItemList - 所有绘制项的列表
  DisplayItemList display_item_list_;
  
  // 2. PaintChunks - 绘制项的分组
  //    每个 chunk 具有相同的 property tree state
  //    用于后续合成时确定哪些项应该在同一个 cc::Layer 中
  PaintChunks chunks_;
  
  // 3. 调试信息
  DebugInfo debug_info_;
};
```

#### PaintChunk 的结构

```cpp
struct PaintChunk {
  // 该 chunk 中第一个 DisplayItem 的索引
  wtf_size_t begin_index;
  
  // 该 chunk 中最后一个 DisplayItem 之后的索引
  wtf_size_t end_index;
  
  // chunk 的唯一标识符
  PaintChunk::Id id;
  
  // 该 chunk 中所有项的边界矩形
  gfx::Rect bounds;
  
  // 该 chunk 的属性树状态
  // 包含：transform、clip、effect 等
  PropertyTreeState properties;
  
  // 是否需要重新光栅化
  PaintInvalidationReason invalidation_reason;
};
```

### PaintController：Paint 的关键管理器

**文件**: [third_party/blink/renderer/platform/graphics/paint/paint_controller.h](third_party/blink/renderer/platform/graphics/paint/paint_controller.h)

```cpp
class PaintController {
 public:
  // 1. 更新当前 paint chunk 的属性
  void UpdateCurrentPaintChunkProperties(const PropertyTreeStateOrAlias&);
  
  // 2. 创建并添加 DisplayItem
  template <typename DisplayItemClass, typename... Args>
  void CreateAndAppend(const DisplayItemClient& client, Args&&... args);
  
  // 3. 尝试使用缓存的项
  bool UseCachedItemIfPossible(const DisplayItemClient&, DisplayItem::Type);
  
  // 4. 尝试使用缓存的子序列
  bool UseCachedSubsequenceIfPossible(const DisplayItemClient&);
  
  // 5. 完成 paint 并生成 PaintArtifact
  const PaintArtifact& PaintArtifact() const;
  
  // 6. 提交新的 DisplayItems
  void CommitNewDisplayItems();
};
```

#### PaintController 的工作流程

```
LocalFrameView::Paint(GraphicsContext&)
  ↓
PaintController::BeginPaint()
  ↓ 开始记录新的绘制项
  ↓
LayoutView::Paint()
  ├─ for each LayoutObject:
  │   └─ LayoutObject::Paint()
  │       └─ PaintController::CreateAndAppend() ← 添加 DisplayItems
  └─ 递归绘制子元素
  ↓
PaintController::CommitNewDisplayItems()
  ↓ 生成 PaintArtifact，包含 DisplayItemList 和 PaintChunks
  ↓
返回 PaintArtifact 给 Compositor
```

---

## Compositor 处理

### Compositor 的角色

Compositor 将 Paint 输出的 PaintArtifact 转换为 cc::Layers，并构建属性树。

#### 文件位置
- **PaintArtifactCompositor**: [third_party/blink/renderer/platform/graphics/compositing/paint_artifact_compositor.h](third_party/blink/renderer/platform/graphics/compositing/paint_artifact_compositor.h)
- **PaintChunksToCcLayer**: [third_party/blink/renderer/platform/graphics/compositing/paint_chunks_to_cc_layer.h](third_party/blink/renderer/platform/graphics/compositing/paint_chunks_to_cc_layer.h)

### PaintArtifactCompositor 的核心逻辑

```cpp
class PaintArtifactCompositor {
 public:
  // 更新合成层
  // 输入：PaintArtifact（Paint 的输出）
  // 输出：cc::Layers 树和属性树
  void Update(const PaintArtifact&, 
              const ViewportProperties&);
  
  // 更新的类型
  enum class UpdateType {
    kNone,
    
    // 快速路径：仅因滚动失效
    kRasterInducingScroll,
    
    // 快速路径：已有层但内容改变
    kRepaintAfterPaint,
    
    // 完整更新：重建层树
    kFull,
  };
};
```

### Compositor 处理 PaintArtifact 的步骤

```
PaintArtifactCompositor::Update(PaintArtifact)
  ↓
遍历每个 PaintChunk
  ├─ 决定合成策略
  │  ├─ 是否需要创建新的 cc::Layer？
  │  ├─ 是否可以合并到现有 Layer？
  │  └─ 是否需要特殊的外部 Layer（如 Canvas）？
  │
  ├─ 创建或重用 cc::PictureLayer
  │
  ├─ 转换 PaintChunk 到 cc::DisplayItemList
  │   └─ PaintChunksToCcLayer::ConvertInto()
  │
  └─ 设置 cc::Layer 的属性
      ├─ 变换（Transform）
      ├─ 裁剪（Clip）
      ├─ 效果（Effect）
      └─ 其他属性（Scroll 等）

↓
生成 cc::PropertyTrees
  ├─ TransformTree - 所有变换节点
  ├─ ClipTree - 所有裁剪节点
  └─ EffectTree - 所有效果节点（透明度、模糊等）

↓
返回 cc::LayerList 给 LayerTreeHost
```

### PaintChunksToCcLayer 的转换

**文件**: [third_party/blink/renderer/platform/graphics/compositing/paint_chunks_to_cc_layer.h](third_party/blink/renderer/platform/graphics/compositing/paint_chunks_to_cc_layer.h)

```cpp
class PaintChunksToCcLayer {
 public:
  // 将 Blink PaintChunks 转换为 cc::DisplayItemList
  static void ConvertInto(
      const PaintChunkSubset& chunks,
      const PropertyTreeState& layer_state,
      const gfx::Vector2dF& layer_offset,
      RasterUnderInvalidationCheckingParams*,
      cc::DisplayItemList& output);  // ← 输出 cc::DisplayItemList
};
```

#### 转换过程的关键操作

1. **状态扁平化** (State Flattening)
   - 当 PaintChunk 的 PropertyTreeState 与目标 Layer 的状态不同时
   - 需要插入 Begin/End 操作来调整绘制状态

2. **PaintOp 复制**
   - 从 Blink DisplayItemList 复制 PaintOps 到 cc::DisplayItemList
   - 保留所有绘制操作的顺序和内容

3. **属性树映射**
   - 映射 Blink 的属性树节点到 cc 的属性树节点

---

## Rasterization 流程

### Rasterization 的定义

Rasterization 是将矢量绘制操作转换为像素位图的过程。

### 关键参与者

#### 1. cc::DisplayItemList
**文件**: [cc/paint/display_item_list.h](cc/paint/display_item_list.h)

```cpp
class DisplayItemList : public base::RefCountedThreadSafe<DisplayItemList> {
 public:
  // 核心方法：将 DisplayItemList 光栅化为 SkCanvas
  void Raster(SkCanvas* canvas,
              ImageProvider* image_provider = nullptr,
              const ScrollOffsetMap* raster_inducing_scroll_offsets = nullptr) const;
  
  void Raster(SkCanvas* canvas, const PlaybackParams& params) const;
  
  // 确定需要光栅化哪些操作
  std::vector<size_t> OffsetsOfOpsToRaster(SkCanvas* canvas) const;
  
  // 开始记录新的绘制操作
  void StartPaint();
  
  // 推送绘制操作 (SaveOp, DrawRectOp, DrawTextOp 等)
  template <typename T, typename... Args>
  size_t push(Args&&... args);
  
  // 完成记录
  void Finalize();
};
```

#### 2. RasterInvalidator
**文件**: [third_party/blink/renderer/platform/graphics/paint/raster_invalidator.h](third_party/blink/renderer/platform/graphics/paint/raster_invalidator.h)

```cpp
class RasterInvalidator : public GarbageCollected<RasterInvalidator> {
 public:
  // 生成光栅失效区域
  void Generate(const PaintChunkSubset& chunks,
                const gfx::Vector2dF& layer_offset,
                const gfx::Size& layer_bounds,
                const PropertyTreeState& layer_state);
  
  // 回调接口：当需要重新光栅化时
  class Callback {
    virtual void InvalidateRect(const gfx::Rect&) = 0;
  };
};
```

### Rasterization 的流程

```
LayerTreeHostImpl::WillBeginImplFrame()
  ↓
TileManager::PrepareTiles()
  ↓ 遍历每个 cc::PictureLayer
  ↓
ContentLayerClientImpl::PaintContentsToDisplayList()
  │ ← 返回 cc::DisplayItemList
  │   包含所有该 Layer 的绘制操作
  ↓
TileManager::CreateOrUpdateTiles()
  ↓ 对每个需要光栅化的 Tile：
  ↓
Rasterizer::RasterizeAndFinalize()
  ├─ 创建 SkCanvas（对应 Tile 的像素）
  ├─ DisplayItemList::Raster(canvas)
  │   ├─ 遍历 PaintOps
  │   ├─ 播放操作到 SkCanvas
  │   └─ 生成像素数据
  └─ 上传到 GPU 纹理
  
↓
Compositor::DrawFrame()
  ├─ 合成所有 Tiles
  └─ 显示到屏幕
```

### 光栅化的关键优化

#### 1. 区域光栅化 (Partial Rasterization)
```cpp
// 只光栅化需要更新的区域，而不是整个 Layer
std::vector<size_t> DisplayItemList::OffsetsOfOpsToRaster(SkCanvas* canvas) {
  // 使用 R-tree 查询与 canvas bounds 相交的操作
  rtree_.Search(canvas->getLocalClipBounds(), offsets);
  return offsets;
}
```

#### 2. PaintOp 缓存
- DisplayItemList 中的 PaintOp 被缓存
- 如果内容未改变，直接复用缓存的 DisplayItemList
- 由 RasterInvalidator 跟踪哪些区域需要更新

#### 3. Tile 管理
- 大的 Layer 被分割为多个 Tiles
- 每个 Tile 独立光栅化和缓存
- 只更新改变的 Tiles

---

## 关键类和方法

### Blink 侧（Paint 生成）

| 类 | 文件 | 主要方法 | 功能 |
|---|---|---|---|
| `LayoutObject` | `core/layout/layout_object.h` | `Paint()`, `PaintSelf()` | 递归绘制对象树 |
| `PaintController` | `platform/graphics/paint/paint_controller.h` | `CreateAndAppend()`, `CommitNewDisplayItems()` | 管理 Paint 过程和缓存 |
| `DisplayItem` | `platform/graphics/paint/display_item.h` | 无（数据结构） | 单个绘制项 |
| `DisplayItemList` | `platform/graphics/paint/display_item_list.h` | `push()`, `Finalize()` | 绘制项集合 |
| `PaintArtifact` | `platform/graphics/paint/paint_artifact.h` | `GetDisplayItemList()`, `GetPaintChunks()` | Paint 输出容器 |
| `PaintChunk` | `platform/graphics/paint/paint_chunk.h` | 无（数据结构） | DisplayItems 分组 |

### Compositing 侧（层管理）

| 类 | 文件 | 主要方法 | 功能 |
|---|---|---|---|
| `PaintArtifactCompositor` | `platform/graphics/compositing/paint_artifact_compositor.h` | `Update()` | Paint → cc::Layers 转换 |
| `PaintChunksToCcLayer` | `platform/graphics/compositing/paint_chunks_to_cc_layer.h` | `ConvertInto()`, `Convert()` | PaintChunk → cc::DisplayItemList |
| `PropertyTreeManager` | `platform/graphics/compositing/property_tree_manager.h` | 无（内部使用） | 管理 cc 属性树 |
| `RasterInvalidator` | `platform/graphics/paint/raster_invalidator.h` | `Generate()` | 追踪光栅失效 |
| `ContentLayerClientImpl` | `platform/graphics/compositing/content_layer_client_impl.h` | `PaintContentsToDisplayList()` | cc::ContentLayerClient 实现 |

### cc 侧（光栅和显示）

| 类 | 文件 | 主要方法 | 功能 |
|---|---|---|---|
| `DisplayItemList` (cc) | `cc/paint/display_item_list.h` | `Raster()`, `StartPaint()`, `push()` | PaintOp 容器 |
| `PaintOp` | `cc/paint/paint_op.h` | 各 Op 的 PlaybackType | 原始绘制操作 |
| `TileManager` | `cc/tiles/tile_manager.h` | `PrepareTiles()` | Tile 光栅化管理 |
| `Rasterizer` | `cc/raster/rasterizer.h` | `RasterizeAndFinalize()` | 执行光栅化 |
| `LayerTreeHostImpl` | `cc/trees/layer_tree_host_impl.h` | `ActivatePendingTree()` | 合成树管理（impl thread） |

---

## 数据流向

### 完整的 Paint 到 Rasterization 流程

```
┌─────────────────────────────────────────────────────────┐
│ 1. Layout 完成                                          │
│    LayoutObject 树已确定                                 │
└────────────────────┬──────────────────────────────────┘

                     ↓

┌─────────────────────────────────────────────────────────┐
│ 2. Paint 阶段 (Blink Main Thread)                       │
│                                                         │
│    LocalFrameView::Paint(GraphicsContext)              │
│      ├─ PaintController::BeginPaint()                  │
│      ├─ LayoutView::Paint()                            │
│      │   └─ 递归调用 LayoutObject::Paint()            │
│      │       └─ PaintController::CreateAndAppend()    │
│      │           → 生成 DisplayItems                  │
│      └─ PaintController::CommitNewDisplayItems()      │
│          → 生成 PaintArtifact                         │
│              ├─ DisplayItemList                       │
│              └─ PaintChunks                           │
└────────────────────┬──────────────────────────────────┘

                     ↓

┌─────────────────────────────────────────────────────────┐
│ 3. PaintArtifact 结构                                   │
│                                                         │
│    PaintArtifact                                        │
│    ├─ DisplayItemList                                  │
│    │   ├─ PaintOps: SaveOp, DrawRectOp, ...           │
│    │   └─ visual_rects: 每个 Op 的边界               │
│    └─ PaintChunks[]                                    │
│        ├─ chunk[0]: begin_index=0, end_index=10       │
│        │            properties={transform, clip, ...}  │
│        ├─ chunk[1]: begin_index=10, end_index=20      │
│        └─ ...                                          │
└────────────────────┬──────────────────────────────────┘

                     ↓

┌─────────────────────────────────────────────────────────┐
│ 4. Compositor 处理 (Blink Main Thread)                 │
│                                                         │
│    PaintArtifactCompositor::Update(PaintArtifact)      │
│      ├─ 遍历每个 PaintChunk                           │
│      ├─ 决定合成策略（创建/合并 cc::Layer）           │
│      ├─ PaintChunksToCcLayer::ConvertInto()          │
│      │   └─ Blink DisplayItems → cc DisplayItems     │
│      └─ 构建 cc::PropertyTrees                        │
│          ├─ TransformTree                            │
│          ├─ ClipTree                                 │
│          └─ EffectTree                               │
│                                                       │
│    输出：                                              │
│    ├─ cc::LayerList                                  │
│    └─ cc::PropertyTrees                              │
└────────────────────┬──────────────────────────────────┘

                     ↓

┌─────────────────────────────────────────────────────────┐
│ 5. LayerTreeHost Commit (Main → Impl Thread)           │
│                                                         │
│    LayerTreeHost::CommitChanges()                      │
│      ├─ 复制 PendingTree 到 ActiveTree               │
│      ├─ 同步 cc::Layers 和属性树                     │
│      └─ 标记需要重新光栅化的 Tiles                   │
└────────────────────┬──────────────────────────────────┘

                     ↓

┌─────────────────────────────────────────────────────────┐
│ 6. Rasterization (Compositor Impl Thread)              │
│                                                         │
│    LayerTreeHostImpl::ActivatePendingTree()             │
│      ↓                                                  │
│    TileManager::PrepareTiles()                         │
│      ├─ 遍历所有 PictureLayer 和 Tiles              │
│      ├─ 识别需要光栅化的 Tiles                       │
│      └─ RasterTask 入队                              │
│          ↓                                            │
│    RasterWorkerPool::RasterizeAndFinalize()          │
│      ├─ ContentLayerClientImpl::                      │
│      │   PaintContentsToDisplayList()                │
│      │   → 返回 cc::DisplayItemList                 │
│      │                                               │
│      ├─ SkCanvas::Create(Tile bounds)               │
│      ├─ DisplayItemList::Raster(canvas)             │
│      │   ├─ OffsetsOfOpsToRaster()                  │
│      │   │   → R-tree 查询相交的 PaintOps         │
│      │   ├─ PaintOpBuffer::Playback(canvas)        │
│      │   │   → 执行每个 PaintOp                   │
│      │   └─ 生成像素数据                            │
│      └─ GPU::UploadTile(pixels)                     │
│          → 上传到 GPU 纹理                          │
└────────────────────┬──────────────────────────────────┘

                     ↓

┌─────────────────────────────────────────────────────────┐
│ 7. 合成和显示                                          │
│                                                         │
│    Compositor::DrawFrame()                            │
│      ├─ 根据 RenderPass 列表合成                    │
│      ├─ 读取光栅化的 Tiles 纹理                     │
│      ├─ 应用 PropertyTrees 的变换和效果              │
│      └─ 输出到屏幕                                   │
│          ↓                                            │
│    屏幕显示更新内容                                   │
└─────────────────────────────────────────────────────────┘
```

### 数据结构变换

```
LayoutTree (LayoutObjects)
    ↓ Paint()
DisplayItemList (PaintOps)
    ↓ PaintController::CommitNewDisplayItems()
PaintArtifact (DisplayItemList + PaintChunks)
    ↓ PaintArtifactCompositor::Update()
cc::Layers (cc::DisplayItemList + PropertyTrees)
    ↓ LayerTreeHost::CommitChanges()
RenderPass (由 LayerTreeHostImpl 生成)
    ↓ TileManager::PrepareTiles()
Tiles
    ↓ Rasterizer::RasterizeAndFinalize()
GPU Textures (光栅化的像素数据)
    ↓ Compositor::DrawFrame()
Screen (屏幕输出)
```

---

## 关键文件位置速查

### Paint 相关
- [third_party/blink/renderer/core/paint/](third_party/blink/renderer/core/paint/) - 各类 Painter
- [third_party/blink/renderer/platform/graphics/paint/paint_controller.h](third_party/blink/renderer/platform/graphics/paint/paint_controller.h) - Paint 控制器
- [third_party/blink/renderer/platform/graphics/paint/display_item.h](third_party/blink/renderer/platform/graphics/paint/display_item.h) - DisplayItem 定义

### DisplayItem 和 PaintArtifact
- [third_party/blink/renderer/platform/graphics/paint/display_item_list.h](third_party/blink/renderer/platform/graphics/paint/display_item_list.h) - DisplayItemList
- [third_party/blink/renderer/platform/graphics/paint/paint_artifact.h](third_party/blink/renderer/platform/graphics/paint/paint_artifact.h) - PaintArtifact
- [third_party/blink/renderer/platform/graphics/paint/paint_chunk.h](third_party/blink/renderer/platform/graphics/paint/paint_chunk.h) - PaintChunk

### Compositing
- [third_party/blink/renderer/platform/graphics/compositing/paint_artifact_compositor.h](third_party/blink/renderer/platform/graphics/compositing/paint_artifact_compositor.h) - PaintArtifactCompositor
- [third_party/blink/renderer/platform/graphics/compositing/paint_chunks_to_cc_layer.h](third_party/blink/renderer/platform/graphics/compositing/paint_chunks_to_cc_layer.h) - 转换逻辑
- [third_party/blink/renderer/platform/graphics/compositing/content_layer_client_impl.h](third_party/blink/renderer/platform/graphics/compositing/content_layer_client_impl.h) - cc::ContentLayerClient 实现

### Rasterization
- [cc/paint/display_item_list.h](cc/paint/display_item_list.h) - cc DisplayItemList
- [cc/paint/paint_op.h](cc/paint/paint_op.h) - PaintOp 定义
- [cc/tiles/tile_manager.h](cc/tiles/tile_manager.h) - Tile 管理
- [third_party/blink/renderer/platform/graphics/paint/raster_invalidator.h](third_party/blink/renderer/platform/graphics/paint/raster_invalidator.h) - 光栅失效追踪

---

## 性能优化点

### 1. DisplayItem 缓存
- PaintController 缓存不变的 DisplayItems
- 避免重复生成和存储相同的绘制操作

### 2. PaintChunk 分组
- 自动将兼容的 DisplayItems 分组为 PaintChunks
- 减少后续合成时的层创建

### 3. 属性树共享
- 相同的变换、裁剪、效果被多个层共享
- 减少属性树节点数量

### 4. Tile 光栅化
- 大的 Layer 被分割为小 Tiles
- 并行光栅化多个 Tiles
- 只更新改变的 Tiles

### 5. 区域光栅化 (Partial Rasterization)
- 使用 R-tree 快速查询相交的 PaintOps
- 只光栅化需要更新的区域

### 6. RasterInvalidator 追踪
- 精确追踪哪些 Tiles 需要重新光栅化
- 避免不必要的光栅化工作

---

## 常见问题

### Q: Paint 和 Composite 的区别？
A: 
- **Paint**: 生成绘制操作列表（DisplayItems），由 LayoutObjects 驱动，运行在 main thread
- **Composite**: 将 DisplayItems 组织为 cc::Layers，决定哪些内容需要什么样的层，运行在 main thread，但基于 Paint 输出

### Q: 为什么需要 PaintChunks？
A: PaintChunks 将具有相同属性树状态的 DisplayItems 分组，这样 Compositor 可以：
- 决定是否可以合并到同一个 cc::Layer
- 快速识别哪些项需要一起光栅化
- 优化属性树的大小

### Q: DisplayItem 和 PaintOp 的区别？
A: 
- **DisplayItem**: Blink 侧的绘制单位，包含一个或多个 PaintOps，具有元数据（边界、ID 等）
- **PaintOp**: cc 侧的原始绘制操作（SaveOp、DrawRectOp 等），可序列化并光栅化

### Q: 光栅化何时发生？
A: 在 Compositor impl thread 的 PrepareTiles 阶段，当 TileManager 识别到某个 Tile 需要重新光栅化时，发起 RasterTask，由 RasterWorkerPool 执行

