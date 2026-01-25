# HTML 解析到 Layout Tree 构建完成的函数调用流程

> 详细追踪从 HTML 文档开始解析到最终 Layout Tree 构建完成的每一步

---

## 概述

HTML 解析到 Layout Tree 构建的流程涉及以下关键阶段：

```
HTML Data Input
  ↓
HTML Parsing (Tokenization + Tree Building)
  ↓
DOM Tree Construction
  ↓
CSS 解析与样式计算
  ↓
Layout Tree 构建（从 DOM Tree 创建 LayoutObjects）
  ↓
Layout 计算（位置、大小、尺寸）
  ↓
Paint Phase（生成绘制指令）
```

---

## 第一阶段：HTML 解析入口

### 1.1 document.write() / Append HTML

**文件**: `third_party/blink/renderer/core/html/parser/html_document_parser.h`

**入口点**:

```cpp
// 主文档解析器创建
HTMLDocumentParser::HTMLDocumentParser(HTMLDocument&,
                                       ParserSynchronizationPolicy,
                                       ParserPrefetchPolicy);

// 追加 HTML 内容
void HTMLDocumentParser::Append(const String&) override;
void HTMLDocumentParser::AppendBytes(base::span<const uint8_t> bytes) override;
```

**关键类**:
- `HTMLDocumentParser`: 文档级 HTML 解析器
- `HTMLInputStre am`: 输入流管理
- `HTMLTokenizer`: 词元化引擎

### 1.2 文档加载阶段

**文档位置**: `content/renderer/render_frame_impl.cc`

```cpp
// 帧创建时的初始化流程
void RenderFrameImpl::Initialize()
  ↓ 创建 WebLocalFrameImpl
  ↓ 创建 Document
  ↓ 创建 DocumentParser

// 收到 HTML 数据时
void RenderFrameImpl::DidReceiveData(const char* data, size_t data_length)
  ↓ RenderFrameImpl::CommitNavigation()
  ↓ Document::open()
  ↓ HTMLDocumentParser::Append()
```

---

## 第二阶段：Tokenization（词元化）

### 2.1 HTML Tokenizer

**文件**: `third_party/blink/renderer/core/html/parser/html_tokenizer.h`

**流程**:

```cpp
class HTMLTokenizer {
  // 状态机处理
  State NextToken(SegmentedString& source, HTMLToken& token);
  
  // 关键状态:
  // - DataState: 处理文本数据
  // - TagNameState: 处理标签名
  // - AttributeNameState: 处理属性名
  // - AttributeValueState: 处理属性值
  // - CommentStartState: 处理注释
  // ... 更多状态
};
```

**调用链**:

```
HTMLDocumentParser::Append(String)
  ↓
HTMLDocumentParser::PumpTokenizer()
  ↓
HTMLTokenizer::NextToken()
  ↓ 返回 HTMLToken (StartTag, EndTag, Character, EOF, etc.)
  ↓
HTMLTreeBuilder::ReceiveToken(AtomicHTMLToken)
```

**关键函数**:

```cpp
// 在 html_document_parser.cc 中
void HTMLDocumentParser::PumpTokenizer() {
  if (IsStopped())
    return;
  
  // 从输入流中获取下一个 token
  HTMLToken token;
  HTMLTokenizer::State state = m_tokenizer.nextToken(m_input, token);
  
  // 将 token 转为原子形式（优化）
  AtomicHTMLToken atomicToken(token);
  
  // 传递给 Tree Builder
  m_treeBuilder->receiveToken(atomicToken);
  
  // 如果需要，继续处理下一个 token
  if (state == HTMLTokenizer::DataState || 
      state == HTMLTokenizer::RCDATAState)
    PumpTokenizer();
}
```

### 2.2 Tokenizer 的关键状态

| 状态 | 目的 | 例子 |
|------|------|------|
| `DataState` | 处理普通文本 | `hello` |
| `TagNameState` | 解析标签名 | `<div` |
| `BeforeAttributeNameState` | 标签属性前 | `<div ` |
| `AttributeNameState` | 属性名 | `class` |
| `AttributeValueState` | 属性值 | `"myclass"` |
| `CommentStartState` | 注释 | `<!-- comment -->` |
| `ScriptDataState` | 脚本内容 | `<script>...` |

