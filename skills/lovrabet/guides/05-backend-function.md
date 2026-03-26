# Backend Function 脚本编写规范

> **目标**：学会编写符合 lovrabet-runtime 平台规范的动态脚本，用于数据权限过滤、动态脱敏、数据增强和复杂业务逻辑。
>
> **前置阅读（必读）**：
>
> - `06-data-api-guidelines.md` - 数据接口访问规范（字段分析、依赖关系、**性能优化**）
> - **性能优化是编写 Backend Function 的必读内容，不可跳过**

---

## 核心概念

Backend Function 是运行在 lovrabet-runtime 平台上的动态脚本，分为两类：

| 类型              | 说明               | 触发方式                                    |
| ----------------- | ------------------ | ------------------------------------------- |
| **HOOK 脚本**     | 依附于标准数据接口 | Before（前置）/ After（后置）               |
| **ENDPOINT 脚本** | 独立业务端点       | POST `/api/{appCode}/endpoint/{scriptName}` |

---

## 平台配置地址

| 脚本类型                 | 平台配置地址格式                                                           | 说明                                                              |
| ------------------------ | -------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| **HOOK**（Before/After） | `https://app.lovrabet.com/app/{appCode}/data/dataset/{datasetId}#api-list` | datasetId 为数据集 ID（整型数字，从 MCP get_dataset_detail 获取） |
| **ENDPOINT**（独立端点） | `https://app.lovrabet.com/app/{appCode}/data/backend-function`             | appCode 为应用编码，其他固定                                      |

---

## 系统自动维护字段

调用标准 SDK/API（create/update）时，以下字段由平台自动维护，**代码中无需设置**：

| 字段名        | 说明     | 处理方式       |
| ------------- | -------- | -------------- |
| `id`          | 主键ID   | 创建时自动生成 |
| `create_time` | 创建时间 | 创建时自动设置 |
| `modify_time` | 修改时间 | 更新时自动更新 |
| `create_by`   | 创建人   | 创建时自动设置 |
| `modify_by`   | 修改人   | 更新时自动设置 |

**代码编写规范**：

```javascript
// ❌ 错误：手动设置系统自动维护的字段
await orderDS.create({
  id: uuid(), // 不要设置
  create_time: Date.now(), // 不要设置
  order_no: "ORD001",
  amount: 100,
});

// ✅ 正确：只设置业务字段
await orderDS.create({
  order_no: "ORD001",
  amount: 100,
  status: "pending",
});
```

---

## 6 步强制工作流程

```
需求分析 → 字段分析 → 编写脚本 → 字段验证 → 添加注释 → 代码校验
```

### 步骤 1: 需求分析与数据集获取

**操作**：

1. 分解用户需求，识别：数据集、脚本类型（Before/After/Endpoint）、接口操作、业务逻辑
2. 使用 MCP 工具 `get_dataset_detail` 获取数据集详情

**示例**：

```
用户需求："非管理员只能看到自己创建的数据"
分析结果：数据集 users，Before，filter 接口，权限过滤
```

### 步骤 2: 字段分析与推理

从数据集详情中提取字段信息，根据业务需求推理所需字段。

**推理示例**：

```
场景：权限过滤
需求："非管理员只能查看自己创建的数据"
推理：判断角色需要 context.userInfo.role
      过滤创建人需要 created_by 字段
      获取当前用户需要 context.userInfo.id
```

```
场景：数据脱敏
需求："手机号中间四位脱敏"
推理：需要 phone 字段，After 脚本，操作 params.tableData
```

```
场景：关联查询
需求："订单列表显示用户名称"
推理：订单表 user_id 关联用户表 id
      追加 username 字段到订单数据
```

**获取数据集详情后，确认**：

| 检查项       | 说明                          |
| ------------ | ----------------------------- |
| 字段名       | 精确匹配，区分大小写          |
| 字段类型     | 与操作匹配                    |
| 必填字段     | create 时包含（系统字段除外） |
| 外键关系     | 关联查询/数据校验             |
| **系统字段** | **不要在代码中设置**          |

### 步骤 3: 编写脚本

确定脚本类型和接口类型，编写函数签名 `export default async function`，实现业务逻辑，确保返回值。

