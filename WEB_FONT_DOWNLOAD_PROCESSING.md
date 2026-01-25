# Web 字体下载完成后的处理流程

## 1. 高层流程图

```
字体下载完成
    ↓
  [Resource 加载完成回调]
    ↓
RemoteFontFaceSource::NotifyFinished()
    ↓
┌─────────────────────────────────────────────┐
│ 1. 检查文档状态和完整性                       │
│    - 检查是否分离/销毁                       │
│    - SRI 完整性检查                         │
└─────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────┐
│ 2. 获取字体数据并解码                        │
│    - FontResource::GetCustomFontData()      │
│    - 后台解码或主线程解码                    │
│    - OTS 验证                               │
└─────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────┐
│ 3. 处理错误（如有）                         │
│    - 解码错误 → 输出警告信息                 │
│    - OTS 解析错误 → 输出详细信息             │
└─────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────┐
│ 4. 更新字体加载状态                         │
│    - CSSFontFace::FontLoaded()             │
│    - 设置 LoadStatus = Loaded 或 Error     │
└─────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────┐
│ 5. 通知字体选择器和渲染引擎                  │
│    - FontFaceInvalidated()                  │
│    - 触发重新计算样式和布局                  │
│    - 重新渲染文本                           │
└─────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────┐
│ 6. 发送调试/监控信息                        │
│    - probe::FontsUpdated()                  │
│    - 记录性能指标                           │
└─────────────────────────────────────────────┘
```

## 2. 详细处理步骤

### 步骤 1: 文档状态检查

