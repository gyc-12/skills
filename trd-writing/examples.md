# 使用示例

> 场景：给用户管理页面添加"按角色筛选"功能。以下展示完整的 TRD 编写流程。

---

## Step 1 & 2: 事实收集 + 事实表

假设项目中已有 `UserListPage` 组件，通过 CodeGraph 查询到以下事实：

### 事实表

| # | 符号 | 文件 | 行号 | 当前职责 | 调用方 | 被调用方 | 数据字段来源 |
|---|------|------|------|----------|--------|----------|-------------|
| 1 | UserListPage | src/pages/users/UserListPage.tsx | 12-89 | 渲染用户列表，管理分页状态 | AppRouter (L45) | useUserList, UserTable, Pagination | — |
| 2 | useUserList | src/hooks/useUserList.ts | 8-56 | 封装用户列表数据获取与缓存 | UserListPage (L15) | userApi.fetchUsers | — |
| 3 | userApi.fetchUsers | src/api/userApi.ts | 23-45 | 调用 GET /api/users 接口 | useUserList (L22) | httpClient.get | — |
| 4 | UserTable | src/components/UserTable.tsx | 10-67 | 渲染用户数据表格 | UserListPage (L52) | UserRow | — |
| 5 | UserRole | src/types/user.ts | 5-5 | 用户角色枚举定义 | userApi.fetchUsers, UserTable, UserRow | — | ✅ **唯一来源** src/types/user.ts:5 |
| 6 | httpClient | src/utils/httpClient.ts | 3-40 | 统一 HTTP 请求封装，注入 token | userApi.fetchUsers | axios | — |
| 7 | FilterBar | src/components/FilterBar.tsx | 8-34 | 通用筛选栏容器组件 | ProductListPage, OrderListPage | — | — |

### 调用关系图

```
AppRouter
  └─→ UserListPage
       ├─→ useUserList
       │    └─→ userApi.fetchUsers
       │         └─→ httpClient
       ├─→ UserTable
       │    └─→ UserRow
       └─→ Pagination

已有可参考：
ProductListPage → FilterBar  ← 同类页面已使用
OrderListPage  → FilterBar  ← 同类页面已使用
```

### 数据流路径

```
[URL query params] → [useUserList 解析参数] → [userApi.fetchUsers 发起请求] → [UserTable 渲染响应数据]
```

---

## Step 3: 判断

### 可复用模块

| 模块/符号 | 复用方式 | 说明 |
|-----------|----------|------|
| FilterBar | 组合 | ProductListPage/OrderListPage 已在使用，直接复用 |
| UserRole (enum) | 引用 | 作为筛选选项的数据源 |

### 需改动点

| 文件 | 符号 | 改动原因 | 改动范围 |
|------|------|----------|----------|
| src/hooks/useUserList.ts | useUserList | 增加 role 参数传递 | 入参、fetchUsers 调用处 |
| src/api/userApi.ts | fetchUsers | API 支持 role 查询参数 | 参数类型和传递逻辑 |
| src/pages/users/UserListPage.tsx | UserListPage | 集成 FilterBar 组件 | 新增 FilterBar 渲染，连接筛选状态到 useUserList |

### 禁止改动区域

| 模块/文件 | 禁止原因 |
|-----------|----------|
| src/types/user.ts | UserRole 枚举是唯一数据来源，多处依赖 |
| src/utils/httpClient.ts | 通用基础设施，无关本需求 |
| src/components/FilterBar.tsx | 通用组件，已在多处稳定使用 |

### 数据唯一来源确认

| 数据字段 | 唯一定义位置 | 是否安全 |
|----------|-------------|----------|
| UserRole | src/types/user.ts:5 | ✅ |
| role 查询参数 | 本需求新增 | ⚠️ 需确保命名与后端一致 |

---

## Step 3.5: 复杂度判定

**判定结果：Simple**

