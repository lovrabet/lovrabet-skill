# SDK 编码规则与 API 约束

> 目标：约束 AI 在前端与 Node 环境中生成 Lovrabet SDK 代码时的用法，杜绝凭借通用经验瞎猜 API 结构。
>
> 适用范围：所有调用 `@lovrabet/sdk` 的场景。

## 何时使用

当任务涉及以下动作时，必须阅读并遵守本指南：
* 初始化 Lovrabet Client
* 编写模型/数据集的查询与写入代码
* 在前端或中台服务中执行自定义 SQL
* 在前端调用 Backend Function (BFF)
* 处理 API 返回的错误或业务异常

## 初始化规则

必须使用 `createClient` 命名导出，禁止使用 `new LovrabetClient()`：

```typescript
import { createClient } from "@lovrabet/sdk";

const client = createClient({
  appCode: "your-app-code",
  accessKey: process.env.LOVRABET_ACCESS_KEY, // 仅在服务端使用
  models: [
    { tableName: "users", datasetCode: "abc123def456", alias: "users" }
  ],
});
```

## 1. 模型查询 (Filter API)

这是操作模型（表）的**最高优** API。

### 强制参数名称
绝不允许用错以下参数名：
* ❌ `fields` -> ✅ `select`
* ❌ `sort` -> ✅ `orderBy`
* ❌ `page` / `limit` -> ✅ `currentPage` / `pageSize`

### 强制操作符
`where` 条件中**禁止直接写值**，必须使用操作符：
* ❌ `where: { status: 'active' }`
* ✅ `where: { status: { $eq: 'active' } }`

支持的操作符：`$eq`, `$ne`, `$gte`, `$lte`, `$gt`, `$lt`, `$contain`, `$startWith`, `$endWith`, `$in`。

逻辑组合：
```typescript
where: {
  $and: [
    { age: { $gte: 18 } },
    { $or: [{ status: { $eq: "pending" } }, { status: { $eq: "processing" } }] }
  ]
}
```

### 多表关联（自动 JOIN）
* 仅支持 1:1 或 N:1
* 关联字段引用必须使用**表名**（如 `profile.username`），不能用 datasetCode。
* 只有提前通过 MCP 分析出有外键关联的，才能这么写。

```typescript
// 示例：查询文章及其关联的作者信息
const result = await client.models.article.filter({
  select: ["id", "title", "profile.username"],
  where: { "profile.is_signed": { $eq: true } }
});
```

## 2. 自定义 SQL (SQL API)

### 强制返回值处理
SDK 中的 SQL API 返回的是**业务数据层**，包含 `execSuccess` 和 `execResult`：
* **必须**判断 `execSuccess`，不能直接读结果。
* 这与 BFF 环境中直接返回数组（不带 execResult）有本质区别，**不要混用**。

```typescript
// ✅ 前端/Node SDK 调用 SQL：
const data = await client.sql.execute<MyRowType>({
  sqlCode: "fc8e7777-06e3847d",
  params: { userId: "123" },
});

if (data.execSuccess && data.execResult) {
  console.log(data.execResult); // T[]
} else {
  console.error("业务级SQL执行失败");
}
```

## 3. 后端函数 (BFF API)

### 强制返回值处理
调用 BFF 返回的直接是你在脚本中 `return` 的业务数据对象。
* 这里**没有** `execSuccess` 或 `execResult`。

```typescript
// ✅ 调用 BFF
const dashboard = await client.bff.execute<DashboardData>({
  scriptName: "getUserDashboard",
  params: { userId: "123" },
});

// 直接使用业务数据
console.log(dashboard.userCount);
```

## 异常处理底线

所有通过 `client` 发起的网络调用都可能抛出 HTTP 级别的错误，AI 必须养成使用 `try...catch` 包裹代码的习惯，并识别 `LovrabetError`。

```typescript
import { LovrabetError } from "@lovrabet/sdk";

try {
  const result = await client.models.users.getOne(id);
} catch (error) {
  if (error instanceof LovrabetError) {
    console.error("HTTP/框架级错误:", error.message, error.code);
  } else {
    console.error("其他运行时错误:", error);
  }
}
```

## AI 自检清单
每次生成 SDK 相关代码时，AI 应隐式自问：
* [ ] 是否把 `select` 写成了 `fields`？
* [ ] 是否把 `orderBy` 写成了 `sort`？
* [ ] `where` 对象里是否老老实实带了 `$eq` 等操作符？
* [ ] 处理 SQL 的返回值时，判断了 `execSuccess` 吗？
* [ ] 处理 BFF 的返回值时，是不是直接使用了业务数据？
* [ ] 加入了 `try...catch` 块防止整个应用崩溃吗？
