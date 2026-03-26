---
# SQL 创建工作流

> 目标：约束 AI 在 Lovrabet 项目中创建、修改、验证、保存 SQL 时的行为，避免凭空编造字段、误存高风险语句、覆盖他人资源。
>
> 前置阅读：`02-mcp-sql-workflow.md`、`06-data-api-guidelines.md`

## 何时使用

当任务满足任一条件时，必须阅读并遵守本指南：

* 创建新的自定义 SQL
* 修改已有 SQL
* 将本地 SQL 保存到平台
* 处理 SQL 命名冲突、blocked 或自动命名兜底
* 处理 SQL 验证失败或执行失败

## 核心原则

* 本地文件优先，平台保存次之
* 先校验数据集和字段，再写 SQL
* SQL 名称由 AI 根据业务语义生成，不默认交给用户决定
* 未验证通过前，不得保存到平台
* 遇到 `blocked` 时，不得自动重试或绕过

## 本地文件约定

自定义 SQL 本地稿放在 `src/custom_sql/`：

* 文件按业务功能组织
* 文件名可与 `@lovrabet.sqlName` 一致
* 调用 `validate_sql_content`、`save_or_update_custom_sql` 时，使用磁盘上已保存的当前内容

禁止行为：

* 只在对话中贴 SQL，不在工作区落盘
* 把不同业务功能混进一个 `insert.sql`、`update.sql` 之类的通用文件
* 未保存本地文件就假定“已同步到平台”

## `@lovrabet` 头注释

所有 AI 生成的 SQL 必须以以下注释开头：

```sql
-- @lovrabet.sqlName: <business-name>
-- @lovrabet.description: <one-line-description>
```

规则：

* `sqlName` 由 AI 按业务语义生成，格式建议为 `<领域>-<用途>`
* `description` 用一句话说明查询目的
* 头注释必须位于 SQL 文件最前面
* 平台保存时，MCP 会优先从头注释提取 `sqlName` 和 `description`

### 自动命名兜底

若保存结果出现 `sqlNameSource: "auto"`：

* 说明这条 SQL 缺少有业务语义的名字
* AI 必须提醒开发者补充 `@lovrabet.sqlName`
* 不应把自动生成的结构化名字视为最终结果

## 强制工作流

```text
理解需求 -> 查询现有 SQL -> 校验数据集与字段 -> 本地落盘 -> 验证 -> 保存 -> 测试执行
```

### Step 1：理解需求

开始写 SQL 前，AI 必须确认：

* 查询目标是什么
* 需要哪些字段
* 筛选条件是什么
* 是否需要排序或分页
* 是否需要 JOIN
* 是新建还是修改现有 SQL

若这些信息缺失，必须先问清楚，再继续。

### Step 2：查询现有 SQL

开始创建或修改前，必须先调用 `list_sql_queries`。

目的：

* 检查是否已有同名 SQL
* 检查是否已有相似语义 SQL
* 决定这次是新建还是修改

若已命中现有 SQL：

* 必须先确认是沿用现有 SQL 还是另起新名字
* 未确认前，不得默认新建一个语义重复的 SQL

### Step 3：校验数据集、表和字段

写 SQL 前，必须先调用：

* `list_datasets` 或 `search_datasets`
* `get_dataset_detail`

必须确认：

* 数据集真实存在
* 表名真实存在
* 字段真实存在
* 字段类型匹配 SQL 用法
* INSERT / UPDATE 涉及的必填字段已确认

禁止行为：

* 未调用 `get_dataset_detail` 就直接写字段名
* 凭经验猜表名、字段名
* 把其他项目里的字段名直接套用到当前 SQL

### Step 4：本地落盘

AI 必须先在工作区创建或修改本地 `.sql` 文件，再进入验证和保存流程。

示例路径：

```text
src/custom_sql/customer-active-list.sql
```

本地 SQL 文件必须包含：

* `@lovrabet.sqlName`
* `@lovrabet.description`
* 可执行的 SQL 主体

### Step 5：验证

保存到平台前，必须调用 `validate_sql_content`。

验证目标：

* SQL 语法正确
* 字段名、表名正确
* 参数化写法正确
* 不包含明显的非法语句结构

如果验证失败，AI 必须：

1. 解释失败原因
2. 回到本地文件修正 SQL
3. 重新验证

未验证通过前，不得继续保存。

### Step 6：保存到平台

保存时使用 `save_or_update_custom_sql`。

保存前必须保证：

* 本地 SQL 文件已保存
* `sqlContent` 与本地文件一致
* 新建场景已拿到 `dbId`
* 命名和描述语义清晰

### Step 7：测试执行

保存成功后，必须使用 `execute_custom_sql` 测试执行结果。

如果测试失败，必须形成闭环：

```text
修改本地 SQL -> validate_sql_content -> save_or_update_custom_sql -> execute_custom_sql
```

## 非 SELECT 语句的处理

DELETE、DDL（如 DROP / ALTER / CREATE / TRUNCATE）等高风险语句，不应按普通查询 SQL 直接自动保存到平台。

AI 必须：

* 先提醒开发者该语句属于高风险类型
* 继续保留本地文件稿
* 不得假装它能像普通 SELECT 一样直接走自动保存路径

这类 SQL 仍建议保留在 `src/custom_sql/` 中，作为仓库内记录与评审对象。

## 冲突与 blocked

`save_or_update_custom_sql` 可能返回：

* `blocked`
* `message`
* `sqlNameSource`

如果返回 `blocked: true`：

* 说明同名 SQL 上次由其他人提交，或当前操作被平台规则阻断
* AI 不得自动重试
* AI 不得编造任何绕过参数
* AI 必须直接告知开发者：请前往平台手动处理，或改一个新名字

## AI 输出要求

当 AI 完成 SQL 生成或修改后，输出中至少包含：

* 本地 SQL 文件路径
* 当前 SQL 是新建还是修改
* 已校验的数据集与字段
* 当前 `sqlName`
* 是否已验证通过
* 是否可以继续保存到平台
* 若不能保存，明确阻塞原因

若保存成功，还应补充：

* SQL 标识信息
* 下一步测试方式
* 在前端或 BFF 中的调用方式

## 调用方式差异

### 前端 SDK

```typescript
const result = await client.sql.execute({
  sqlCode: "customer-active-list",
  params: {
    startDate: "2024-01-01",
    pageSize: 100,
  },
});

if (result.execSuccess && result.execResult) {
  console.log(result.execResult);
}
```

### Backend Function

```javascript
const rows = await context.client.sql.execute({
  sqlCode: "customer-active-list",
  params,
});

// rows 直接是数组
```

不要把前端 SDK 的 `{ execSuccess, execResult }` 结构套用到 BFF。

## 常见问题

### Q：是否要先查现有 SQL？

A：对于新建或修改 SQL，优先先通过平台查询现状，避免重复创建或误改现有资源。

### Q：验证失败怎么办？

A：先修正本地 SQL，再重新验证；不要跳过验证直接保存。

### Q：`blocked` 怎么办？

A：停止自动推进，告知开发者手动处理或改名，不要重试。

### Q：为什么 SQL 名称是自动生成的？

A：说明缺少 `@lovrabet.sqlName` 头注释。AI 应补上语义化命名，再重新保存。

## 相关指南

* `02-mcp-sql-workflow.md`
* `06-data-api-guidelines.md`
* `09-conflict-detection.md`
* `10-best-practices.md`
