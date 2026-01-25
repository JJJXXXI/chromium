# RenderingNG Paint & Compositor 快速参考卡片

## 🎯 核心概念速览

```
LayoutTree (HTML & CSS computed)
    ↓ Paint() 递归调用
DisplayItems (绘制操作列表)
    ↓ PaintController 管理
PaintArtifact (DisplayItemList + PaintChunks)
    ↓ 合成处理
cc::Layers (分组合成)
    ↓ 光栅化
GPU Textures (像素数据)
    ↓ 显示
Screen (屏幕输出)
```

---

## 📋 关键类一览

### Paint 阶段（Blink Main Thread）

| 类 | 职责 | 关键方法 |
|---|---|---|
| `PaintController` | Paint 过程管理和缓存 | `BeginPaint()`, `CreateAndAppend()`, `CommitNewDisplayItems()` |
| `LayoutObject` | 递归绘制 | `Paint(PaintInfo&)`, `PaintSelf()` |
| `DisplayItem` | 单个绘制项 | 数据容器，无关键方法 |
| `DisplayItemList` | 绘制项集合 | `push()`, `Finalize()` |
| `PaintArtifact` | Paint 输出 | `GetDisplayItemList()`, `GetPaintChunks()` |
| `PaintChunk` | DisplayItems 分组 | 数据容器 |

### Compositing 阶段（Blink Main Thread）

| 类 | 职责 | 关键方法 |
|---|---|---|
| `PaintArtifactCompositor` | 合成决策和层创建 | `Update()` |
| `PaintChunksToCcLayer` | 格式转换 | `ConvertInto()`, `Convert()` |
| `PropertyTreeManager` | 属性树管理 | 内部使用 |
| `ContentLayerClientImpl` | cc::ContentLayerClient 实现 | `PaintContentsToDisplayList()`, `UpdateCcPictureLayer()` |
| `RasterInvalidator` | 光栅失效追踪 | `Generate()`, `UpdateForRasterInducingScroll()` |

### cc 侧（Compositor Thread）

| 类 | 职责 | 关键方法 |
|---|---|---|
| `cc::DisplayItemList` | PaintOp 容器 | `Raster()`, `OffsetsOfOpsToRaster()` |
| `PaintOp` | 原始绘制操作 | 各子类 (SaveOp, DrawRectOp 等) |
| `TileManager` | Tile 管理 | `PrepareTiles()` |
| `Rasterizer` | 光栅化执行 | `RasterizeAndFinalize()` |

---

## 📁 关键文件位置速查

### Paint 相关
```
third_party/blink/renderer/core/paint/
├── layout_object.h              ← Paint 入口
├── block_painter.h              ← 块级元素 painter
├── inline_box_fragment_painter.h ← 内联元素 painter
└── ...

third_party/blink/renderer/platform/graphics/paint/
├── paint_controller.h            ← Paint 管理器 ⭐
├── display_item.h                ← DisplayItem 定义
├── display_item_list.h           ← DisplayItemList
└── paint_artifact.h              ← PaintArtifact ⭐
```

### Compositing 相关
```
third_party/blink/renderer/platform/graphics/compositing/
├── paint_artifact_compositor.h       ← 合成器 ⭐
├── paint_chunks_to_cc_layer.h        ← 格式转换
├── content_layer_client_impl.h       ← cc 适配器
└── raster_invalidator.h              ← 失效追踪
```

### cc 相关
```
cc/paint/
├── display_item_list.h           ← cc DisplayItemList ⭐
├── paint_op.h                    ← PaintOp 定义
└── ...

cc/tiles/
└── tile_manager.h                ← Tile 管理

cc/raster/
└── rasterizer.h                  ← 光栅化执行
```

---

## 🔄 完整数据流

### 1️⃣ Paint 输入输出

**输入**:
- LayoutTree（LayoutObjects）
- ComputedStyle（CSS 值）
- 无效区域（从之前的 Paint 继承）

**输出**:
- `PaintArtifact` 包含：
  - `DisplayItemList`: 所有 DisplayItems 和 PaintOps
  - `PaintChunks`: DisplayItems 的分组
  - 调试信息

**关键方法**:
```cpp
void LayoutObject::Paint(const PaintInfo&);
void PaintController::CommitNewDisplayItems();
const PaintArtifact& PaintController::PaintArtifact();
```

### 2️⃣ Compositing 输入输出

**输入**:
- `PaintArtifact` 和 viewport 属性

