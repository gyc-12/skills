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
PRD/需求 → [Step 1: 代码事实收集] → [Step 2: 输出事实表] → [Step 3: 基于事实表判断]
                                                                              ↓
                                                                  [Step 3.5: 复杂度判定]
                                                                   ↙ Simple        ↘ Complex
[Step 4: 撰写 TRD (精简)]    [Step 4: 撰写 TRD (完整)]
                    ↘       ↙
              [Step 5: 质量自检]
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

## Step 3.5: 复杂度判定

在开始写 TRD 前，根据需求规模选择输出深度：

| 判定维度 | Simple | Complex |
|----------|--------|---------|
| 涉及组件数 | 单个组件 | 多个组件协作 |
| 数据模型 | 无新增 / 复用已有 | 新增数据模型或实体 |
| 外部集成 | 无 | 有外部 API/服务集成 |
| 开发周期 | < 1 天 | > 1 天 |
| 已有模式 | 完全沿用已有模式 | 需要设计新模式 |

**Simple 模式**输出章节：1-7节 + 16-18节（架构概览 → 不做事项 + 风险 → 确认清单）
**Complex 模式**输出全部章节：1-18节（额外包含数据模型、API设计、组件交互、集成点、测试策略、部署、可观测性、环境配置）

> 如果不确定，选 Complex。过度设计可以通过删除章节修复；信息缺失则无法修补。

---

## Step 4: 撰写 TRD

基于前三步的代码事实和判断，按 [trd-template.md](trd-template.md) 撰写 TRD。
根据不同复杂度，模板中带 `<!-- COMPLEX ONLY -->` 标记的章节选择性填写。

### 输出路径
保存到 `docs/plans/YYYY-MM-DD-<feature>-trd.md`

### Simple 模式（必填章节）

#### 4.1 架构概览
2-3段话描述整体方案思路和已有模式复用情况。

#### 4.2 现有代码依据
设计方案**依据了代码库中哪些已有实现**，引用具体文件路径和符号名（来自事实表）。

#### 4.3 数据流
从数据入口到出口，画出完整的数据流转路径。

#### 4.4 技术选型
每项选型必须在事实表中有已有代码依据。

#### 4.5 关键决策
每个决策给出理由和备选方案（及弃用原因）。

#### 4.6 改动边界
精确到文件级别的改动清单：新增文件、修改文件、需关注但不改动的文件。

#### 4.7 不做事项（Non-Goals）
明确列出**本次不做**的事情及拒绝理由，防止 scope creep。

### Complex 模式（额外章节）

在 Simple 基础上增加：数据模型设计（§8）、API/接口设计（§9）、组件交互（§10）、集成点（§11）、测试策略（§12）、部署方案（§13）、可观测性（§14）、环境配置（§15），以及更详细的风险分析（§16.1-16.3）。

---

## Step 5: 质量自检

TRD 写完后，用以下清单逐项自查（对照 [trd-template.md](trd-template.md) 第18节确认清单）：

```
Simple 模式自检：
- [ ] 每个设计决策在事实表中都有对应的代码依据？
- [ ] 数据流完整可追溯，入口→处理→出口无断点？
- [ ] 技术选型每项都有已有代码依据？
- [ ] 改动边界精确到文件级别，没有"等""相关文件"之类的模糊表述？
- [ ] 不做事项有明确理由？
- [ ] 验证方法可执行、可复现？
- [ ] 是否有破坏概念完整性的风险点？

Complex 模式额外自检：
- [ ] 数据模型支持 PRD 中所有用户故事？
- [ ] API 设计覆盖所有用户旅程？
- [ ] 组件交互流程与已有代码调用关系一致？
- [ ] 部署方案、可观测性、环境配置均已规划？
```

---

## TRD 生命周期

| 状态 | 含义 | 何时进入 |
|------|------|----------|
| **Draft** | 撰写中 | TRD 开始编写 |
| **Confirmed** | 设计已确认，可开发 | Step 5 自检通过 + 人工确认 |
| **In Progress** | 开发进行中 | 开始按 TRD 实现代码 |
| **Completed** | 功能已上线 | 代码合入主分支并部署 |
| **Archived** | 已归档 | 移至 `docs/archive/` |

---

## TRD 完成后

### Simple 项目
> TRD 完成，将其保存为 `docs/plans/YYYY-MM-DD-<feature>-trd.md`，状态设为 **Confirmed**。
> 下一步：按 TRD 的改动边界和数据流直接进入开发。

### Complex 项目
> TRD 完成，将其保存为 `docs/plans/YYYY-MM-DD-<feature>-trd.md`，状态设为 **Draft**。
> 建议进行设计评审后再设为 **Confirmed** 进入开发。
> 复杂项目建议按 TRD 中的单一职责原则拆分为多个独立 session 实现。

---

## 核心原则

这些原则来自真实项目中的教训，贯穿整个 TRD 编写过程：

1. **概念完整性优先**：新增代码必须与代码库已有风格相洽。宁可不做新功能，也不能引入风格分裂。
2. **事实先于设计**：看不到代码事实就不要动笔写方案。
3. **单一职责拆解**：一份 PRD 对应多份 TRD 切片，每个切片是一个独立 session 的开发单元。
4. **人定方向，agent 做执行**：TRD 的方向由人把控，agent 负责填充细节和生成代码。
5. **双视角 review**：写完代码后，用系统视角（架构影响）和用户视角（体验变化）双重审视。
6. **自适应深度**：简单项目不过度设计，复杂项目不遗漏关键决策。

---

## 其他资源

- [fact-table-template.md](fact-table-template.md) — 事实表模板，Step 2 使用
- [trd-template.md](trd-template.md) — TRD 输出模板，Step 4 使用
- [examples.md](examples.md) — 完整示例，展示一个真实场景的完整流程