**注意**：普通接口直接设置 `params.fieldName = value`；filter 接口操作 `params.where`，使用操作符语法。

### 步骤 4: 字段验证

检查字段名精确匹配（区分大小写）、类型正确、无拼写错误。

**常见错误**：

- 错误：`params.createBy` → 正确：`created_by`
- 错误：`params.userId` → 正确：`user_id`

### 步骤 5: 添加注释

#### ⚠️ 占位符替换（易错）

顶部注释中的占位符 **必须替换为实际值**，这是最常见的错误：

| 占位符 | 替换为 | 示例 |
|-------|-------|------|
| `{appCode}` | 应用编码 | `app-99571871` |
| `{datasetCode}` | 数据集编码 | `users` |
| `{datasetId}` | 数据集 ID（整型数字） | `1000372` |
| `{operation}` | 操作名 | `filter` |
| `{scriptName}` | 脚本名 | `createOrder` |

```javascript
// ❌ 错误：占位符未替换
// [接口路径] POST /api/{appCode}/{datasetCode}/{operation}

// ✅ 正确：替换为实际值
// [接口路径] POST /api/app-99571871/users/filter
```

#### 顶部注释格式（强制）

**HOOK 脚本（Before/After）**：

```javascript
/**
 * 脚本功能描述
 *
 * [接口路径] POST /api/{appCode}/{datasetCode}/{operation}
 * 规则：/api/{应用编码}/{数据集编码}/{操作名}
 *
 * [平台配置] https://app.lovrabet.com/app/{appCode}/data/dataset/{datasetId}#api-list
 * 说明：appCode 为应用编码，datasetId 为数据集 ID（整型数字，从 MCP get_dataset_detail 获取）
 *
 * [HTTP 请求体参数]
 * { "field1": "字段1说明", "field2": "字段2说明" }
 *
 * [返回数据结构]
 * HOOK: 返回修改后的 params 对象
 *
 * @param {Object} params - 请求参数
 * @param {string} params.field1 - 字段1
 * @param {Object} context - 执行上下文（平台自动注入，调用时无需传递）
 * @param {Object} context.userInfo - 当前用户信息
 * @param {Object} context.client - 数据库操作入口
 * @returns {Object} 返回结果说明
 */
```

**ENDPOINT 脚本（独立端点）**：

- 顶部注释必须包含脚本功能描述、脚本调用地址、平台对应的配置地址、HTTP请求参数和返回结果的数据结构

```javascript
/**
 * {脚本功能描述}
 *
 * [接口路径] POST /api/{appCode}/endpoint/{scriptName}
 * [平台配置] https://app.lovrabet.com/app/{appCode}/data/backend-function
 *
 * [HTTP 请求体参数]
 * { "field1": "字段1说明", "field2": "字段2说明" }
 *
 * [返回数据结构]
 * ENDPOINT: 返回业务数据对象（会被包装在响应的 data 字段中）
 *
 * @param {Object} params - 请求参数
 * @param {string} params.field1 - 字段1
 * @param {Object} context - 执行上下文（平台自动注入，调用时无需传递）
 * @param {Object} context.userInfo - 当前用户信息
 * @param {Object} context.client - 数据库操作入口
 * @returns {Object} 返回结果说明
 */
```

#### 数据集调用注释（强制）

```javascript
// 数据集: {数据集名称} | 数据表: {数据表名称}
const ds = context.client.models.dataset_XXXXXXXXXX;

// 多数据库场景（数据库名.表名）
// 数据集: {数据集名称} | 数据表: {数据库名.数据表名称}
const ds = context.client.models.dataset_XXXXXXXXXX;
```

### 步骤 6: 代码校验

**校验清单**：

- [ ] 文件名格式正确：HOOK `{tablename}_before{Operation}.js`，ENDPOINT `endpoint_{scriptName}.js`
- [ ] 函数名与 scriptName 一致，驼峰命名
- [ ] **顶部注释占位符已替换**（`{appCode}`、`{datasetCode}`、`{datasetId}` 等）
- [ ] 顶部注释包含接口路径和平台配置地址
- [ ] context 标注为"平台自动注入"
- [ ] 每个数据集调用前有 `// 数据集: ... | 数据表: ...` 注释
- [ ] 使用 32 位数据集编码（不是 datasetCode）
- [ ] 查询其他表用 filter 而非 getList
- [ ] 所有数据库操作使用 await
- [ ] HOOK 返回 params，ENDPOINT 返回业务对象

