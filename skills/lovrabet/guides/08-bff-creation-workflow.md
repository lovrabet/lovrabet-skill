# BFF 工作流规则

前置知识：`05-backend-function.md`

## 工作流

```
确认需求 → 校验字段 → [按需]查平台 → 本地落盘 → 自检 → 保存平台
```

### 1. 确认需求
写码前必须明确：类型（ENDPOINT/HOOK）、函数名、入参、返回结构、涉及的数据集。缺失则先问用户。

### 2. 校验字段
调用 `get_dataset_detail` 确认字段名、类型、枚举值、关联关系。禁止凭经验猜字段名。

### 3. 查平台（按需）
* 新建 → 跳过（平台校验 functionName 唯一性）
* 修改已有 → 调 `list_bff_scripts` + `get_bff_script_info` 取 `id` 和最新内容
* 本地已有 `id` → 跳过
* 不确定 → 查一下

命中同名脚本时，停下问用户：修改还是另起新名。

### 4. 本地落盘
先写本地文件再保存平台。路径规则：
* ENDPOINT → `src/backend-function/endpoint/endpoint_<name>.js`
* HOOK → `src/backend-function/<tableName>/`

禁止：只输出代码不落盘、用 `/tmp/`、未落盘就调保存。

### 5. 自检
* 方法名与文件名匹配
* 单条查询用 `getOne`
* BFF 中 `sql.execute` 返回数组，不是 `{ execSuccess, execResult }`
* 参数校验、错误处理、脱敏

### 6. 保存平台
调用 `save_or_update_bff_script`：
* 新建不传 `id`，更新传 `id`
* 必传 `scriptContent`、`description`

**`scriptContent` 传参规则（违反即为 BUG）**：
MCP 工具调用是结构化参数传递，框架自动处理 JSON 序列化，你不需要也不应该手工构造 JSON。
正确做法：读取本地文件内容 → 作为 `scriptContent` 参数值直接传入 MCP tool call → 结束。
就像调用函数传字符串参数一样简单，不需要任何中间步骤。

以下行为全部禁止：
* 压缩、minify、uglify、删注释、删空行
* 手动 JSON.stringify / JSON-escape / 转义换行符
* 写 .json 载荷文件再读取或解析
* 用 python / shell / node / bun -e 等外部命令构造参数或绕行调用
* 截断或摘要（不管多长都传完整源码）
* 以"生成合法载荷"为由做任何额外处理

## 冲突处理

返回 `blocked: true` → 告知用户去平台手动操作，不重试。
返回 `confirmationRequired: true` → 停下说明风险，等用户确认，不自动重试。

## 修改已有脚本

`list_bff_scripts` → `get_bff_script_info` → 合并到本地 → 改 → 带 `id` 保存。禁止不拉最新就覆盖。

## BFF 语义差异

| 场景 | 前端 SDK | BFF (context.client) |
|------|---------|---------------------|
| SQL 返回值 | `{ execSuccess, execResult }` | 直接返回数组 |
| 单条查询 | `getOne(id)` | `getOne(id)` |
| 前端调 BFF | `client.bff.execute({ scriptName, params })` 返回业务数据 | — |

## MCP 不可用时

看不到 `save_or_update_bff_script` 工具 → 提醒用户启用 `--enable-bff-save`，不假装能保存。
