# MCP SQL 创建工作流

> **目标**：掌握创建自定义 SQL 的正确流程，避免因 SQL 错误导致的问题
>
> **前置阅读**：`06-data-api-guidelines.md` - 数据接口访问规范（涵盖字段分析、依赖关系、性能优化）

## 📋 5 步工作流

第1步：查询现有 SQL
第2步：获取表结构
第3步：验证 SQL 内容
第4步：保存 SQL
第5步：测试执行

---

## 本地文件（与以上步骤配合）

编写或修改自定义 SQL 时，先在 **`src/custom_sql/`** 下按业务功能保存 `.sql` 文件（可与 `sqlName` 同名或按功能命名），再将**该文件已保存的内容**用于 `validate_sql_content` 与 `save_or_update_custom_sql`。不要在未写盘的情况下假定「已同步到平台」。目录组织见 `07-sql-creation-workflow.md`。

---

## 步骤详解

### 第 1 步：查询现有 SQL

**目标**：确认 SQL 是否已存在，避免重复创建。

```typescript
const sqlList = await list_sql_queries({
  keyword: "用户统计",
});

const existingSql = sqlList.sqls.find((s) => s.sqlName.includes("用户统计"));
if (existingSql) {
  console.log("SQL 已存在:", existingSql.sqlCode);
  // 可以直接使用，或进入第3步验证
}
```

### 第 2 步：获取表结构与字段核对

**目标**：编写 SQL 前，先获取数据集字段详情，逐项核对字段信息。

**⚠️ 核心原则**：禁止凭空编造字段名。SQL 中使用的每一个字段都必须来自 `get_dataset_detail` 返回的真实字段列表。

**强制要求**：AI 助手在编写任何 SQL 前，必须先执行以下操作：

1. 使用 MCP 工具 `get_dataset_detail` 获取数据集完整字段列表
2. 打印所有可用字段供用户核对
3. 逐项核对 SQL 中使用的每个字段的：名称（区分大小写）、类型、是否必填
4. 确认 SQL 中使用的所有字段都存在且拼写正确

| 核对项 | 说明 | 错误后果 |
|-------|------|---------|
| **字段名称** | 精确匹配，区分大小写 | SQL 执行失败，字段不存在 |
| **字段类型** | VARCHAR/INTEGER/DECIMAL/DATE 等 | 类型不匹配导致查询失败 |
| **是否必填** | required=true 的字段必须有值 | INSERT 时必填字段缺失报错 |
| **主键字段** | primaryKey=true 的字段用于唯一标识 | 无法正确关联或更新数据 |

**字段信息核对清单**（AI 编写 SQL 前必须确认）：

- [ ] 已调用 `get_dataset_detail` 获取真实字段列表
- [ ] 字段名拼写正确（区分大小写）
- [ ] 字段类型与 SQL 操作匹配
- [ ] INSERT 语句包含所有必填字段（required=true）
- [ ] WHERE 条件中的字段确实存在

### 第 3 步：验证 SQL 内容

**目标**：在保存前验证 SQL 语法正确性。

```typescript
const validation = await validate_sql_content({
  sqlContent: `
    SELECT id, customer_name, create_time
    FROM customer
    <where>
        <if test="status != null">
            AND status = #{status, jdbcType=INTEGER}
        </if>
    </where>
  `,
  dbId: detail.basic.database.dbId,
});

if (!validation.validation.valid) {
  console.error("SQL 验证失败:", validation.validation.errors);
  return;
}
```

**验证结果说明**：

| 检测项                 | 说明                       |
| ---------------------- | -------------------------- |
| `valid`                | SQL 语法是否正确           |
| `sqlType.isSelectOnly` | 是否为纯 SELECT 查询       |
| `errors`               | 语法错误列表（含修复建议） |

- **SELECT 查询**：验证通过后可保存到平台
- **INSERT/UPDATE/DELETE/DDL**：不会自动保存，需手动处理（见第4步B）

### 第 4 步：保存 SQL