---

## 数据库操作 API

通过 `context.client.models.dataset_` + **32位数据集编码** 访问数据集。

### 数据集变量定义（必须）

为了代码可读性，必须在函数开头定义数据集映射：

```javascript
// 数据集编码映射表（函数开头定义）
// 注释格式：数据集: {数据集名称} | 数据表: {数据表名称}
const TABLES = {
  fm_protocol_extends: "dataset_7e9eb1d6f53142f49a391addb79a809d", // 数据集: 协议继承处理 | 数据表: fm_protocol_extends
  fm_protocol_trace: "dataset_44da4410ec6d4c47875defb42f7c0daf", // 数据集: 协议 | 数据表: fm_protocol_trace
  customers: "dataset_9d33b702fd1a4e27b45e07399f823828", // 数据集: 客户 | 数据表: customers
};

// 提取 models 简化调用
const models = context.client.models;

// 后续调用：models[TABLES.表名]
const result = await models[TABLES.fm_protocol_extends].getOne(123);
```

### 常用方法

| 方法              | 说明     | 返回值  |
| ----------------- | -------- | ------- |
| `getOne(id)`     | 按主键查询单条 | Object  |
| `filter(params)`  | 高级过滤 | List    |
| `create(data)`    | 创建数据 | ID      |
| `update(data)`    | 更新数据 | Boolean |
| `delete(params)`  | 删除数据 | Boolean |

**重要**：

- 所有数据库操作必须使用 `await`
- 必须在函数开头定义数据集映射
- 每个数据集映射后添加注释说明数据表名称

### filter 接口返回格式

```javascript
{
  "tableData": [],           // 数据列表
  "paging": {
    "currentPage": 1,
    "totalCount": 0,
    "pageSize": 30
  },
  "tableColumns": []         // 列定义
}
```

### 自定义 SQL 调用

通过 `context.client.sql.execute()` 调用自定义 SQL：

**基本语法**：

```javascript
const result = await context.client.sql.execute({
  sqlCode: "sql-code-id",  // SQL 配置的 code（32位编码）
  params: {                // SQL 参数（MyBatis 格式）
    param1: value1,
    param2: value2,
  },
});
// result 直接是查询结果数组，无需取 execResult
```

> **⚠️ 核心差异：与前端 SDK 的返回值不同**
>
> | 环境 | 调用方式 | 返回值 | 数据获取方式 |
> |------|---------|--------|-------------|
> | **前端 SDK** | `client.sql.execute()` | `{ execSuccess, execResult }` | 需检查 `execSuccess`，从 `execResult` 取数据 |
> | **Backend Function** | `context.client.sql.execute()` | 直接返回数组 `[{ col: val }, ...]` | 直接使用结果，无 `execResult` 字段 |
>
> ```javascript
> // ❌ 错误：BFF 中不存在 execResult
> const result = await context.client.sql.execute({ sqlCode: 'xxx' });
> const rows = result.execResult;  // undefined!
>
> // ✅ 正确：BFF 返回值直接就是数组
> const rows = await context.client.sql.execute({ sqlCode: 'xxx' });
> const firstRow = rows[0];  // { col1: val1, col2: val2, ... }
> ```

**sqlCode 获取方式**：

| 方式 | 说明 |
|------|------|
| **平台配置** | 进入应用 → 自定义 SQL → 查看 sqlCode |
| **MCP 工具** | 使用 `list_sql_queries` 查询所有 SQL |

**完整示例**：

```javascript
/**
 * 前置校验函数 - 调用自定义 SQL 查询员工信息
 *
 * [接口路径] POST /api/{appCode}/endpoint/beforeFilter
 */
export default async function beforeFilter(params, context) {
  // 调用自定义 SQL
  const result = await context.client.sql.execute({
    sqlCode: "8cbdd953-8d5fa8e5",  // SQL 配置的 code
    params: {
      storeId: params.storeId,     // SQL 参数
    },
  });

  // result 是查询结果数组
  params.employees = result;
  return params;
}
```

