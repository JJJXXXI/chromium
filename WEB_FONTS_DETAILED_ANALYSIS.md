# Web 字体与自定义字体的详细代码分析

> **目的**: 补充 Android_Font_Rendering_Code_Paths.md 中关于 Web 字体和自定义字体的详细实现，基于实际代码而非假设。

---

## 1. FontFace 类的完整分析

### 1.1 FontFace 构造与初始化

**文件**: `third_party/blink/renderer/core/css/font_face.h/cc`

```cpp
// font_face.h - 类定义
enum FontLoadStatus : uint8_t { 
  kUnloaded,  // 初始状态：规则存在但未加载
  kLoading,   // 进行中：网络请求或解码中
  kLoaded,    // 完成：字体可用
  kError      // 失败：网络错误、格式错误等
};

class FontFace : public ScriptWrappable,
                 public ExecutionContextClient {
 private:
  // 字体描述符
  AtomicString family_;           // 字体家族名（如 "MyCustomFont"）
  Member<const CSSValue> style_;  // font-style 描述符
  Member<const CSSValue> weight_; // font-weight 描述符
  Member<const CSSValue> stretch_;// font-stretch 描述符
  Member<const CSSValue> unicode_range_;  // unicode-range 描述符
  Member<const CSSValue> variant_;        // font-variant 描述符
  Member<const CSSValue> feature_settings_; // font-feature-settings
  Member<const CSSValue> variation_settings_; // font-variation-settings
  Member<const CSSValue> display_; // font-display 描述符
  
  // 字体数据
  Member<CSSFontFace> css_font_face_;  // 实际处理加载逻辑的对象
  Member<DOMException> error_;  // 加载失败时的错误对象
  
  // 加载状态
  LoadStatusType status_;  // 当前加载状态
  
  // Promise 相关
  Member<LoadedProperty> loaded_property_;  // Promise<FontFace>
  HeapVector<Member<LoadFontCallback>> callbacks_;  // 加载完成回调
  
  // 来源追踪
  CascadeLayered<const StyleRuleFontFace> style_rule_;
  bool is_user_style_;  // 用户样式表中的字体
};
```

### 1.2 创建 FontFace 对象的三种方式

#### 方式 1: 从 CSS @font-face 规则

```cpp
// font_face.cc 行 ~149
FontFace* FontFace::Create(
    Document* document,
    const CascadeLayered<const StyleRuleFontFace>& layered_font_face_rule,
    bool is_user_style) {
  
  const StyleRuleFontFace* font_face_rule = layered_font_face_rule.value;
  const CSSPropertyValueSet& properties = font_face_rule->Properties();

  // ① 提取 font-family 属性（必需）
  auto* family = DynamicTo<CSSFontFamilyValue>(
      properties.GetPropertyCSSValue(AtRuleDescriptorID::FontFamily));
  if (!family) {
    return nullptr;  // font-family 缺失，规则无效
  }
  
  // ② 提取 src 属性（必需）
  const CSSValue* src = properties.GetPropertyCSSValue(AtRuleDescriptorID::Src);
  if (!src || !src->IsValueList()) {
    return nullptr;  // src 缺失或无效，规则无效
  }

  // ③ 创建 FontFace 对象
  FontFace* font_face = MakeGarbageCollected<FontFace>(
      document->GetExecutionContext(), 
      layered_font_face_rule,      // 保留原始 CSS 规则
      is_user_style);              // 是否来自用户样式表
  
  font_face->SetFamilyValue(*family);  // 设置字体家族

  // ④ 解析并设置所有描述符
  if (font_face->SetPropertyFromStyle(properties, AtRuleDescriptorID::FontStyle) &&
      font_face->SetPropertyFromStyle(properties, AtRuleDescriptorID::FontWeight) &&
      font_face->SetPropertyFromStyle(properties, AtRuleDescriptorID::FontStretch) &&
      // ... 其他描述符 ...
      font_face->GetFontSelectionCapabilities().IsValid()) {
    
    // ⑤ 初始化内部的 CSSFontFace
    font_face->InitCSSFontFace(document->GetExecutionContext(), *src);
    return font_face;
  }
  return nullptr;  // 验证失败
}
```

**关键流程**:
1. 解析 CSS 规则中的所有描述符
2. 验证 `font-family` 和 `src` 必需属性
3. 验证所有属性值有效
4. 创建 FontFace 对象并初始化 CSSFontFace

#### 方式 2: 从 JavaScript API（new FontFace()）

