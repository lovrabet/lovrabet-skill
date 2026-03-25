# Lovrabet Skill（开放生态版）

版本：1.0.0  
来源：[lovrabet-skill](https://github.com/lovrabet/lovrabet-skill)

## 简介

Lovrabet（云兔）是一个 AI-Native B 端开发平台，本 skill 包含 Lovrabet 开发的工作流规范、SQL / BFF 创建规范和 MCP 工具路由指引，帮助 AI 助手（Cursor / Claude 等）在 Lovrabet 项目中正确执行任务。

## 安装方式

### 方式 1：平台内安装（推荐，完整功能）

```bash
lovrabet skill install
```

适用场景：

* 已使用 `lovrabet-cli`
* 需要和 rules、flags、config 联动
* 需要完整受控安装流程

### 方式 2：开放生态安装（快速接入）

```bash
npx skills add lovrabet/lovrabet-skill
```

适用场景：

* 希望快速接入通用 agent 技能生态
* 希望先试用 Lovrabet skill 能力
* 不一定使用 lovrabet-cli

## 能力矩阵

| 能力 | 无 MCP | 有 MCP |
|------|--------|--------|
| 阅读 guide 文档 | 支持 | 支持 |
| 本地编写 SQL 文件（src/custom_sql/） | 支持 | 支持 |
| 本地编写 BFF 文件（src/backend-function/） | 支持 | 支持 |
| 查询平台数据集结构 | 不支持 | 支持（list_datasets / get_dataset_detail） |
| 验证 SQL 语法 | 不支持 | 支持（validate_sql_content） |
| 保存 SQL 到平台 | 不支持 | 支持（save_or_update_custom_sql） |
| 保存 BFF 到平台 | 不支持 | 支持（save_or_update_bff_script） |
| 执行 SQL | 不支持 | 支持（execute_custom_sql） |
| 冲突检测 | 不支持 | 支持（自动） |

## 无 MCP 时的风险提示

如果你的 AI 助手没有配置 Lovrabet MCP 服务：

* 可以阅读 guides 文档，了解规范和工作流
* 可以在本地 `src/custom_sql/` 编写 `.sql` 草稿文件
* 可以在本地 `src/backend-function/` 编写 BFF 草稿
* **不能**完成 SQL / BFF 的平台验证、保存和执行
* **不能**查询平台数据集的字段结构
* AI 在无 MCP 时必须明确告知用户上述限制，不得假装已完成平台写入

安装 MCP：

```bash
lovrabet mcp install
```

## Guides 文档（本版本共 11 个）

安装后，guides 位于 `guides/` 目录下：

| 文件 | 说明 |
|------|------|
| `01-typescript-sdk.md` | TypeScript SDK 用法 |
| `02-mcp-sql-workflow.md` | MCP SQL 工作流（必读） |
| `03-frontend-development.md` | React 前端开发 |
| `04-troubleshooting.md` | 故障排查 |
| `05-backend-function.md` | BFF 脚本开发 |
| `06-data-api-guidelines.md` | 数据接口规范 |
| `07-sql-creation-workflow.md` | SQL 创建工作流 |
| `07-bff-sync-workflow.md` | BFF 同步与协作 |
| `08-bff-creation-workflow.md` | BFF 创建工作流 |
| `09-conflict-detection.md` | 冲突检测处理 |
| `10-best-practices.md` | 团队协作最佳实践 |

## 本地稿目录约定

* SQL：`src/custom_sql/<name>.sql`
* BFF ENDPOINT：`src/backend-function/endpoint/endpoint_<scriptName>.ts`
* BFF HOOK：`src/backend-function/<tableName>/hook_<hookType>.ts`

以上目录建议纳入 Git。

## 许可

MIT © Lovrabet Team