**目标**：将验证通过的 SQL 保存到平台（现在由服务端 toolbox 统一完成冲突检测）。

#### 4A. 保存 SELECT 查询

```typescript
const result = await save_or_update_custom_sql({
  sqlName: "客户统计查询",
  dbId: detail.basic.database.dbId,
  sqlContent: `...`,
});
console.log("SQL 保存成功:", result.data?.id);
```

#### 4A-1. 处理 blocked（非本人资源场景）

当同名 SQL 上次由其他人提交时，toolbox 会返回 `blocked: true` 并阻断保存，此时应告知用户手动操作，**不能重试**：

```typescript
if (result.blocked) {
  // 直接告知用户，不重试
  console.log(result.message);
  // → "SQL「客户统计查询」上次由其他人提交，请手动操作"
}
```

保存规则说明：

| 场景 | 行为 |
|------|------|
| 无同名 SQL | 直接创建 |
| 同名且上次是本人提交 | 直接更新 |
| 同名但上次非本人提交 | `blocked: true`，阻断，提示用户手动操作 |

#### 4B. 处理非 SELECT SQL

如果 SQL 是 INSERT/UPDATE/DELETE/DDL 类型，MCP 可能阻止或未开放自动保存到平台。此时仍须落在**本地仓库** **`src/custom_sql/`**，按**业务功能**分文件，与可经 MCP 保存的 SELECT 稿同一约定：

```
✅ 正确：按功能组织
src/custom_sql/
  ├── user-registration.sql      # 用户注册相关的所有 SQL
  └── order-settlement.sql        # 订单结算相关的所有 SQL

❌ 错误：按 CRUD 类型组织
src/custom_sql/
  ├── insert.sql                  # 所有 INSERT 混在一起
  └── update.sql                  # 所有 UPDATE 混在一起
```

本地 SQL 文件格式：

```sql
-- ================================================
-- 功能: 用户注册
-- 说明: 用户注册流程涉及的所有数据库操作
-- ================================================

INSERT INTO users (username, email, password_hash, status)
VALUES (#{username}, #{email}, #{passwordHash}, 1);

INSERT INTO user_profiles (user_id, nickname, avatar)
VALUES (LAST_INSERT_ID(), #{nickname}, #{avatar});
```

### 第 5 步：测试执行

**目标**：验证 SQL 能正确执行并返回预期数据。

```typescript
const execResult = await execute_custom_sql({
  sqlCode: "customer_stats",
  params: { status: 1 },
});

if (execResult.execSuccess) {
  console.log("SQL 执行成功，返回", execResult.rowCount, "行");
} else {
  console.error("SQL 执行失败:", execResult.execError);
}
```

#### 测试失败：迭代修正流程

测试失败时，必须回到第 3 步重新修改 SQL，形成闭环迭代：

```
第 3 步：验证 SQL → 第 4 步：保存 SQL → 第 5 步：测试执行
       ↑                        ↓
       └────────────── 失败 ──────────────┘
```

**迭代步骤**：
1. 第 5 步发现错误 → 修改 SQL
2. 第 3 步：重新验证 (`validate_sql_content`)
3. 第 4 步：重新保存 (`save_or_update_custom_sql`，使用已有 `sqlCode`)
4. 第 5 步：重新测试 (`execute_custom_sql`)

```typescript
// 测试失败后
if (!testResult.execSuccess) {
  // 修改 SQL → 重新验证 → 重新保存 → 重新测试
  const fixedSql = "修正后的 SQL";
  await validate_sql_content({ sqlContent: fixedSql, dbId });
  await save_or_update_custom_sql({ sqlCode: "xxx", sqlContent: fixedSql });
  await execute_custom_sql({ sqlCode: "xxx" });
}
```

**注意**：
- **SELECT 查询**：可以直接在 MCP 中迭代修改和测试
- **非 SELECT SQL**：无法通过 MCP 执行，需手动在 Lovrabet 平台测试
- **多个 SQL 场景**：每保存一个 SQL 都要测试执行，不能只测试第一个

---

## ⚠️ 禁止行为