```cpp
// font_face.cc 行 ~117
FontFace* FontFace::Create(
    ExecutionContext* execution_context,
    const AtomicString& family,
    const V8UnionArrayBufferOrArrayBufferViewOrString* source,
    const FontFaceDescriptors* descriptors) {
  
  // ① 处理 source 参数的三种类型
  switch (source->GetContentType()) {
    case V8UnionArrayBufferOrArrayBufferViewOrString::ContentType::kArrayBuffer:
      // source 是 ArrayBuffer（二进制数据）
      return Create(execution_context, family,
                    source->GetAsArrayBuffer()->ByteSpan(), descriptors);
    
    case V8UnionArrayBufferOrArrayBufferViewOrString::ContentType::kArrayBufferView:
      // source 是 TypedArray（如 Uint8Array）
      return Create(execution_context, family,
                    source->GetAsArrayBufferView()->ByteSpan(), descriptors);
    
    case V8UnionArrayBufferOrArrayBufferViewOrString::ContentType::kString:
      // source 是字符串（URL 或 data: URI）
      return Create(execution_context, family, 
                    source->GetAsString(), descriptors);
  }
}

// 字符串情况：URL 或 data: URI
FontFace* FontFace::Create(ExecutionContext* context,
                           const AtomicString& family,
                           const String& source,
                           const FontFaceDescriptors* descriptors) {
  FontFace* font_face =
      MakeGarbageCollected<FontFace>(context, family, descriptors);
  
  // 解析 src 属性
  const CSSValue* src = ParseCSSValue(context, source, AtRuleDescriptorID::Src);
  if (!src || !src->IsValueList()) {
    // 格式错误
    font_face->SetError(MakeGarbageCollected<DOMException>(
        DOMExceptionCode::kSyntaxError,
        StrCat({"The source provided ('", source,
                "') could not be parsed as a value list."})));
  }

  font_face->InitCSSFontFace(context, *src);
  return font_face;
}

// 二进制数据情况：ArrayBuffer/TypedArray
FontFace* FontFace::Create(ExecutionContext* context,
                           const AtomicString& family,
                           base::span<const uint8_t> data,
                           const FontFaceDescriptors* descriptors) {
  FontFace* font_face =
      MakeGarbageCollected<FontFace>(context, family, descriptors);
  
  // 直接初始化为二进制数据
  font_face->InitCSSFontFace(context, data);
  return font_face;
}
```

**三种 source 类型**:
1. **ArrayBuffer/TypedArray**: 字体文件的原始二进制数据
   - 无网络 I/O
   - 立即解码（可能导致 jank）
2. **URL string**: 字体文件的远程 URL
   - 触发网络请求
   - 异步加载（非阻塞）
3. **data: URI**: 内联字体数据（base64 编码）
   - 无网络 I/O
   - 先解码 base64，再解析字体

#### 方式 3: 描述符对象

```cpp
// font_face.cc 行 ~198
FontFace::FontFace(ExecutionContext* context,
                   const AtomicString& family,
                   const FontFaceDescriptors* descriptors)
    : ActiveScriptWrappable<FontFace>({}),
      ExecutionContextClient(context),
      family_(family),
      status_(kUnloaded) {
  
  // 从 FontFaceDescriptors IDL 对象解析所有描述符
  SetPropertyFromString(context, descriptors->style(),
                        AtRuleDescriptorID::FontStyle);
  SetPropertyFromString(context, descriptors->weight(),
                        AtRuleDescriptorID::FontWeight);
  SetPropertyFromString(context, descriptors->stretch(),
                        AtRuleDescriptorID::FontStretch);
  SetPropertyFromString(context, descriptors->unicodeRange(),
                        AtRuleDescriptorID::UnicodeRange);
  SetPropertyFromString(context, descriptors->variant(),
                        AtRuleDescriptorID::FontVariant);
  SetPropertyFromString(context, descriptors->featureSettings(),
                        AtRuleDescriptorID::FontFeatureSettings);
  SetPropertyFromString(context, descriptors->variationSettings(),
                        AtRuleDescriptorID::FontVariationSettings);
  SetPropertyFromString(context, descriptors->display(),
                        AtRuleDescriptorID::FontDisplay);
  SetPropertyFromString(context, descriptors->ascentOverride(),
                        AtRuleDescriptorID::AscentOverride);
  // ... 等等 ...
}

// JavaScript API 用法
const fontFace = new FontFace('MyFont', 
  new ArrayBuffer(...),  // 或 "url(...)"
  {
    style: 'normal',
    weight: '400',
    stretch: 'normal',
    // 等等...
  }
);
```

---

## 2. CSSFontFace 与 CSSFontFaceSource 系统

### 2.1 CSSFontFace 的架构

