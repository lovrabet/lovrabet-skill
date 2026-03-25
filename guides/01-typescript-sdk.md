# TypeScript SDK 使用指南

> **目标**：掌握 Lovrabet TypeScript SDK 的三大核心 API：Filter 高级查询、SQL 自定义查询、BFF 端点调用

## SDK 安装与初始化

```bash
npm install @lovrabet/sdk
# 或
bun add @lovrabet/sdk
```

```typescript
import { createClient } from "@lovrabet/sdk";

const client = createClient({
  appCode: "your-app-code",
  accessKey: process.env.LOVRABET_ACCESS_KEY, // 服务端使用
  models: [
    { tableName: "users", datasetCode: "abc123def456", alias: "users" },
    { tableName: "orders", datasetCode: "xyz789ghi012", alias: "orders" },
  ],
});
```

---

## 1️⃣ Filter 高级查询

`filter()` 是 SDK 中最强大的查询方法，支持复杂条件、字段选择、排序等。

### 基本结构

```typescript
const result = await client.models.users.filter({
  where: {
    /* 查询条件 */
  },
  select: ["id", "name", "email"], // 返回指定字段
  orderBy: [{ createTime: "desc" }], // 排序
  currentPage: 1,
  pageSize: 20,
});

console.log(result.tableData); // 数据列表
console.log(result.total); // 总数
```

### 操作符列表

| 操作符       | 说明         | 示例                                 |
| ------------ | ------------ | ------------------------------------ |
| `$eq`        | 等于         | `{ status: { $eq: 'active' } }`      |
| `$ne`        | 不等于       | `{ status: { $ne: 'deleted' } }`     |
| `$gte`       | 大于等于     | `{ age: { $gte: 18 } }`              |
| `$lte`       | 小于等于     | `{ age: { $lte: 65 } }`              |
| `$gt`        | 大于         | `{ price: { $gt: 100 } }`            |
| `$lt`        | 小于         | `{ price: { $lt: 1000 } }`          |
| `$contain`   | 包含（模糊） | `{ name: { $contain: 'keyword' } }` |
| `$startWith` | 开头匹配     | `{ email: { $startWith: 'admin' } }` |
| `$endWith`   | 结尾匹配     | `{ domain: { $endWith: '.com' } }`   |
| `$in`        | 包含于       | `{ type: { $in: ['A', 'B', 'C'] } }` |

### 逻辑操作符

```typescript
// $and - 所有条件都要满足
where: {
  $and: [{ age: { $gte: 18 } }, { status: { $eq: "active" } }];
}

// $or - 任一条件满足即可
where: {
  $or: [{ status: { $eq: "pending" } }, { status: { $eq: "processing" } }];
}

// 嵌套组合
where: {
  $and: [
    { age: { $gte: 18 } },
    {
      $or: [{ country: { $eq: "中国" } }, { country: { $eq: "美国" } }],
    },
  ];
}
```

### 常见错误

```typescript
// ❌ 错误：直接写值
where: { status: 'active' }
// ✅ 正确：使用 $eq
where: { status: { $eq: 'active' } }

// ❌ 错误：使用 fields
fields: ['id', 'name']
// ✅ 正确：使用 select
select: ['id', 'name']

// ❌ 错误：使用 sort
sort: [{ createTime: 'desc' }]
// ✅ 正确：使用 orderBy
orderBy: [{ createTime: 'desc' }]

// ❌ 错误：使用 page/limit
page: 1, limit: 20
// ✅ 正确：使用 currentPage/pageSize
currentPage: 1, pageSize: 20
```

### 实战示例

```typescript
// 带搜索的用户列表
async function searchUsers(keyword: string, page: number = 1) {
  const result = await client.models.users.filter({
    where: keyword
      ? {
          $or: [
            { name: { $contain: keyword } },
            { email: { $contain: keyword } },
          ],
        }
      : undefined,
    select: ["id", "name", "email", "status"],
    orderBy: [{ createTime: "desc" }],
    currentPage: page,
    pageSize: 20,
  });

  return {
    list: result.tableData,
    total: result.total,
    hasMore: page * 20 < result.total,
  };
}

// 筛选订单列表
async function getOrders(filters: {
  status?: string;
  dateStart?: string;
  dateEnd?: string;
}) {
  const conditions = [];

  if (filters.status) {
    conditions.push({ status: { $eq: filters.status } });
  }
  if (filters.dateStart) {
    conditions.push({ createTime: { $gte: filters.dateStart } });
  }
  if (filters.dateEnd) {
    conditions.push({ createTime: { $lte: filters.dateEnd } });
  }

  const result = await client.models.orders.filter({
    where: conditions.length > 0 ? { $and: conditions } : undefined,
    orderBy: [{ createTime: "desc" }],
    currentPage: 1,
    pageSize: 50,
  });

  return result.tableData;
}
```