---

## 第三阶段：Tree Building（树构建）

### 3.1 HTMLTreeBuilder 核心流程

**文件**: `third_party/blink/renderer/core/html/parser/html_tree_builder.h`

```cpp
class HTMLTreeBuilder final : public GarbageCollected<HTMLTreeBuilder> {
 public:
  // 接收 token 并构建树
  void ReceiveToken(const AtomicHTMLToken&);
  
  // 关键处理方法
  void ProcessToken(const AtomicHTMLToken&);
  void ProcessStartTag(const AtomicHTMLToken&);
  void ProcessEndTag(const AtomicHTMLToken&);
  void ProcessCharacter(const AtomicHTMLToken&);
  void ProcessEOF(const AtomicHTMLToken&);
  
 private:
  // 内部构建状态
  InsertionMode insertion_mode_;  // 当前插入模式
  HTMLElementStack element_stack_;  // 打开元素栈
  HTMLConstructionSite construction_site_;  // DOM 节点创建
};
```

### 3.2 Insertion Mode（插入模式）

HTML5 规范定义了多种插入模式来处理不同的上下文：

| 插入模式 | 用途 |
|---------|------|
| `InitialMode` | 初始状态，处理 DOCTYPE |
| `BeforeHtmlMode` | HTML 元素前 |
| `BeforeHeadMode` | HEAD 元素前 |
| `InHeadMode` | HEAD 内部 |
| `InBodyMode` | BODY 内部（最常用） |
| `TextMode` | 原始文本（SCRIPT、STYLE） |
| `InTableMode` | TABLE 内部 |
| `InTableBodyMode` | TABLE BODY 内部 |

### 3.3 树构建关键函数

**文件**: `third_party/blink/renderer/core/html/parser/html_tree_builder.cc`

```cpp
// 1. 接收 token
void HTMLTreeBuilder::ReceiveToken(const AtomicHTMLToken& token) {
  // 根据 insertion_mode 和 token 类型调用处理函数
  ProcessToken(token);
}

// 2. 处理不同的 token 类型
void HTMLTreeBuilder::ProcessToken(const AtomicHTMLToken& token) {
  switch (token.Type()) {
    case HTMLToken::StartTag:
      ProcessStartTag(token);
      break;
    case HTMLToken::EndTag:
      ProcessEndTag(token);
      break;
    case HTMLToken::Character:
      ProcessCharacter(token);
      break;
    case HTMLToken::EndOfFile:
      ProcessEOF(token);
      break;
    case HTMLToken::Comment:
      ProcessComment(token);
      break;
    // ... 其他类型
  }
}

// 3. 创建 DOM 元素
void HTMLTreeBuilder::InsertElement(const AtomicHTMLToken& token) {
  // 根据 token 的标签名创建元素
  Element* element = construction_site_.CreateElement(token);
  
  // 添加到 DOM 树中
  current_node_->AppendChild(element);
  
  // 更新栈
  element_stack_.Push(element);
}

// 4. 关闭元素
void HTMLTreeBuilder::ProcessEndTag(const AtomicHTMLToken& token) {
  // 查找匹配的打开标签
  Element* element = element_stack_.Find(token.TagName());
  
  if (element) {
    // 关闭所有中间的标签
    element_stack_.PopUntilElementPopped(element);
  }
}
```

### 3.4 HTMLConstructionSite（DOM 节点创建）

**文件**: `third_party/blink/renderer/core/html/parser/html_construction_site.h`

```cpp
class HTMLConstructionSite {
 public:
  // 创建 Element
  Element* CreateElement(const AtomicHTMLToken& token) {
    // 1. 确定命名空间（HTML、SVG、MathML）
    QualifiedName tag_name = TagToQualifiedName(token);
    
    // 2. 创建元素对象
    Element* element = CreateElementForToken(token);
    
    // 3. 设置属性
    SetAttributes(element, token);
    
    // 4. 调用 CustomElement 回调
    MaybeRunCustomElementDefinedCallback(element);
    
    return element;
  }
  
  // 插入文本节点
  void InsertText(const String& text, InsertionMode mode);
  
  // 添加子节点
  void AppendChild(Node* child, Element* parent);
};
```

### 3.5 DOM Tree 构建完成的关键点

