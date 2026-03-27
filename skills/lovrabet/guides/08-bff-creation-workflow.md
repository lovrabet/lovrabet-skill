# BFF 工作流规则

前置知识：`05-backend-function.md`

## 核心原则

平台是唯一 source of truth。AI 在内存中生成脚本后直接提交平台。
提交成功后回写一份本地文件，供人类阅读参考。
修改已有脚本时，从平台拉取最新内容，不读本地文件。

## 工作流

```
确认需求 → 校验字段 → [按需]查平台 → 生成脚本 → 自检 → 提交平台 → 回写本地
```

### 1. 确认需求
写码前必须明确：类型（ENDPOINT/HOOK）、函数名、入参、返回结构、涉及的数据集。缺失则先问用户。

### 2. 校验字段
调用 `get_dataset_detail` 确认字段名、类型、枚举值、关联关系。禁止凭经验猜字段名。

### 3. 查平台（按需）
* 新建 → 跳过
* 修改已有 → 调 `list_bff_scripts` + `get_bff_script_info` 取 `id` 和最新内容到内存
* 不确定 → 查一下

命中同名脚本时，停下问用户：修改还是另起新名。

### 4. 生成脚本
在内存（对话上下文）中生成完整脚本。不需要写入本地文件。

### 5. 自检
* 方法名正确
* 单条查询用 `getOne`
* BFF 中 `sql.execute` 返回数组，不是 `{ execSuccess, execResult }`
* 参数校验、错误处理、脱敏

### 6. 提交平台
调用 `save_or_update_bff_script`：
* 新建不传 `id`，更新传 `id`
* 必传 `scriptContent`（完整源码）、`description`、`functionName`

**`scriptContent` 传参规则（违反即为 BUG）**：
MCP 工具调用是结构化参数传递，框架自动处理序列化。
直接把内存中的完整脚本字符串作为 `scriptContent` 参数传入，结束。

以下行为全部禁止：
* 压缩、minify、uglify、删注释、删空行
* 手动 JSON.stringify / JSON-escape / 转义换行符
* 写任何临时文件或中间文件再读取
* 用 python / shell / node / bun -e 等外部命令构造参数或绕行调用
* 截断或摘要（不管多长都传完整源码）

### 7. 回写本地
提交成功后，将脚本内容写入本地文件，供人类查阅。
* ENDPOINT → `src/backend-function/endpoint/endpoint_<name>.js`
* HOOK → `src/backend-function/<tableName>/<tableName>_<hookName>.js`
* 若文件已存在，直接覆盖（无需警告，平台版本即最新）

## 冲突处理

返回 `blocked: true` → 将当前脚本内容写入本地草稿文件，告知用户手动处理，不重试。
草稿路径：`src/backend-function/endpoint/endpoint_<name>.draft.js` 或 HOOK 对应目录。

## 本地文件

正常流程：提交平台成功后回写本地，路径同 Step 7。
例外场景（直接写本地，不走正常回写）：
* 保存被 blocked → 写草稿（`.draft.js`），供人工处理
* 用户主动要求"同步平台最新到本地" → 从平台拉取 → 覆盖本地

修改已有脚本时，永远从平台拉取最新内容，不读本地文件。

## 修改已有脚本

`list_bff_scripts` → `get_bff_script_info` 拉最新内容到内存 → 修改 → 带 `id` 提交 → 回写本地。
永远从平台拉取，不读本地文件。

## BFF 语义差异

| 场景 | 前端 SDK | BFF (context.client) |
|------|---------|---------------------|
| SQL 返回值 | `{ execSuccess, execResult }` | 直接返回数组 |
| 单条查询 | `getOne(id)` | `getOne(id)` |
| 前端调 BFF | `client.bff.execute({ scriptName, params })` 返回业务数据 | — |

## MCP 不可用时

看不到 `save_or_update_bff_script` 工具 → 提醒用户启用 `--enable-bff-save`，不假装能保存。
