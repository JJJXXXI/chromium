# RenderingNG Paint & Compositor 完整资源索引

## 📚 文档目录

本资源库包含三份完整的文档，针对不同的学习需求：

### 1. 📖 [RENDERING_NG_PAINT_COMPOSITOR_GUIDE.md](RENDERING_NG_PAINT_COMPOSITOR_GUIDE.md)
**完整综合指南** - 深度讲解架构和流程

**包含内容**:
- Paint 阶段详细分析
  - Paint 入口和流程
  - 关键阶段划分
  - PaintInfo 结构
- DisplayItem 和 PaintArtifact 详解
  - DisplayItem 的类型和内容
  - PaintArtifact 的组成
  - PaintChunk 的结构
  - PaintController 的工作流程
- Compositor 处理机制
  - PaintArtifactCompositor 的角色
  - Compositor 处理步骤
  - PaintChunksToCcLayer 的转换
- Rasterization 完整流程
  - 光栅化的定义和流程
  - DisplayItemList 和 RasterInvalidator
  - 光栅化的关键优化
- 关键类和方法表格
- 完整的数据流向图
- 文件位置速查

**适用于**: 想要全面理解 RenderingNG Paint 和 Compositor 架构的开发者

---

### 2. 💻 [RENDERING_NG_PAINT_COMPOSITOR_CODE_EXAMPLES.md](RENDERING_NG_PAINT_COMPOSITOR_CODE_EXAMPLES.md)
**代码示例和实战指南** - 包含详细的代码片段

**包含内容**:
- Paint 代码示例
  - 基本 Paint 流程实现
  - PaintController 使用示例
  - Cull Rect 优化示例
- DisplayItem 示例
  - 自定义 DisplayItem 类
  - 创建和使用 PaintRecord
  - PaintArtifact 查询
- Compositor 代码示例
  - PaintArtifactCompositor 更新流程
  - PaintChunksToCcLayer 转换
  - ContentLayerClientImpl 实现
- Rasterization 示例
  - DisplayItemList Rasterization
  - Tile 光栅化流程
  - RasterInvalidator 失效生成
- 关键方法速查表
  - PaintController 常用方法
  - cc::DisplayItemList 常用方法
  - RasterInvalidator 常用方法
- 调试技巧和工具
  - 启用调试标志
  - 添加调试输出
  - DevTools 和 about:tracing 使用
  - 关键断点位置
- 性能分析实例
  - Paint 时间长的诊断
  - Rasterization 耗时的诊断
- 常见代码模式

**适用于**: 需要实际代码示例和性能调试的工程师

---

### 3. ⚡ [RENDERING_NG_QUICK_REFERENCE.md](RENDERING_NG_QUICK_REFERENCE.md)
**快速参考卡片** - 快速查找信息

**包含内容**:
- 核心概念速览（数据流图）
- 关键类一览表（按角色分类）
- 关键文件位置速查
- 完整数据流（Paint → Composite → Rasterization）
- 数据结构变换图
- 性能优化核心（最佳实践）
- 常见问题排查（问答格式）
- 时间线示例
- 学习路径（初级→中级→高级→实战）
- 常用编译和测试命令
- 相关链接和版本信息
- 提示和最佳实践

**适用于**: 需要快速查找信息、已有基础知识的开发者

---

## 🎯 快速开始

### 根据你的情况选择合适的文档

**我是新手，想全面了解**
→ 从 [RENDERING_NG_PAINT_COMPOSITOR_GUIDE.md](RENDERING_NG_PAINT_COMPOSITOR_GUIDE.md) 开始

**我想看代码示例和调试方法**
→ 去 [RENDERING_NG_PAINT_COMPOSITOR_CODE_EXAMPLES.md](RENDERING_NG_PAINT_COMPOSITOR_CODE_EXAMPLES.md)

**我想快速查找信息**
→ 使用 [RENDERING_NG_QUICK_REFERENCE.md](RENDERING_NG_QUICK_REFERENCE.md)

**我想深入某个特定领域**
→ 使用下面的 "按主题索引" 部分

---

## 📑 按主题索引

### Paint 阶段