**文件**: `third_party/blink/renderer/core/dom/document.cc`

```cpp
// 1. 解析完成时调用
void Document::FinishParsing() {
  // 标记解析完成
  SetReadyState(kComplete);
  
  // 移除解析中的标志
  RemoveParsingBlockingScript();
  
  // 触发 DOMContentLoaded 事件
  DispatchEvent(Event::Create(event_type_names::kDomcontentloaded));
  
  // 触发 load 事件（需要等待所有资源）
  // RequestsFinished() → DispatchLoad()
}

// 2. 文档可用时的回调
void Document::DocumentElementAvailable() {
  // 此时 <html> 元素已创建
  // Layout Engine 可以开始准备工作
}
```

---

## 第四阶段：CSS 解析与样式计算

### 4.1 CSS 加载与解析

**文件**: `third_party/blink/renderer/core/css/style_engine.h`

```cpp
class StyleEngine {
 public:
  // 添加样式表
  void AddStyleSheet(CSSStyleSheet*);
  
  // 更新样式（在 DOM 修改后调用）
  void UpdateActiveStyle();
  
  // 处理 <link> 标签
  void DidAddPendingStylesheet();
  void DidRemovePendingStylesheet();
};
```

### 4.2 样式计算入口

```cpp
// 在 html_tree_builder.cc 中
void HTMLTreeBuilder::InsertElement(const AtomicHTMLToken& token) {
  Element* element = construction_site_.CreateElement(token);
  
  // 元素插入 DOM 后
  parent->AppendChild(element);
  
  // 如果是 <link rel="stylesheet"> 或 <style>，触发样式计算
  if (element->IsStyleElement() || element->IsLinkElement()) {
    StyleEngine* style_engine = document_->GetStyleEngine();
    style_engine->UpdateActiveStyle();  // ← 关键：触发样式重新计算
  }
}
```

### 4.3 计算样式（Computed Style）

**文件**: `third_party/blink/renderer/core/css/style_resolver.h`

```cpp
class StyleResolver {
 public:
  // 为元素计算最终样式
  const ComputedStyle* StyleForElement(Element*);
  
  // 关键步骤：
  // 1. 收集应用于该元素的所有规则（内联、样式表、用户代理）
  // 2. 根据特异性和级联解决冲突
  // 3. 计算最终的 ComputedStyle
};
```

**调用流程**:

```
Element 插入 DOM
  ↓
Document::DidInsertElement()
  ↓
Style Invalidation（标记需要重新计算）
  ↓
Document::UpdateStyleIfNeeded()
  ↓
StyleResolver::StyleForElement()
  ↓ 返回 ComputedStyle*
  ↓
Element::SetComputedStyle()
```

---

## 第五阶段：DOM 到 LayoutTree 的转换

### 5.1 LayoutObject 创建（DOM → LayoutTree）

**文件**: `third_party/blink/renderer/core/layout/layout_object.h`

```cpp
class LayoutObject : public GarbageCollected<LayoutObject> {
 public:
  // 创建 LayoutObject（根据 ComputedStyle）
  static LayoutObject* CreateObject(Element*, const ComputedStyle&);
  
  // 重要方法
  virtual void Layout() = 0;
  virtual void Paint(const PaintInfo&) = 0;
  virtual LayoutSize MinMaxSize() = 0;
};

// 常见的 LayoutObject 子类：
// - LayoutBox: 有尺寸的元素
// - LayoutBlockFlow: 块级元素
// - LayoutInline: 内联元素
// - LayoutText: 文本内容
// - LayoutTable: 表格
// - LayoutFlexibleBox: Flexbox 容器
// - LayoutGrid: Grid 容器
```

### 5.2 LayoutTree 构建流程

**文件**: `third_party/blink/renderer/core/layout/layout_tree_builder.h`

```cpp
class LayoutTreeBuilder {
 public:
  // 为 DOM 树构建对应的 Layout 树
  LayoutObject* CreateLayout(Node*);
  
  // 关键方法：
  // 1. 遍历 DOM 树的每个节点
  // 2. 跳过某些节点（display:none、伪元素等）
  // 3. 创建对应的 LayoutObject
  // 4. 建立 LayoutObject 树的父子关系
};
```

**调用流程**:

```
Document::UpdateLayout()
  ↓
Document::UpdateLayoutTree()  ← 开始构建 LayoutTree
  ↓
LayoutTreeBuilder::CreateLayout()
  ↓
for each Element in DOM:
  ├─ Get ComputedStyle
  ├─ Determine display value
  ├─ Create appropriate LayoutObject
  ├─ Set Parent/Child relationships
  └─ Handle pseudo-elements
  ↓
Root LayoutObject 树完成
```

### 5.3 Style-to-Layout 映射

**文件**: `third_party/blink/renderer/core/layout/layout_tree_builder.cc`

```cpp
LayoutObject* CreateLayout(Node* node) {
  // 1. 获取计算后的样式
  const ComputedStyle* style = node->ComputedStyleRef();
  
  if (!style)
    return nullptr;
  
  // 2. 检查 display 属性
  EDisplay display = style->GetDisplay();
  
  switch (display) {
    case EDisplay::kNone:
      // 不创建 LayoutObject
      return nullptr;
      
    case EDisplay::kBlock:
    case EDisplay::kFlowRoot:
      // 创建 LayoutBlockFlow
      return new LayoutBlockFlow(node);
      
    case EDisplay::kInline:
      // 创建 LayoutInline
      return new LayoutInline(node);
      
    case EDisplay::kInlineBlock:
      // 创建 LayoutBlockFlow（但作为内联元素）
      return new LayoutBlockFlow(node);
      
    case EDisplay::kTable:
      // 创建 LayoutTable
      return new LayoutTable(node);
      
    case EDisplay::kFlex:
      // 创建 LayoutFlexibleBox
      return new LayoutFlexibleBox(node);
      
    case EDisplay::kGrid:
      // 创建 LayoutGrid
      return new LayoutGrid(node);
      
    // ... 更多 display 类型
  }
}
```

### 5.4 关键的 LayoutObject 类型

| LayoutObject 类型 | 对应 CSS | 例子 |
|-----------------|---------|------|
| `LayoutBlockFlow` | `display: block` | `<div>`, `<p>` |
| `LayoutInline` | `display: inline` | `<span>`, `<a>` |
| `LayoutInlineBlock` | `display: inline-block` | `<input>` |
| `LayoutTable` | `display: table` | `<table>` |
| `LayoutTableRow` | `display: table-row` | `<tr>` |
| `LayoutTableCell` | `display: table-cell` | `<td>` |
| `LayoutFlexibleBox` | `display: flex` | flex 容器 |
| `LayoutGrid` | `display: grid` | grid 容器 |
| `LayoutText` | 文本节点 | "hello" |
| `LayoutImage` | 替换元素 | `<img>` |

---

## 第六阶段：Layout（布局计算）

### 6.1 Layout 入口

**文件**: `third_party/blink/renderer/core/dom/document.cc`

```cpp
void Document::UpdateLayout() {
  // 1. 检查是否已经在 layout 中
  if (lifecycle_.StateTransitionDisallowed()) {
    return;  // 避免重入
  }
  
  // 2. 确保样式已计算
  UpdateStyleIfNeeded();
  
  // 3. 开始 Layout（关键！）
  LayoutTreeBuilder builder;
  LayoutObject* root = builder.CreateLayout(this);
  
  if (root) {
    // 4. 调用根 LayoutObject 的 Layout 方法
    LayoutView* layout_view = root->View();
    layout_view->UpdateLayout();  // ← 开始递归布局计算
  }
  
  // 5. 标记 lifecycle 状态
  lifecycle_.AdvanceTo(DocumentLifecycle::kInLayoutSubtreeLayout);
  lifecycle_.AdvanceTo(DocumentLifecycle::kLayoutClean);
}
```

### 6.2 Layout 的递归调用

**文件**: `third_party/blink/renderer/core/layout/layout_object.cc`

```cpp
void LayoutObject::Layout() {
  // 1. 检查是否需要 layout
  if (!NeedsLayout()) {
    return;
  }
  
  // 2. 执行自身的布局逻辑
  PerformLayout();
  
  // 3. 递归调用所有子元素的 Layout
  for (LayoutObject* child = FirstChild(); child; child = child->NextSibling()) {
    if (child->NeedsLayout()) {
      child->Layout();  // ← 递归调用
    }
  }
  
  // 4. 清除 needs_layout 标志
  ClearNeedsLayout();
  
  // 5. 计算尺寸（高度、宽度）
  // 6. 计算位置（x, y 坐标）
  // 7. 处理溢出（overflow）
}
```

