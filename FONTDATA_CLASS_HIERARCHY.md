# FontData 类继承体系分析

## 1. 继承关系图

```
                    GarbageCollected<FontData>
                            ↓
                        FontData (抽象基类)
                      /              \
                     /                \
          SimpleFontData          SegmentedFontData
          (单个字体)               (多个字体集合)
```

---

## 2. 三个主要的 FontData 相关类

### 2.1 **FontData** - 抽象基类

**位置**: [font_data.h](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/platform/fonts/font_data.h)

**职责**: 定义字体数据的通用接口

```cpp
class PLATFORM_EXPORT FontData : public GarbageCollected<FontData> {
 public:
  virtual ~FontData() = default;

  // 核心查询接口
  virtual const SimpleFontData* FontDataForCharacter(UChar32) const = 0;

  // 状态查询
  virtual bool IsCustomFont() const = 0;      // 是否自定义字体
  virtual bool IsLoading() const = 0;         // 是否正在加载
  virtual bool IsLoadingFallback() const = 0; // 是否加载 fallback
  virtual bool IsSegmented() const = 0;       // 是否分段字体
  virtual bool ShouldSkipDrawing() const = 0; // 是否应跳过绘制
};
```

**关键方法**:
- `FontDataForCharacter(UChar32 c)` - **核心虚函数**
  - 给定一个 Unicode 字符
  - 返回能渲染该字符的 SimpleFontData
  - 子类可实现不同的查询策略

---

### 2.2 **SimpleFontData** - 具体实现：单个字体