**文件**: `third_party/blink/renderer/core/css/css_font_face.h/cc`

```cpp
// css_font_face.h
class CSSFontFace final : public GarbageCollected<CSSFontFace> {
 public:
  // 源列表（优先级顺序）
  HeapDeque<Member<CSSFontFaceSource>> sources_;
  
  // Unicode 范围（用于字符选择）
  Member<const UnicodeRangeSet> ranges_;
  
  // 反向引用
  Member<FontFace> font_face_;
  
  // 关联的分段字体
  HeapHashSet<Member<CSSSegmentedFontFace>> segmented_font_faces_;
  
  // 核心操作
  void AddSource(CSSFontFaceSource*);  // 添加源（优先级顺序）
  bool FontLoaded(CSSFontFaceSource*); // 源加载完成回调
  bool MaybeLoadFont(const FontDescription&, const String&); // 按需加载
  void Load();  // 手动触发所有源加载
};
```

### 2.2 CSSFontFaceSource 的继承层级

```
CSSFontFaceSource (抽象基类)
  ├─ RemoteFontFaceSource (网络字体)
  │   └─ 处理 URL 加载、超时、缓存
  ├─ BinaryDataFontFaceSource (二进制数据)
  │   └─ 处理 ArrayBuffer/data:URI
  └─ LocalFontFaceSource (本地系统字体)
      └─ 处理 local() 字体引用
```

### 2.3 RemoteFontFaceSource 的完整生命周期

**文件**: `third_party/blink/renderer/core/css/remote_font_face_source.h/cc`

```cpp
class RemoteFontFaceSource final : public CSSFontFaceSource,
                                   public FontResourceClient {
 private:
  // 字体显示阶段
  enum DisplayPeriod : uint8_t {
    kBlockPeriod,        // 隐藏，等待加载（通常 0-3s）
    kSwapPeriod,         // 显示回退，尽快加载（通常 3-10s）
    kFailurePeriod,      // 使用回退，加载失败
    kNotApplicablePeriod // 内存缓存（无需等待）
  };
  
  // font-display CSS 属性的实现
  FontDisplay display_;  // auto, block, swap, fallback, optional
  DisplayPeriod period_;  // 当前阶段
  
  // 超时管理
  enum Phase : uint8_t {
    kNoLimitExceeded,
    kShortLimitExceeded,    // 3 秒超时
    kLongLimitExceeded      // 10 秒超时
  };
  Phase phase_;
  
  // 资源
  Member<FontResource> font_resource_;  // 网络资源
  Member<FontCustomPlatformData> custom_font_data_;  // 解码后的字体
  
  // 直方图统计
  FontLoadHistograms histograms_;  // UMA 上报
};

// 生命周期回调
class FontLoadHistograms {
 public:
  enum DataSource {
    kFromUnknown,
    kFromDataURL,       // data: URI
    kFromMemoryCache,   // 内存缓存
    kFromDiskCache,     // 磁盘缓存
    kFromNetwork        // 网络加载
  };
  
  void LoadStarted();                      // 开始加载
  void FallbackFontPainted(DisplayPeriod); // 绘制回退字体
  void LongLimitExceeded();                // 10 秒超时
  void RecordFallbackTime();               // 记录回退时间
  void RecordRemoteFont(const FontResource*);  // 记录字体加载
};
```

**加载流程**:

```cpp
// remote_font_face_source.cc
void RemoteFontFaceSource::BeginLoadIfNeeded() {
  if (IsLoading()) {
    return;  // 已在加载中
  }
  
  // 构建 FetchParameters
  ResourceRequest request(url_);
  request.SetFetchCredentialsMode(
      network::mojom::CredentialsMode::kSameOrigin);
  
  FetchParameters params(request);
  params.SetOriginRestricted(true);  // 跨域检查
  
  // 发起网络请求
  font_resource_ = FontResource::Fetch(
      params,
      font_selector_->GetResourceFetcher(),
      this);  // this 作为 FontResourceClient
  
  // 记录开始时间
  histograms_.LoadStarted();
  
  // 启动超时计时器（3s 和 10s）
  StartLoadLimitTimers();
}

// 网络请求完成回调
void RemoteFontFaceSource::NotifyFinished(Resource* resource) {
  // 检查网络状态
  if (font_resource_->LoadFailedOrCanceled()) {
    SetError(font_resource_->ErrorDetails());
    return;
  }
  
  // 检查 CORS
  if (!font_resource_->PassesAccessControlCheck()) {
    SetError("CORS error");
    return;
  }
  
  // 获取字体数据
  SharedBuffer* data = font_resource_->Data();
  if (!data || data->size() == 0) {
    SetError("Empty font data");
    return;
  }
  
  // 解码字体格式（WOFF2/WOFF/TTF/OTF）
  String ots_message;
  custom_font_data_ = FontCustomPlatformData::Create(
      data,
      ots_message);  // OTS 解析器输出消息
  
  if (!custom_font_data_) {
    SetError(ots_message);  // 格式错误
    return;
  }
  
  // 成功
  SetLoadStatus(FontFace::kLoaded);
}

// 3 秒超时回调
void RemoteFontFaceSource::FontLoadShortLimitExceeded() {
  phase_ = kShortLimitExceeded;
  
  // 更新显示阶段
  if (display_ == FontDisplay::kAuto) {
    period_ = kSwapPeriod;  // 从 block → swap
  }
  
  // 触发页面重排（使用回退字体）
  NotifySegmentedFontFaces();
}

// 10 秒超时回调
void RemoteFontFaceSource::FontLoadLongLimitExceeded() {
  phase_ = kLongLimitExceeded;
  
  // 更新显示阶段（最终）
  if (display_ == FontDisplay::kAuto ||
      display_ == FontDisplay::kBlock) {
    period_ = kFailurePeriod;  // 进入失败阶段
  }
  
  histograms_.LongLimitExceeded();
  NotifySegmentedFontFaces();  // 最后一次重排
}
```