### 6.3 各类 LayoutObject 的 Layout 实现

#### LayoutBlockFlow::Layout()
```cpp
void LayoutBlockFlow::Layout() {
  // 1. 计算内容宽度
  LogicalExtentComputedValues computed_values;
  ComputeLogicalWidth(computed_values);
  
  // 2. 布局子元素
  for (LayoutBox* child = FirstChildBox(); child; 
       child = child->NextSiblingBox()) {
    // 处理边距折叠（margin collapsing）
    LayoutUnit margin_before = ComputeMarginBefore(child);
    
    // 设置子元素位置
    child->SetLogicalTop(current_logical_top + margin_before);
    
    // 递归布局
    child->Layout();
    
    // 更新当前高度
    current_logical_top += child->LogicalHeight() + margin_after;
  }
  
  // 3. 计算自身的总高度
  LogicalExtentComputedValues height_values;
  ComputeLogicalHeight(height_values);
}
```

#### LayoutInline::Layout()
```cpp
void LayoutInline::Layout() {
  // Inline 元素不参与 Layout，由 LayoutBlockFlow 处理行布局
  // 只需要标记子元素
  for (LayoutObject* child = FirstChild(); child; 
       child = child->NextSibling()) {
    child->ClearNeedsLayout();
  }
}
```

#### LayoutFlexibleBox::Layout()
```cpp
void LayoutFlexibleBox::Layout() {
  // 1. 主轴方向布局
  PerformLayout();
  
  // 2. 处理 flex item
  for (LayoutBox* child = FirstChildBox(); child; 
       child = child->NextSiblingBox()) {
    // 计算 flex basis
    // 计算 flex grow/shrink
    // 分配空间
    child->Layout();
  }
}
```

### 6.4 Layout 计算的关键参数

每个 LayoutObject 需要计算：

```cpp
class LayoutBox : public LayoutBoxModelObject {
 private:
  // 位置
  LayoutUnit x_;  // 相对于父元素
  LayoutUnit y_;  // 相对于父元素
  
  // 尺寸
  LayoutUnit width_;
  LayoutUnit height_;
  
  // 盒子模型
  LayoutBoxExtent border_;
  LayoutBoxExtent padding_;
  LayoutBoxExtent margin_;
  
  // 其他
  LayoutUnit overflow_width_;
  LayoutUnit overflow_height_;
};
```

---

## 第七阶段：Paint（绘制）

### 7.1 Paint 入口

**文件**: `third_party/blink/renderer/core/layout/layout_object.h`

```cpp
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
  
  // 5. Paint 子元素
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

### 7.2 Paint 的关键阶段

```
Paint(LayoutObject)
  ├─ PaintPhase::BackgroundPhase
  │   └─ 绘制背景色、背景图片
  ├─ PaintPhase::BorderPhase
  │   └─ 绘制边框
  ├─ PaintPhase::ForegroundPhase
  │   ├─ 绘制文本
  │   ├─ 绘制替换元素（图片等）
  │   └─ PaintChildren()  ← 递归调用子元素
  └─ PaintPhase::OutlinePhase
      └─ 绘制 outline
```

---

## 完整的函数调用链

### 链路 1：从 HTML 数据到 Tokenization

```
RenderFrameImpl::OnReceiveMessage(kCommitNavigation)
  ↓
RenderFrameImpl::CommitNavigation()
  ↓
LocalFrame::Navigate()
  ↓
Document::open()
  ↓ 创建 HTMLDocumentParser
  ↓
HTMLDocumentParser::HTMLDocumentParser()
  ├─ 初始化 HTMLTokenizer
  ├─ 初始化 HTMLTreeBuilder
  └─ 初始化 HTMLPreloadScanner

DocumentLoader::DidReceiveData(const char* data, int length)
  ↓
RenderFrameImpl::DidReceiveData()
  ↓
HTMLDocumentParser::AppendBytes(data)
  ↓
HTMLDocumentParser::Append(decoded_string)
  ↓
