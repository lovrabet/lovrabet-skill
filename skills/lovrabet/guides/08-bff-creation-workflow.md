---
requireFlags:
  - enable-bff-save
---
# Backend Function 创建与同步工作流

> 目标：约束 AI 在 Lovrabet 项目中创建、修改、同步 Backend Function 时的行为，避免凭空编造、误写平台脚本、覆盖他人修改。
>
> 前置阅读：`05-backend-function.md`

## 何时使用

当任务满足任一条件时，必须阅读并遵守本指南：

* 创建新的 Backend Function
* 修改已有 Backend Function
* 将本地 BFF 脚本同步到平台
* 检查平台是否已有同名或相似 BFF
* 处理 BFF 保存冲突、作者校验或强制更新

## 核心原则

* 本地文件优先，平台保存次之
* 先校验数据集和字段，再写业务逻辑
* 先检查重名和现有脚本，再决定新建还是修改
* 未确认前，不得强制覆盖平台脚本
* BFF 场景下，前后端单条查询统一使用 `getOne`

## 本地文件约定

与 `05-backend-function.md` 保持一致：

* 根目录：`src/backend-function/`
* ENDPOINT：`src/backend-function/endpoint/endpoint_<scriptName>.ts` 或 `.js`
* HOOK：`src/backend-function/<tableName>/`
* 本地文件内容必须与提交到平台的 `scriptContent` 一致

禁止行为：

* 只在对话中输出整段 BFF 代码，不在工作区落盘
* 使用临时目录如 `/tmp/` 存放业务脚本
* 未保存本地文件就直接调用 `save_or_update_bff_script`

## MCP 与 BFF 写入能力

如果当前环境看不到以下工具：

* `list_bff_scripts`
* `get_bff_script_info`
* `save_or_update_bff_script`

则说明当前 MCP 未启用 BFF 写入能力。此时 AI 必须：

1. 明确提醒开发者：当前环境无法读取或保存平台 BFF
2. 不得假装可以保存到平台
3. 仅在明确需要 BFF 同步/保存能力时，建议按需启用 `--enable-bff-save`

可用参数：

* 新参数：`--enable-bff-save`
* 旧参数：`--dangerously-bff-save`

示例：

```json
"args": ["-y", "@lovrabet/dataset-mcp-server@latest", "--enable-bff-save"]
```

风险提醒：

* 该参数会开启 BFF 相关读写能力
* 不需要 BFF 保存能力时，不应默认开启
* 开启后仍需遵守本指南的检查重名、冲突确认、先本地后平台等规则

## 强制工作流

```text
理解需求 -> 校验数据集与字段 -> 检查平台现状 -> 本地落盘 -> 验证 -> 保存到平台
```

### Step 1：理解需求

开始写代码前，AI 必须确认：

* 脚本类型：ENDPOINT 还是 HOOK
* 脚本名称或目标文件名
* 输入参数
* 返回数据结构
* 涉及哪些数据集
* 是否有权限校验、数据脱敏、事务、外部依赖

若这些信息缺失，必须先问清楚，再继续。

### Step 2：校验数据集、字段与关系

写任何业务逻辑前，必须先调用：

* `list_datasets` 或 `search_datasets`
* `get_dataset_detail`

必须确认：

* 字段真实存在
* 字段类型正确
* 枚举值真实存在
* 关联关系真实存在

禁止行为：

* 未调用 `get_dataset_detail` 就直接写字段名
* 凭经验猜 `user_id`、`status`、`deleted` 等常见字段
* 把前端 SDK 或其他项目里的字段名直接套到当前数据集

### Step 3：检查平台现状

写入平台前，必须先调用 `list_bff_scripts`。

目的：

* 检查是否已有同名脚本
* 检查是否已有相似功能脚本
* 决定这次是新建还是修改

若命中同名或相似脚本，必须继续调用 `get_bff_script_info` 拉取最新内容。

此时 AI 必须停止自动推进，并请开发者确认：

* 修改现有脚本
* 另起一个新脚本名

未确认前，不得直接生成“默认新建”方案。

### Step 4：本地落盘

AI 必须先在工作区创建或修改本地文件，再进入保存流程。

ENDPOINT 示例路径：

```text
src/backend-function/endpoint/endpoint_create-order.ts
```

本地文件必须包含：

* 文件头注释
* `export default async function`
* 必要的参数校验
* 明确的返回结构
* 数据集映射
* 必要时的版本记录块

### Step 5：验证

保存到平台前，至少完成以下检查：

* 方法名与文件名匹配
* 顶部注释占位符已替换
* 单条查询使用 `getOne`
* SQL 调用理解 BFF 返回值是数组，不是 `{ execSuccess, execResult }`
* 敏感数据脱敏已处理
* 权限控制已处理
* 涉及批量操作或事务时，逻辑边界清晰