| 禁止行为              | 后果                   | 正确做法       |
| --------------------- | ---------------------- | -------------- |
| 跳过第1步，直接创建   | 可能创建重复的 SQL     | 先查询现有 SQL |
| 跳过第2步，直接写 SQL | 字段名可能错误或编造   | 先获取表结构   |
| 凭空编造字段名        | SQL 执行失败           | 使用 get_dataset_detail 确认 |
| 忽略字段大小写        | 字段不存在错误         | 精确匹配字段名 |
| 跳过第3步，直接保存   | 语法错误导致保存失败   | 先验证 SQL     |
| **只测试第一个 SQL**  | **后续 SQL 可能有问题** | **每个 SQL 都要测试** |
| 保存后不测试          | 运行时才发现错误       | 必须测试执行   |
| 非SELECT SQL 强行保存 | MCP 会阻止，需手动处理 | 保存到本地文件 |

---

## 📚 SQL 语法参考

### 两种写法（不可混用）

| 类型             | 参数语法                  | jdbcType | 适用场景     |
| ---------------- | ------------------------- | -------- | ------------ |
| 简单 SQL         | `#{param}`                | ❌       | 固定条件查询 |
| MyBatis 动态 SQL | `#{param, jdbcType=TYPE}` | ✅       | 可选条件查询 |

### 简单 SQL 示例

```sql
-- ✅ 正确：不使用 jdbcType
SELECT id, name FROM company WHERE status = #{status}

-- ❌ 错误：简单 SQL 不支持 jdbcType
WHERE status = #{status, jdbcType=INTEGER}
```

### MyBatis 动态 SQL 示例（推荐）

```sql
-- ✅ 使用动态标签 + jdbcType
SELECT id, name, status_code FROM company
<where>
    <if test="statusCode != null and statusCode != ''">
        AND status_code = #{statusCode, jdbcType=VARCHAR}
    </if>
    <if test="name != null and name != ''">
        AND name LIKE CONCAT('%', #{name, jdbcType=VARCHAR}, '%')
    </if>
    <if test="startDate != null">
        AND created_at &gt;= #{startDate, jdbcType=DATE}
    </if>
</where>
ORDER BY id DESC
```

### 常用标签

| 标签           | 说明                                 |
| -------------- | ------------------------------------ |
| `<where>`      | 自动处理 WHERE 子句，去除多余 AND/OR |
| `<if test="">` | 条件判断                             |
| `<foreach>`    | 循环处理（如 IN 查询）               |

### jdbcType 对照表

| jdbcType    | 说明   |
| ----------- | ------ |
| `VARCHAR`   | 字符串 |
| `INTEGER`   | 整数   |
| `DECIMAL`   | 小数   |
| `DATE`      | 日期   |
| `TIMESTAMP` | 时间戳 |
| `TINYINT`   | 小整数 |

### XML 转义

```sql
-- SQL 内容中需要转义
AND created_at &gt;= #{startDate}  -- 大于等于
AND created_at &lt;= #{endDate}    -- 小于等于

-- <if test=""> 属性内不需要转义
<if test="startDate != null">
```

---

## ✅ 自检清单

创建自定义 SQL 时，确认：

- [ ] 第1步：查询了现有 SQL，避免重复
- [ ] 第2步：**已调用 `get_dataset_detail` 获取真实字段列表**
- [ ] 第2步：**SQL 中每个字段都来自真实字段列表，未编造任何字段**
- [ ] 第2步：逐项核对字段名称（区分大小写）、类型、是否必填
- [ ] 第2步：确认了必填字段列表（INSERT 时必须包含）
- [ ] 第3步：验证了 SQL 语法
- [ ] 第4步：SELECT 保存到平台，非 SELECT 保存到本地
- [ ] 第5步：测试执行成功

---

## 🔗 相关资源

- MCP 工具：使用 `lovrabet mcp install` 配置
- 数据集详情：通过 MCP `get_dataset_detail` 获取表结构
- Lovrabet 平台：手动导入非 SELECT SQL
