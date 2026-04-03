---
name: lovrabet
description: 当用户请求涉及 Lovrabet/云兔/yuntoo 平台的数据集查询、SQL 创建、BFF 开发时使用。触发词：数据集、数据表、自定义 SQL、filter、sql.execute、bff.execute、get_dataset_detail、validate_sql_content、save_or_update_custom_sql、save_or_update_bff_script、@lovrabet/sdk、MCP SQL 工作流、多表关联、lovrabet 开发。
---

# Lovrabet 开发工作流

Lovrabet 中文名为**云兔**，也常写作 **yuntoo**。

## 核心原则

* 平台是唯一 source of truth；修改已有 SQL / BFF 时从平台拉最新内容
* 不要臆测 API、字段名、数据集 code，写代码前先用 MCP 获取元数据
* 直接调用 MCP 工具完成查询、验证、保存，不要用命令行绕行 MCP 协议
* SQL / BFF 写好后，通过 MCP 工具保存到平台，同时保留本地文件供版本管理

## 前置条件：MCP 可用检测

尝试调用任何 Lovrabet MCP 工具（如 `list_datasets`）。找到则继续；未找到则提示用户 `lovrabet mcp install`。

## MCP 工具速查

| 场景 | MCP Tool |
|------|----------|
| 列出数据集 | `list_datasets` |
| 搜索数据集 | `search_datasets` |
| 获取数据集详情 | `get_dataset_detail` |
| 生成 SDK 代码 | `generate_sdk_code` |
| 生成 SQL 代码 | `generate_sql_code` |
| 列出 SQL | `list_sql_queries` |
| 验证 SQL | `validate_sql_content` |
| 保存 SQL | `save_or_update_custom_sql` |
| 执行 SQL | `execute_custom_sql` |
| 列出 BFF（ENDPOINT） | `list_bff_scripts` |
| 列出公共函数（COMMON） | `list_bff_scripts`（type=COMMON） |
| 获取 BFF 详情 | `get_bff_script_info` |
| 保存 BFF | `save_or_update_bff_script` |

## SDK 核心规则

### 初始化

```typescript
import { createClient } from "@lovrabet/sdk";
const client = createClient({
  appCode: "your-app-code",
  accessKey: process.env.LOVRABET_ACCESS_KEY,
  models: [{ tableName: "users", datasetCode: "abc123", alias: "users" }],
});
```

### Filter 查询（最常用）

```typescript
const result = await client.models.users.filter({
  where: { status: { $eq: "active" } },
  select: ["id", "name"],
  orderBy: [{ createTime: "desc" }],
  currentPage: 1,
  pageSize: 20,
});
// result.tableData, result.total
```

参数名强制约束：`select`（非 fields）、`orderBy`（非 sort）、`currentPage`/`pageSize`（非 page/limit）

where 条件强制使用操作符：`$eq` `$ne` `$gte` `$lte` `$gt` `$lt` `$in` `$contain` `$startWith` `$endWith`

逻辑组合用 `$and` / `$or`，可嵌套。

### SQL 调用

```typescript
const data = await client.sql.execute<MyRow>({ sqlCode: "xxx", params: { key: "val" } });
if (data.execSuccess && data.execResult) {
  console.log(data.execResult); // T[]
}
```

返回 `{ execSuccess, execResult }`，应用层必须检查 `execSuccess`。

### BFF 调用

```typescript
const result = await client.bff.execute<DashboardData>({
  scriptName: "getUserDashboard",
  params: { userId: "123" },
});
console.log(result.userCount); // 直接是业务数据
```

返回业务数据对象，没有 `execSuccess` 层。

### 错误处理

所有 SDK 调用必须 try-catch，HTTP 级错误抛出 `LovrabetError`。

### 前端 vs BFF 关键差异

| | 前端 SDK | BFF (context.client) |
|---|---------|---------------------|
| SQL 返回值 | `{ execSuccess, execResult }` | 直接返回数组 `T[]` |
| 单条查询 | `getOne(id)` | `getOne({id: 1})` |
| 调 BFF | `client.bff.execute({ scriptName, params })` | — |

### 批量操作（SDK >= 1.2.0）

批量更新/删除单次最多 1000 条：
```typescript
await client.models.customer.update({ ids: [1, 2, 3], data: { status: "active" } });
await client.models.customer.delete({ ids: [1, 2, 3] });
```

### 性能约束

* 禁止循环单条查询（N+1），用 `filter + $in` 批量查询或多表关联
* 禁止循环写入，用自定义 SQL 批量处理
* 多表关联：`select: ["id", "profile.username"]`（1:1 或 N:1，使用表名引用）

## SQL 工作流

```
确认需求 → [按需]查现有 SQL → 校验字段 → 编写 SQL → 验证 → 保存到平台 → 测试 → 写入本地
```

1. 明确查询目标、字段、条件、是否 JOIN、新建还是修改
2. 修改已有 → `list_sql_queries` 找目标拉内容；不确定 → 查一下
3. `get_dataset_detail` 确认表名、字段名、`dbId`，禁止凭经验猜
4. 编写完整 SQL，头注释含 `@lovrabet.sqlName`、`@lovrabet.description`
5. `validate_sql_content` 验证，失败则修正后重新验证
6. `save_or_update_custom_sql` 保存（新建传 `dbId`，更新传 `id`；`sqlContent` 直接传原始 SQL 文本，不要做压缩、转义或写临时文件）
7. `execute_custom_sql` 测试执行
8. 写入本地 `src/custom_sql/<sqlName>.sql`（含 `@lovrabet.sqlCode`），纳入 Git