### 多表关联查询（JOIN）

filter 支持多表关联查询，可以在一次查询中获取关联表的数据。

**关联规则**
| 规则 | 说明 |
|------|------|
| JOIN 类型 | 统一使用 LEFT JOIN，保证主表数据不丢失 |
| 字段引用 | 使用表名（如 `profile`），不使用 datasetCode |
| 支持关系 | 1:1、N:1（反向 1:N 暂不支持） |
| 嵌套层级 | 最多 5 层 |
| 关联检测 | 隐式关联，系统自动分析（基于外键等） |

**基础用法**：`tableName.fieldName`

```typescript
// 数据集: article | 数据表: article
// 主表：article（文章），关联表：profile（作者），1:1 关联
const result = await client.models.article.filter({
  select: [
    "id",
    "title",
    "content",
    "create_at",
    "profile.username", // 关联表字段：表名.字段名
    "profile.avatar_url",
  ],
  where: {
    status: { $eq: "published" },
    "profile.is_signed": { $eq: true }, // 关联表过滤条件
  },
  orderBy: [{ create_at: "desc" }, { "profile.username": "desc" }],
  currentPage: 1,
  pageSize: 20,
});

// 返回结果（嵌套结构）
{
  id: 1,
  title: "文章标题",
  profile: {
    username: "张三",
    avatar_url: "https://..."
  }
}
```

**注意事项**：

- 使用数据库**表名**（如 `customer`），不是数据集编码（如 `dataset_xxx`）
- 系统自动检测表关联关系，无需手动指定
- 返回的关联表数据会嵌套在对应对象中

---

## 2️⃣ SQL 自定义查询

通过 `client.sql.execute()` 执行已注册的自定义 SQL 查询。

### 基本用法

```typescript
// 对象参数格式
const data = await client.sql.execute({
  sqlCode: "fc8e7777-06e3847d",
  params: { userId: "123", startDate: "2025-01-01" },
});
```

### 返回值处理

```typescript
// SDK 返回的是业务数据层：{ execSuccess, execResult }
const data = await client.sql.execute({
  sqlCode: "fc8e7777-06e3847d",
});

// 应用层检查业务状态（应用层职责）
if (data.execSuccess && data.execResult) {
  data.execResult.forEach((row) => {
    console.log(row);
  });
} else {
  console.error("SQL执行失败");
}
```

**重要说明**：

- SDK 返回的是 `{ execSuccess, execResult }`（业务数据层），不是完整的响应对象
- HTTP 错误会抛出 `LovrabetError` 异常
- 应用层负责检查 `execSuccess` 状态，处理业务结果

### 错误处理

```typescript
import { createClient, LovrabetError } from "@lovrabet/sdk";

// HTTP 级别错误（网络、认证等）会抛出 LovrabetError
try {
  const data = await client.sql.execute({
    sqlCode: "fc8e7777-06e3847d",
  });

  // 业务级别错误（SQL 语法、参数等）通过 execSuccess 判断（应用层职责）
  if (data.execSuccess && data.execResult) {
    data.execResult.forEach((row) => console.log(row));
  } else {
    console.error("SQL 查询失败");
  }
} catch (error) {
  if (error instanceof LovrabetError) {
    console.error("HTTP 请求失败:", error.message);
  }
}
```

### 带类型提示

```typescript
// 定义结果类型
interface PageStat {
  creation_date: string;
  page_count: number;
}

// 使用泛型获得类型安全
const data = await client.sql.execute<PageStat>({
  sqlCode: "fc8e7777-06e3847d",
});

// 应用层检查业务状态
if (data.execSuccess && data.execResult) {
  data.execResult.forEach((stat) => {
    console.log(stat.creation_date); // TypeScript 自动补全
    console.log(stat.page_count);
  });
}
```

### 注意事项

- SQL 需提前通过 Lovrabet 平台或 MCP 工具创建（参见 `02-mcp-sql-workflow.md`）
- SQL 中的参数使用 `#{paramName}` 语法
- **返回值结构**：`{ execSuccess: boolean, execResult?: T[] }`（业务数据层）
- **HTTP 错误**：会抛出 `LovrabetError` 异常
- **业务错误**：通过 `execSuccess` 判断
- **应用层职责**：检查 `execSuccess` 状态和处理结果