| 判定维度 | 情况 |
|----------|------|
| 涉及组件数 | 单个页面（UserListPage） |
| 数据模型 | 无新增，复用 UserRole 枚举 |
| 外部集成 | 无新增外部服务 |
| 已有模式 | FilterBar 已有多处使用先例 |

→ 选 **Simple 模式**，输出 §1-7 + §16-18。

---

## Step 4: TRD（按 Simple 模板输出）

**输出路径**：`docs/plans/2026-06-30-user-role-filter-trd.md`
**复杂度**：Simple
**状态**：Draft → Confirmed

### 1. 架构概览

在 UserListPage 中集成已有的 FilterBar 通用组件，将筛选状态通过 URL query params 传递，复用 useUserList → userApi.fetchUsers → httpClient 的已有数据获取链路。整个过程不引入新组件、不修改通用组件、不新增数据模型。

方案选择理由：ProductListPage 和 OrderListPage 已用相同方式集成 FilterBar，沿用已有模式保证概念一致性。

### 2. 现有代码依据

| 引用依据 | 文件路径 | 与本需求的关系 |
|----------|----------|---------------|
| FilterBar 组件 | src/components/FilterBar.tsx | 已有通用筛选栏，直接复用 |
| UserRole 枚举 | src/types/user.ts:5 | 筛选选项的唯一数据源 |
| useUserList hook | src/hooks/useUserList.ts | 数据获取层，需扩展参数 |
| ProductListPage 参考 | src/pages/products/ProductListPage.tsx | 同类页面已集成 FilterBar，参考其集成方式 |

### 3. 数据流设计

**入口**：用户在 FilterBar 选择角色
**数据结构**：`UserRole` 枚举值（来自 src/types/user.ts:5）

```
[用户在 FilterBar 选择角色]
  → [UserListPage 将 role 写入 URL query params]
    → [useUserList 读取 role 参数，传入 fetchUsers]
      → [userApi.fetchUsers 附加 ?role=xxx 到 GET /api/users]
        → [API 返回筛选后的用户列表]
          → [UserTable 渲染]
```

**出口**：UserTable 渲染筛选后的用户列表

### 4. 技术选型

| 维度 | 选择 | 理由 | 已有代码依据 |
|------|------|------|-------------|
| 筛选组件 | FilterBar | 已有通用组件 | src/components/FilterBar.tsx |
| 状态传递 | URL query params | 与已有分页参数模式一致 | useUserList (params 模式) |
| 数据获取 | useUserList → fetchUsers | 已有数据链路 | src/hooks/useUserList.ts |

### 5. 关键决策

**决策 1：筛选参数传递方式**
- **决策**：通过 URL query params 传递 role 参数
- **理由**：与已有分页参数（page, pageSize）传递方式一致，保持 URL 可分享、可回退
- **备选方案**：组件内部 state — 弃用，因与已有分页参数模式不一致，且 URL 不可分享
- **代码依据**：useUserList 已有 query param 解析逻辑

**决策 2：FilterBar 集成方式**
- **决策**：组合方式集成，不修改 FilterBar 源码
- **理由**：FilterBar 是通用组件，在 ProductListPage/OrderListPage 中已稳定使用
- **备选方案**：修改 FilterBar 增加角色专属逻辑 — 弃用，会破坏通用性，影响其他页面
- **代码依据**：事实表 row 7，FilterBar 已被两处调用

### 6. 改动边界

**新增文件**：无

**修改文件**：
| 文件路径 | 修改范围 | 修改原因 |
|----------|----------|----------|
| src/hooks/useUserList.ts | 入参增加 role?: UserRole | 传递筛选条件 |
| src/api/userApi.ts | fetchUsers 参数增加 role | 透传到 API |
| src/pages/users/UserListPage.tsx | 引入 FilterBar，连接状态 | 提供筛选交互 |

