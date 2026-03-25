---
# SQL 创建工作流

**适用场景**: 创建新的 SQL 查询或修改现有 SQL
**关键原则**: **先在规定目录写本地 `.sql` 文件并保存**，再经 MCP 验证与保存；头注释规范与冲突规避仍适用

---

## 本地文件优先（强制）

* **目录**：与 `02-mcp-sql-workflow.md` 一致，自定义 SQL 本地稿放在项目 **`src/custom_sql/`** 下，按**业务功能**组织为 `.sql` 文件（可一文件多语句，勿按 CRUD 混在一个文件）。
* **命名**：文件名可与头注释中 `@lovrabet.sqlName` 一致（如 `active-clue-list.sql`），便于与平台资源对应。
* **流程**：新建或修改时，**先创建/编辑上述本地文件并保存**；调用 `validate_sql_content`、`save_or_update_custom_sql` 时，使用**该文件在磁盘上的当前内容**（与用户确认路径一致）。禁止仅在对话里贴出大段 SQL 却不在工作区落盘。
* **版本管理**：**默认纳入 Git**（`git add` / `commit`），与业务代码一并评审；若团队不提交草稿，须在仓库内统一约定并配置 `.gitignore`。

---

## 🎯 目标

* 生成带规范头注释的 SQL 代码
* 验证语法和字段引用
* 检测并避免命名冲突
* 安全保存到平台

---

## 📌 @lovrabet 头注释规范

**所有 AI 生成的 SQL 必须以以下注释开头**：

```sql
-- @lovrabet.sqlName: <业务语义名>
-- @lovrabet.description: <一句话描述>
```

规则：
* `sqlName` 由 AI 根据业务语义生成，格式 `<领域>-<用途>`（全小写，`-` 分隔），例如：`active-clue-list`、`order-settlement-summary`、`customer-detail-by-id`
* `description` 一句话说明查询目的
* 注释必须放在 SQL 文件最前面，SQL 主体在注释之后
* MCP 会自动从注释提取 `sqlName` 和 `description`，**无需用户手动填参数**

**三级兜底策略**（当 AI 或用户没有提供时）：

| 情况 | 名字来源 | `sqlNameSource` | 名字质量 |
|------|---------|----------------|---------|
| AI 写了头注释（主路径） | 注释 `@lovrabet.sqlName` | `comment` | 有业务语义，最佳 |
| 用户手动传了 `sqlName` 参数 | 显式参数 | `param` | 取决于用户 |
| 两者都没有（兜底） | MCP/toolbox 自动生成 | `auto` | 仅结构性（`clue-select-1k7z`），无业务语义 |

> 当 `sqlNameSource: 'auto'` 时，返回 message 里会提示：`sqlName was auto-generated`。AI 应立即提示用户为这条 SQL 补充有意义的名字，或重新发起一次带 `-- @lovrabet.sqlName:` 注释的保存。

---

## 📋 完整工作流

### Step 1: 理解需求

**AI 的职责**:
```markdown
在生成 SQL 之前，AI 应该：

1. **询问关键信息**：
   - 查询什么数据？
   - 需要哪些字段？
   - 有什么筛选条件？
   - 排序和分页需求？

2. **确认数据集**：
   - 涉及哪些表？
   - 是否需要 JOIN？
   - 数据集的 code 是什么？

3. **明确用途**（AI 主动决策，无需问用户）：
   - AI 根据业务语义自行生成 sqlName，写入头注释
   - 数据库 ID（dbId）—— 仅首次创建时需要；更新已有 SQL 时可省略（toolbox 会回填）
```

**示例对话**:
```
用户: "帮我创建一个查询活跃线索的 SQL"

AI 应该询问:
- 需要哪些字段？（id, name, status...）
- 什么算"活跃"？（status = 'active'?）
- 需要排序吗？（按创建时间倒序?）
- 数据集是 clue 吗？

注意：AI 不需要问用户"给 SQL 起什么名字"，AI 根据业务语义自行生成。
```

### Step 2: 生成 SQL 代码

**AI 的职责**:
```markdown
生成 SQL 时必须：

0. **本地落盘**：在 `src/custom_sql/<sqlName>.sql` 创建或打开文件（`sqlName` 与即将写入的头注释一致），后续所有生成与修改都在该文件中进行并保存。

1. **在 SQL 最前面输出标准头注释**（必须，哪怕用户没说要名字）：
   ```sql
   -- @lovrabet.sqlName: active-clue-list
   -- @lovrabet.description: 查询活跃线索列表
   ```
   - `sqlName` 由 AI 根据业务语义生成，格式：`<表/领域>-<用途>`（全小写，`-` 分隔）
   - `description` 一句话描述查询目的
   - MCP 会自动从注释中提取，无需用户手动填 sqlName 参数

2. **使用正确的表名和字段名**：
   - 参考数据集的实际字段
   - 避免字段名拼写错误
   - 注意大小写敏感

3. **考虑性能**：
   - 添加合适的索引字段
   - 避免 SELECT *
   - 添加 LIMIT 限制

4. **参数化查询**：
   - 使用 MyBatis 语法: #{param}
   - 支持动态条件: <if test="">
```

