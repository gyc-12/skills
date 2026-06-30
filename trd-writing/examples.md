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

## Step 4: TRD（精简版示例）

### 2. 现有代码依据

| 引用依据 | 文件路径 | 与本需求的关系 |
|----------|----------|---------------|
| FilterBar 组件 | src/components/FilterBar.tsx | 已有通用筛选栏，直接复用 |
| UserRole 枚举 | src/types/user.ts:5 | 筛选选项的唯一数据源 |
| useUserList hook | src/hooks/useUserList.ts | 数据获取层，需扩展参数 |
| ProductListPage 参考 | src/pages/products/ | 同类页面已集成 FilterBar，参考其集成方式 |

### 3. 数据流设计

```
[用户在 FilterBar 选择角色]
  → [UserListPage 将 role 写入 URL query params]
    → [useUserList 读取 role 参数，传入 fetchUsers]
      → [userApi.fetchUsers 附加 ?role=xxx 到 GET /api/users]
        → [API 返回筛选后的用户列表]
          → [UserTable 渲染]
```

### 4. 改动边界

**新增文件**：无

**修改文件**：
| 文件路径 | 修改范围 | 修改原因 |
|----------|----------|----------|
| src/hooks/useUserList.ts | 入参增加 role?: UserRole | 传递筛选条件 |
| src/api/userApi.ts | fetchUsers 参数增加 role | 透传到 API |
| src/pages/users/UserListPage.tsx | 引入 FilterBar，连接状态 | 提供筛选交互 |

### 5. 不做事项

| 不做事项 | 拒绝理由 |
|----------|----------|
| 多条件组合筛选 | 本次需求仅角色单一维度 |
| 筛选条件持久化到 URL 之外 | 当前 URL query 方案已满足需求 |
| FilterBar 组件修改 | 通用组件，不应为单一页面定制 |

---

## Step 5: 质量自检

```
TRD 质量自检清单：
- [x] 每个设计决策在事实表中都有对应的代码依据？
      → FilterBar 复用依据 rows 1,7；UserRole 依据 row 5
- [x] 数据流完整可追溯，入口→处理→出口无断点？
      → FilterBar → URL params → useUserList → fetchUsers → API → UserTable
- [x] 改动边界精确到文件级别？
      → 3 个文件修改，0 个新增文件
- [x] 不做事项有明确理由？
      → 见上表
- [x] 验证方法可执行、可复现？
- [x] 新增代码与已有代码的交互方式是否明确？
      → FilterBar 通过组合方式集成，useUserList 通过参数扩展
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