### 2.4 font-display 描述符的行为

```
font-display: auto
  Block Period: 0-3s（隐藏文本）
  Swap Period:  3-10s（显示回退）
  Failure Period: 10s+（使用回退）

font-display: block
  Block Period: 0-∞（等待完成）
  Swap Period: ∞（从不交换）
  Failure Period: ∞（永不失败）

font-display: swap
  Block Period: 0s（立即显示回退）
  Swap Period: ∞（字体加载立即交换）
  Failure Period: ∞（永不失败）

font-display: fallback
  Block Period: 0.1s（极短隐藏）
  Swap Period: ∞（3-10s 内可交换）
  Failure Period: 10s+（使用回退）

font-display: optional
  Block Period: 0.1s（极短隐藏）
  Swap Period: 0s（不交换）
  Failure Period: 0s（立即使用回退）
```

---

## 3. BinaryDataFontFaceSource - 直接字体数据

**文件**: `third_party/blink/renderer/core/css/binary_data_font_face_source.h/cc`

```cpp
class BinaryDataFontFaceSource final : public CSSFontFaceSource {
 private:
  Member<const FontCustomPlatformData> custom_platform_data_;
};

// 构造函数
BinaryDataFontFaceSource::BinaryDataFontFaceSource(
    CSSFontFace* css_font_face,
    SharedBuffer* data,
    String& format) {
  
  // 直接解码二进制数据
  String ots_message;
  custom_platform_data_ = FontCustomPlatformData::Create(
      data,
      ots_message);
  
  if (!custom_platform_data_) {
    // 解码失败
    SetError(ots_message);
    return;
  }
  
  // 设置为已加载（无网络等待）
  SetLoadStatus(FontFace::kLoaded);
}

// 字体数据源
const SimpleFontData* BinaryDataFontFaceSource::CreateFontData(
    const FontDescription& font_description,
    const FontSelectionCapabilities&) {
  
  if (!IsValid()) {
    return nullptr;
  }
  
  // 从已解码的平台数据创建字体
  return MakeGarbageCollected<SimpleFontData>(
      custom_platform_data_->GetSkTypeface(),
      font_description.ComputedSize());
}
```

**特点**:
- 无网络 I/O
- 同步解码（可能导致 jank）
- 用于 ArrayBuffer/TypedArray/data:URI

---

## 4. FontCustomPlatformData - 跨平台字体包装

**文件**: `third_party/blink/renderer/platform/fonts/font_custom_platform_data.h/cc`