---

## 3️⃣ BFF 端点调用

通过 `client.bff.execute()` 调用 Backend For Frontend 端点。

### 基本用法

```typescript
const result = await client.bff.execute({
  scriptName: "getUserDashboard",
  params: { userId: "123" },
});
```

### 带类型提示

```typescript
interface DashboardData {
  userCount: number;
  orderCount: number;
  totalAmount: number;
}

const dashboard = await client.bff.execute<DashboardData>({
  scriptName: "getUserDashboard",
});

console.log(dashboard.userCount);
```

### 完整示例

```typescript
// 获取用户仪表盘数据
async function getUserDashboard(userId: string) {
  try {
    const dashboard = await client.bff.execute<{
      userCount: number;
      orderCount: number;
      totalAmount: number;
    }>({
      scriptName: "getUserDashboard",
      params: { userId },
    });

    return {
      success: true,
      data: dashboard,
    };
  } catch (error) {
    if (error instanceof LovrabetError) {
      console.error("BFF调用失败:", error.message);
      return {
        success: false,
        error: error.message,
      };
    }
    throw error;
  }
}
```

### 错误处理

```typescript
import { createClient, LovrabetError } from "@lovrabet/sdk";

try {
  // SDK 返回的是业务数据对象（直接是业务数据层）
  const dashboard = await client.bff.execute({
    scriptName: "someEndpoint",
    params: {
      /* ... */
    },
  });

  // 直接使用返回的业务数据
  console.log(dashboard.userCount);
} catch (error) {
  if (error instanceof LovrabetError) {
    console.error("错误码:", error.code);
    console.error("错误信息:", error.message);
  }
}
```

**重要说明**：

- SDK 返回的是业务数据对象（直接是业务数据层），不是包装器
- HTTP 错误会抛出 `LovrabetError` 异常
- 应用层直接使用返回的业务数据，无需检查 `execSuccess`（BFF 没有此字段）

---

## 📊 三大 API 对比

| 特性           | Filter                                  | SQL                            | BFF                      |
| -------------- | --------------------------------------- | ------------------------------ | ------------------------ |
| **用途**       | 数据集高级查询                          | 自定义 SQL 查询                | 调用后端函数             |
| **适用场景**   | 列表查询、条件筛选、多表关联（1:1/N:1） | 复杂统计、跨表关联（任意JOIN） | 业务逻辑、事务处理       |
| **API 调用**   | `client.models.<name>.filter()`         | `client.sql.execute()`         | `client.bff.execute()`   |
| **返回值**     | `{ tableData, total, ... }`             | `{ execSuccess, execResult }`  | 业务数据对象             |
| **参数格式**   | `{ where, select, orderBy, ... }`       | `{ sqlCode, params }`          | `{ scriptName, params }` |
| **类型支持**   | ✅ 泛型                                 | ✅ 泛型                        | ✅ 泛型                  |
| **错误处理**   | 抛出 LovrabetError                      | execSuccess + LovrabetError    | 抛出 LovrabetError       |
| **应用层职责** | 直接使用返回数据                        | 检查 `execSuccess` 状态        | 直接使用返回数据         |

---

## ✅ 自检清单

### Filter 检查

- [ ] where 条件使用了操作符（`$eq`、`$gte` 等）
- [ ] 参数名正确（`select` 不是 `fields`）
- [ ] `orderBy` 和 `select` 是数组格式
- [ ] 分页参数名是 `currentPage`/`pageSize`
- [ ] 多表关联使用表名（`tableName.fieldName`），不是数据集编码

### SQL 检查

- [ ] 使用对象参数格式 `{ sqlCode, params }`
- [ ] 检查 `execSuccess` 状态
- [ ] SQL 已提前通过平台或 MCP 创建

### BFF 检查

- [ ] 使用 `client.bff.execute({ scriptName, params })`
- [ ] 添加错误处理（LovrabetError）
- [ ] 使用泛型获得类型提示

---

## 🔗 相关资源

- Lovrabet SDK 文档：https://www.npmjs.com/package/@lovrabet/sdk
- MCP SQL 创建工作流：参见 `02-mcp-sql-workflow.md`
- Backend Function 编写规范：参见 `05-backend-function.md`