### 事务使用

事务用于保证多个数据库操作的原子性（全部成功或全部回滚）。

#### 快速开始

```javascript
await context.client.db.transaction(async (tx) => {
  // 在这里执行数据库操作
  // 正常结束自动提交，抛出异常自动回滚
});
```

#### 使用规范

| 规则 | 说明 |
|------|------|
| **必须使用 await** | `await context.client.db.transaction(...)` |
| **事务函数必须是 async** | `async (tx) => { ... }` |
| **不要在事务外使用 tx 对象** | 所有操作必须在事务回调内完成 |
| **不要在事务中执行耗时操作** | 外部 API 调用放在事务外 |
| **必须正确处理异常** | 异常要向上抛出，让事务自动回滚 |

#### 基本用法

**1. 执行自定义 SQL（事务内）**：

```javascript
await context.client.db.transaction(async (tx) => {
  const result = await tx.sql.execute({
    sqlCode: "insertUser",
    params: { nickname: "Alice", phone: "13800138000" },
  });
});
```

**2. 使用数据集 API（事务内）**：

```javascript
await context.client.db.transaction(async (tx) => {
  const models = context.client.models;

  // 创建记录
  const id = await models[TABLES.users].create({
    name: "Bob",
    age: 25,
  });

  // 查询记录
  const user = await models[TABLES.users].getOne(id);

  // 更新记录
  await models[TABLES.users].update({ id, age: 26 });
});
```

**3. 混合使用**：

```javascript
await context.client.db.transaction(async (tx) => {
  const models = context.client.models;

  // 使用自定义 SQL
  await tx.sql.execute({
    sqlCode: "insertUser",
    params: { nickname: "Charlie", phone: "13700137000" },
  });

  // 使用数据集 API
  await models[TABLES.orders].create({
    user_id: userId,
    amount: 100,
  });
});
```

#### 常见场景示例

**场景 1：用户注册（单数据源）**：

```javascript
export default async function registerUser(params, context) {
  const TABLES = {
    users: "dataset_XXXXXXXXXX",     // 数据集: 用户 | 数据表: users
    user_points: "dataset_YYYYYYYYYY", // 数据集: 用户积分 | 数据表: user_points
    messages: "dataset_ZZZZZZZZZZ",   // 数据集: 消息 | 数据表: messages
  };
  const models = context.client.models;

  await context.client.db.transaction(async (tx) => {
    // 1. 插入用户信息
    const userId = await models[TABLES.users].create({
      nickname: params.nickname,
      phone: params.phone,
    });

    // 2. 初始化用户积分
    await models[TABLES.user_points].create({
      user_id: userId,
      points: 0,
    });

    // 3. 发送欢迎消息
    await models[TABLES.messages].create({
      user_id: userId,
      content: "欢迎注册",
    });

    return { success: true, userId };
  });
}
```

**场景 2：订单创建（多表操作）**：

```javascript
export default async function createOrder(params, context) {
  const TABLES = {
    orders: "dataset_XXXXXXXXXX",       // 数据集: 订单 | 数据表: orders
    order_items: "dataset_YYYYYYYYYY",  // 数据集: 订单明细 | 数据表: order_items
    products: "dataset_ZZZZZZZZZZ",     // 数据集: 商品 | 数据表: products
  };
  const models = context.client.models;

  await context.client.db.transaction(async (tx) => {
    // 1. 创建订单
    const orderId = await models[TABLES.orders].create({
      user_id: params.userId,
      total_amount: params.amount,
      status: "pending",
    });

    // 2. 创建订单明细
    for (const item of params.items) {
      await models[TABLES.order_items].create({
        order_id: orderId,
        product_id: item.productId,
        quantity: item.quantity,
        price: item.price,
      });
    }

    // 3. 扣减库存（使用自定义 SQL）
    await tx.sql.execute({
      sqlCode: "decreaseStock",
      params: { items: params.items },
    });

    return { success: true, orderId };
  });
}
```

**场景 3：转账操作（带校验）**：

