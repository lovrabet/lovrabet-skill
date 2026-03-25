# 问题排查指南

> 遇到 Lovrabet 开发问题时，按照本指南快速定位和解决问题。

## 🔍 快速诊断流程

```
1. 确认问题类型
   ↓
2. 查看错误信息
   ↓
3. 定位问题根源
   ↓
4. 应用解决方案
   ↓
5. 验证问题已解决
```

---

## 常见问题

### 问题 1: Filter 查询没有结果

**现象**：
```typescript
const result = await client.models.users.filter({
  where: { status: "active" },  // ❌ 忘记操作符！
});
console.log(result.tableData); // 空数组
console.log(result.total);      // 0
```

**原因**：忘记使用操作符

**解决方案**：
```typescript
// ✅ 正确：使用 $eq 操作符
const result = await client.models.users.filter({
  where: { status: { $eq: "active" } },
});

console.log(result.tableData); // 正确获取数据
```

---

### 问题 2: 报错 "参数格式不正确"

**现象**：
```typescript
const result = await client.models.users.filter({
  fields: ["id", "name"],  // ❌ 错误：应该是 select
  sort: [{ id: "desc" }],  // ❌ 错误：应该是 orderBy
});
```

**原因**：参数名错误

**解决方案**：
```typescript
// ✅ 正确的参数名
const result = await client.models.users.filter({
  select: ["id", "name"],     // 正确
  orderBy: [{ id: "desc" }], // 正确
  currentPage: 1,
  参数名错误：fields → select
  参数名错误：sort → orderBy
  参数名错误：page → currentPage
  参数名：limit → pageSize
```

---

### 问题 3: MCP 工具调用失败

**现象**：
```
使用 MCP 工具时返回错误：Error: Dataset not found
```

**原因**：
1. appcode 配置错误
2. 数据集代码错误
3. 未登录或 Cookie 过期

**解决方案**：

1. 检查配置文件
   ```typescript
   // .lovrabetrc
   {
     "appcode": "app-xxx",
     "env": "online"
   }
   ```

2. 使用 list_datasets 验证
   ```typescript
   // 先验证数据集是否存在
   const datasets = await list_datasets();
   console.log(datasets.datasets.map(d => d.code)); // 查看所有数据集代码
   ```

3. 检查登录状态
   ```bash
   # 检查是否已登录
   lovrabet auth
   ```

---

### 问题 4: SQL 执行错误

**现象**：
```
execSuccess: false
execError: Table 'wrong_table' doesn't exist
```

**原因**：SQL 中的表名或字段名与实际不符

**解决方案**：

1. **先验证表结构**
   ```typescript
   // 必须先获取表结构
   const detail = await get_dataset_detail({
     datasetCode: "users",
   });

   // 检查实际表名
   console.log("表名:", detail.basic.tableName); // 使用实际的表名

   // 检查字段名
   const fieldCodes = detail.fields.map(f => f.code);
   console.log("可用字段:", fieldCodes);
   ```

2. **验证 SQL 内容**
   ```typescript
   // 验证 SQL 语法
   const validation = await validate_sql_content({
     sqlContent: `
       SELECT id, name
       FROM correct_table_name  -- 使用实际的表名
       WHERE status = 1
     `,
     dbId: detail.basic.database.dbId,
   });

   if (!validation.validation.valid) {
     console.log("SQL 错误:", validation.validation.errors);
   }
   ```

3. **修复并保存**
   ```typescript
   // 使用正确的表名和字段名
   const sqlContent = `
     SELECT id, name
     FROM users            -- 实际表名
     WHERE status = 1
   `;

   await save_or_update_custom_sql({
     sqlContent,
     sqlCode: "user_list",
     sqlName: "用户列表",
     dbId: detail.basic.database.dbId,
   });
   ```

---

### 问题 5: SDK 返回错误

**现象**：
```typescript
try {
  // 支持两种参数格式
  const user = await client.models.users.getOne(userId);
  // 或
  const user = await client.models.users.getOne({ id: userId });
} catch (error) {
  console.log(error); // LovrabetError 对象
}
```

**解决方案**：

1. **查看错误信息**
   ```typescript
   if (error instanceof LovrabetError) {
     console.error("业务错误:", error.message);      // 业务错误信息
     console.error("错误代码:", error.code);        // 错误代码
     console.error("HTTP状态:", error.status);      // HTTP 状态码
     console.error("服务端响应:", error.response); // 完整响应
   }
   ```

2. **常见错误代码**
   ```
   - "0000": 成功
   - "API_ERROR": API 处理错误
   - "AUTH_FAILED": 认证失败
   - "PARAMETER_ERROR": 参数错误
   - "PERMISSION_DENIED": 权限不足
   ```

---

### 问题 6: 类型定义错误

**现象**：
```typescript
import { createClient } from '@lovrabet/sdk';

// 类型错误
const client = createClient({
  appCode: "app-xxx",
  models: {
    users: {
      tableName: "users",
      datasetCode: "xxx",
    },
  },
});

// ❌ 类型错误：类型不匹配
const user: User = await client.models.users.getOne(id);
```

**解决方案**：

1. **定义正确的类型**
   ```typescript
   // 定义与返回值匹配的类型
   interface User {
     id: string;
     name: string;
     email: string;
     createdAt: string;
   }

   // 使用泛型
   const user = await client.models.users.getOne<User>(id);
   ```

