# BFF 工作流规则

前置知识：`05-backend-function.md`

## 核心原则

平台是唯一 source of truth。修改已有脚本时，从平台拉取最新内容。

## 工作流

```
速查公共函数 → 确认需求 → 校验字段 → [按需]查平台 → 编写脚本 → 自检 → 保存到平台 → 写入本地
```

### 0. 速查公共函数
调用 `list_bff_scripts`（type=COMMON）查看已有公共函数。如有可复用的工具函数，在后续脚本中直接 import，避免重复实现。

### 1. 确认需求
写码前必须明确：类型（ENDPOINT/HOOK/COMMON）、函数名、入参、返回结构、涉及的数据集。缺失则先问用户。

### 2. 校验字段
调用 `get_dataset_detail` 确认字段名、类型、枚举值、关联关系。禁止凭经验猜字段名。

### 3. 查平台（按需）
* 新建 → 跳过
* 修改已有 → 调 `list_bff_scripts`（ENDPOINT 或 COMMON） + `get_bff_script_info` 取 `id` 和最新内容
* 不确定 → 查一下

命中同名脚本时，停下问用户：修改还是另起新名。

### 4. 编写脚本
编写完整脚本。

### 5. 自检
* 方法名正确
* 单条查询用 `getOne`
* BFF 中 `sql.execute` 返回数组，不是 `{ execSuccess, execResult }`
* 参数校验、错误处理、脱敏

### 6. 保存到平台
调用 `save_or_update_bff_script`：
* 新建不传 `id`，更新传 `id`
* 必传 `scriptContent`、`description`、`scriptName`（与脚本中 `export default async function` 后的函数名一致）、`scriptType`（`ENDPOINT` 或 `COMMON`）
* `scriptContent` 直接传完整源码原文，不要做压缩、JSON 转义、缩短变量名或写临时文件

### 7. 写入本地
保存成功后，将脚本内容写入本地文件，纳入 Git 版本管理。
* ENDPOINT → `src/backend-function/endpoint/endpoint_<name>.js`
* HOOK → `src/backend-function/<tableName>/<tableName>_<hookName>.js`
* COMMON → `src/backend-function/common/common_<name>.js`
* 若文件已存在，直接覆盖

## 冲突处理

返回 `blocked: true` → 将当前脚本内容写入本地草稿文件，告知用户手动处理，不重试。
草稿路径：`src/backend-function/endpoint/endpoint_<name>.draft.js` 或 HOOK 对应目录。

## 本地文件

正常流程：保存平台成功后写入本地，路径同 Step 7。
例外场景：
* 保存被 blocked → 写草稿（`.draft.js`），供人工处理
* 用户主动要求"同步平台最新到本地" → 从平台拉取 → 覆盖本地

修改已有脚本时，从平台拉取最新内容。

## 修改已有脚本

`list_bff_scripts` → `get_bff_script_info` 拉最新内容 → 修改 → 带 `id` 保存 → 写入本地。

## BFF 语义差异

| 场景 | 前端 SDK | BFF (context.client) |
|------|---------|---------------------|
| SQL 返回值 | `{ execSuccess, execResult }` | 直接返回数组 |
| 单条查询 | `getOne(id)` | `getOne(id)` |
| 前端调 BFF | `client.bff.execute({ scriptName, params })` 返回业务数据 | — |

## MCP 不可用时

看不到 `save_or_update_bff_script` 工具 → 提醒用户启用 `--enable-bff-save`，不假装能保存。
