---
name: trd-writing
description: Write Technical Requirement Documents (TRD) grounded in existing codebase facts. Forces CodeGraph exploration before any design work, produces fact tables mapping symbols/files/responsibilities, and ensures TRDs respect architectural integrity and conceptual consistency with the codebase. Use when the user mentions writing TRDs, technical design documents, translating PRDs to TRDs, 技术方案, 技术设计, or planning non-trivial code changes.
---

# TRD Writing — 基于代码事实的技术方案编写

## 核心理念

Coding agent 本质是多人接力协作。每一轮对话等同于换了一个"人"来接手。如果不加约束，agent 写的 TRD 极易脱离代码库实际实现风格和架构风格，破坏系统的**概念完整性**（《人月神话》）。

本 skill 强制在写 TRD 之前先通过 CodeGraph 获取代码事实，让设计"长在"已有代码之上，而非凭空臆想。

## 工具适配

本 skill 以 CodeGraph 为参考工具，但核心要求是"获取代码事实"，而非绑定特定工具。当 CodeGraph 不可用时，使用以下等价能力替代：

| CodeGraph 能力 | 等价替代方案 |
|---|---|
| 按业务概念查询符号 | `SearchCodebase` + `SearchSymbol` |
| 查询调用方/被调用方 | `SearchSymbol` (relation: calls/called_by) |
| 查询文件路径和行号 | `Grep` + `Glob` |
| 查询数据字段定义 | `Grep` 搜索类型定义文件 |

## 工作流概览

```
PRD/需求 → [Step 1: 代码事实收集] → [Step 2: 输出事实表] → [Step 3: 基于事实表判断] → [Step 4: 撰写 TRD] → [Step 5: 质量自检]
```

**重要**：不允许跳过任何步骤。TRD 是"概念完整性"的守门人。

---

## Step 1: 代码事实收集

在构思任何方案之前，必须先跑通代码事实。优先使用 CodeGraph，不可用时使用 Qoder 内置工具（SearchCodebase / SearchSymbol / Grep）等价替代：

### 1.1 查询相关业务概念

```
查询与 [需求关键词] 相关的：
- 页面/路由/入口组件
- service 层 / 业务逻辑模块
- 数据模型 / schema / 类型定义
- API 接口 / 数据获取层
```

### 1.2 查询调用关系

对每个关键符号，向上追溯调用方、向下追溯被调用方，形成调用链全貌。

### 1.3 查询数据来源

对需求涉及的数据字段，追溯其定义位置、流转路径、持久化位置。

> **约束**：此步骤只收集事实，不做任何判断和设计。不要在这步就开始想方案。

---

## Step 2: 输出事实表

基于 Step 1 的查询结果，输出结构化的"事实表"。格式见 [fact-table-template.md](fact-table-template.md)。

事实表必须包含以下字段：

| 列 | 说明 |
|---|---|
| 符号 | 类名/函数名/变量名 |
| 文件 | 文件绝对路径 |
| 行号 | 符号定义所在行号范围 |
| 当前职责 | 该符号当前承担的业务职责（一句话） |
| 调用方 | 哪些符号调用了它 |
| 被调用方 | 它调用了哪些符号 |
| 数据字段来源 | 若涉及数据字段，标注其定义位置和唯一来源 |

---

## Step 3: 基于事实表判断

基于事实表，做出四个维度的判断，填入事实表下方的判断区：

### 3.1 可复用模块
标记哪些现有模块可以直接复用，说明复用的具体方式（调用/继承/组合）。

### 3.2 需改动点
标记哪些文件/符号需要修改，说明改动原因和改动范围。

### 3.3 禁止改动区域
标记**绝对不能动**的模块：
- 作为唯一数据来源的字段定义
- 被多处依赖的核心工具函数
- 已在其他需求中定版的模块

### 3.4 数据唯一来源确认
确认需求涉及的数据字段的单一数据源（Single Source of Truth），防止数据源分裂。

---

## Step 4: 撰写 TRD

基于前三步的代码事实和判断，开始写 TRD。使用 [trd-template.md](trd-template.md) 中的模板。

TRD 必须包含以下五个要素：

### 4.1 现有代码依据
说明设计方案**依据了代码库中哪些已有实现**，引用具体文件路径和符号名。

### 4.2 数据流
从数据入口到出口，画出完整的数据流转路径：
- 数据从哪入（用户输入/API 响应/数据库查询）
- 经过哪些处理节点（函数/组件）
- 最终落到哪（渲染/持久化/传递给下游）

### 4.3 改动边界
精确到文件级别的改动清单：
- 新增文件列表
- 修改文件列表及修改范围
- 不改动但需要关注的文件

### 4.4 不做事项（Non-Goals）
明确列出**本次不做**的事情，防止 scope creep：
- 拒绝理由
- 如果后续要做，前置条件是什么

### 4.5 验证方法
如何验证实现是否符合 TRD：
- 关键数据流断点检查
- 回归范围说明
- 手动验证步骤

---

## Step 5: 质量自检

TRD 写完后，用以下清单逐项自查：

```
TRD 质量自检清单：
- [ ] 每个设计决策在事实表中都有对应的代码依据？
- [ ] 数据流完整可追溯，入口→处理→出口无断点？
- [ ] 改动边界精确到文件级别，没有"等""相关文件"之类的模糊表述？
- [ ] 不做事项有明确理由？
- [ ] 验证方法可执行、可复现？
- [ ] 新增代码与已有代码的交互方式是否明确（调用、继承、组合）？
- [ ] 是否有破坏概念完整性的风险点（引入与现有风格不一致的模式）？
```

---

## 核心原则

这些原则来自真实项目中的教训，贯穿整个 TRD 编写过程：

1. **概念完整性优先**：新增代码必须与代码库已有风格相洽。宁可不做新功能，也不能引入风格分裂。
2. **事实先于设计**：看不到代码事实就不要动笔写方案。
3. **单一职责拆解**：一份 PRD 对应多份 TRD 切片，每个切片是一个独立 session 的开发单元。
4. **人定方向，agent 做执行**：TRD 的方向由人把控，agent 负责填充细节和生成代码。
5. **双视角 review**：写完代码后，用系统视角（架构影响）和用户视角（体验变化）双重审视。

---

## 其他资源

- [fact-table-template.md](fact-table-template.md) — 事实表模板，Step 2 使用
- [trd-template.md](trd-template.md) — TRD 输出模板，Step 4 使用
- [examples.md](examples.md) — 完整示例，展示一个真实场景的完整流程
