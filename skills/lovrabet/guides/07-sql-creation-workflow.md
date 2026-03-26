# SQL 工作流规则

前置知识：`02-mcp-sql-workflow.md`、`06-data-api-guidelines.md`

## 工作流

```
确认需求 → [按需]查现有 SQL → 校验字段 → 本地落盘 → 验证 → 保存 → 测试
```

### 1. 确认需求
写 SQL 前必须明确：查询目标、字段、筛选条件、排序分页、是否 JOIN、新建还是修改。缺失则先问用户。

### 2. 查现有 SQL（按需）
* 新建 → 可跳过
* 修改已有 → 调 `list_sql_queries` 找到目标，确认 sqlCode
* 不确定 → 查一下

命中同名或同语义 SQL 时，停下问用户：沿用还是另建。

### 3. 校验字段
调用 `get_dataset_detail` 确认表名、字段名、字段类型。禁止凭经验猜。

### 4. 本地落盘
先写本地 `.sql` 文件再验证保存。

文件命名规则：统一用 `sqlName` 命名，路径 `src/custom_sql/<sqlName>.sql`
* 新建 → `sqlName` 由 AI 按业务语义生成，格式 `<领域>-<用途>`
* 从平台拉取已有 SQL → 用平台返回的 `sqlName` 命名

文件必须以头注释开头：
```sql
-- @lovrabet.sqlName: <领域>-<用途>
-- @lovrabet.sqlCode: <平台返回的 sqlCode，新建时不写此行>
-- @lovrabet.description: <一句话说明>
```

规则：
* `sqlName` 由 AI 按业务语义生成，保存后若返回 `sqlNameSource: "auto"`，说明缺 `@lovrabet.sqlName`，需补上
* `sqlCode` 由平台分配，新建时不写；保存成功或从平台拉取后，补上 `@lovrabet.sqlCode` 行
* 从平台拉取的 SQL 若缺头注释，落盘时用平台返回的 `name`、`sqlCode`、`description` 补全

禁止：只在对话贴 SQL 不落盘、未落盘就调保存。

### 5. 验证
调用 `validate_sql_content`。未通过则修正本地文件后重新验证。未验证通过禁止保存。

### 6. 保存
调用 `save_or_update_custom_sql`：
* 新建需传 `dbId`
* `sqlContent` 与本地文件一致

**`sqlContent` 传参规则（违反即为 BUG）**：
MCP 工具调用是结构化参数传递，框架自动处理 JSON 序列化，你不需要也不应该手工构造 JSON。
正确做法：读取本地 SQL 文件内容 → 作为 `sqlContent` 参数值直接传入 MCP tool call → 结束。
就像调用函数传字符串参数一样简单，不需要任何中间步骤。

以下行为全部禁止：
* 压缩、删注释、删空行、合并为单行
* 手动 JSON.stringify / JSON-escape / 转义换行符
* 写 .json 载荷文件再读取或解析
* 用 python / shell / node / bun -e 等外部命令构造参数或绕行调用
* 截断或摘要（不管多长都传完整内容）
* 以"生成合法载荷"为由做任何额外处理

### 7. 测试
保存成功后调 `execute_custom_sql` 验证。失败则闭环：改本地 → validate → save → execute。

## 非 SELECT 语句

DELETE / DDL（DROP / ALTER / CREATE / TRUNCATE）属高风险，不走自动保存。提醒用户后保留本地文件作评审记录。

## 冲突处理

返回 `blocked: true` → 告知用户去平台手动操作或改名，不重试，不绕过。

## SQL 调用差异

| 场景 | 前端 SDK | BFF (context.client) |
|------|---------|---------------------|
| 返回值 | `{ execSuccess, execResult }` | 直接返回数组 |
| 调用 | `client.sql.execute({ sqlCode, params })` | `context.client.sql.execute({ sqlCode, params })` |

## MCP 不可用时

看不到 `save_or_update_custom_sql` 工具 → 提醒用户检查 MCP 配置，不假装能保存。
