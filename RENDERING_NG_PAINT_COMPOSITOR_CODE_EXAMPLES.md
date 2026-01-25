# RenderingNG Paint & Compositor 代码示例和快速参考

## 目录
1. [Paint 代码示例](#paint-代码示例)
2. [DisplayItem 示例](#displayitem-示例)
3. [Compositor 代码示例](#compositor-代码示例)
4. [Rasterization 示例](#rasterization-示例)
5. [关键方法速查](#关键方法速查)
6. [调试技巧](#调试技巧)

---

## Paint 代码示例

### 1. 基本 Paint 流程

**文件**: [third_party/blink/renderer/core/paint/block_painter.cc](third_party/blink/renderer/core/paint/block_painter.cc)

```cpp
void BlockPainter::Paint(const PaintInfo& paint_info,
                         const LayoutPoint& paint_offset) {
  // 检查是否需要绘制
  if (box_.HasSelfPaintingLayer())
    return;

  // 绘制背景
  if (ShouldPaintBackground(paint_info)) {
    PaintBoxDecorationBackground(paint_info, paint_offset);
  }

  // 绘制子元素
  PaintContents(paint_info, paint_offset);

  // 绘制边框和轮廓
  if (ShouldPaintSelfOutline(paint_info.phase)) {
    PaintOutline(paint_info, paint_offset);
  }
}

void BlockPainter::PaintContents(const PaintInfo& paint_info,
                                 const LayoutPoint& paint_offset) {
  // 递归绘制子元素
  for (LayoutBox* child = box_.FirstChildBox(); child;
       child = child->NextSiblingBox()) {
    PaintChild(*child, paint_info, paint_offset);
  }
}
```

### 2. 使用 PaintController 添加 DisplayItem

**文件**: [third_party/blink/renderer/core/paint/inline_box_fragment_painter.cc](third_party/blink/renderer/core/paint/inline_box_fragment_painter.cc)

```cpp
void InlineBoxFragmentPainter::PaintSelectionBackground(
    const PaintInfo& paint_info,
    const gfx::Rect& rect) {
  
  // 1. 创建 DisplayItem
  if (paint_info.phase == PaintPhase::kForeground) {
    // 2. 使用 PaintController 添加绘制项
    paint_controller.CreateAndAppend<SelectionBackgroundDisplayItem>(
        fragment_client_,  // 所有者
        color,             // 颜色参数
        rect);             // 矩形参数
    
    // 或者使用缓存的项
    if (!paint_controller.UseCachedItemIfPossible(
            fragment_client_, 
            DisplayItem::kSelectionBackground)) {
      paint_controller.CreateAndAppend<SelectionBackgroundDisplayItem>(
          fragment_client_, color, rect);
    }
  }
}
```

### 3. Paint 的 Cull Rect 优化

```cpp
class InlineBoxFragmentPainter {
 private:
  // 检查绘制区域是否在 cull rect 内
  bool ShouldPaint(const PaintInfo& paint_info) {
    gfx::Rect fragment_rect = 
        fragment_.LocalVisualRect();
    
    // 与 cull rect 相交检查
    return fragment_rect.Intersects(paint_info.cull_rect);
  }
};
```

---

## DisplayItem 示例

### 1. 自定义 DisplayItem 类

**文件**: [third_party/blink/renderer/platform/graphics/paint/drawing_display_item.h](third_party/blink/renderer/platform/graphics/paint/drawing_display_item.h)

```cpp
// DisplayItem 的示例：绘制矩形
class DrawingDisplayItem : public DisplayItem {
 public:
  DrawingDisplayItem(const DisplayItemClient& client,
                     Type type,
                     sk_sp<const PaintRecord> record)
      : DisplayItem(client, type, sizeof(*this)),
        record_(std::move(record)) {}

  sk_sp<const PaintRecord> GetPaintRecord() const {
    return record_;
  }

 private:
  sk_sp<const PaintRecord> record_;
};
```

### 2. 创建 DisplayItem 的 PaintRecord

```cpp
// 在 Painter 中创建绘制操作
void SomePainter::Paint(const PaintInfo& paint_info) {
  // 1. 开始记录
  paint_info.context->GetPaintController().BeginPaint();
  
  // 2. 添加绘制操作（通过 PaintContext）
  cc::PaintCanvas* canvas = paint_info.context->Canvas();
  
  // 绘制矩形
  cc::PaintFlags flags;
  flags.setColor(SK_ColorRED);
  canvas->drawRect(SkRect::MakeXYWH(10, 10, 50, 50), flags);
  
  // 3. 创建 DisplayItem
  paint_info.context->GetPaintController()
      .CreateAndAppend<DrawingDisplayItem>(
          *this,
          DisplayItem::kDrawingFirst,
          paint_info.context->GetPaintRecord());
}
```

### 3. PaintArtifact 的查询

```cpp
// 遍历 PaintArtifact 中的所有项
void AnalyzePaintArtifact(const PaintArtifact& artifact) {
  const DisplayItemList& items = artifact.GetDisplayItemList();
  const PaintChunks& chunks = artifact.GetPaintChunks();
  
  // 遍历每个 chunk
  for (const auto& chunk : chunks) {
    printf("Chunk: %d items from %zu to %zu\n",
           chunk.id.unique_id,
           chunk.begin_index,
           chunk.end_index);
    
    // 遍历 chunk 内的 items
    auto item_range = artifact.DisplayItemsInChunk(
        chunk_index);
    for (const auto& item : item_range) {
      printf("  Item type: %d\n", item.GetType());
    }
  }
}
```

---

## Compositor 代码示例

### 1. PaintArtifactCompositor 更新

**文件**: [third_party/blink/renderer/platform/graphics/compositing/paint_artifact_compositor.cc](third_party/blink/renderer/platform/graphics/compositing/paint_artifact_compositor.cc)

```cpp
void PaintArtifactCompositor::Update(
    const PaintArtifact& paint_artifact,
    const ViewportProperties& viewport_properties) {
  
  // 1. 确定更新类型
  UpdateType update_type = DetermineUpdateType(paint_artifact);
  
  // 2. 根据更新类型执行对应的操作
  if (update_type == UpdateType::kFull) {
    // 完整重建
    FullUpdate(paint_artifact, viewport_properties);
  } else if (update_type == UpdateType::kRepaintAfterPaint) {
    // 快速更新（仅内容改变）
    RepaintUpdate(paint_artifact);
  } else if (update_type == UpdateType::kRasterInducingScroll) {
    // 滚动相关的快速更新
    RasterInducingScrollUpdate(paint_artifact);
  }
  
  // 3. 更新属性树
  property_tree_manager_->UpdatePropertyTrees(
      paint_artifact, viewport_properties);
}

void PaintArtifactCompositor::FullUpdate(
    const PaintArtifact& paint_artifact,
    const ViewportProperties& viewport_properties) {
  
  // 遍历每个 PaintChunk
  for (const auto& chunk : paint_artifact.GetPaintChunks()) {
    // 检查是否需要外部层（Foreign Layer）
    if (RequiresForeignLayer(chunk)) {
      // 创建或重用 foreign layer
      CreateForeignLayer(chunk);
    } else {
      // 创建或重用 picture layer
      scoped_refptr<cc::PictureLayer> layer = 
          CreateOrReuseLayer(chunk);
      
      // 转换绘制内容
      PaintChunksToCcLayer::ConvertInto(
          chunk_subset,
          chunk.properties,
          layer_offset,
          raster_under_invalidation_checking_params,
          layer->PaintContentsToDisplayList());
    }
  }
}
```

### 2. PaintChunksToCcLayer 转换

**文件**: [third_party/blink/renderer/platform/graphics/compositing/paint_chunks_to_cc_layer.cc](third_party/blink/renderer/platform/graphics/compositing/paint_chunks_to_cc_layer.cc)

```cpp
void PaintChunksToCcLayer::ConvertInto(
    const PaintChunkSubset& paint_chunks,
    const PropertyTreeState& layer_state,
    const gfx::Vector2dF& layer_offset,
    RasterUnderInvalidationCheckingParams* checking_params,
    cc::DisplayItemList& cc_display_items) {
  
  // 1. 创建转换上下文
  ConversionContext context(
      layer_state,
      layer_offset,
      checking_params);
  
  // 2. 遍历每个 PaintChunk
  for (const auto& chunk : paint_chunks) {
    // 3. 转换状态（如有必要）
    context.SwitchToChunkState(chunk.properties);
    
    // 4. 复制绘制项
    CopyDisplayItems(chunk, 
                    chunk_subset,
                    context,
                    cc_display_items);
  }
  
  // 5. 完成转换
  context.Finish(cc_display_items);
}

// 辅助函数：复制 DisplayItems
void CopyDisplayItems(
    const PaintChunk& chunk,
    const PaintChunkSubset& paint_chunks,
    ConversionContext& context,
    cc::DisplayItemList& cc_display_items) {
  
  // 获取该 chunk 中的所有项
  auto item_range = paint_chunks.GetDisplayItems(chunk);
  
  // 逐个复制
  for (const auto& item : item_range) {
    // 记录调试信息（如有必要）
    if (checking_params) {
      checking_params->tracking.AddChunk(
          chunk.id, 
          item.VisualRect());
    }
    
    // 复制 PaintOps
    CopyPaintOpsTo(item, cc_display_items);
  }
}
```

### 3. ContentLayerClientImpl 的实现

**文件**: [third_party/blink/renderer/platform/graphics/compositing/content_layer_client_impl.cc](third_party/blink/renderer/platform/graphics/compositing/content_layer_client_impl.cc)

```cpp
void ContentLayerClientImpl::UpdateCcPictureLayer(
    const PendingLayer& pending_layer) {
  
  // 1. 转换 PaintChunk 到 cc::DisplayItemList
  cc_display_item_list_ = base::MakeRefCounted<cc::DisplayItemList>();
  
  PaintChunksToCcLayer::ConvertInto(
      pending_layer.GetPaintChunkSubset(),
      pending_layer.GetPropertyTreeState(),
      pending_layer.GetLayerOffset(),
      raster_under_invalidation_checking_params,
      *cc_display_item_list_);
  
  // 2. 完成 DisplayItemList
  cc_display_item_list_->Finalize();
  
  // 3. 生成光栅失效
  raster_invalidator_->Generate(
      pending_layer.GetPaintChunkSubset(),
      pending_layer.GetLayerOffset(),
      cc_picture_layer_->bounds().size(),
      pending_layer.GetPropertyTreeState());
  
  // 4. 设置 Layer 属性
  PaintChunksToCcLayer::UpdateLayerProperties(
      *cc_picture_layer_,
      pending_layer.GetPropertyTreeState(),
      pending_layer.GetPaintChunkSubset(),
      layer_selection,
      false);
}
```

---

## Rasterization 示例

### 1. DisplayItemList Rasterization

**文件**: [cc/paint/display_item_list.cc](cc/paint/display_item_list.cc)

```cpp
void DisplayItemList::Raster(
    SkCanvas* canvas,
    ImageProvider* image_provider,
    const ScrollOffsetMap* raster_inducing_scroll_offsets) const {
  
  // 1. 创建播放参数
  PlaybackParams params(image_provider);
  params.raster_inducing_scroll_offsets = 
      raster_inducing_scroll_offsets;
  
  // 2. 调用另一个 Raster 重载
  Raster(canvas, params);
}

void DisplayItemList::Raster(
    SkCanvas* canvas,
    const PlaybackParams& params) const {
  
  // 1. 查询需要光栅化的操作
  std::vector<size_t> offsets = 
      OffsetsOfOpsToRaster(canvas);
  
  if (offsets.empty()) {
    return;  // 没有相交的操作
  }
  
  // 2. 播放这些操作到 canvas
  paint_op_buffer_.Playback(
      canvas,
      params,
      /*local_ctm=*/true,
      &offsets);  // 只播放这些偏移的操作
}

std::vector<size_t> DisplayItemList::OffsetsOfOpsToRaster(
    SkCanvas* canvas) const {
  
  // 1. 获取 canvas 的边界（通常是 tile bounds）
  SkRect canvas_bounds = SkRect::Make(canvas->getDeviceClipBounds());
  gfx::Rect query_rect = gfx::SkRectToRect(canvas_bounds);
  
  // 2. 使用 R-tree 查询相交的操作
  std::vector<size_t> offsets;
  rtree_.Search(query_rect, &offsets);
  
  return offsets;
}
```

### 2. Tile 光栅化流程

**文件**: [cc/raster/rasterizer.cc](cc/raster/rasterizer.cc)

```cpp
void Rasterizer::RasterizeAndFinalize(
    const RasterBuffer& raster_buffer,
    const RasterSource& raster_source,
    const gfx::Rect& raster_full_rect,
    const gfx::Rect& playback_rect,
    float scale,
    const RasterColorSpace& raster_color_space,
    bool include_step_into_analysis,
    RasterColorSpace::SkImageColorSpaceType type) {
  
  // 1. 创建 SkCanvas（对应 Tile 的像素缓冲）
  SkImageInfo image_info = SkImageInfo::Make(
      playback_rect.width(),
      playback_rect.height(),
      kN32_SkColorType,
      kPremul_SkAlphaType);
  
  auto canvas = raster_buffer.AcquireSkCanvas(image_info);
  
  // 2. 应用缩放
  canvas->scale(scale, scale);
  canvas->translate(-playback_rect.x(), -playback_rect.y());
  
  // 3. 播放光栅操作
  raster_source.PlaybackToCanvas(
      canvas.get(),
      raster_full_rect,
      playback_rect,
      scale,
      raster_color_space.color_space.get());
  
  // 4. 完成并上传到 GPU
  raster_buffer.ReleaseSkCanvas();
}
```

### 3. RasterInvalidator 生成失效区域

**文件**: [third_party/blink/renderer/platform/graphics/paint/raster_invalidator.cc](third_party/blink/renderer/platform/graphics/paint/raster_invalidator.cc)

```cpp
void RasterInvalidator::Generate(
    const PaintChunkSubset& paint_chunks,
    const gfx::Vector2dF& layer_offset,
    const gfx::Size& layer_bounds,
    const PropertyTreeState& layer_state) {
  
  // 1. 比较新旧 PaintChunks
  Vector<PaintChunkInfo> new_chunks_info;
  
  for (const auto& chunk : paint_chunks) {
    // 2. 尝试匹配到旧的 chunk
    wtf_size_t old_index = 
        MatchNewChunkToOldChunk(chunk, previous_index);
    
    // 3. 生成差异失效
    if (old_index != kNotFound) {
      const PaintChunk& old_chunk = GetOldChunk(old_index);
      IncrementallyInvalidateChunk(
          old_chunks_info[old_index],
          new_chunk_info,
          chunk.id.client_id);
    } else {
      // 新 chunk，全部失效
      callback_->InvalidateRect(new_chunk_info.bounds_in_layer);
    }
  }
  
  // 4. 保存新的 chunks 信息用于下次对比
  old_chunks_info_ = new_chunks_info;
}

void RasterInvalidator::IncrementallyInvalidateChunk(
    const PaintChunkInfo& old_chunk_info,
    const PaintChunkInfo& new_chunk_info,
    DisplayItemClientId client_id) {
  
  // 1. 检查属性是否改变
  if (ChunkPropertiesChanged(
          old_chunk_info, new_chunk_info)) {
    // 属性改变，整个 chunk 失效
    callback_->InvalidateRect(new_chunk_info.bounds_in_layer);
  } else if (old_chunk_info.bounds_in_layer != 
             new_chunk_info.bounds_in_layer) {
    // 边界改变，计算并集失效
    gfx::Rect combined = gfx::UnionRects(
        old_chunk_info.bounds_in_layer,
        new_chunk_info.bounds_in_layer);
    callback_->InvalidateRect(combined);
  } else {
    // 边界不变，不需要失效
  }
}
```

---

## 关键方法速查

### PaintController 常用方法

```cpp
class PaintController {
 public:
  // 开始新的 paint 循环
  void BeginPaint();
  
  // 为新项创建和追加
  template <typename DisplayItemClass, typename... Args>
  void CreateAndAppend(const DisplayItemClient& client, Args&&... args);
  
  // 尝试使用缓存项
  bool UseCachedItemIfPossible(
      const DisplayItemClient& client,
      DisplayItem::Type type);
  
  // 开始/结束子序列
  wtf_size_t BeginSubsequence(const DisplayItemClient&);
  void EndSubsequence(wtf_size_t subsequence_index);
  
  // 获取生成的 PaintArtifact
  const PaintArtifact& PaintArtifact() const;
  
  // 提交新项到 artifact
  void CommitNewDisplayItems();
  
  // 更新当前 paint chunk 属性
  void UpdateCurrentPaintChunkProperties(
      const PropertyTreeStateOrAlias&);
};
```

### cc::DisplayItemList 常用方法

```cpp
class DisplayItemList : public base::RefCountedThreadSafe<DisplayItemList> {
 public:
  // 开始记录新的绘制项
  void StartPaint();
  
  // 推送绘制操作（各种 Op 类型）
  template <typename T, typename... Args>
  size_t push(Args&&... args);
  
  // SaveLayer 操作的边界更新
  void UpdateSaveLayerBounds(size_t id, const SkRect& bounds);
  
  // 标记不成对操作的结束
  void EndPaintOfUnpaired(const gfx::Rect& visual_rect);
  
  // 标记成对操作（如 SaveLayer）的开始/结束
  void EndPaintOfPairedBegin();
  void EndPaintOfPairedEnd();
  
  // 完成记录
  void Finalize();
  
  // 光栅化到 canvas
  void Raster(
      SkCanvas* canvas,
      ImageProvider* image_provider = nullptr,
      const ScrollOffsetMap* raster_inducing_scroll_offsets = nullptr) const;
  
  // 查询需要光栅化的操作偏移
  std::vector<size_t> OffsetsOfOpsToRaster(SkCanvas* canvas) const;
};
```

### RasterInvalidator 常用方法

```cpp
class RasterInvalidator : public GarbageCollected<RasterInvalidator> {
 public:
  // 生成光栅失效区域
  void Generate(
      const PaintChunkSubset& chunks,
      const gfx::Vector2dF& layer_offset,
      const gfx::Size& layer_bounds,
      const PropertyTreeState& layer_state);
  
  // 更新滚动相关的 chunk 信息
  void UpdateForRasterInducingScroll(
      const PaintChunkSubset& chunks);
  
  // 设置旧的 PaintArtifact
  void SetOldPaintArtifact(const PaintArtifact&);
  
  // 启用光栅失效追踪
  void SetTracksRasterInvalidations(bool);
  
  // 获取光栅失效追踪信息
  RasterInvalidationTracking* GetTracking() const;
  
  // 清除旧状态
  void ClearOldStates();
};
```

---

## 调试技巧

### 1. 启用 Paint 调试标志

```bash
# 启用 paint 的逐阶段输出
gfx.prefers_reduced_motion = false
blink.enable_paint_invalidation_annotations = true
```

### 2. 在代码中添加调试输出

```cpp
// 打印 PaintArtifact 的结构
void DebugPaintArtifact(const PaintArtifact& artifact) {
  DLOG(INFO) << "PaintArtifact with " 
             << artifact.GetPaintChunks().size() 
             << " chunks";
  
  for (size_t i = 0; i < artifact.GetPaintChunks().size(); ++i) {
    const auto& chunk = artifact.GetPaintChunks()[i];
    DLOG(INFO) << "  Chunk " << i 
               << ": items [" << chunk.begin_index 
               << ", " << chunk.end_index 
               << ") bounds=" << chunk.bounds;
  }
}

// 追踪 DisplayItem 创建
void DebugDisplayItem(const DisplayItem& item) {
  DLOG(INFO) << "DisplayItem type=" << static_cast<int>(item.GetType())
             << " client=" << item.GetId().client_id;
}
```

### 3. Chrome DevTools 中查看 Paint 性能

1. 打开 DevTools → Performance 标签
2. 记录页面操作
3. 查看 Paint 事件的耗时
4. 点击 Paint 事件查看涉及的区域和元素

### 4. 使用 about:tracing 追踪详细信息

```
chrome://tracing
  ↓ 点击 Record
  ↓ 执行操作
  ↓ 停止录制
  ↓ 搜索 Paint、Composite、Raster 事件
```

### 5. 在 Blink 中启用光栅失效追踪

```cpp
// 在 RasterInvalidator 创建时
raster_invalidator_->SetTracksRasterInvalidations(true);

// 后续可以访问追踪信息
auto* tracking = raster_invalidator_->GetTracking();
if (tracking) {
  // 遍历所有被失效的矩形
  for (const auto& invalidation : tracking->GetInvalidations()) {
    DLOG(INFO) << "Invalidation: " << invalidation.rect 
               << " reason=" << invalidation.reason;
  }
}
```

### 6. 关键断点位置

| 位置 | 文件 | 用途 |
|---|---|---|
| `PaintController::CommitNewDisplayItems()` | `paint_controller.cc` | 观察 PaintArtifact 生成 |
| `PaintArtifactCompositor::Update()` | `paint_artifact_compositor.cc` | 观察合成决策 |
| `PaintChunksToCcLayer::ConvertInto()` | `paint_chunks_to_cc_layer.cc` | 观察 Blink→cc 转换 |
| `RasterInvalidator::Generate()` | `raster_invalidator.cc` | 观察失效追踪 |
| `DisplayItemList::Raster()` | `cc/paint/display_item_list.cc` | 观察光栅化执行 |

---

## 性能分析实例

### 问题：Paint 时间过长

**诊断步骤**：

```cpp
// 1. 启用 Paint 计时
TRACE_EVENT0("blink.paint", "LayoutObject::Paint");

// 2. 检查是否有不必要的 Paint 调用
if (!ShouldPaint()) {
  // 跳过不可见或不需要的元素
  return;
}

// 3. 检查 cull rect 是否有效
if (!fragment_rect.Intersects(paint_info.cull_rect)) {
  // 跳过不在视口内的内容
  return;
}

// 4. 使用缓存的 DisplayItems
if (paint_controller.UseCachedItemIfPossible(this, type)) {
  return;  // 成功使用缓存
}

// 5. 只有必要时才创建新的 DisplayItem
paint_controller.CreateAndAppend<DrawingDisplayItem>(...);
```

### 问题：Rasterization 耗时

**诊断步骤**：

```cpp
// 1. 检查 R-tree 查询效率
std::vector<size_t> offsets = 
    display_item_list->OffsetsOfOpsToRaster(canvas);
DLOG(INFO) << "Total ops: " << display_item_list->TotalOpCount()
           << " Need to raster: " << offsets.size();

// 2. 检查 Tile 大小是否合理
DLOG(INFO) << "Tile size: " << tile_size
           << " Layer size: " << layer_bounds;

// 3. 监控光栅化失效的频率
if (raster_invalidator->GetTracking()) {
  DLOG(INFO) << "Invalidations in last frame: " 
             << raster_invalidator->GetTracking()->GetInvalidations().size();
}
```

---

## 常见代码模式

### 模式 1：添加新的 DisplayItem 类型

```cpp
// 1. 在 display_item.h 中定义类型
enum Type {
  // ...
  kMyCustomDrawing,  // 新类型
  // ...
};

// 2. 创建 DisplayItem 子类
class MyCustomDisplayItem : public DisplayItem {
 public:
  MyCustomDisplayItem(const DisplayItemClient& client, 
                      MyData* data)
      : DisplayItem(client, kMyCustomDrawing, sizeof(*this)),
        data_(data) {}
  
  MyData* GetData() const { return data_; }
  
 private:
  raw_ptr<MyData> data_;
};

// 3. 在 Painter 中使用
void MyPainter::Paint(const PaintInfo& paint_info) {
  paint_controller->CreateAndAppend<MyCustomDisplayItem>(
      *this, my_data);
}
```

### 模式 2：条件性 Invalidation

```cpp
void RasterInvalidator::IncrementallyInvalidateChunk(
    const PaintChunkInfo& old_chunk,
    const PaintChunkInfo& new_chunk) {
  
  // 只在真正改变时失效
  if (old_chunk.bounds_in_layer != new_chunk.bounds_in_layer) {
    gfx::Rect combined = gfx::UnionRects(
        old_chunk.bounds_in_layer,
        new_chunk.bounds_in_layer);
    callback_->InvalidateRect(combined);
  }
}
```

### 模式 3：批量 Paint 操作

```cpp
void BlockPainter::PaintContents(
    const PaintInfo& paint_info,
    const LayoutPoint& paint_offset) {
  
  // 使用 subsequence 优化多个项
  wtf_size_t subsequence_index = 
      paint_controller->BeginSubsequence(*this);
  
  // 批量绘制子元素
  for (LayoutBox* child = box_.FirstChildBox(); child;
       child = child->NextSiblingBox()) {
    PaintChild(*child, paint_info, paint_offset);
  }
  
  paint_controller->EndSubsequence(subsequence_index);
}
```