**处理**:
- 决定哪些 PaintChunks 需要各自的 cc::Layer
- 转换 Blink DisplayItems 为 cc::DisplayItemList
- 构建属性树（Transform/Clip/Effect）

**输出**:
- `cc::LayerList`（合成层）
- `cc::PropertyTrees`（属性信息）

**关键方法**:
```cpp
void PaintArtifactCompositor::Update(const PaintArtifact&, ...);
void PaintChunksToCcLayer::ConvertInto(..., cc::DisplayItemList&);
```

### 3️⃣ Rasterization 输入输出

**输入**:
- `cc::DisplayItemList`（PaintOps）
- Tile bounds（光栅化目标区域）

**处理**:
- 使用 R-tree 查询相交的 PaintOps
- 播放 PaintOps 到 SkCanvas
- 生成像素数据

**输出**:
- GPU 纹理（光栅化结果）

**关键方法**:
```cpp
void cc::DisplayItemList::Raster(SkCanvas*);
void Rasterizer::RasterizeAndFinalize(...);
```

---

## 🚀 性能优化核心

### ✅ Paint 优化

```cpp
// 1. 使用 Cull Rect 跳过不可见内容
if (!visual_rect.Intersects(paint_info.cull_rect))
  return;

// 2. 使用缓存避免重复 Paint
if (paint_controller.UseCachedItemIfPossible(this, type))
  return;

// 3. 使用 subsequence 优化批量项
auto idx = paint_controller.BeginSubsequence(*this);
// ... paint multiple items ...
paint_controller.EndSubsequence(idx);

// 4. 避免不必要的 Paint
if (!ShouldPaint())
  return;
```

### ✅ Composite 优化

```cpp
// 1. PaintChunks 自动分组兼容项
// 2. 属性树节点共享
// 3. 外部层（Foreign Layers）优化特殊内容

// 关键决策：何时创建新 Layer
- 需要特殊效果（掩码、滤镜）→ 新 Layer
- 不能合并（属性冲突）→ 新 Layer
- 可以合并 → 使用现有 Layer
```

### ✅ Rasterization 优化

```cpp
// 1. 区域光栅化（部分更新）
std::vector<size_t> offsets = 
    display_item_list->OffsetsOfOpsToRaster(canvas);
// 只播放相交的操作

// 2. Tile 分割
// 大 Layer 分成多个 Tiles，并行光栅化

// 3. R-tree 加速查询
rtree_.Search(query_rect, &op_indices);

// 4. RasterInvalidator 追踪
// 精确确定需要重新光栅化的区域
```

---

## 🔍 常见问题排查

### Q: Paint 时间过长？

```cpp
// ✅ 检查清单
1. PaintController::UseCachedItemIfPossible() 是否使用了缓存？
2. 是否有不必要的 Paint 调用（ShouldPaint() 检查）？
3. Cull Rect 是否正确（避免绘制视口外的内容）？
4. 是否有频繁的 Layout 触发 Paint（避免重排）？

// 诊断
TRACE_EVENT("blink.paint", "..."); // 添加追踪
paint_controller->SetRecordDebugInfo(true);
```

### Q: Composite 层过多？

```cpp
// ✅ 检查清单
1. PaintChunks 的数量是否过多？
   → 减少频繁改变属性树状态的操作
2. 是否有不必要的 Foreign Layers？
   → 检查 RequiresForeignLayer() 逻辑
3. 属性树节点是否重复创建？
   → 使用 ReusePropertyTreeNodes()

// 优化
- 合并兼容的 PaintChunks
- 减少属性树深度
- 避免频繁的效果创建
```

### Q: Rasterization 耗时？

```cpp
// ✅ 检查清单
1. OffsetsOfOpsToRaster() 是否返回过多操作？
   → 优化 PaintOp 数量或大小
2. R-tree 查询是否有效？
   → 检查 visual_rects 的精度
3. Tile 大小是否合理？
   → 调整 Tile 大小（通常 256x256 或 512x512）
4. InvalidateRect() 调用频率？
   → 减少不必要的失效

// 诊断
RasterInvalidator::SetTracksRasterInvalidations(true);
auto* tracking = raster_invalidator->GetTracking();
```

---

## 📊 时间线示例

### Paint 阶段时间线
```
BeginFrame
  ├─ BeginPaint()                          [0ms]
  ├─ LayoutView::Paint()                   [0-10ms]
  │   ├─ LayoutObject::Paint() x N         [并行]
  │   └─ CreateAndAppend() x M             [并行]
  ├─ CommitNewDisplayItems()               [10-11ms]
  │   └─ 生成 PaintArtifact
  └─ Paint 完成                             [11ms]
```