```cpp
class FontCustomPlatformData {
 public:
  // 核心方法：从二进制数据创建
  static sk_sp<FontCustomPlatformData> Create(
      SharedBuffer* buffer,  // 字体文件内容
      String& ots_parse_message);  // OTS 验证消息
  
  // 获取 Skia 字体对象
  sk_sp<SkTypeface> GetSkTypeface() const { 
    return typeface_; 
  }
  
  // 其他平台相关方法
  #if BUILDFLAG(IS_WIN)
    IDWriteFont* GetIDWriteFont() const;
  #elif BUILDFLAG(IS_APPLE)
    CTFontRef GetCTFont() const;
  #endif
  
 private:
  sk_sp<SkTypeface> typeface_;  // 核心：Skia 字体对象
  #if BUILDFLAG(IS_WIN)
    Member<IDWriteFont> font_;
  #elif BUILDFLAG(IS_APPLE)
    RetainPtr<CTFontRef> font_;
  #endif
};

// 实现：Android/Linux
sk_sp<FontCustomPlatformData> FontCustomPlatformData::Create(
    SharedBuffer* buffer,
    String& ots_parse_message) {
  
  // ① OTS（OpenType Sanitizer）验证
  size_t font_size = buffer->size();
  const uint8_t* font_data = 
      reinterpret_cast<const uint8_t*>(buffer->Data());
  
  ots::OtsContext ots_context;
  if (!ots_context.Process(font_data, font_size)) {
    ots_parse_message = "OTS validation failed: " + 
                        ots_context.GetError();
    return nullptr;  // 不安全的字体格式
  }
  
  // ② 创建 SkTypeface
  sk_sp<SkData> sk_data = SkData::MakeWithCopy(
      font_data,
      font_size);
  
  sk_sp<SkTypeface> typeface =
      SkTypeface::MakeFromData(sk_data);
  
  if (!typeface) {
    ots_parse_message = "Failed to create SkTypeface";
    return nullptr;
  }
  
  // ③ 包装为 FontCustomPlatformData
  auto* custom_data = 
      MakeGarbageCollected<FontCustomPlatformData>();
  custom_data->typeface_ = typeface;
  return custom_data;
}
```

**关键流程**:
1. **OTS 验证**: 检查字体文件结构安全性
2. **SkTypeface 创建**: 从字体数据创建 Skia 对象
3. **包装**: 为不同平台的差异提供统一接口

---

## 5. WebFontDecoder - 字体格式转换

**文件**: `third_party/blink/renderer/platform/fonts/web_font_decoder.h/cc`

```cpp
class WebFontDecoder {
 public:
  // 支持的格式
  enum FontFormat {
    kFormatUnknown,
    kFormatWoff2,
    kFormatWoff,
    kFormatTrueType,
    kFormatOpenType,
    kFormatSvg,
  };
  
  // 核心方法：解码字体
  static sk_sp<SkTypeface> Decode(
      base::span<const uint8_t> font_data,
      FontFormat);
};

// 实现
sk_sp<SkTypeface> WebFontDecoder::Decode(
    base::span<const uint8_t> font_data,
    FontFormat format) {
  
  switch (format) {
    case kFormatWoff2:
      return DecodeWoff2(font_data);
    
    case kFormatWoff:
      return DecodeWoff(font_data);
    
    case kFormatTrueType:
    case kFormatOpenType:
      // TTF/OTF：直接使用
      return SkTypeface::MakeFromData(
          SkData::MakeWithCopy(font_data.data(), font_data.size()));
    
    case kFormatSvg:
      return DecodeSvg(font_data);
    
    default:
      return nullptr;
  }
}

// WOFF2 解码
sk_sp<SkTypeface> WebFontDecoder::DecodeWoff2(
    base::span<const uint8_t> woff2_data) {
  
  // 使用 Brotli 解压
  std::vector<uint8_t> decompressed;
  size_t max_size = 30 * 1024 * 1024;  // 30MB 上限
  
  if (!brotli::BrotliDecode(
          woff2_data.data(), 
          woff2_data.size(),
          &decompressed, 
          max_size)) {
    return nullptr;  // 解压失败
  }
  
  // 得到 TTF 数据后，继续处理
  return SkTypeface::MakeFromData(
      SkData::MakeWithCopy(
          decompressed.data(), 
          decompressed.size()));
}

// WOFF 解码
sk_sp<SkTypeface> WebFontDecoder::DecodeWoff(
    base::span<const uint8_t> woff_data) {
  
  // 使用 zlib 解压
  std::vector<uint8_t> decompressed;
  
  if (!zlib::Decompress(woff_data, &decompressed)) {
    return nullptr;  // 解压失败
  }
  
  return SkTypeface::MakeFromData(
      SkData::MakeWithCopy(
          decompressed.data(), 
          decompressed.size()));
}

// SVG 字体（特殊处理）
sk_sp<SkTypeface> WebFontDecoder::DecodeSvg(
    base::span<const uint8_t> svg_data) {
  // SVG 字体通常需要特殊渲染器
  // 这里简化处理
  return nullptr;  // 当前实现不支持
}
```

**支持的格式**:
- **WOFF2**: WebOpen Font Format 2（使用 Brotli 压缩）
- **WOFF**: WebOpen Font Format 1（使用 deflate 压缩）
- **TTF**: TrueType 格式（直接使用）
- **OTF**: OpenType 格式（直接使用）
- **SVG**: SVG 字体（特殊处理，通常不支持）

