# Lovrabet Skill

面向 [skills](https://www.npmjs.com/package/skills) 开放生态的 **Lovrabet（云兔）** Agent Skill 发布仓库。

[Lovrabet](https://www.lovrabet.com)（中文名云兔）是 企业级 AI-Native 业务系统生成平台，通过首创的数据库逆向分析引擎，自动理解业务模型，实现端到端的业务系统自动开发，将企业沉睡的数据资产，直接转化为可生长、可智能交互的AI驱动能力，此外还能让最懂业务的非技术人员直接参与系统构建，缩短决策链路，释放企业创新潜能。

这个仓库发布的是 Lovrabet 在开放 Agent Skills 生态中的官方 skill，用于让 Cursor、Claude Code 等 AI 助手更准确地理解和执行 Lovrabet 项目中的开发任务。

## 为什么是这个 Skill

在 Lovrabet 项目里，很多任务并不是“写一段代码”这么简单，而是需要同时遵守平台约束、数据结构、MCP 工具流程和多人协作规范。这个 skill 的价值在于把这些关键经验沉淀成可复用的工作流，让 AI 助手在真实项目里少走弯路。

它重点覆盖：

* Lovrabet TypeScript SDK 的正确使用方式
* SQL / Backend Function 的创建、校验、保存和冲突处理
* MCP 工具路由与数据集字段校验
* `src/custom_sql/`、`src/backend-function/` 等本地稿目录约定
* 团队协作中的安全保存与防覆盖规范

## 安装

推荐在项目根目录使用 `skills` 安装：

```bash
npx skills add lovrabet/lovrabet-skill
```

安装后，skill 会被识别为 `skills/lovrabet/` 下的标准技能包结构，便于与其他品牌或团队自定义 skill 区分。

## 仓库结构

* 根目录：`README.md`（本说明）与 `LICENSE`
* Skill 本体：`skills/lovrabet/`
* Skill 入口：`skills/lovrabet/SKILL.md`
* Guides 与配套说明：`skills/lovrabet/guides/`、`skills/lovrabet/README.md`


## 适用场景

如果你在做这些事情，这个 skill 会特别有帮助：

* 在 Lovrabet 项目中查询或修改数据集
* 编写自定义 SQL 并通过 MCP 校验、保存和测试
* 创建或更新 Backend Function
* 在多成员协作场景下避免覆盖他人的 SQL / BFF 脚本
* 让 AI 助手按 Lovrabet 规范生成更可靠的前端或服务端代码

## 进一步了解

* 官网：[www.lovrabet.com](https://www.lovrabet.com)
* 开发文档：[open.lovrabet.com](https://open.lovrabet.com/)
* Skill 详细说明：`skills/lovrabet/README.md`