HTMLDocumentParser::PumpTokenizer()
  ├─ HTMLTokenizer::NextToken()
  │   ├─ 状态机处理（DataState, TagNameState等）
  │   └─ 返回 HTMLToken
  └─ HTMLTreeBuilder::ReceiveToken()
```

### 链路 2：从 Token 到 DOM Tree

```
HTMLTreeBuilder::ReceiveToken(AtomicHTMLToken token)
  ↓
HTMLTreeBuilder::ProcessToken()
  ├─ if StartTag → ProcessStartTag()
  │   ├─ HTMLConstructionSite::CreateElement()
  │   │   ├─ 创建 DOM Element
  │   │   ├─ 设置属性
  │   │   └─ 调用 CustomElement 回调
  │   ├─ element->AppendChild()  // 添加到 DOM
  │   └─ element_stack_.Push()
  ├─ if EndTag → ProcessEndTag()
  │   └─ element_stack_.PopUntilElementPopped()
  └─ if Character → ProcessCharacter()
      ├─ 创建 Text Node
      └─ current_element->AppendChild(text_node)

// 当 </html> 或 EOF 时
HTMLTreeBuilder::ProcessEOF()
  ↓
Document::FinishParsing()
  ├─ SetReadyState(kComplete)
  ├─ DispatchEvent(DOMContentLoaded)
  └─ RequestsFinished() → DispatchEvent(load)
```

### 链路 3：从 DOM Tree 到样式计算

```
Element::AppendChild(child)
  ↓
Document::DidInsertElement()
  ↓
StyleEngine::DidAddElement(element)
  ├─ InvalidateStyle(element)
  └─ SetNeedsStyleRecalc()

Document::UpdateStyleIfNeeded()  // 可被多个地方调用
  ↓
StyleEngine::RecalcStyleIfNeeded()
  ├─ for each Element in DOM:
  │   └─ StyleResolver::StyleForElement(element)
  │       ├─ 收集规则（inline、stylesheets、UA）
  │       ├─ 解决级联冲突
  │       └─ 创建 ComputedStyle
  └─ SetLifecycleState(StyleClean)
```

### 链路 4：从 DOM+ComputedStyle 到 LayoutTree

```
Document::UpdateLayout()
  ↓
Document::UpdateLayoutTree()
  ├─ LayoutTreeBuilder::CreateLayout(document)
  │   └─ for each Element in DOM:
  │       ├─ Get ComputedStyle
  │       ├─ Check display property
  │       └─ Create LayoutObject
  │           ├─ LayoutBlockFlow
  │           ├─ LayoutInline
  │           ├─ LayoutText
  │           └─ ...
  └─ root_layout_object_created

// 建立 LayoutObject 树的关系
LayoutObject* parent = element->GetLayoutObject();
for each child in element->children():
  LayoutObject* child_layout = child->GetLayoutObject();
  parent->AddChild(child_layout);  // 链接树关系
```

### 链路 5：从 LayoutTree 到 Layout 计算

```
LocalFrameView::UpdateLayout()
  ↓ 进入 Layout 周期
  ↓
LayoutView::UpdateLayout()  // 根 LayoutObject
  ↓ 计算视口尺寸
  ↓
LayoutObject::Layout()  // 开始递归
  ├─ LayoutBlockFlow::Layout()
  │   ├─ ComputeLogicalWidth()  // 计算宽度
  │   ├─ for each child:
  │   │   ├─ MarginCollapsing()
  │   │   ├─ SetLogicalTop()
  │   │   └─ child->Layout()  ← 递归
  │   └─ ComputeLogicalHeight()  // 计算高度
  ├─ LayoutFlexibleBox::Layout()
  │   ├─ PerformLayout()  // flex 布局算法
  │   └─ for each item: item->Layout()
  └─ LayoutGrid::Layout()
      ├─ ComputeGridLayout()  // grid 布局算法
      └─ for each item: item->Layout()

// Layout 完成
SetLifecycleState(LayoutClean)
```

### 链路 6：从 LayoutTree 到绘制命令

```
LocalFrameView::Paint(GraphicsContext&)
  ↓