非 SELECT / blocked → 写草稿 `<sqlName>.draft.sql`，告知用户手动处理

### MyBatis 动态 SQL 要点

* 固定参数 `#{param}`，动态参数（`<if>` 内）`#{param, jdbcType=TYPE}`
* `<where>` 自动处理 AND/OR 前缀
* SQL 正文中 `>` 写 `&gt;`、`<` 写 `&lt;`；`<if test="">` 属性内不需要转义
* jdbcType：VARCHAR / INTEGER / DECIMAL / DATE / TIMESTAMP

## BFF 工作流

```
确认需求 → 校验字段 → [按需]查平台 → 编写脚本 → 自检 → 保存到平台 → 写入本地
```

1. 明确类型（ENDPOINT/HOOK）、函数名、入参、返回结构、涉及的数据集
2. `get_dataset_detail` 确认字段名、类型、枚举值、关联关系
3. 修改已有 → `list_bff_scripts` + `get_bff_script_info` 取 `id` 和最新内容
4. 编写完整脚本，使用 `export default async function`
5. 自检：方法名、`getOne` 统一、BFF 中 SQL 返回数组（非 `{ execSuccess, execResult }`）
6. `save_or_update_bff_script`（新建不传 `id`，更新传 `id`；必传 `scriptContent`、`description`、`scriptName`、`scriptType`；`scriptName` 是 `export default function` 后的函数名；`scriptType` 为 `ENDPOINT` 或 `COMMON`；`scriptContent` 直接传完整源码，不要做压缩、转义或写临时文件）
7. 写入本地：ENDPOINT → `src/backend-function/endpoint/endpoint_<name>.js`，HOOK → `src/backend-function/<tableName>/<tableName>_<hookName>.js`，COMMON → `src/backend-function/common/common_<name>.js`，纳入 Git

blocked → 写草稿 `.draft.js`，告知用户手动处理

### BFF 脚本类型

| 类型 | 用途 | 查询方式 |
|------|------|----------|
| ENDPOINT | 独立业务端点，前端通过 `client.bff.execute()` 调用 | `list_bff_scripts`（默认） |
| HOOK | 挂在标准数据接口前后执行（Before/After） | 通过数据集详情查看 |
| COMMON | 公共函数，供其他 BFF 脚本 import 复用 | `list_bff_scripts`（type=COMMON） |

写 BFF 前，先查一下公共函数列表（`list_bff_scripts` type=COMMON），看是否有可复用的工具函数，避免重复实现。

### 公共函数调用

在 BFF 中调用公共函数：`await context.client.bff.execute({ scriptName: 'commonXxx', params: {} })`

* 命名建议加 `common` 前缀，便于区分
* `scriptName` 必须精确匹配（大小写敏感）
* `params` 会被深拷贝，不影响调用方原始对象
* 禁止循环调用（A 调 B，B 又调 A）
* 异常会向上传播，调用方需 try-catch
* 修改公共函数出入参会影响所有引用方，建议新建 V2 逐步切换

### BFF 脚本要点

* HOOK Before：修改 `params`（filter/aggregate 改 `params.where`）
* HOOK After：修改 `params.tableData`
* ENDPOINT：独立业务端点，返回业务对象
* COMMON：公共工具函数，被其他脚本 import
* 数据集调用：`context.client.models.dataset_xxx`，推荐用 TABLES 映射表
* 系统字段（id、create_time、modify_time、create_by、modify_by）不要手动设置
* 事务：`await context.client.db.transaction(async (tx) => { ... })`

## 冲突处理

`save_or_update_custom_sql` / `save_or_update_bff_script` 返回 `blocked: true` 时：
* 告知用户手动在平台操作
* 将内容写入本地草稿文件
* 禁止重试、禁止绕过

未保存成功时必须明确告知用户，禁止含糊带过。

## 前端页面规则

* CLI 生成的页面顶部注释必须保留，追加说明用 `@modified`
* 页面组件设置 `displayName`
* 写页面前必须 `get_dataset_detail` 确认字段和关系
* 外键字段数据来自关联数据集接口，枚举字段来自定义的 options
* 禁止 emoji、AI 味文案、感叹号、花哨颜色；文案简洁专业
* 图标用 `@ant-design/icons`，颜色用 AntD token

## 接口选型优先级

遇到新需求时按优先级选择实现方式：
1. 标准 SDK 接口（filter/getOne/create 等）— 能用就不写 SQL
2. Aggregate 聚合接口 — 简单分组汇总
3. 自定义 SQL — 复杂 JOIN、数据库函数
4. BFF — 外部系统调用、跨表事务、复杂业务编排

## 深入指南

以下 guide 文件提供各主题的详细说明、完整示例和边界情况，遇到具体问题时可查阅：

| 主题 | Guide |
|------|-------|
| SDK 完整参数与返回值 | `./guides/01-typescript-sdk.md` |
| MCP SQL 工具与 MyBatis 语法 | `./guides/02-mcp-sql-workflow.md` |
| 前端页面开发约束 | `./guides/03-frontend-development.md` |
| 故障诊断手册 | `./guides/04-troubleshooting.md` |
| BFF 脚本编写规范 | `./guides/05-backend-function.md` |
| 数据接口访问规范 | `./guides/06-data-api-guidelines.md` |
| SQL 创建工作流细则 | `./guides/07-sql-creation-workflow.md` |
| BFF 创建工作流细则 | `./guides/08-bff-creation-workflow.md` |
| 冲突检测与处理 | `./guides/09-conflict-detection.md` |
| 开发质量与最佳实践 | `./guides/10-best-practices.md` |