### Composite 阶段时间线
```
Paint 完成 [11ms]
  ├─ PaintArtifactCompositor::Update()     [11-12ms]
  │   ├─ 决策合成策略                      
  │   ├─ PaintChunksToCcLayer::ConvertInto()
  │   └─ 构建属性树
  └─ Composite 完成 [12ms]
     
Commit 到 Impl Thread [12-13ms]
  └─ cc::PropertyTrees 和 Layers 同步
```

### Rasterization 阶段时间线
```
PrepareTiles() [Impl Thread]
  ├─ 遍历 Layers 和 Tiles              [13-14ms]
  ├─ 标记需要光栅化的 Tiles
  └─ 入队 RasterTasks

RasterWorkerPool (后台线程)
  ├─ OffsetsOfOpsToRaster()             [14-14.5ms]
  ├─ SkCanvas::drawXxx()                [并行 x CPU核数]
  ├─ GPU::UploadTile()
  └─ Rasterization 完成 [14.5ms]

DrawFrame() [Impl Thread]
  ├─ 读取光栅化 Tiles               [14.5-15ms]
  ├─ 合成 RenderPass
  └─ 显示 [15ms]
```

---

## 🎓 学习路径

### 初级：理解基本流程
1. 阅读 `Paint` 函数的递归逻辑
2. 理解 `DisplayItem` 和 `DisplayItemList` 的关系
3. 认识 `PaintController` 的缓存机制

### 中级：深入 Compositing
1. 学习 `PaintArtifactCompositor` 的决策逻辑
2. 理解 `PaintChunksToCcLayer` 的转换过程
3. 研究属性树的构建

### 高级：性能优化
1. 实现 `RasterInvalidator` 的失效追踪
2. 优化 Tile 光栅化流程
3. 设计新的合成策略

### 实战：增加新功能
1. 在 `paint_artifact_compositor.h` 中定义新的合成规则
2. 在 `paint_chunks_to_cc_layer.cc` 中实现转换逻辑
3. 添加单元测试验证正确性

---

## 📝 常用命令

### 编译相关
```bash
# 只编译 Paint 相关代码
autoninja -C out/Default blink_platform_unittests

# 编译 cc 相关代码
autoninja -C out/Default cc_unittests

# 编译整个 Chromium
autoninja -C out/Default chrome
```

### 测试相关
```bash
# 运行 Paint 相关测试
out/Default/blink_platform_unittests --gtest_filter="*Paint*"

# 运行 Compositor 测试
out/Default/blink_platform_unittests --gtest_filter="*Compositor*"

# 启用调试日志
out/Default/chrome --vmodule="paint_controller=2,raster_invalidator=2"
```

### 调试相关
```bash
# 使用 gdb 调试
gdb out/Default/chrome
(gdb) b PaintController::CommitNewDisplayItems
(gdb) run

# Chrome 内部追踪
chrome://tracing → 记录 → 搜索 Paint/Composite/Raster
```

---

## 🔗 相关链接

### 文档
- [Blink Compositing Architecture](https://docs.google.com/document/d/1mWc5v9lF0FFfZQ0dxsKr4mCkOcKYO3L2IvCz0hBB2Eo/edit)
- [Paint Architecture](https://chromium.googlesource.com/chromium/src/+/main/third_party/blink/renderer/platform/graphics/paint/README.md)
- [Compositing README](https://chromium.googlesource.com/chromium/src/+/main/third_party/blink/renderer/platform/graphics/compositing/README.md)

### 关键的 Gerrit 审查
- Paint Controller 相关 changes
- RasterInvalidator 相关 changes
- PaintArtifactCompositor 相关 changes

### 相关的 Issue
- 搜索 "Paint"、"Compositor"、"Rasterization" 在 monorail.chromium.org

---

## ⏱️ 版本信息

**最后更新**: 2026-01-18
**适用于**: Chromium M125+
**Paint 架构版本**: RenderingNG (CAP v2)

---

## 💡 提示

1. **始终检查 visual_rect**: DisplayItem 的边界决定了合成和光栅化的范围
2. **优先使用缓存**: `UseCachedItemIfPossible()` 可以显著提升性能
3. **监控 PaintChunk 数量**: 过多的 chunks 会增加合成和光栅化的开销
4. **使用 Cull Rect**: 跳过不在视口内的内容可以大幅减少 Paint 工作
5. **定期运行 Perfetto**: 使用系统级追踪工具了解性能瓶颈