**位置**: [remote_font_face_source.cc#L180-L193](remote_font_face_source.cc#L180-L193)

```cpp
void RemoteFontFaceSource::NotifyFinished(Resource* resource) {
  ExecutionContext* execution_context = font_selector_->GetExecutionContext();
  if (!execution_context) {
    return;
  }
  DCHECK(execution_context->IsContextThread());
  // 防止在文档销毁时返回 promise reject
  auto* window = DynamicTo<LocalDOMWindow>(execution_context);
  if (window && window->document()->IsDetached()) {
    return;  // 文档已分离，不处理
  }
  
  auto* font = To<FontResource>(resource);
  histograms_.RecordRemoteFont(font);  // 记录统计信息
}
```

**作用**: 确保在有效的文档环境中处理字体

---

### 步骤 2: 完整性验证 (SRI)

**位置**: [remote_font_face_source.cc#L194-L207](remote_font_face_source.cc#L194-L207)

```cpp
bool force_integrity_checks = resource->ForceIntegrityChecks();
if (force_integrity_checks) {
  resource->IntegrityReport().SendReports(execution_context);
}

// 如果 SRI 检查失败，不更新 custom_font_data_（保持 nullptr）
DCHECK(!custom_font_data_);
if (resource->PassedIntegrityChecks() || !force_integrity_checks) {
  custom_font_data_ = font->GetCustomFontData();  // ← 获取并解码字体
}
url_ = resource->Url().GetString();
```

**作用**: 
- 验证 `integrity` 属性（如果有）
- 如果 SRI 校验失败，视为网络错误

---

### 步骤 3: 字体数据获取与解码

**位置**: [font_resource.cc#L367-L401](font_resource.cc#L367-L401)

#### 主线程解码 (同步)

```cpp
const FontCustomPlatformData* FontResource::GetCustomFontData() {
  if (font_data_ || ErrorOccurred() || IsLoading()) {
    return font_data_;
  }
  
  if (Data()) {
    if (background_decode_result_or_error_) {
      // 使用后台解码的结果
      if (background_decode_result_or_error_->has_value()) {
        font_data_ = FontCustomPlatformData::Create(
            std::move((*background_decode_result_or_error_)->sk_typeface),
            (*background_decode_result_or_error_)->decoded_size);
      } else {
        ots_parsing_message_ = background_decode_result_or_error_->error();
      }
    } else {
      // 主线程解码
      auto decode_start_time = base::TimeTicks::Now();
      font_data_ = FontCustomPlatformData::Create(Data(), ots_parsing_message_);
      base::UmaHistogramMicrosecondsTimes(
          "Blink.Fonts.DecodeTime", 
          base::TimeTicks::Now() - decode_start_time);
    }
  }
  
  if (!font_data_) {
    SetStatus(ResourceStatus::kDecodeError);  // 标记为解码错误
  }
  return font_data_;
}
```

#### 后台解码 (异步)

**位置**: [font_resource.cc#L200-L246](font_resource.cc#L200-L246)

```cpp
class FontResource::BackgroundFontProcessor {
  void OnDataComplete() {
    // 在后台线程解码
    PostCrossThreadTask(
        *GetFontDecodingTaskRunner(), FROM_HERE,
        CrossThreadBindOnce(
            &FontResource::BackgroundFontProcessor::DecodeOnBackgroundThread,
            std::move(buffer_), background_task_runner_,
            weak_factory_.GetWeakPtr()));
  }

  static void DecodeOnBackgroundThread(
      SegmentedBuffer data,
      scoped_refptr<base::SequencedTaskRunner> background_task_runner,
      base::WeakPtr<BackgroundFontProcessor> weak_this) {
    // 在工作线程/字体线程上解码
    base::expected<DecodedResult, String> result_or_error = DecodeFont(&data);
    
    // 结果回传到主线程
    PostCrossThreadTask(
        *background_task_runner, FROM_HERE,
        CrossThreadBindOnce(
            &FontResource::BackgroundFontProcessor::OnDecodeComplete,
            std::move(weak_this), std::move(result_or_error), std::move(data)));
  }
  
  void OnDecodeComplete(
      base::expected<DecodedResult, String> result_or_error,
      SegmentedBuffer data) {
    // 回到主线程处理结果
    client_->PostTaskToMainThread(CrossThreadBindOnce(
        &FontResource::OnBackgroundDecodeFinished,
        MakeUnwrappingCrossThreadWeakHandle(std::move(resource_handle_)),
        std::move(result_or_error)));
  }
};
```

**解码器**: [font_resource.cc#L77-L88](font_resource.cc#L77-L88)

```cpp
base::expected<FontResource::DecodedResult, String> DecodeFont(
    SegmentedBuffer* buffer) {
  if (buffer->empty()) {
    return base::unexpected("");
  }
  WebFontDecoder decoder;  // OTS 验证器
  auto decode_start_time = base::TimeTicks::Now();
  sk_sp<SkTypeface> typeface = decoder.Decode(buffer);  // 解码为 Skia typeface
  base::UmaHistogramMicrosecondsTimes(
      "Blink.Fonts.BackgroundDecodeTime",
      base::TimeTicks::Now() - decode_start_time);
  
  if (typeface) {
    return FontResource::DecodedResult(std::move(typeface),
                                       decoder.DecodedSize());
  }
  return base::unexpected(decoder.GetErrorString());  // OTS 验证失败
}
```

**作用**:
- 将二进制字体数据转换为可用的 SkTypeface（Skia 字体对象）
- 执行 OTS（OpenType Sanitizer）安全验证
- 记录解码耗时

---

### 步骤 4: 错误处理

**位置**: [remote_font_face_source.cc#L208-L220](remote_font_face_source.cc#L208-L220)

```cpp
if (font->GetStatus() == ResourceStatus::kDecodeError) {
  // 输出解码错误警告
  execution_context->AddConsoleMessage(MakeGarbageCollected<ConsoleMessage>(
      mojom::ConsoleMessageSource::kOther,
      mojom::ConsoleMessageLevel::kWarning,
      StrCat({"Failed to decode downloaded font: ",
              font->Url().ElidedString()})));
  
  // 输出 OTS 详细错误信息
  if (!font->OtsParsingMessage().empty()) {
    execution_context->AddConsoleMessage(MakeGarbageCollected<ConsoleMessage>(
        mojom::ConsoleMessageSource::kOther,
        mojom::ConsoleMessageLevel::kWarning,
        StrCat({"OTS parsing error: ", font->OtsParsingMessage()})));
  }
}
```

**输出示例**:
```
Failed to decode downloaded font: https://fonts.gstatic.com/s/roboto/v29/KFOmCnqEu92Fr1Mu4mxK.woff2
OTS parsing error: Sanitizer rejected font file
```

---

### 步骤 5: 状态更新

**位置**: [remote_font_face_source.cc#L221-L248](remote_font_face_source.cc#L221-L248)

```cpp
ClearResource();  // 释放网络资源
ClearTable();     // 清除字体缓存表

if (GetDocument()) {
  // 记录字体完成的时机
  if (!GetDocument()->RenderingHasBegun()) {
    finished_before_document_rendering_begin_ = true;
  }
  if (!FontFaceSetDocument::From(*GetDocument())->HasReachedLCPLimit()) {
    finished_before_lcp_limit_ = true;
  }
}

if (FinishedFromMemoryCache()) {
  period_ = kNotApplicablePeriod;  // 从缓存完成，无需等待期
} else {
  UpdatePeriod();  // 更新 font-display 周期
}

// 调用 CSSFontFace::FontLoaded()
if (face_->FontLoaded(this)) {
  font_selector_->FontFaceInvalidated(
      FontInvalidationReason::kFontFaceLoaded);
  
  if (custom_font_data_) {
    probe::FontsUpdated(execution_context, face_->GetFontFace(),
                        resource->Url().GetString(), custom_font_data_.Get());
  }
}
```

---

### 步骤 6: CSSFontFace 的反应

**位置**: [css_font_face.cc#L70-L99](css_font_face.cc#L70-L99)

```cpp
bool CSSFontFace::FontLoaded(CSSFontFaceSource* source) {
  if (!IsValid() || source != sources_.front()) {
    return false;
  }

  if (LoadStatus() == FontFace::kLoading) {
    if (source->IsValid()) {
      // 字体有效 → 标记为加载完成
      SetLoadStatus(FontFace::kLoaded);
    } else if (source->IsInFailurePeriod()) {
      // 在失败期内 → 标记为错误，停止加载
      sources_.clear();
      SetLoadStatus(FontFace::kError);
    } else {
      // 继续尝试下一个 source
      sources_.pop_front();
      Load();  // 递归加载下一个源
    }
  }

  // 通知所有使用该字体的分段字体面
  for (CSSSegmentedFontFace* segmented_font_face : segmented_font_faces_) {
    segmented_font_face->FontFaceInvalidated();
  }

  // 记录性能数据
  const FontCustomPlatformData* platform_data = source->GetCustomPlaftormData();
  if (LoadStatus() == FontFace::kLoaded && platform_data) {
    TRACE_EVENT("devtools.timeline", "RemoteFontLoaded", "url",
                source->GetURL(), "name",
                platform_data->GetPostScriptNameOrFamilyNameForInspector());
  }

  return true;
}
```

---

### 步骤 7: 样式和布局重新计算

**触发链**:

```
font_selector_->FontFaceInvalidated(FontInvalidationReason::kFontFaceLoaded)
    ↓
CSSFontSelector::FontFaceInvalidated()
    ↓
RuleSets 更新
    ↓
Document::UpdateStyleAndLayout()
    ↓
Style Recalculation (样式重新计算)
    ↓
Layout Recalculation (布局重新计算)
    ↓
Paint Invalidation (绘制失效)
    ↓
Repaint (重绘)
```

---

## 3. 超时控制 (font-display 周期)

**位置**: [font_resource.cc#L78-L84](font_resource.cc#L78-L84)

```cpp
constexpr base::TimeDelta kFontLoadWaitShort = base::Milliseconds(100);
constexpr base::TimeDelta kFontLoadWaitLong = base::Milliseconds(3000);
```

**超时回调**:

- **100ms 超时**: `FontLoadShortLimitCallback()` → 进入 "short limit exceeded" 状态
- **3s 超时**: `FontLoadLongLimitCallback()` → 进入 "long limit exceeded" 状态

根据 `font-display` 值：
- `block`: 在短期内阻塞，然后显示 fallback
- `swap`: 立即显示 fallback，等待字体完成
- `fallback`: 100ms 后显示 fallback，3s 后放弃
- `optional`: 如果超过超时时间，可能完全不使用

---

## 4. 性能优化

### 后台解码线程

- **优点**: 避免阻塞主线程
- **适用**: 大型字体文件
- **控制**: `kBackgroundFontResponseProcessor` feature flag

### 内存缓存优化

- 从内存缓存加载时，直接进入 "swapped" 状态
- 不需要等待 100ms 或 3s 的超时

### LCP 考虑

- 字体加载会影响 Largest Contentful Paint (LCP)
- 如果字体加载过慢，字体选择器会改变行为以避免 LCP 恶化

---

## 5. 关键时间点记录

```cpp
finished_before_document_rendering_begin_  // 在文档开始渲染前完成
finished_before_lcp_limit_                 // 在 LCP 窗口期内完成
FinishedFromMemoryCache()                  // 是否从内存缓存加载
```

这些信息用于性能分析和优化决策。

---

## 总结

字体下载完成后的处理流程：

1. ✅ **验证**: 检查文档、SRI 完整性
2. 🔧 **解码**: 主线程或后台线程将二进制转换为 Skia typeface
3. ⚠️ **错误处理**: OTS 验证，输出错误信息
4. 📊 **状态更新**: 从 Loading → Loaded/Error
5. 🔔 **通知**: 字体选择器和渲染引擎
6. 🎨 **重绘**: 触发样式/布局/绘制重新计算
7. 📈 **监控**: 记录性能指标和调试信息