---

## 6. FontResource - 网络资源管理

**文件**: `third_party/blink/renderer/core/loader/resource/font_resource.h/cc`

```cpp
class FontResource final : public Resource {
 public:
  // 获取已解码的字体
  const FontCustomPlatformData* GetCustomFontData();
  
  // 加载超时状态
  bool IsShortLimitExceeded() const { 
    return load_limit_state_ >= kShortLimitExceeded; 
  }
  bool IsLongLimitExceeded() const {
    return load_limit_state_ >= kLongLimitExceeded;
  }
  
  // CORS 信息
  bool IsCrossOrigin() const;
  bool PassesAccessControlCheck() const;
  
  // 清理原始数据（节省内存）
  void ClearData() {
    data_.Clear();  // 删除网络下载的原始数据
  }
  
 private:
  enum class LoadLimitState {
    kLoadNotStarted,
    kUnderLimit,
    kShortLimitExceeded,    // 3 秒
    kLongLimitExceeded      // 10 秒
  };
  
  Member<FontCustomPlatformData> font_data_;
  LoadLimitState load_limit_state_;
  bool cors_failed_;
  TaskHandle font_load_short_limit_;
  TaskHandle font_load_long_limit_;
};

// 核心方法：加载字体资源
FontResource* FontResource::Fetch(FetchParameters& params,
                                   ResourceFetcher* fetcher,
                                   FontResourceClient* client) {
  
  params.SetDefer(FetchParameters::kNoDefer);  // 高优先级
  
  // 设置 CORS
  params.SetCrossOriginAccessControl(
      fetcher->GetDocument()->GetSecurityOrigin(),
      kCrossOriginSameOrigin);
  
  // 发起网络请求
  FontResource* resource = 
      static_cast<FontResource*>(
          fetcher->RequestResource(params, client));
  
  return resource;
}

// 字体解码
void FontResource::NotifyFinished() {
  String ots_message;
  
  if (Data()) {
    // 解码字体数据
    font_data_ = FontCustomPlatformData::Create(
        GetData(),
        ots_message);
  }
  
  if (!font_data_) {
    // 解码失败
    ots_parsing_message_ = ots_message;
    NotifyClientsError();
    return;
  }
  
  // 成功
  NotifyClientsFinished();
  
  // 清理原始数据（字体通常 > 1MB）
  ClearData();
}
```

**关键特性**:
- **优先级管理**: 字体资源高优先级
- **超时管理**: 3s 和 10s 超时计时器
- **CORS 支持**: 跨域字体加载检查
- **内存优化**: 解码后清除原始数据

---

## 7. 自定义字体的完整加载流程

### 7.1 JavaScript 用户代码示例

```javascript
// 示例 1: 从 URL 加载
const fontFace = new FontFace('MyFont', 
  "url('https://example.com/font.woff2')",
  {
    weight: '400',
    style: 'normal'
  }
);

document.fonts.add(fontFace);

fontFace.load().then(() => {
  console.log('Font loaded!');
  // 现在可以安全地使用 CSS: font-family: MyFont
}).catch(error => {
  console.error('Font load failed:', error);
});

// 示例 2: 从 ArrayBuffer 加载
const fontBuffer = new ArrayBuffer(123456);
const fontFace2 = new FontFace('MyFont2', fontBuffer);

// 示例 3: 等待所有字体加载
document.fonts.ready.then(() => {
  console.log('All fonts loaded!');
  // 页面布局已经重新计算
});
```

### 7.2 内部执行流程

```
JavaScript: fontFace.load()
  ↓
FontFace::load(ScriptState*)
  ↓
检查状态：
  ✓ kUnloaded → 继续
  ✓ kLoading → 返回现有 Promise
  ✓ kLoaded → 解决 Promise
  ✓ kError → 拒绝 Promise
  ↓
css_font_face_->Load()
  ↓
CSSFontFace::Load()
  ↓
遍历 sources_ 列表：
  对每个 CSSFontFaceSource：
    ├─ RemoteFontFaceSource → BeginLoadIfNeeded()
    │   ├─ 构建 FetchParameters
    │   ├─ FontResource::Fetch()
    │   ├─ 启动网络请求
    │   └─ 启动 3s/10s 超时计时器
    │
    ├─ BinaryDataFontFaceSource → 直接 CreateFontData()
    │   └─ 返回已解码的 SimpleFontData
    │
    └─ LocalFontFaceSource → 查询系统字体
        └─ 返回匹配的系统字体
  ↓
等待第一个源成功加载
  ↓
FontResource::NotifyFinished()
  ├─ 验证 CORS
  ├─ 调用 FontCustomPlatformData::Create()
  ├─ 触发 OTS 验证
  ├─ 创建 SkTypeface
  └─ 设置状态 → kLoaded
  ↓
RemoteFontFaceSource::FontLoaded()
  ├─ 通知所有 CSSSegmentedFontFace
  ├─ 解除 Block Period
  ├─ 触发页面重排（从回退切换到 Web 字体）
  └─ 解决 Promise<FontFace>
  ↓
JavaScript 回调执行
```

