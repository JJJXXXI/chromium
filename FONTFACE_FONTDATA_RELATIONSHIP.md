# FontFace 和 FontData 的关系

## 1. 快速概览

```
FontFace (JavaScript API + CSS @font-face)
    ↓ 管理
CSSFontFace (CSS 字体面的内部表示)
    ↓ 包含多个源
CSSFontFaceSource (字体源：本地/远程/数据URL)
    ↓ 加载并转换为
SimpleFontData (平台字体对象)
    ↓ 用于
绘制文本
```

---

## 2. 三层架构

### **第 1 层: FontFace (暴露给JavaScript/CSS)**

**位置**: [font_face.h#L75](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/core/css/font_face.h#L75)

```cpp
class CORE_EXPORT FontFace : public ScriptWrappable,
                             public ActiveScriptWrappable<FontFace>,
                             public ExecutionContextClient {
 public:
  enum LoadStatusType : uint8_t { 
    kUnloaded,    // 未加载
    kLoading,     // 正在加载
    kLoaded,      // 加载完成
    kError        // 加载失败
  };

  // JavaScript API
  const AtomicString& familyNameUnquoted() const { return family_; }
  V8FontFaceLoadStatus status() const;
  ScriptPromise<FontFace> loaded(ScriptState* script_state);
  ScriptPromise<FontFace> load(ScriptState*);
  
  // 内部
  CSSFontFace* CssFontFace() { return css_font_face_.Get(); }
  
 private:
  AtomicString family_;  // 字体族名
  Member<CSSFontFace> css_font_face_;  // ← 指向内部表示
  LoadStatusType status_;  // 加载状态
};
```

**职责**:
- 暴露给 JavaScript API
- 管理加载状态
- 提供 Promise 接口

**使用例**:
```javascript
const font = new FontFace('Roboto', 'url(...)', {weight: 400});
font.load().then(() => {
  document.fonts.add(font);
});
```

---

### **第 2 层: CSSFontFace (CSS 字体面的内部表示)**

**位置**: [css_font_face.h#L47](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/core/css/css_font_face.h#L47)

```cpp
class CORE_EXPORT CSSFontFace final : public GarbageCollected<CSSFontFace> {
 public:
  CSSFontFace(FontFace* font_face, HeapVector<UnicodeRange>&& ranges)
      : ranges_(MakeGarbageCollected<UnicodeRangeSet>(std::move(ranges))),
        font_face_(font_face) {}

  // 关键方法：获取字体数据
  const SimpleFontData* GetFontData(const FontDescription&);

  // 添加字体源（可能有多个备选源）
  void AddSource(CSSFontFaceSource*);
  void SetDisplay(FontDisplay);

  // 生命周期
  void DidBeginLoad();
  bool FontLoaded(CSSFontFaceSource*);
  bool FallbackVisibilityChanged(RemoteFontFaceSource*);

  // 管理分段字体
  void AddSegmentedFontFace(CSSSegmentedFontFace*);

  // 加载控制
  void Load();
  void Load(const FontDescription&);
  bool MaybeLoadFont(const FontDescription&, const String&);
  bool UpdatePeriod();

 private:
  Member<FontFace> font_face_;  // ← 指向 FontFace
  HeapDeque<Member<CSSFontFaceSource>> sources_;  // 字体源队列
  Member<const UnicodeRangeSet> ranges_;  // Unicode 范围
  HeapHashSet<Member<CSSSegmentedFontFace>> segmented_font_faces_;
};
```

**职责**:
- 连接 FontFace 和 SimpleFontData
- 管理多个备选字体源（fallback）
- 控制 font-display 生命周期
- 决定何时加载字体

---

### **第 3 层: CSSFontFaceSource (字体源)**

**位置**: [css_font_face_source.h](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/core/css/css_font_face_source.h)

```cpp
class PLATFORM_EXPORT CSSFontFaceSource
    : public GarbageCollected<CSSFontFaceSource> {
 public:
  virtual bool IsLocalNonBlocking() const { return false; }
  virtual bool IsLoading() const { return false; }
  virtual bool IsLoaded() const { return true; }
  virtual bool IsValid() const { return true; }

  virtual String GetURL() const { return g_null_atom; }
  virtual const FontCustomPlatformData* GetCustomPlaftormData() const {
    return nullptr;
  }

  // 最终返回 SimpleFontData
  const SimpleFontData* GetFontData(const FontDescription&,
                                    const FontSelectionCapabilities&);

  virtual void BeginLoadIfNeeded() {}
  virtual void SetDisplay(FontDisplay) {}
  virtual bool IsInBlockPeriod() const { return false; }
  virtual bool IsInFailurePeriod() const { return false; }
  virtual bool UpdatePeriod() { return false; }
};
```

**两个主要实现**:

1. **LocalFontFaceSource** - 本地系统字体
   ```cpp
   src: local('Arial');
   ```

2. **RemoteFontFaceSource** - 远程网络字体
   ```cpp
   src: url('https://fonts.gstatic.com/s/roboto.woff2');
   ```

---

## 3. 数据流过程

### **场景: 渲染使用 @font-face 字体的文本**

```
step 1: 页面解析 CSS
┌──────────────────────────────────────────┐
│ @font-face {                             │
│   font-family: 'MyFont';                 │
│   src: url('myfont.woff2');              │
│   font-display: swap;                    │
│ }                                        │
└──────────────────────────────────────────┘
         ↓
step 2: 创建对象链
FontFace* 
  ├─ status_ = kUnloaded
  ├─ family_ = "MyFont"
  └─ css_font_face_ → CSSFontFace*
                       ├─ ranges_ = UnicodeRangeSet (全部)
                       ├─ sources_ = [RemoteFontFaceSource*]
                       │               └─ is_loading_ = false
                       └─ segmented_font_faces_ = []

step 3: 渲染引擎需要该字体
CSSFontFace::GetFontData(font_description)
  ↓
if (LoadStatus() == FontFace::kUnloaded) {
  Load(font_description);  // 触发下载
}
  ↓
RemoteFontFaceSource::BeginLoadIfNeeded()
  ├─ 创建 FontResource
  ├─ 网络请求 'myfont.woff2'
  ├─ is_loading_ = true
  └─ FontFace::status_ = kLoading

step 4: 字体下载完成
RemoteFontFaceSource::NotifyFinished(resource)
  ├─ 解码为 SkTypeface
  ├─ 创建 FontCustomPlatformData
  ├─ 创建 SimpleFontData
  │   ├─ platform_data_ = SkTypeface
  │   ├─ font_metrics_ = 测量数据
  │   └─ custom_font_data_ = CSSCustomFontData
  ├─ is_loading_ = false
  ├─ CSSFontFace::FontLoaded()
  │   └─ FontFace::status_ = kLoaded
  └─ font_selector_->FontFaceInvalidated()
      └─ 触发重新计算样式和重绘

step 5: 再次渲染相同的文本
CSSFontFace::GetFontData(font_description)
  ↓
LoadStatus() == FontFace::kLoaded (✓)
  ↓
返回 SimpleFontData* 
  ├─ GlyphForCharacter('M')
  ├─ WidthForGlyph(glyph_id)
  └─ 调用 Skia 进行栅格化和绘制
```

---

## 4. 关键转换点

### **FontFace → SimpleFontData 的转换路径**

```
FontFace (JavaScript 对象)
    ↓
    └─→ CSSFontFace::GetFontData()
        ↓
        └─→ 检查 LoadStatus()
            ├─ kUnloaded: 触发 Load()
            ├─ kLoading: 返回 nullptr (使用 fallback)
            ├─ kLoaded: 返回 SimpleFontData*
            └─ kError: 返回 nullptr
```

### **CSSFontFace 的角色**

```
┌─────────────────────────────────────────────────────┐
│  CSSFontFace (协调者)                               │
├─────────────────────────────────────────────────────┤
│                                                     │
│  输入:   FontDescription (字体大小、样式、重量等)   │
│                                                     │
│  处理:   1. 检查加载状态                           │
│          2. 触发加载（如果需要）                   │
│          3. 查询源列表（fallback chain）           │
│          4. 返回可用的 SimpleFontData              │
│                                                     │
│  输出:   SimpleFontData* (用于渲染)               │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 5. 源列表管理（Fallback Chain）

```
@font-face {
  font-family: 'CustomFont';
  src: local('CustomFont'),
       local('CustomFont-Regular'),
       url('https://example.com/font.woff2'),
       url('https://example.com/font.woff');
}

创建的源链:
sources_[0] → LocalFontFaceSource('CustomFont')
sources_[1] → LocalFontFaceSource('CustomFont-Regular')  
sources_[2] → RemoteFontFaceSource('font.woff2')
sources_[3] → RemoteFontFaceSource('font.woff')

CSSFontFace::GetFontData() 的查询逻辑:

const SimpleFontData* GetFontData(...) {
  while (!sources_.empty()) {
    CSSFontFaceSource& source = sources_.front();
    
    if (source->IsValid()) {
      // 尝试从这个源获取数据
      SimpleFontData* result = source->GetFontData(...);
      if (result) {
        return result;  // 成功！
      }
    }
    
    sources_.pop_front();  // 移到下一个源
  }
  
  return nullptr;  // 没有可用的源
}
```

---

## 6. 加载状态管理

### **状态转换图**

```
         FontFace 创建
              ↓
        kUnloaded 状态
        /     |     \
       /      |      \
  调用   文本   JavaScript
  load() 需要   API 加载
   |     |        |
   v     v        v
  kLoading ←─────┘
   |     |
   |    成功
   |     |
   ├────→ kLoaded ←─── 使用之前加载的字体
   |
  失败
   |
   v
  kError
```

### **代码流程**

```cpp
// 步骤 1: 创建
FontFace* font = new FontFace(context, family, source, descriptors);
font->status_ = FontFace::kUnloaded;

// 步骤 2: 触发加载
if (css_font_face->GetFontData(description)) {
  // 内部调用 CSSFontFace::Load()
}
css_font_face->Load();
// FontFace::status_ = kLoading

// 步骤 3: 加载完成回调
RemoteFontFaceSource::NotifyFinished() {
  css_font_face->FontLoaded(this);
  // FontFace::status_ = kLoaded
}

// 步骤 4: 获取数据
const SimpleFontData* data = css_font_face->GetFontData(description);
if (data) {
  // 可以使用真实字体
} else if (font->IsLoading()) {
  // 还在加载，使用 fallback
}
```

---

## 7. 实际使用示例

### **示例 1: CSS @font-face**

```css
@font-face {
  font-family: 'Roboto';
  src: url('roboto.woff2') format('woff2');
  font-display: swap;
}

body {
  font-family: Roboto, Arial, sans-serif;
}
```

**内部流程**:

```
1. 解析 @font-face
   → 创建 FontFace + CSSFontFace
   
2. 解析 font-family: Roboto
   → CSSFontSelector 记录要使用 Roboto
   
3. 渲染 <body> 文本
   → 需要 Roboto 字体数据
   → CSSFontFace::GetFontData()
   → 触发下载
   → 返回 SimpleFontData (延迟)
   
4. 字体下载完成
   → SimpleFontData 创建完成
   → FontFace::status_ = kLoaded
   → 触发重绘
   
5. 再次渲染
   → 使用真实的 SimpleFontData
   → 显示 Roboto 文本
```

### **示例 2: JavaScript FontFace API**

```javascript
// 创建字体对象
const font = new FontFace(
  'MyFont',
  'url(https://fonts.example.com/myfont.woff2)',
  { weight: '400', style: 'normal' }
);

// 等同于内部:
// FontFace* obj = new FontFace(context, 'MyFont', ...);
// obj->status_ = FontFace::kUnloaded;

// 加载字体
font.load().then(() => {
  // FontFace::status_ = kLoaded
  document.fonts.add(font);
  // 现在可以使用 'MyFont'
}).catch(() => {
  // FontFace::status_ = kError
  console.error('Font failed to load');
});

// 在 CSS 中使用
document.body.style.fontFamily = 'MyFont, serif';
```

---

## 8. 关键理解

### **FontFace vs FontData 的区别**

| 方面 | FontFace | FontData |
|------|---------|---------|
| **层级** | 高层 (用户 API) | 底层 (平台实现) |
| **来源** | CSS @font-face 或 JS API | 加载后的字体资源 |
| **职责** | 声明字体 + 管理生命周期 | 存储字体具体数据 |
| **何时创建** | 解析时 | 加载完成时 |
| **可用性** | 需要加载 | 加载完成后可用 |
| **查询方式** | font.status / font.loaded | font_data->IsLoading() |
| **用途** | JavaScript 控制 | 渲染文本 |

### **关系层次**

```
用户告诉浏览器         FontFace
"我要用 Roboto"    (声明字体)
    ↓                   ↓
浏览器找到字体源      CSSFontFace
并加载它           (获取字体源)
    ↓                   ↓
字体下载并解码        SimpleFontData
成可用格式         (字体数据)
    ↓                   ↓
调用操作系统         Skia/平台 API
绘制文本            (实际渲染)
```

---

## 9. 核心流程总结

```cpp
// 核心转换函数
const SimpleFontData* CSSFontFace::GetFontData(
    const FontDescription& font_description) {
  
  // 如果还没有加载，触发加载
  if (LoadStatus() == FontFace::kUnloaded) {
    Load(font_description);
    // FontFace::status_ = kLoading
    // 返回 nullptr，使用 fallback
  }
  
  // 如果正在加载
  if (LoadStatus() == FontFace::kLoading) {
    // 根据 font-display 决定是否使用 fallback
    if (ShouldUseInvisibleFallback()) {
      return nullptr;  // 使用 fallback (不显示)
    }
    return nullptr;  // 使用 fallback (显示)
  }
  
  // 如果加载完成
  if (LoadStatus() == FontFace::kLoaded) {
    return GetValidFontData();  // ← SimpleFontData*
  }
  
  // 如果加载失败
  if (LoadStatus() == FontFace::kError) {
    return nullptr;  // 使用 fallback
  }
}
```

---

## 10. 总结表

| 组件 | 类型 | 职责 | 何时存在 |
|------|------|------|--------|
| **FontFace** | JavaScript 对象 + C++ 绑定 | 暴露 API，管理状态 | 一直 |
| **CSSFontFace** | CSS 处理对象 | 协调加载和查询 | 一直 |
| **CSSFontFaceSource** | 字体源 | 提供字体数据 | 一直 |
| **SimpleFontData** | 平台字体对象 | 存储字形和度量 | 加载完成后 |
| **SkTypeface** | Skia 对象 | 平台字体接口 | 加载完成后 |

---

## 11. 关键文件导航

| 文件 | 用途 |
|------|------|
| [font_face.h](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/core/css/font_face.h) | FontFace (JS 暴露) |
| [css_font_face.h](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/core/css/css_font_face.h) | CSSFontFace (协调) |
| [css_font_face_source.h](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/core/css/css_font_face_source.h) | 字体源 |
| [remote_font_face_source.cc](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/core/css/remote_font_face_source.cc) | 远程字体加载 |
| [font_resource.h](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/core/loader/resource/font_resource.h) | 字体资源 |
| [simple_font_data.h](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/platform/fonts/simple_font_data.h) | SimpleFontData |