```javascript
export default async function transfer(params, context) {
  const TABLES = {
    accounts: "dataset_XXXXXXXXXX",  // 数据集: 账户 | 数据表: accounts
    transfers: "dataset_YYYYYYYYYY", // 数据集: 转账记录 | 数据表: transfers
  };
  const models = context.client.models;

  await context.client.db.transaction(async (tx) => {
    // 1. 查询转出账户余额
    const fromAccount = await models[TABLES.accounts].getOne(
      params.fromAccountId,
    );

    // 2. 余额校验
    if (fromAccount.balance < params.amount) {
      throw new Error("余额不足");
    }

    // 3. 扣减转出账户
    await models[TABLES.accounts].update({
      id: params.fromAccountId,
      balance: fromAccount.balance - params.amount,
    });

    // 4. 增加转入账户
    const toAccount = await models[TABLES.accounts].getOne(
      params.toAccountId,
    );
    await models[TABLES.accounts].update({
      id: params.toAccountId,
      balance: toAccount.balance + params.amount,
    });

    // 5. 记录转账流水
    await models[TABLES.transfers].create({
      from_account_id: params.fromAccountId,
      to_account_id: params.toAccountId,
      amount: params.amount,
    });

    return { success: true };
  });
}
```

#### 注意事项

**事务范围尽量小**：

```javascript
// ✅ 好的做法：先准备数据，再开启事务
const data = await fetchExternalApi();
await context.client.db.transaction(async (tx) => {
  await tx.sql.execute({ sqlCode: "insert", params: data });
});

// ❌ 不好的做法：事务中包含外部调用
await context.client.db.transaction(async (tx) => {
  const data = await fetchExternalApi();  // 外部 API 很慢
  await tx.sql.execute({ sqlCode: "insert", params: data });
});
```

**异常处理**：

```javascript
// ✅ 正确：异常要向上抛出，让事务回滚
try {
  await context.client.db.transaction(async (tx) => {
    await tx.sql.execute({ sqlCode: "xxx", params: {} });
  });
} catch (e) {
  // 在这里处理异常
  return { success: false, error: e.message };
}

// ❌ 错误：吞掉异常，导致事务状态不明确
await context.client.db.transaction(async (tx) => {
  try {
    await tx.sql.execute({ sqlCode: "xxx", params: {} });
  } catch (e) {
    console.log(e);  // 只打印日志，不抛出异常
  }
});
```

---

## 性能优化（必读）

**编写 Backend Function 前，必须先阅读 `06-data-api-guidelines.md` 的性能优化章节**。

性能问题是最常见的 Backend Function 问题根源，包括：

- 循环查询单条（N 次数据库调用）
- 循环写入（N 次数据库调用）
- 嵌套循环查询

**优化方案**：详见 `06-data-api-guidelines.md` → 步骤 4（批量查询）和步骤 5（批量操作）

**单次脚本数据库调用限制**：**50 次**

---

## Before 脚本编写规则

Before 脚本用于在接口执行前修改请求参数。

### 普通接口 Before

直接设置字段值：

```javascript
/**
 * 分页查询前置脚本 - 自动过滤创建人
 *
 * [接口路径] POST /api/{appCode}/{datasetCode}/getList
 * 规则：/api/{应用编码}/{数据集编码}/{操作名}
 *
 * [平台配置] https://app.lovrabet.com/app/{appCode}/data/dataset/{datasetId}#api-list
 *
 * @param {Object} params - 请求参数
 * @param {Object} context - 执行上下文（平台自动注入）
 * @returns {Object} 修改后的 params
 */
export default async function beforeGetList(params, context) {
  if (context.userInfo.role !== "admin") {
    params.created_by = context.userInfo.id;
  }
  return params;
}
```

### Filter / Aggregate 接口 Before

必须操作 `params.where`，使用操作符语法：

```javascript
/**
 * 高级查询前置脚本 - 权限过滤
 *
 * [接口路径] POST /api/{appCode}/{datasetCode}/filter
 *
 * @param {Object} params - filter 请求参数
 * @param {Object} context - 执行上下文（平台自动注入）
 * @returns {Object} 修改后的 params
 */
export default async function beforeFilter(params, context) {
  if (context.userInfo.role !== "admin") {
    params.where = params.where || {};
    const originalWhere = { ...params.where };
    params.where = {
      $and: [originalWhere, { created_by: { $eq: context.userInfo.id } }],
    };
  }
  return params;
}
```