2. **使用 MCP 获取元数据**
   ```typescript
   // 先获取数据集详情，确保类型正确
   const detail = await get_dataset_detail({
     datasetCode: "users",
   });

   // 基于 fields 生成类型
   interface User {
     // 从 MCP 获取的字段动态生成类型
   }

   // 支持两种参数格式
   const user = await client.models.users.getOne<User>(id);
   // 或
   const user = await client.models.users.getOne<User>({ id });
   ```

---

### 问题 7: AntD 样式问题

**现象**：
- 团队反馈：界面看起来不专业
- 需求文档中提到 "不要 AI 味道"

**解决方案**：

1. **检查清单**
   - [ ] 没有 emoji
   - [ ] 没有感叹号
   - [ ] 文案简洁专业（"保存"不是"保存吧！"）
   - [ ] 使用 AntD token 颜色
   - [ ] 使用 @ant-design/icons

2. **代码示例**
   ```typescript
   // ✅ 正确
   <Button type="primary">保存</Button>
   <Button danger>删除</Button>
   <Message success>操作成功</Message>
   <Modal title="确认删除">...</Modal>
   <Empty description="暂无数据" />

   // ❌ 错误（AI 常见错误）
   <Button>保存！</Button>
   <Message success="🎉 操作成功！"></Message>
   <Modal title="确认删除？"><Icon type="question-circle" /></Modal>
   <Empty description="还没有数据哦~"></Empty>
   ```

---

## 🔧 排查工具

### 1. 检查 SDK 配置

```bash
# 查看 SDK 初始化代码
cat src/api/client.ts

# 检查 appcode 配置
cat .lovrabetrc

# 检查 MCP 配置（IDE 不同，配置文件位置不同）
cat .cursor/mcp.json       # Cursor
# 或
cat .claude/config.json    # claude code
```

### 2. 查看日志

```bash
# 查看 Lovrabet 日志
lovrabet logs

# 清空日志
lovrabet logs --clear
```

### 3. 使用 MCP 工具验证

```typescript
// 验证数据集是否存在
const datasets = await list_datasets();
const exists = datasets.datasets.some(d => d.code === "users");

// 验证字段是否存在
const detail = await get_dataset_detail({
  datasetCode: "users",
});
const hasField = detail.fields.some(f => f.code === "email");
```

### 4. 检查类型定义

```typescript
// 打印实际返回的数据结构（支持两种参数格式）
const user = await client.models.users.getOne("test-id");
// 或
const user = await client.models.users.getOne({ id: "test-id" });
console.log("实际返回:", JSON.stringify(user, null, 2));
```

---

## 📋 检查清单

在提交代码前，确认：

### SDK 部分
- [ ] Filter 查询使用了操作符 (`$eq`, `$gte` 等)
- [ ] 参数名正确（`select` 不是 `fields`）
- [ ] 排序参数名正确（`orderBy` 不是 `sort`）
- [ ] 分页参数名正确（`currentPage`/`pageSize` 不是 `page`/`limit`）
- [ ] 添加了数据集和数据表注释
- [ ] 所有 SDK 调用都包含在 try-catch 中

### MCP 部分
- [ ] 创建 SQL 前验证表名和字段名
  - 使用 `get_dataset_detail` 获取表结构
  - 对比 SQL 内容与实际表结构
- [ ] 创建 SQL 时执行了所有 5 个步骤
  - 查询 → 生成 → 验证 → 保存 → 测试
- [ ] SQL 中只使用 SELECT 查询
  - 没有 INSERT/UPDATE/DELETE

### UI 部分
- [ ] 没有 emoji
- [ ] 没有感叹号
- [ ] 文案简洁专业
- [ ] 使用 AntD token 颜色
- [ ] 使用 @ant-design/icons（不是 emoji）

---

## 🚨 常见错误速查表

| 错误现象 | 可能原因 | 解决方案 |
|---------|---------|---------|
| 查询无结果 | 忘记使用操作符 | `where: { status: { $eq: "active" } }` |
| 参数不正确 | 参数名错误 | `fields` → `select`，`sort` → `orderBy` |
| SQL 表不存在 | 表名或字段名错误 | 先用 `get_dataset_detail` 验证 |
| 认证失败 | Cookie 过期或 appcode 错误 | 运行 `lovrabet auth` 重新登录 |
| 类型错误 | 类型定义与实际返回值不匹配 | 使用 `getOne<类型>()` 泛型 |
| MCP 调用失败 | MCP 服务未配置 | 运行 `lovrabet mcp install` 配置 |

---

## 🔗 获取帮助

### CLI 命令
```bash
lovrabet logs          # 查看日志
lovrabet logs --clear  # 清空日志
lovrabet auth          # 重新登录
```

### MCP 工具
```typescript
// 验证配置和数据集
const datasets = await list_datasets();
const detail = await get_dataset_detail({ datasetCode: "users" });
```

### 开发者指南
```bash
# 查看 TypeScript SDK 使用指南
cat lovrabet-skill/guides/01-typescript-sdk.md

# 查看 MCP SQL 工作流指南
cat lovrabet-skill/guides/02-mcp-sql-workflow.md
```

---

**记住**：
- 遇到问题先看日志：`lovrabet logs`
- 验证 SQL 前先验证表结构：`get_dataset_detail`
- 检查错误代码：`error.code`
- 使用 MCP 工具验证：`list_datasets`