**示例 SQL**:
```sql
-- @lovrabet.sqlName: customer-active-list
-- @lovrabet.description: 查询活跃客户列表，用于客户管理主页面

SELECT
  id,
  name,
  email,
  phone,
  status,
  created_at
FROM customer
WHERE status = 'active'
  AND created_at > #{startDate}
ORDER BY created_at DESC
LIMIT #{pageSize}
```

### Step 3: 提示验证步骤（关键！）

**AI 必须提示用户验证**:
```markdown
AI 生成 SQL 后，应该提供以下信息：

---

**✅ SQL 代码已生成！**

接下来请按以下步骤验证和保存：

**1. 确认本地稿路径**

* 可保存到平台的语句（SELECT / INSERT / UPDATE / WITH 等）：**单一事实源**为 `src/custom_sql/<sqlName>.sql`，请先保存文件，再执行下列 MCP 步骤。
* DELETE 和 DDL（DROP/ALTER/CREATE/TRUNCATE 等）会被 MCP 拦截，**无法**自动保存到平台：同样建议落在 **`src/custom_sql/`**（可用独立文件名区分，如 `maintenance-truncate-xxx.sql`），仅作仓库内记录与评审，勿只放在聊天记录里。

**2. 验证语法和字段**

使用 MCP 工具 `validate_sql_content` 验证：
- 传入 SQL 内容（应与 **`src/custom_sql/` 下已保存文件**一致）
- 可选传入 `validateSchemas` 参数做字段引用校验

如果验证失败，我会帮你修复。

**3. 保存到平台（自动检测提交人）**

使用 MCP 工具 `save_or_update_custom_sql`：
* `sqlContent`: 与本地稿一致的内容（头注释里已有 `@lovrabet.sqlName`，MCP 自动提取，无需另传 `sqlName`）
* `dbId`: 数据库 ID（**新建时必填**，从 `get_dataset_detail` -> `basic.database.dbId` 获取；更新已有 SQL 时可省略）
* `description`: 可选

toolbox 会自动执行以下检测：
* DELETE / DDL → 直接拦截，返回 blocked（**永不保存**）
* 同名 SQL 且上次提交人不是你 → 返回 `blocked: true`，**告知用户手动操作，不重试**
* 同名 SQL 且上次是你 → 直接更新
* 无同名 SQL → 直接创建

---
```

### Step 4: 处理验证结果

**如果验证失败**:
```markdown
用户: "验证失败了，说字段不存在"

AI 应该:
1. 检查字段名是否正确
2. 参考数据集的实际字段
3. 修正 SQL 代码
4. 再次调用 validate_sql_content 验证

示例回复:
"""
抱歉！字段名写错了。让我修正：

**错误原因**: 字段 `phone_number` 不存在，应该是 `phone`

**修正后的 SQL**:
[修正后的代码]

请再次使用 validate_sql_content 验证修正后的 SQL。
"""
```

### Step 5: 处理 blocked（提交人冲突）

当 `save_or_update_custom_sql` 返回 `blocked: true` 时，表示同名 SQL 上次由其他人提交。

> ⛔ **AI 不能重试，不能用任何参数绕过**。必须直接告知用户：

```
该 SQL「xxx」上次由其他人提交。
如需更新，请前往 Lovrabet 平台手动操作，或联系上次提交人。
```

用户的选择：
* 自行去平台操作
* 联系上次提交人协调
* 改一个新名字（AI 可协助重新生成带新 `sqlName` 的 SQL 并保存）

### Step 6: 确认保存成功

**AI 应该确认**:
```markdown
用户: "保存成功了"

AI 应该回复:
"""
✅ **保存成功！**

**SQL 信息**：
- 名称: customer-active-list
- 数据库 ID: 100
- 状态: 已创建/已更新

**后续使用**：

1. **在代码中调用**：
```typescript
// 使用 Frontend SDK
const result = await client.sql.execute({
  sqlCode: 'customer-active-list',
  params: {
    startDate: '2024-01-01',
    pageSize: 100
  }
});

