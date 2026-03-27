# SQL 工作流规则

前置知识：`02-mcp-sql-workflow.md`、`06-data-api-guidelines.md`

## 核心原则

平台是唯一 source of truth。AI 在内存中生成 SQL 后直接提交平台。
提交成功后回写一份本地文件，供人类阅读参考。
修改已有 SQL 时，从平台拉取最新内容，不读本地文件。

## 工作流

```
确认需求 → [按需]查现有 SQL → 校验字段 → 生成 SQL → 验证 → 提交平台 → 测试 → 回写本地
```

### 1. 确认需求
写 SQL 前必须明确：查询目标、字段、筛选条件、排序分页、是否 JOIN、新建还是修改。缺失则先问用户。

### 2. 查现有 SQL（按需）
* 新建 → 可跳过
* 修改已有 → 调 `list_sql_queries` 找到目标，确认 sqlCode，拉取内容到内存
* 不确定 → 查一下

命中同名或同语义 SQL 时，停下问用户：沿用还是另建。

### 3. 校验字段
调用 `get_dataset_detail` 确认表名、字段名、字段类型。禁止凭经验猜。

### 4. 生成 SQL
在内存（对话上下文）中生成完整 SQL。带 `@lovrabet` 头注释：
```sql
-- @lovrabet.sqlName: <领域>-<用途>
-- @lovrabet.description: <一句话说明>
```
不需要写入本地文件。

### 5. 验证
调用 `validate_sql_content` 传入内存中的 SQL 内容。未通过则在内存中修正后重新验证。

### 6. 提交平台
调用 `save_or_update_custom_sql`：
* 新建需传 `dbId`（从 `get_dataset_detail` 获取）
* 传 `sqlContent`（完整 SQL）和 `sqlName`

**`sqlContent` 传参规则（违反即为 BUG）**：
MCP 工具调用是结构化参数传递，框架自动处理序列化。
直接把内存中的完整 SQL 字符串作为 `sqlContent` 参数传入，结束。

以下行为全部禁止：
* 压缩、删注释、删空行、合并为单行
* 手动 JSON.stringify / JSON-escape / 转义换行符
* 写任何临时文件或中间文件再读取
* 用 python / shell / node / bun -e 等外部命令构造参数或绕行调用
* 截断或摘要（不管多长都传完整内容）

### 7. 测试
保存成功后调 `execute_custom_sql` 验证。失败则在内存中修正 → validate → save → execute。

### 8. 回写本地
测试通过后，将 SQL 内容写入本地文件，供人类查阅。
路径：`src/custom_sql/<sqlName>.sql`
头注释须含 `@lovrabet.sqlName`、`@lovrabet.sqlCode`（从保存响应获取）、`@lovrabet.description`。
若文件已存在，直接覆盖（无需警告，平台版本即最新）。

## 非 SELECT 语句

DELETE / DDL（DROP / ALTER / CREATE / TRUNCATE）属高风险，不走自动保存。
将 SQL 写入本地草稿文件，告知用户手动在平台操作。
草稿路径：`src/custom_sql/<sqlName>.draft.sql`

## 冲突处理

返回 `blocked: true` → 将当前 SQL 写入本地草稿文件，告知用户手动处理，不重试。
草稿路径：`src/custom_sql/<sqlName>.draft.sql`

## 本地文件

正常流程：提交平台 + 测试通过后回写本地，路径及格式同 Step 8。
例外场景（直接写本地，不走正常回写）：
* 保存被 blocked 或属于 DELETE/DDL → 写草稿（`.draft.sql`），供人工处理
* 用户主动要求"同步平台最新到本地" → 从平台拉取 → 覆盖本地

修改已有 SQL 时，永远从平台拉取最新内容，不读本地文件。

## SQL 调用差异

| 场景 | 前端 SDK | BFF (context.client) |
|------|---------|---------------------|
| 返回值 | `{ execSuccess, execResult }` | 直接返回数组 |
| 调用 | `client.sql.execute({ sqlCode, params })` | `context.client.sql.execute({ sqlCode, params })` |

## MCP 不可用时

看不到 `save_or_update_custom_sql` 工具 → 提醒用户检查 MCP 配置，不假装能保存。