**位置**: [simple_font_data.h#L79](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/platform/fonts/simple_font_data.h#L79)

**职责**: 代表单个字体文件

```cpp
class PLATFORM_EXPORT SimpleFontData final : public FontData {
 private:
  Member<const FontPlatformData> platform_data_;  // 平台字体对象
  FontMetrics font_metrics_;                       // 字体度量信息
  float max_char_width_ = -1;
  float avg_char_width_ = -1;
  float space_width_ = 0;
  Glyph space_glyph_ = 0;
  Glyph zero_glyph_ = 0;
  Member<const CustomFontData> custom_font_data_;  // 自定义字体标记

 public:
  // 实现虚函数
  const SimpleFontData* FontDataForCharacter(UChar32) const override;
  Glyph GlyphForCharacter(UChar32) const;
  float WidthForGlyph(Glyph) const;
  gfx::RectF BoundsForGlyph(Glyph) const;

  // 状态查询
  bool IsCustomFont() const override { 
    return custom_font_data_; 
  }
  bool IsLoading() const override {
    return custom_font_data_ ? custom_font_data_->IsLoading() : false;
  }
  bool IsSegmented() const override;  // 通常返回 false
};
```

**特点**:
- **直接对应一个字体文件** (如 Roboto.woff2)
- 包含所有字体的具体数据
  - 平台字体对象 (SkTypeface)
  - 字体度量信息
  - 常用字符的缓存 (space, zero)
- `FontDataForCharacter()` 通过查找平台字体来实现
- 标记为 `final` - **不能再继承**

**使用例**:
```cpp
SimpleFontData* font = ...;
Glyph g = font->GlyphForCharacter('A');      // 查询字符 'A' 对应的 Glyph ID
float width = font->WidthForGlyph(g);         // 查询该 Glyph 的宽度
gfx::RectF bounds = font->BoundsForGlyph(g);  // 查询该 Glyph 的边界
```

---

### 2.3 **SegmentedFontData** - 具体实现：多个字体集合

**位置**: [segmented_font_data.h#L39](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/platform/fonts/segmented_font_data.h#L39)

**职责**: 管理多个 SimpleFontData，按 Unicode 范围分配

```cpp
class PLATFORM_EXPORT SegmentedFontData : public FontData {
 private:
  // 存储多个字体 + 其对应的 Unicode 范围
  HeapVector<Member<FontDataForRangeSet>, 1> faces_;

 public:
  void AppendFace(FontDataForRangeSet* font_data_for_range_set) {
    faces_.push_back(std::move(font_data_for_range_set));
  }

  unsigned NumFaces() const { return faces_.size(); }
  FontDataForRangeSet* FaceAt(unsigned i) const { return faces_[i].Get(); }

  // 实现虚函数：查询时遍历所有分段
  const SimpleFontData* FontDataForCharacter(UChar32 c) const override {
    for (const auto& face : faces_) {
      if (face->Contains(c)) {  // 字符是否在这个分段的范围内？
        return face->FontData();  // 返回该分段对应的字体
      }
    }
    return faces_[0]->FontData();  // 默认返回第一个字体
  }

  bool IsSegmented() const override { return true; }
  bool IsCustomFont() const override { return true; }  // 总是自定义字体
};
```

**特点**:
- **包含多个 SimpleFontData**
- 每个 SimpleFontData 关联一个 Unicode 范围
- 查询时按范围查找对应的字体
- 用于支持多字体渲染

**使用例**:
```
SegmentedFontData:
├─ Face 0: SimbleFontData (Roboto)  → U+0000-U+007F (ASCII)
├─ Face 1: SimpleFontData (Noto Sans) → U+0100-U+017F (Latin Extended-A)
└─ Face 2: SimpleFontData (SimSun)  → U+4E00-U+9FFF (CJK)

查询 'A' (U+0041)   → 在 Face 0 范围内 → 返回 Roboto
查询 'Ā' (U+0100)   → 在 Face 1 范围内 → 返回 Noto Sans
查询 '中' (U+4E2D)  → 在 Face 2 范围内 → 返回 SimSun
```

---

### 2.4 **FontDataForRangeSet** - 辅助类

**位置**: [font_data_for_range_set.h#L39](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/platform/fonts/font_data_for_range_set.h#L39)

**职责**: 将单个 SimpleFontData 与 Unicode 范围关联

```cpp
class PLATFORM_EXPORT FontDataForRangeSet
    : public GarbageCollected<FontDataForRangeSet> {
 private:
  Member<const SimpleFontData> font_data_;       // 具体字体
  Member<const UnicodeRangeSet> range_set_;      // 支持的 Unicode 范围

 public:
  bool Contains(UChar32 test_char) const {
    return !range_set_ || range_set_->Contains(test_char);
  }

  const SimpleFontData* FontData() const { 
    return font_data_.Get(); 
  }

  const UnicodeRangeSet* Ranges() const { 
    return range_set_.Get(); 
  }
};
```

**用于**:
- SegmentedFontData 中存储字体 + 范围的对应关系
- 不继承 FontData，是一个辅助容器

---

## 3. 分类的目的和架构设计

### 🎯 **核心设计目的**

```
问题：如何在一个统一的接口下支持：
  1. 单个字体的直接使用
  2. 多个字体的智能选择
  3. 字体加载/准备状态的多样性
  4. 自定义字体 vs 系统字体

解决方案：使用多态性（虚函数）
  ↓
定义 FontData 抽象接口
  ├─ SimpleFontData: 直接查询单个字体
  └─ SegmentedFontData: 按范围分配到不同字体
```

---

### 📊 **查询流程对比**

#### **场景 1: SimpleFontData 查询**

```
请求字符 'A'
  ↓
SimpleFontData::FontDataForCharacter('A')
  ↓
platform_data_->GetGlyph('A')  // 直接查平台字体
  ↓
返回 SimpleFontData* self
```

#### **场景 2: SegmentedFontData 查询**

```
请求字符 '中' (U+4E2D)
  ↓
SegmentedFontData::FontDataForCharacter('中')
  ↓
遍历 faces_:
  ├─ Face 0: 范围 U+0000-U+007F? NO
  ├─ Face 1: 范围 U+0100-U+017F? NO
  └─ Face 2: 范围 U+4E00-U+9FFF? YES ✓
  ↓
返回 Face 2 的 SimpleFontData*
  ↓
调用者继续使用这个 SimpleFontData 查询具体信息
```

---

### 🏗️ **分层架构**

```
渲染引擎
  ↓
FontData 接口 (多态调用)
  ├─ FontDataForCharacter(c) → 得到 SimpleFontData
  └─ IsLoading() / IsCustomFont() 等状态查询
  ↓
┌─────────────────┬──────────────────────┐
│                 │                      │
SimpleFontData    SegmentedFontData      
│                 │
├─ 单个字体       ├─ 多个字体组合
├─ 直接计算       ├─ 按范围分配
├─ 绘制字形       └─ 最终返回 SimpleFontData*
└─ 度量信息           ↓
                  SimpleFontData
                  └─ 最终执行绘制
```

---

## 4. 为什么要这样分类？

### **问题 1: 多字体支持**

❌ **不用继承的问题**:
```cpp
// 如果没有 SegmentedFontData，渲染引擎需要这样处理:
const SimpleFontData* font = fontList[0];
if (character >= 0x4E00 && character <= 0x9FFF) {
  font = fontList[2];  // CJK 字体
} else if (character >= 0x0100 && character <= 0x017F) {
  font = fontList[1];  // Latin Extended 字体
}
// ... 到处都要写这样的判断逻辑
```

✅ **使用继承的好处**:
```cpp
// 统一的接口，多态调用
const SimpleFontData* font = fontData->FontDataForCharacter(c);
// 内部自动选择合适的字体
```

### **问题 2: 状态管理**

不同类型的字体有不同的状态：

| 状态 | SimpleFontData | SegmentedFontData |
|------|----------------|-------------------|
| `IsLoading()` | 检查 custom_font_data_ | 检查任何一个 face |
| `IsSegmented()` | 通常 false | 总是 true |
| `IsCustomFont()` | 如果有 custom_font_data_ | 总是 true |

不能用单一类型表示。

### **问题 3: 接口的统一性**

```cpp
// 字体选择器不需要知道具体是哪种字体
FontData* result = cssFont->GetFontData(description);

// 不管是 Simple 还是 Segmented，都能通过同一接口使用
if (result) {
  const SimpleFontData* simple = result->FontDataForCharacter(c);
  glyph = simple->GlyphForCharacter(c);
}
```

---

## 5. 调用链示例

### **从渲染到具体字形的完整流程**

```
1. 页面需要渲染 "Hello 中国"
   ↓
2. 渲染引擎调用字体选择器
   CSSFontSelector::GetFontData(font_description)
   ↓
3. 返回 FontData* 
   (可能是 SimpleFontData 也可能是 SegmentedFontData)
   ↓
4. 逐字符处理 'H', 'e', 'l', 'l', 'o', ' ', '中', '国'
   
   对于 'H':
   ├─ fontData->FontDataForCharacter('H')  // 虚函数调用
   ├─ (如果是 SegmentedFontData) 查找 'H' 在哪个 face
   ├─ 返回对应的 SimpleFontData*
   ├─ simpleFontData->GlyphForCharacter('H')
   ├─ 返回 Glyph ID (假设 42)
   ├─ simpleFontData->WidthForGlyph(42)  // 字符宽度
   ├─ simpleFontData->BoundsForGlyph(42) // 字符边界
   └─ 进行栅格化和绘制
   
   对于 '中':
   ├─ fontData->FontDataForCharacter('中')  // 虚函数调用
   ├─ (如果是 SegmentedFontData) 发现 '中' 在 CJK face
   ├─ 返回 CJK SimpleFontData*
   └─ ... 同上
```

---

## 6. 代码导航

| 类 | 文件 | 关键方法 |
|---|------|--------|
| **FontData** | [font_data.h](font_data.h) | `FontDataForCharacter()` 虚函数 |
| **SimpleFontData** | [simple_font_data.h](simple_font_data.h) | `GlyphForCharacter()`, `WidthForGlyph()` |
| **SegmentedFontData** | [segmented_font_data.h](segmented_font_data.h) | `FontDataForCharacter()` 实现 |
| **FontDataForRangeSet** | [font_data_for_range_set.h](font_data_for_range_set.h) | 辅助容器 |

---

## 7. 总结表

### **三个类的核心区别**

```
┌─────────────────┬──────────────────┬──────────────────────┐
│     FontData    │ SimpleFontData   │ SegmentedFontData    │
├─────────────────┼──────────────────┼──────────────────────┤
│ 角色            │ 抽象接口         │ 单个字体             │ 多字体集合
├─────────────────┼──────────────────┼──────────────────────┤
│ 职责            │ 定义虚函数       │ 实现字体查询         │ 按范围分配字体
├─────────────────┼──────────────────┼──────────────────────┤
│ FontDataFor     │ 纯虚函数         │ 返回 self            │ 遍历 faces
│ Character()     │                  │                      │ 找匹配范围
├─────────────────┼──────────────────┼──────────────────────┤
│ 包含数据        │ 只有虚函数       │ platform_data_       │ HeapVector<
│                 │                  │ font_metrics_        │   FontDataFor
│                 │                  │ custom_font_data_    │   RangeSet>
├─────────────────┼──────────────────┼──────────────────────┤
│ 是否可继承      │ 可以继承         │ final（不可继承）    │ 不可继承
├─────────────────┼──────────────────┼──────────────────────┤
│ 何时使用        │ 类型检查/转换    │ 渲染单个字体         │ 处理多语言
│                 │ 多态调用         │ 大多数情况           │ 字体回退
└─────────────────┴──────────────────┴──────────────────────┘
```

---

## 8. 关键设计模式

### **策略模式 (Strategy Pattern)**

- **Context**: 渲染引擎
- **Strategy**: FontData 接口
  - **ConcreteStrategy A**: SimpleFontData (单字体策略)
  - **ConcreteStrategy B**: SegmentedFontData (多字体策略)

```cpp
// 渲染引擎不需要知道具体策略
void RendererDraw(FontData* font_data, UChar32 c) {
  const SimpleFontData* simple = 
      font_data->FontDataForCharacter(c);  // 多态调用
  // 后续操作相同
}
```

### **组合模式 (Composite Pattern)**

- SegmentedFontData 是多个 SimpleFontData 的组合
- 对外表现为单个 FontData 接口
- 内部管理复杂的多字体逻辑

```cpp
SegmentedFontData
  ├─ FontDataForRangeSet
  │   ├─ SimpleFontData (Roboto)
  │   └─ UnicodeRangeSet (U+0000-U+007F)
  ├─ FontDataForRangeSet
  │   ├─ SimpleFontData (Noto Sans)
  │   └─ UnicodeRangeSet (U+0100-U+017F)
  └─ ... 更多字体
```

---

## 9. 实际应用场景

### **场景 A: 简单英文网页**
```
使用 SimpleFontData (Roboto Regular)
  ↓
所有字符都在同一字体中
  ↓
直接调用 SimpleFontData::GlyphForCharacter()
```

### **场景 B: 多语言网页 (中文 + 英文 + 日文)**
```
使用 SegmentedFontData:
  ├─ Roboto for ASCII
  ├─ Noto Sans CJK SC for 中文
  └─ Noto Sans CJK JP for 日文
  ↓
字符 'A' → Roboto
字符 '中' → Noto Sans CJK SC
字符  'あ' → Noto Sans CJK JP
  ↓
统一的多态接口处理所有情况
```

### **场景 C: 字体正在加载**
```
SimpleFontData 的 custom_font_data_->IsLoading() = true
  ↓
渲染引擎知道需要等待或使用 fallback
  ↓
加载完成后，IsLoading() = false
  ↓
再次尝试渲染，这次使用真实的自定义字体
```

---

## 10. 深入理解：什么是"自定义字体"

### **定义**

**自定义字体** = 通过 `@font-face` 或 JavaScript API 显式加载的字体

```
对比：
┌──────────────────────┬──────────────────────────────────────────┐
│   系统字体           │  自定义字体                              │
├──────────────────────┼──────────────────────────────────────────┤
│ 来自操作系统         │ 来自网页 CSS 或 JavaScript                │
│ 总是可用             │ 需要动态加载                             │
│ custom_font_data_    │ custom_font_data_ 指向 CSSCustomFontData │
│ = nullptr            │ = CSSCustomFontData*                     │
│ IsCustomFont()       │ IsCustomFont()                           │
│ = false              │ = true (如果有 custom_font_data_)        │
│ IsLoading()          │ 可能返回 true (字体正在加载)             │
│ = false              │ 可能返回 true                             │
└──────────────────────┴──────────────────────────────────────────┘
```

### **代码示例**

#### **例 1: 系统字体（不是自定义字体）**

```cpp
// 用户的系统中存在的字体，如 Arial, Times New Roman
SimpleFontData* font_data = new SimpleFontData(
    platform_data,      // 直接从系统字体库获取
    nullptr,            // ← custom_font_data_ = nullptr
    ...);

font_data->IsCustomFont();  // 返回 false
```

#### **例 2: Web 字体（自定义字体）**

```cpp
// 通过 @font-face 或 fetch API 加载的字体
@font-face {
  font-family: 'MyWebFont';
  src: url('https://fonts.googleapis.com/css?family=Roboto');
}

// 渲染引擎创建：
CSSCustomFontData* custom_font_data = 
    new CSSCustomFontData(css_font_face_source, visibility);

SimpleFontData* font_data = new SimpleFontData(
    platform_data,      // 加载后的字体数据
    custom_font_data,   // ← custom_font_data_ 非 nullptr
    ...);

font_data->IsCustomFont();  // 返回 true
```

### **CustomFontData 的职责**

**位置**: [custom_font_data.h](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/platform/fonts/custom_font_data.h)

```cpp
class PLATFORM_EXPORT CustomFontData : public GarbageCollected<CustomFontData> {
 public:
  // 开始加载字体（如果尚未开始）
  virtual void BeginLoadIfNeeded() const {}
  
  // 字体是否正在加载中？
  virtual bool IsLoading() const { return false; }
  
  // 是否使用 fallback（还没加载完的临时字体）
  virtual bool IsLoadingFallback() const { return false; }
  
  // 是否应该跳过绘制（如：font-display: block 期间）
  virtual bool ShouldSkipDrawing() const { return false; }
  
  // 是否是 data: URL 字体
  virtual bool IsPendingDataUrl() const { return false; }
};
```

### **CSSCustomFontData 的具体实现**

**位置**: [css_custom_font_data.h](https://github.com/chromium/chromium/blob/main/third_party/blink/renderer/core/css/css_custom_font_data.h)

```cpp
class CSSCustomFontData final : public CustomFontData {
 private:
  Member<CSSFontFaceSource> font_face_source_;  // 字体源（文件 URL）
  FallbackVisibility fallback_visibility_;      // 是否显示 fallback
  mutable bool is_loading_ = false;             // 加载状态

 public:
  bool ShouldSkipDrawing() const override {
    // 如果是 block 期间且正在加载，不绘制（使用 fallback）
    return fallback_visibility_ == kInvisibleFallback && is_loading_;
  }

  bool IsLoading() const override { 
    return is_loading_; 
  }

  bool IsLoadingFallback() const override { 
    return true;  // 有 fallback 字体可用
  }
};
```

### **为什么要区分自定义字体？**

| 需求 | 原因 | 处理方式 |
|------|------|--------|
| **延迟加载** | Web 字体很大，可能加载慢 | `IsLoading()` 检查状态 |
| **Fallback 显示** | 等待自定义字体时显示系统字体 | `ShouldSkipDrawing()` 决定何时使用 fallback |
| **font-display** | 控制加载期间的文本显示 | 根据 `font-display` 值调整 visibility |
| **性能监控** | 追踪字体加载时间 | 检查 `IsLoading()` 和完成时间 |

### **完整的加载生命周期**

```
1. 页面解析 @font-face
   @font-face {
     font-family: 'Roboto';
     src: url('roboto.woff2');
   }
   ↓
2. 创建 CSSCustomFontData
   is_loading_ = true
   IsLoading() → true
   ↓
3. 后台下载并解码字体
   ↓
4. 字体加载完成
   is_loading_ = false
   IsLoading() → false
   ↓
5. 渲染引擎重新计算样式 + 重绘
   现在可以使用真实的自定义字体

时间轴：
0ms:     @font-face 声明
50ms:    开始下载
150ms:   下载完成，开始解码
250ms:   解码完成
         IsLoading() 从 true → false
         自动触发重绘
```

### **案例分析**

#### **案例 1: Google Fonts (快速网络)**

```
页面中使用 Roboto 字体
@font-face { src: url('https://fonts.gstatic.com/s/roboto/...') }

0ms:     创建 SimpleFontData + CSSCustomFontData
         is_loading_ = true
         ShouldSkipDrawing() = true (block 期间)
         文本显示为空白（等待字体）
50ms:    字体下载完成
100ms:   字体解码完成
         is_loading_ = false
         ShouldSkipDrawing() = false
         自动重新绘制，显示 Roboto 文本
```

#### **案例 2: 自定义本地字体 (data: URL)**

```
const font = new FontFace('MyFont', 
    'url(data:application/octet-stream;base64,...)');

是否是自定义字体？YES
  ↓
IsPendingDataUrl() = true (正在处理 data: URL)
  ↓
无需网络请求，直接从内存解码
  ↓
解码完成后，IsPendingDataUrl() = false
```

#### **案例 3: 系统字体作为 fallback**

```
CSS:
  font-family: 'MyWebFont', Arial;

如果 'MyWebFont' 是自定义字体且正在加载：
  IsLoading() = true
  ShouldSkipDrawing() = true
  ↓
渲染引擎跳过 'MyWebFont'
  ↓
使用 fallback 字体 Arial（系统字体）
  ↓
Arial->IsCustomFont() = false
  ↓
显示 Arial 文本