| 主题 | 文档位置 | 内容 |
|---|---|---|
| Paint 基础 | [指南 - Paint 阶段](#paint-阶段) | Paint 入口、流程、关键阶段 |
| Paint 代码 | [示例 - Paint 代码示例](#paint-代码示例) | 实际代码片段 |
| Paint 优化 | [快速参考 - Paint 优化](#%EF%B8%8F-paint-优化) | 性能优化技巧 |

### DisplayItem

| 主题 | 文档位置 | 内容 |
|---|---|---|
| DisplayItem 结构 | [指南 - DisplayItem 的结构](#displayitem-的结构) | 类型、内容、属性 |
| DisplayItem 代码 | [示例 - DisplayItem 示例](#displayitem-示例) | 创建和使用 |
| PaintController | [指南 - PaintController](#paintcontroller-paint-的关键管理器) | 工作流程和方法 |

### PaintArtifact

| 主题 | 文档位置 | 内容 |
|---|---|---|
| PaintArtifact 结构 | [指南 - PaintArtifact 的结构](#paintartifact-的结构) | 组成和关系 |
| PaintChunk | [指南 - PaintChunk 的结构](#paintchunk-的结构) | DisplayItems 分组 |
| 查询 PaintArtifact | [示例 - PaintArtifact 的查询](#3-paintartifact-的查询) | 遍历和访问 |

### Compositor

| 主题 | 文档位置 | 内容 |
|---|---|---|
| 合成概述 | [指南 - Compositor 处理](#compositor-处理) | 角色和流程 |
| PaintArtifactCompositor | [指南 - PaintArtifactCompositor 的核心逻辑](#paintartifactcompositor-的核心逻辑) | 更新流程和决策 |
| 合成代码 | [示例 - Compositor 代码示例](#1-paintartifactcompositor-更新) | 实际实现 |
| 格式转换 | [指南 - PaintChunksToCcLayer 的转换](#paintchunkstocccomlayer-的转换) | Blink → cc 转换 |

### Rasterization

| 主题 | 文档位置 | 内容 |
|---|---|---|
| 光栅化概念 | [指南 - Rasterization 的定义](#rasterization-的定义) | 基本概念 |
| DisplayItemList | [指南 - cc::DisplayItemList](#1-ccdisplayitemlist) | 光栅化容器 |
| RasterInvalidator | [指南 - RasterInvalidator](#2-rasterinvalidator) | 失效追踪 |
| 光栅化流程 | [指南 - Rasterization 的流程](#rasterization-的流程) | 完整步骤 |
| 光栅化代码 | [示例 - Rasterization 示例](#rasterization-示例) | 代码实现 |
| 失效生成 | [示例 - RasterInvalidator 生成失效](#3-rasterinvalidator-生成失效区域) | 追踪实现 |

### 数据流

| 主题 | 文档位置 | 内容 |
|---|---|---|
| 完整流程 | [指南 - 数据流向](#数据流向) | 从 LayoutTree 到 Screen |
| 数据变换 | [快速参考 - 数据结构变换](#-数据结构变换) | 结构转换链 |
| 时间线 | [快速参考 - 时间线示例](#%E2%8F%B2%EF%B8%8F-时间线示例) | 各阶段耗时 |

### 关键类和方法

| 主题 | 文档位置 | 内容 |
|---|---|---|
| 所有关键类 | [指南 - 关键类和方法](#关键类和方法) | 完整表格 |
| Paint 类 | [快速参考 - Paint 相关](#paint-相关) | Paint 阶段的类 |
| Composite 类 | [快速参考 - Compositing 相关](#compositing-相关) | 合成阶段的类 |
| cc 类 | [快速参考 - cc 相关](#cc-相关) | cc 侧的类 |
| PaintController 方法 | [示例 - PaintController 常用方法](#paintcontroller-常用方法) | 详细的方法列表 |
| DisplayItemList 方法 | [示例 - cc::DisplayItemList 常用方法](#ccdisplayitemlist-常用方法) | cc 侧方法 |

### 文件位置

| 主题 | 文档位置 | 内容 |
|---|---|---|
| Paint 文件 | [指南 - Paint 相关](#paint-相关) | Paint 代码位置 |
| DisplayItem 文件 | [指南 - DisplayItem 和 PaintArtifact](#displayitem-和-paintartifact) | DisplayItem 相关位置 |
| Composite 文件 | [指南 - Compositor 处理](#compositor-处理) | 合成代码位置 |
| Raster 文件 | [指南 - Rasterization 流程](#rasterization-流程) | 光栅化代码位置 |
| 快速查询 | [快速参考 - 关键文件位置速查](#-关键文件位置速查) | 汇总表格 |

### 调试和性能

| 主题 | 文档位置 | 内容 |
|---|---|---|
| 调试技巧 | [示例 - 调试技巧](#调试技巧) | 调试方法和工具 |
| 性能分析 | [示例 - 性能分析实例](#性能分析实例) | 具体的诊断案例 |
| Paint 优化 | [示例 - Paint 优化](#-paint-优化) | Paint 性能优化 |
| 排查问题 | [快速参考 - 常见问题排查](#-常见问题排查) | 问答式排查 |
| 时间线 | [快速参考 - 时间线示例](#%E2%8F%B2%EF%B8%8F-时间线示例) | 性能时间线 |

### 实战指南

| 主题 | 文档位置 | 内容 |
|---|---|---|
| 代码模式 | [示例 - 常见代码模式](#常见代码模式) | 常用实现模式 |
| 学习路径 | [快速参考 - 学习路径](#-学习路径) | 从入门到精通 |
| 常用命令 | [快速参考 - 常用命令](#-常用命令) | 编译、测试、调试命令 |
| 问题排查 | [快速参考 - 常见问题排查](#-常见问题排查) | 诊断和解决 |

---

## 🔑 关键概念速览

### Paint（绘制）
将 LayoutObjects 转换为绘制指令序列，输出 DisplayItems。
- **发生地点**: Main thread（Blink）
- **输入**: LayoutTree（DOM + CSS）
- **输出**: PaintArtifact（DisplayItems + PaintChunks）
- **文档**: [指南 - Paint 阶段](#paint-阶段)

### DisplayItem（绘制项）
单个绘制单位，包含绘制操作和元数据。
- **作用**: 原子化的绘制操作
- **包含**: PaintOps（实际绘制指令）+ metadata（边界、ID 等）
- **存储**: DisplayItemList
- **文档**: [指南 - DisplayItem 的结构](#displayitem-的结构)

### PaintArtifact（Paint 输出）
Paint 阶段的最终输出，包含所有绘制内容和分组信息。
- **组成**: DisplayItemList + PaintChunks
- **用途**: 提供给 Compositor 进行分层和光栅化
- **缓存**: 可在多个 Paint 循环中复用
- **文档**: [指南 - PaintArtifact 的结构](#paintartifact-的结构)

### Compositor（合成）
将 PaintArtifact 分解为合成层，决定哪些内容使用独立的 cc::Layer。
- **发生地点**: Main thread（Blink）
- **输入**: PaintArtifact
- **输出**: cc::Layers + PropertyTrees
- **优化**: 减少 Tiles、优化合成时间
- **文档**: [指南 - Compositor 处理](#compositor-处理)

### Rasterization（光栅化）
将矢量绘制指令转换为像素数据并上传到 GPU。
- **发生地点**: Compositor thread（cc）
- **输入**: cc::DisplayItemList + Tile bounds
- **输出**: GPU 纹理
- **优化**: 部分光栅化、Tile 缓存、R-tree 加速
- **文档**: [指南 - Rasterization 流程](#rasterization-流程)

---

## 🎓 学习建议

### 初次接触
1. 阅读 [快速参考 - 核心概念速览](#🎯-核心概念速览)
2. 查看 [指南 - 完整的函数调用链](#完整的函数调用链)（如果有）
3. 运行 [快速参考 - 常用命令](#-常用命令) 中的编译命令

### 深入理解
1. 逐章阅读 [完整综合指南](RENDERING_NG_PAINT_COMPOSITOR_GUIDE.md)
2. 查看 [代码示例](RENDERING_NG_PAINT_COMPOSITOR_CODE_EXAMPLES.md) 中的实际实现
3. 参考 [快速参考](RENDERING_NG_QUICK_REFERENCE.md) 中的表格和时间线

### 实战应用
1. 使用 [示例 - 调试技巧](#调试技巧) 设置调试环境
2. 参考 [示例 - 常见代码模式](#常见代码模式) 实现新功能
3. 使用 [快速参考 - 常见问题排查](#-常见问题排查) 诊断问题

### 性能优化
1. 学习 [快速参考 - 性能优化核心](#-性能优化核心) 的优化方法
2. 使用 [示例 - 性能分析实例](#性能分析实例) 的诊断步骤
3. 参考 [快速参考 - 时间线示例](#%E2%8F%B2%EF%B8%8F-时间线示例) 了解瓶颈

---

## 📞 快速问题查找

### 我的问题是...

**关于 Paint 流程**
- Paint 是如何工作的？→ [指南 - Paint 阶段](#paint-阶段)
- Paint 如何缓存？→ [指南 - PaintController 的工作流程](#paintcontroller-的工作流程)
- Paint 时间过长怎么办？→ [快速参考 - Paint 时间过长](#q-paint-时间过长)

**关于 DisplayItem**
- DisplayItem 包含什么？→ [指南 - DisplayItem 的内容](#displayitem-的内容)
- 如何创建自定义 DisplayItem？→ [示例 - 模式 1：添加新的 DisplayItem 类型](#模式-1添加新的-displayitem-类型)
- DisplayItem 和 PaintOp 有什么区别？→ [快速参考 - 常见问题排查](#q-displayitem-和-paintop-的区别)

**关于 Compositor**
- Compositor 做什么？→ [指南 - Compositor 的角色](#compositor-的角色)
- 合成层如何创建？→ [指南 - PaintArtifactCompositor 的核心逻辑](#paintartifactcompositor-的核心逻辑)
- 合成层过多怎么办？→ [快速参考 - Composite 层过多](#q-composite-层过多)

**关于 Rasterization**
- 光栅化何时发生？→ [快速参考 - 常见问题排查](#q-光栅化何时发生)
- 光栅化如何优化？→ [指南 - 光栅化的关键优化](#光栅化的关键优化)
- Rasterization 耗时？→ [快速参考 - Rasterization 耗时](#q-rasterization-耗时)

**关于调试**
- 如何追踪 Paint 性能？→ [示例 - 调试技巧](#调试技巧)
- 如何查看 Paint 结果？→ [示例 - Chrome DevTools 中查看 Paint 性能](#chrome-devtools-中查看-paint-性能)
- 如何启用失效追踪？→ [示例 - 在 Blink 中启用光栅失效追踪](#在-blink-中启用光栅失效追踪)

**关于代码**
- Paint 的关键方法有哪些？→ [示例 - PaintController 常用方法](#paintcontroller-常用方法)
- 如何添加 DisplayItem？→ [示例 - 使用 PaintController 添加 DisplayItem](#2-使用-paintcontroller-添加-displayitem)
- 如何实现光栅化？→ [示例 - DisplayItemList Rasterization](#1-displayitemlist-rasterization)

**关于文件位置**
- Paint 代码在哪里？→ [指南 - Paint 相关](#paint-相关) 或 [快速参考 - Paint 相关](#paint-相关)
- Compositor 代码在哪里？→ [指南 - Compositing 相关](#compositing-相关) 或 [快速参考 - Compositing 相关](#compositing-相关)
- cc 代码在哪里？→ [指南 - cc 侧](#cc-侧) 或 [快速参考 - cc 相关](#cc-相关)

---

## 📊 统计信息

- **总文档数**: 3 个
- **总页面数**: ~60 页
- **代码示例数**: 30+ 个
- **表格数**: 20+ 个
- **图表数**: 10+ 个
- **常见问题**: 15+ 个

---

## 📋 文档使用建议

### 在线阅读
可以直接在 GitHub 或 Chromium 源代码浏览器中查看这些 Markdown 文件。

### 离线查看
```bash
# 克隆或下载文件
git clone <repo>
cd chromium

# 用你喜欢的 Markdown 阅读器打开
code RENDERING_NG_PAINT_COMPOSITOR_GUIDE.md
```

### 快速搜索
所有文档都支持 Markdown 格式，可以使用文本编辑器的搜索功能查找特定内容。

### 更新维护
这些文档基于 Chromium 源代码和最新的 RenderingNG 架构，内容会定期更新。

---

## 🙏 致谢

这份资源基于以下资料编写：
- Chromium 源代码（third_party/blink 和 cc）
- HTML_PARSING_TO_LAYOUT_TREE.md（项目中已有的文档）
- Blink Font Rendering Architecture（项目中已有的文档）
- Chromium Paint Team 的设计文档
- 社区的最佳实践

---

## 📄 许可证

这些文档是 Chromium 项目的一部分，遵循 Chromium 的许可证。

---

**最后更新**: 2026-01-18
**版本**: 1.0
**适用于**: Chromium M125+ 及以上