---

## After 脚本编写规则

After 阶段的 `params` 包含 `tableData`（数据列表）和 `tableColumns`（列定义）。

```javascript
/**
 * 高级查询后置脚本 - 手机号脱敏
 *
 * [接口路径] POST /api/{appCode}/{datasetCode}/filter
 *
 * @param {Object} params - filter 返回结果
 * @param {Array} params.tableData - 数据列表
 * @param {Object} context - 执行上下文（平台自动注入）
 * @returns {Object} 修改后的 params
 */
export default async function afterFilter(params, context) {
  params.tableData?.forEach((record) => {
    if (record.phone) {
      record.phone = record.phone.replace(/(\d{3})\d{4}(\d{4})/, "$1****$2");
    }
  });
  return params;
}
```

---

## ENDPOINT 脚本编写规则

独立端点脚本用于复杂事务逻辑。

**文件名**：`endpoint_{scriptName}.js`
**API 路径**：`POST /api/{appCode}/endpoint/{scriptName}`
**返回值**：业务数据对象（会被包装在响应的 `data` 字段中）

**平台响应结构**：

```javascript
{
  "success": true,      // 平台级别：脚本执行成功
  "data": {             // ENDPOINT 返回的业务数据
    // 你返回的对象内容
  }
}
```

**示例**：

```javascript
/**
 * 创建订单并扣减库存
 *
 * [接口路径] POST /api/{appCode}/endpoint/createOrder
 *
 * [平台配置] https://app.lovrabet.com/app/{appCode}/data/backend-function
 *
 * @param {Object} params - 请求参数
 * @param {string} params.productId - 商品ID
 * @param {number} params.quantity - 购买数量
 * @param {Object} context - 执行上下文（平台自动注入）
 * @returns {Object} { orderId: string, payAmount: number }
 */
export default async function createOrder(params, context) {
  const TABLES = {
    products: "dataset_XXXXXXXXXX", // 数据集: 商品 | 数据表: products
    orders: "dataset_YYYYYYYYYYYY", // 数据集: 订单 | 数据表: orders
  };
  const models = context.client.models;

  // 查询商品
  const product = await models[TABLES.products].getOne(params.productId);
  if (!product || product.stock < params.quantity) {
    throw new Error(`库存不足`);
  }

  // 创建订单（只设置业务字段）
  const orderId = await models[TABLES.orders].create({
    user_id: context.userInfo.id,
    product_id: params.productId,
    quantity: params.quantity,
    amount: product.price * params.quantity,
    status: "pending",
    // 不要设置 id, create_time 等系统字段
  });

  // 扣减库存
  await models[TABLES.products].update({
    id: product.id,
    stock: product.stock - params.quantity,
  });

  return { orderId, payAmount: product.price * params.quantity };
}
```

---

## 文件存放目录

**项目根目录**：`src/backend-function/`

```
src/backend-function/
├── {tablename}/              # HOOK 脚本目录（按表名组织）
│   ├── {tablename}_beforeFilter.js
│   ├── {tablename}_afterFilter.js
│   └── {tablename}_beforeCreate.js
└── endpoint/                 # ENDPOINT 脚本目录
    ├── endpoint_createOrder.js
    └── endpoint_batchDelete.js
```

| 脚本类型                 | 存放目录                            | 说明                                       |
| ------------------------ | ----------------------------------- | ------------------------------------------ |
| **HOOK**（Before/After） | `src/backend-function/{tablename}/` | 按数据表名组织，同一表的 HOOK 脚本放在一起 |
| **ENDPOINT**             | `src/backend-function/endpoint/`    | 所有独立端点脚本放在 endpoint 子目录       |

---

## 文件命名与函数规范