PaintController::PaintArtifact()
  ├─ LayoutView::Paint()
  │   └─ PaintInternal()
  │       ├─ Paint children with different phases:
  │       │   ├─ BackgroundClip
  │       │   ├─ BackgroundColor
  │       │   ├─ Foreground (递归绘制子元素)
  │       │   └─ Outline
  │       └─ for each child: child->Paint()  ← 递归
  └─ 生成 Display Items 列表

// 合成到屏幕
PaintController::CommitNewDisplayItems()
  ↓ 生成 PaintArtifact
  ↓ 发送给 Compositor
  ↓ Compositor::Composite()
  ↓ 光栅化并显示到屏幕
```

---

## 关键的触发点和回调

| 事件 | 触发点 | 关键函数 |
|------|--------|---------|
| HTML 解析开始 | `DocumentLoader` 收到数据 | `HTMLDocumentParser::Append()` |
| Token 生成 | 每个 token 完成 | `HTMLTokenizer::NextToken()` |
| DOM 元素创建 | 遇到 StartTag | `HTMLConstructionSite::CreateElement()` |
| 样式更新 | 元素插入 DOM | `StyleEngine::DidAddElement()` |
| Layout 需求 | 样式改变、DOM 修改 | `Document::SetNeedsLayout()` |
| Layout 执行 | 主事件循环、requestAnimationFrame | `Document::UpdateLayout()` |
| LayoutObject 创建 | Layout 前 | `LayoutTreeBuilder::CreateLayout()` |
| Paint 执行 | Layout 完成后 | `LocalFrameView::Paint()` |
| 屏幕更新 | 每帧 | `Compositor::Composite()` |

---

## 性能关键点

### 重排（Reflow）
重排发生在以下情况：
- DOM 修改（添加、删除、修改元素）
- 样式改变（会影响尺寸/位置的属性）
- 窗口大小改变
- `offsetWidth`、`scrollHeight` 等属性访问
- `layout()` 方法调用

```
避免重排的技巧：
1. 批量修改 DOM（使用 DocumentFragment）
2. 缓存 layout 信息（避免频繁读取）
3. 使用 transform/opacity（不触发重排）
4. 使用 RequestAnimationFrame 批量更新
```

### 代码示例：触发重排的操作
```javascript
// ❌ 坏：每次都触发重排
for (let i = 0; i < 100; i++) {
  element.style.width = (i * 10) + 'px';  // 重排
}

// ✅ 好：只 reflow 一次
element.style.width = '1000px';

// ❌ 坏：读取 offsetWidth 触发重排
for (let i = 0; i < 100; i++) {
  console.log(element.offsetWidth);  // 每次重排
}

// ✅ 好：缓存值
const width = element.offsetWidth;
console.log(width);
```

---

## 调试工具和技巧

### Chrome DevTools Timeline

1. 打开 DevTools → Performance 标签
2. 点击录制
3. 执行操作
4. 查看时间线：
   - `Parse` - HTML 解析
   - `Evaluate Script` - JavaScript 执行
   - `Recalculate Style` - 样式计算
   - `Layout` - 布局计算
   - `Paint` - 绘制
   - `Composite` - 合成

### 代码中的调试

```cpp
// Blink 源码中的调试
void LayoutObject::Layout() {
  // 打印调试信息
  #if DCHECK_IS_ON()
  DLOG(INFO) << "Layout: " << DebugName() 
             << " size=" << Size()
             << " pos=(" << X() << "," << Y() << ")";
  #endif
  
  // ... layout 逻辑
}
```

---

## 总结

完整的 HTML 解析到 Layout Tree 构建流程涉及 7 个主要阶段：

```
1. HTML 解析入口 (HTML Data → Parser)
   ↓ Tokenization
2. Token 处理 (HTMLTokenizer.NextToken())
   ↓ Tree Building
3. DOM 树构建 (HTMLTreeBuilder.ReceiveToken())
   ↓ Style Calculation
4. 样式计算 (StyleResolver.StyleForElement())
   ↓ LayoutTree Creation
5. Layout 树创建 (LayoutTreeBuilder.CreateLayout())
   ↓ Layout Computation
6. 布局计算 (LayoutObject.Layout())
   ↓ Paint
7. 绘制 (LayoutObject.Paint())
```

每个阶段都涉及递归遍历树结构，确保所有元素都被正确处理。理解这个流程对于优化渲染性能至关重要。