**需关注但不改动的文件**：
| 文件路径 | 关注原因 |
|----------|----------|
| src/components/FilterBar.tsx | 确认其接口满足角色筛选需求 |
| src/types/user.ts | 确认 UserRole 枚举值覆盖所有筛选选项 |

### 7. 不做事项

| 不做事项 | 拒绝理由 | 如果后续要做，前置条件 |
|----------|----------|----------------------|
| 多条件组合筛选 | 本次需求仅角色单一维度 | PRD 明确多条件需求后 |
| 筛选条件持久化到 URL 之外 | URL query 方案已满足需求 | 出现跨页面筛选状态共享需求时 |
| FilterBar 组件修改 | 通用组件，不应为单一页面定制 | 所有使用方达成一致后统一升级 |

### 16. 风险与缓解

| 风险 | 影响 | 可能性 | 缓解措施 |
|------|------|--------|----------|
| 破坏概念完整性 | 中 | 低 | FilterBar 复用方式与 ProductListPage 完全一致 |
| 数据源分裂 | 低 | 低 | UserRole 枚举定义在唯一位置 src/types/user.ts |

### 17. 验证方法

**数据流断点检查**：
| 检查点 | 预期行为 | 如何触发 |
|--------|----------|----------|
| FilterBar 渲染 | 角色选项来自 UserRole 枚举 | 打开 UserListPage |
| URL 参数同步 | 选择角色后 URL 出现 ?role=xxx | 选择角色，观察地址栏 |
| API 请求 | GET /api/users?role=xxx | 打开 Network 面板 |
| 列表刷新 | 表格仅显示所选角色用户 | 切换角色选项 |

**回归范围**：
- 不受影响：UserTable 渲染、Pagination 分页、httpClient 请求封装
- 重点回归：useUserList 的已有调用方（确认新增可选参数不影响已有行为）

### 18. 确认清单

- [x] 架构方案已确认
- [x] 技术选型已确认（每项有代码依据）
- [x] 每个设计决策都有事实表依据
- [x] 数据流完整可追溯
- [x] 改动边界精确到文件级别（3个修改，0个新增）
- [x] 不做事项有明确理由
- [x] 验证方法可执行、可复现
- [x] 无破坏概念完整性的风险点

**TRD 状态**：Draft → **Confirmed** ✅

---

## Step 5: 质量自检

```
Simple 模式自检：
- [x] 每个设计决策在事实表中都有对应的代码依据？
      → FilterBar 复用依据 row 7；UserRole 依据 row 5；URL params 模式依据 row 2
- [x] 数据流完整可追溯，入口→处理→出口无断点？
      → FilterBar → URL params → useUserList → fetchUsers → API → UserTable
- [x] 技术选型每项都有已有代码依据？
      → 筛选组件/状态传递/数据获取均有事实表依据
- [x] 改动边界精确到文件级别，没有模糊表述？
      → 3 个文件修改，0 个新增
- [x] 不做事项有明确理由？
      → 3 项不做事项，每项有理由和前置条件
- [x] 验证方法可执行、可复现？
      → 4 个数据流断点 + 回归范围明确
- [x] 是否有破坏概念完整性的风险点？
      → 无，FilterBar 复用方式与 ProductListPage 完全一致
```

---

## 反面案例（常见错误）

以下是通过本 skill 要**避免**的做法：

### ❌ 错误做法：跳过事实收集直接写 TRD

```
"需求是加角色筛选，我觉得可以在 UserListPage 里加一个下拉框，
然后写一个新的 useUserFilter hook 来管理筛选状态，
再在 API 层加个参数就行了。"
```

**问题**：
1. 没发现 FilterBar 组件已存在 → 重复造轮子
2. 新写 useUserFilter hook → 违背 useUserList 已有的参数设计模式
3. 没确认 UserRole 的唯一来源 → 可能在多处重复定义角色枚举

### ✅ 正确做法：走完五步流程

即上文展示的完整流程。每个决策都有代码事实支撑，不会引入概念不一致的设计。