| 脚本类型     | 文件名格式                         | 函数名              | 示例                                      |
| ------------ | ---------------------------------- | ------------------- | ----------------------------------------- |
| HOOK（前置） | `{tablename}_before{Operation}.js` | `before{Operation}` | `users_beforeFilter.js` → `beforeFilter`  |
| HOOK（后置） | `{tablename}_after{Operation}.js`  | `after{Operation}`  | `users_afterFilter.js` → `afterFilter`    |
| ENDPOINT     | `endpoint_{scriptName}.js`         | `{scriptName}`      | `endpoint_createOrder.js` → `createOrder` |

**常见 Operation**：`Filter`、`GetList`、`getOne`、`Create`、`Update`、`Delete`、`Aggregate`

---

## 参数说明

| 阶段         | params 内容                             |
| ------------ | --------------------------------------- |
| **Before**   | API 请求参数                            |
| **After**    | API 返回结果（tableData、tableColumns） |
| **Endpoint** | HTTP 请求体 JSON 数据                   |

**context 属性**：

- `context.userInfo` - 当前用户信息（id、username、role、tenantCode）
- `context.appCode` - 当前应用编码
- `context.tenantCode` - 当前租户编码
- `context.client` - 数据库操作入口

---

## 操作符参考

### where 查询操作符

| 操作符     | 说明           | 示例                            |
| ---------- | -------------- | ------------------------------- |
| `$eq`      | 等于           | `{ field: { $eq: value } }`     |
| `$ne`      | 不等于         | `{ field: { $ne: value } }`     |
| `$gte`     | 大于等于       | `{ field: { $gte: value } }`    |
| `$lte`     | 小于等于       | `{ field: { $lte: value } }`    |
| `$in`      | 在集合内       | `{ field: { $in: [v1, v2] } }`  |
| `$contain` | 包含（模糊）   | `{ field: { $contain: "kw" } }` |
| `$and`     | 且（所有满足） | `{ $and: [条件1, 条件2] }`      |

---

## 自检清单

### 基础规范

- [ ] 函数签名：`export default async function`
- [ ] 返回值：HOOK 返回 `params`，ENDPOINT 返回业务数据
- [ ] 所有数据库操作使用 `await`
- [ ] Filter 接口用 `params.where` 和操作符
- [ ] 保留原有 `where`（用 `$and` 组合）
- [ ] 查询其他表用 `filter` 而非 `getList`
- [ ] 数据集映射定义：函数开头定义 `TABLES`，使用 `models[TABLES.表名]`
- [ ] 每个数据集映射后添加注释：`// 数据集: xxx | 数据表: xxx`
- [ ] 使用 32 位数据集编码
- [ ] 文件名和函数名格式正确
- [ ] **顶部注释占位符已替换**（`{appCode}`、`{datasetCode}`、`{datasetId}` 等）
- [ ] 顶部注释包含接口路径和平台配置地址

### 编写前检查（详见 `06-data-api-guidelines.md`）

- [ ] 已获取数据集详情，核对字段名、类型、是否必填
- [ ] 已分析依赖关系，理解主外键约束
- [ ] 已识别外键字段，正确处理关联查询/校验
- [ ] 系统字段未设置（id、create_time、modify_time、create_by、modify_by）

### 性能优化（详见 `06-data-api-guidelines.md`）

- [ ] 无循环查询单条（使用 filter + $in）
- [ ] 无循环写入（使用自定义 SQL）
- [ ] 无嵌套循环查询
- [ ] 数据库调用次数 < 50 次
- [ ] 批量操作已优化

---

## 快速对照表

| 场景       | 普通接口               | Filter 接口                                          |
| ---------- | ---------------------- | ---------------------------------------------------- |
| 权限过滤   | `params.field = value` | `params.where = { $and: [..., { field: { $eq } }] }` |
| 支持操作符 | 不支持                 | 支持 `$eq`、`$gte`、`$contain` 等                    |

| 阶段       | params 内容           | 典型用途                       |
| ---------- | --------------------- | ------------------------------ |
| **Before** | 请求参数              | 权限过滤、参数校验、自动填充   |
| **After**  | 响应数据（tableData） | 数据脱敏、字段计算、敏感列隐藏 |

---

## 🔗 相关指南

- **数据接口访问规范**：`06-data-api-guidelines.md`
- **前端页面开发**：`03-frontend-development.md`
- **SQL 创建工作流**：`02-mcp-sql-workflow.md`