if (result.execSuccess && result.execResult) {
  console.log('查询结果:', result.execResult);
}
```

2. **在 BFF 中调用**：
```typescript
// Backend Function 中直接返回数组
export default async function(params: any, context: any) {
  const rows = await context.client.sql.execute({
    sqlCode: 'customer-active-list',
    params: params
  });
  // rows 是 T[] 数组，不是包装对象

  return {
    success: true,
    data: rows
  };
}
```

3. **测试 SQL**：
使用 `execute_custom_sql` MCP 工具：
```
sqlCode: "customer-active-list"
params: { startDate: "2024-01-01" }
```

**记录**：
- SQL Code: `customer-active-list`
- 创建时间: 2024-03-20

建议将此信息记录到项目文档中。
"""
```

---

## 🚨 关键检查点

### 检查点 1: 生成前

```
✅ 确认数据集信息
✅ 确认字段名称
✅ 确认查询条件
✅ AI 已准备好根据业务语义命名（无需问用户）
```

### 检查点 2: 验证前

```
✅ SQL 头部有 -- @lovrabet.sqlName: <业务语义名>
✅ SQL 头部有 -- @lovrabet.description: <用途描述>
✅ SQL 语法正确
✅ 字段名拼写正确
✅ 表名正确
✅ 参数化查询使用正确
```

### 检查点 3: 保存前

```
✅ 验证已通过
✅ SQL 类型不是 DELETE / DDL（否则需改为本地文件）
✅ 了解同名资源情况（toolbox 会自动检测，无需提前查）
```

### 检查点 4: 保存后

```
✅ 保存成功确认
✅ 记录 SQL Code
✅ 提供使用示例
✅ 建议测试方法
```

---

## 📖 示例场景

### 场景 1: 全新创建（无冲突）

```
用户: "创建一个查询活跃客户的 SQL"

AI:
1. 询问字段需求
2. 生成 SQL 代码
3. 提示验证步骤
4. 提示保存步骤

用户:
1. 执行验证脚本
2. 执行保存脚本
3. 确认成功

结果: ✅ 无冲突，直接保存成功
```

### 场景 2: 修改自己的 SQL

```
用户: "给客户查询增加状态筛选"

AI:
1. 获取原 SQL（通过 sqlCode 或名称定位）
2. 修改 SQL 内容
3. 调用 save_or_update_custom_sql

结果: ✅ toolbox 判断上次提交是本人，直接更新成功
```

### 场景 3: 同名 SQL 上次是他人提交

```
用户: "修改客户查询 SQL"

AI:
1. 调用 save_or_update_custom_sql
2. toolbox 返回 blocked: true

AI 回复用户:
  「customer-query」上次由其他人提交，请手动前往平台操作，
  或考虑改一个新名字，我可以帮你生成。

结果: ✅ 未覆盖他人资源，用户可选择手动或改名
```

---

## 💡 AI 的核心职责

### 必须做的 ✅

1. **生成前确认** - 询问关键数据信息（字段、条件、表名）
2. **自主命名** - 根据业务语义生成 `-- @lovrabet.sqlName`，不问用户
3. **提供验证步骤** - 调用 `validate_sql_content` 验证 SQL
4. **提供保存步骤** - 调用 `save_or_update_custom_sql` 保存
5. **处理 blocked** - 返回 `blocked: true` 时，告知用户手动操作或改名，不重试
6. **提供使用示例** - 告诉用户如何使用保存的 SQL
7. **处理 auto 名字警告** - 若返回 `data.sqlNameSource === 'auto'`，提示用户补充语义名

### 禁止做的 ❌

1. **不要询问用户 SQL 名字** - AI 根据语义自行决定
2. **不要生成没有头注释的 SQL** - 每条 SQL 必须有 `@lovrabet.sqlName`
3. **不要忽略验证步骤** - 必须调用 `validate_sql_content` 验证
4. **不要在 blocked 后重试** - 无论加什么参数都不行，只能告知用户
5. **不要省略使用示例** - 保存后必须提供示例

---

## 🎯 成功标准

### 用户体验

```
✅ 用户知道如何验证代码
✅ 用户知道如何保存代码
✅ 用户理解冲突信息
✅ 用户能做出安全的决策
✅ 用户知道如何使用保存的 SQL
```

### 安全保障

```
✅ 代码经过验证
✅ 冲突经过检测
✅ 操作有记录
✅ 风险有提示
✅ 用户有选择
```

### 协作效率

```
✅ 不会意外覆盖他人代码
✅ 团队成员可以协调
✅ 资源有清晰的所有权
✅ 修改有审计记录
```

---

## 📚 相关文档

- **冲突检测指南**: `09-conflict-detection.md`
- **最佳实践**: `10-best-practices.md`

---

**记住**: AI 是助手，用户是决策者！