### 7.3 错误处理流程

```
错误发生：
  ├─ 网络失败（CORS, 404, timeout)
  ├─ 格式错误（OTS 验证失败）
  ├─ 解码失败（无效的 WOFF2/WOFF）
  └─ 内存不足
  ↓
RemoteFontFaceSource::SetError()
  ├─ 记录错误消息
  ├─ 设置状态 → kError
  └─ 继续尝试下一个源
  ↓
是否有其他源？
  ✓ Yes → 尝试下一个源
  ✓ No → 所有源都失败
  ↓
CSSFontFace 最终失败
  ├─ 通知 FontFace：设置状态 → kError
  ├─ 设置 DOMException 对象
  ├─ 拒绝 Promise<FontFace>
  └─ 字体显示进入 Failure Period
  ↓
JavaScript catch 处理
```

---

## 8. 自定义字体与系统字体的交互

### 8.1 字体选择的完整优先级

```cpp
// third_party/blink/renderer/core/css/css_font_selector.cc

const SimpleFontData* CSSFontSelector::GetFontData(
    const FontDescription& font_description,
    const AtomicString& family_name) {
  
  // ① 检查 CSS 值关键字（serif, sans-serif 等）
  bool is_generic_family = 
      font_family_names::IsGenericFamilyKeyword(family_name);
  
  if (is_generic_family) {
    // 映射到系统通用家族
    // 例: "serif" → "Noto Serif"
    return GetFontDataForGenericFamily(
        font_description, family_name);
  }
  
  // ② 查询 Web 字体（FontFaceCache）
  SimpleFontData* font_data = 
      font_face_cache_->Get(font_description, family_name);
  if (font_data) {
    return font_data;  // ✓ Web 字体命中
  }
  
  // ③ 查询系统字体（FontCache）
  font_data = FontCache::Get().GetFontData(
      font_description,
      family_name);
  if (font_data) {
    return font_data;  // ✓ 系统字体命中
  }
  
  // ④ 字体回退链
  // 例: font-family: "MyFont", serif
  // "MyFont" 未找到 → 使用 serif
  return GetFontDataForGenericFamily(
      font_description, 
      GetGenericFamilyForFamily(family_name));
}

// 用户设置字体（Android WebView）
const SimpleFontData* CSSFontSelector::GetFontDataForGenericFamily(
    const FontDescription& font_description,
    const AtomicString& generic_family) {
  
  // ① 从用户设置查询
  String user_font_name = 
      GetUserPreferredFontName(generic_family);
  
  if (!user_font_name.IsEmpty()) {
    SimpleFontData* font_data = 
        FontCache::Get().GetFontData(
            font_description,
            AtomicString(user_font_name));
    
    if (font_data) {
      return font_data;  // ✓ 用户设置字体
    }
  }
  
  // ② 查询系统通用字体
  return FontCache::Get().GetFontData(
      font_description,
      generic_family);  // "serif", "sans-serif" 等
}
```

### 8.2 字体加载时序

```
时间      Web 字体状态     系统字体状态        页面显示
────────────────────────────────────────────────────
  0ms    kUnloaded        Available      [系统字体 + 回退]
         ↓ 网络请求
         kLoading                        
  50ms                                   [回退字体]
         ↓ 检查超时
         
 100ms                                   [回退字体]
         
3000ms   3s 超时                         [系统字体替换回退]
         ↓ font-display: auto
           进入 swap 期
         
5000ms   ✓ 字体加载完成                  [Web 字体显示]
         kLoaded
         ↓ 重排
         
5100ms                                   [Web 字体 + 系统字体]
         FOUT (Flash of Unstyled Text)
         或 FOIT (Flash of Invisible Text)
```

---

## 9. 关键问题的代码级回答

### Q: Web 字体加载失败时会发生什么？

**答案**:

```cpp
// 在 RemoteFontFaceSource 中
void RemoteFontFaceSource::NotifyFinished(Resource* resource) {
  // 检查各种失败条件
  
  if (font_resource_->LoadFailedOrCanceled()) {
    // 网络错误
    SetError("Network error: " + font_resource_->ErrorDetails());
    return;
  }
  
  if (!font_resource_->PassesAccessControlCheck()) {
    // CORS 失败
    SetError("CORS error");
    return;
  }
  
  SharedBuffer* data = font_resource_->Data();
  if (!data || data->size() == 0) {
    SetError("Empty font data");
    return;
  }
  
  String ots_message;
  custom_font_data_ = FontCustomPlatformData::Create(
      data, ots_message);
  
  if (!custom_font_data_) {
    // OTS 或 SkTypeface 创建失败
    SetError(ots_message);
    return;
  }
  
  // 成功流
  SetLoadStatus(FontFace::kLoaded);
}

void RemoteFontFaceSource::SetError(const String& error_message) {
  // ① 记录错误
  error_message_ = error_message;
  SetLoadStatus(FontFace::kError);
  
  // ② 检查是否有下一个源
  if (css_font_face_->HasNextSource()) {
    // 尝试下一个源（如 local() 字体）
    css_font_face_->LoadNextSource();
  } else {
    // 所有源都失败
    // ③ 通知所有依赖字体的分段
    NotifySegmentedFontFaces();  // 进入 Failure Period
    
    // ④ 拒绝 JavaScript Promise
    font_face_->SetError(
        MakeGarbageCollected<DOMException>(
            DOMExceptionCode::kNetworkError,
            "Failed to load font: " + error_message));
  }
}
```

### Q: 为什么 Web 字体比系统字体慢？

**答案**:

1. **网络 I/O** (~100-1000ms)
   - DNS 查询、TCP 连接、TLS 握手、HTTP 请求/响应
   - 受网络延迟影响

2. **解压缩** (~10-50ms)
   - WOFF2 使用 Brotli（CPU 密集）
   - WOFF 使用 zlib（较快）

3. **OTS 验证** (~5-20ms)
   - 安全检查：检查字体文件结构
   - 防止恶意字体

4. **SkTypeface 创建** (~1-10ms)
   - FreeType 初始化
   - 字体元数据加载

5. **系统字体** (~0-5ms 总计)
   - 已驻留内存
   - 无解压缩或网络 I/O

---

## 10. 性能优化建议

### 10.1 减少 Web 字体 FOUT

```css
/* 方法 1: font-display 策略 */
@font-face {
  font-family: "MyFont";
  src: url("myfont.woff2") format("woff2");
  font-display: swap;  /* 立即显示回退，字体加载后交换 */
}

/* 方法 2: Unicode Range 分割 */
@font-face {
  font-family: "MyFont";
  src: url("latin.woff2") format("woff2");
  unicode-range: U+0000-U+00FF;  /* 仅 Latin */
}

@font-face {
  font-family: "MyFont";
  src: url("cjk.woff2") format("woff2");
  unicode-range: U+4E00-U+9FFF, U+3040-U+309F;  /* CJK */
}
```

### 10.2 预加载策略

```html
<!-- 方法 1: 使用 <link rel="preload"> -->
<link rel="preload" as="font" href="myfont.woff2" 
      type="font/woff2" crossorigin>

<!-- 方法 2: 使用 <link rel="prefetch"> -->
<link rel="prefetch" href="myfont.woff2" as="font" crossorigin>

<!-- 方法 3: 内联小字体 -->
@font-face {
  font-family: "EmbeddedFont";
  src: url("data:font/woff2;base64,d09GMgAB...") format("woff2");
}
```

### 10.3 减少字体体积

```bash
# 使用 WOFF2 而不是 TTF（压缩率 30-50%）
woff2_compress font.ttf

# 仅包含必要的语言
pyftsubset font.ttf --unicodes=U+0000-U+007F  # 仅 ASCII
```

---

## 总结

| 特性 | 系统字体 | Web 字体 | 自定义字体（JS） |
|------|---------|---------|-------------|
| **初始化** | Renderer 启动 | Document 解析 | JavaScript 执行 |
| **数据来源** | /system/fonts | 网络 URL | ArrayBuffer/data:URI/URL |
| **加载时间** | 0-5ms | 100-1000ms+ | 1-10ms（本地）/ 100-1000ms+（网络） |
| **缓存** | Skia FontMgr + FreeType | FontFaceSet + HTTP Cache | FontFaceSet |
| **生命周期** | 进程级 | Document 级 | Document 级 |
| **Promise 支持** | ❌ | ✓ | ✓ |
| **错误处理** | 自动回退 | 可配置（font-display） | JavaScript 异常 |
| **CORS 限制** | ❌ | ✓ | ✓ |
| **代码复杂度** | 简单 | 复杂（超时、缓存、备选源） | 中等 |