若发现以下问题，必须先修复再保存：

* 字段未校验
* 返回结构不清晰
* 误用前端 SDK 语义
* 存在明显覆盖风险

### Step 6：保存到平台

保存平台时使用 `save_or_update_bff_script`：

* 新建：不传 `id`
* 更新：传 `id`

必填信息至少包括：

* `appCode`
* `scriptContent`
* `scriptType`
* `description`

保存前必须保证：

* 平台上的目标脚本已检查
* 本地文件已保存
* `scriptContent` 与本地文件一致

## 冲突与作者校验

平台不是传统的“加锁编辑”模式，但新的 MCP 保存时会做作者校验与冲突检测。

AI 必须关注以下返回信号：

* `blocked`
* `confirmationRequired`
* `nextAction`
* `existing`

处理规则：

* LOW：通常是自己的脚本，可继续处理
* MEDIUM：提示可能重名或相似功能，优先建议改名或人工确认
* HIGH：必须停下，先展示冲突信息，再请求开发者确认

如果返回 `confirmationRequired: true`：

* 不得自动重试
* 不得自动加 `forceUpdate: true`
* 必须先向开发者说明风险，再等待确认

## 修改已有脚本的最小流程

1. `list_bff_scripts`
2. `get_bff_script_info`
3. 合并平台最新内容到本地文件
4. 修改本地文件
5. 再调用 `save_or_update_bff_script`

禁止行为：

* 基于旧缓存内容直接覆盖平台
* 未读取平台最新版本就直接保存

## BFF 特有注意事项

### SQL 返回值差异

前端 SDK：

```javascript
const data = await client.sql.execute({ sqlCode: "xxx" });
if (data.execSuccess) {
  const rows = data.execResult || [];
}
```

Backend Function：

```javascript
const rows = await context.client.sql.execute({ sqlCode: "xxx" });
// rows 直接是数组
```

### 单条查询方法

前端和 Backend Function 保持一致：

```javascript
const record = await models[TABLES.orders].getOne(id);
```

不要再写：

```javascript
const record = await models[TABLES.orders].findOne({ id });
```

### 前端调用方式

前端 SDK 调 BFF 时，直接使用 `client.bff.execute()` 返回的业务数据：

```typescript
const result = await client.bff.execute({
  scriptName: "create-order",
  params: {
    productId: "123",
    quantity: 2,
  },
});

console.log(result);
```

不要写成前端 SDK 返回 `{ success, data }` 的形式。

### 风险清单

保存前必须自检：

* 参数验证是否完整
* 错误处理是否明确
* 敏感数据是否脱敏
* 权限是否校验
* SQL 返回值是否按 BFF 语义处理
* 是否存在明显性能问题
* 是否可能误覆盖平台脚本

## AI 输出要求

当 AI 完成 BFF 生成或修改后，输出中应至少包含：

* 本地文件路径
* 当前脚本是新建还是修改
* 已校验的数据集与字段
* 是否已检查平台同名脚本
* 下一步是否可以保存到平台
* 如不能保存，明确说明阻塞原因

如果可以直接保存，仍应说明：

* 将调用哪个 MCP 工具
* 是否存在冲突风险
* 是否需要开发者确认

## 常见问题

### Q：多人同时修改同一脚本怎么办？

A：新的 MCP 保存时会做作者校验和冲突检测，不应再简单理解为“后保存者一定覆盖前者”。AI 仍应：

* 修改前先拉取最新内容
* 保存时关注 `confirmationRequired`、`blocked` 和作者校验结果
* 遇到高风险冲突时停止并请求确认

### Q：如何回滚到之前的版本？

A：平台支持查看 Backend Function 历史版本：

`https://app.lovrabet.com/app/{appCode}/data/backend-function`

其中 `appCode` 为当前应用的 `appCode`。

同时建议：

* 本地保留脚本备份
* 重要脚本提交到 Git 仓库
* 使用版本记录块追踪修改历史

### Q：脚本保存后如何生效？

A：

* 保存后立即生效
* 建议先在测试环境验证
* 可用前端调用或 HTTP 端点验证

### Q：为什么看不到 `list_bff_scripts` 或 `save_or_update_bff_script`？

A：

* 检查 MCP 参数是否启用了 `--enable-bff-save`
* 旧环境也可能使用 `--dangerously-bff-save`
* 若当前只是普通数据集开发，不必默认开启
* 若明确需要 BFF 保存能力，再按需开启，并提醒开发者自控误写风险

## 相关指南

* `05-backend-function.md`
* `06-data-api-guidelines.md`
* `07-sql-creation-workflow.md`
* `09-conflict-detection.md`
* `10-best-practices.md`
