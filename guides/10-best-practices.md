# 团队协作最佳实践

**适用场景**: SQL 和 BFF 多人协作开发
**关键原则**: 命名清晰、沟通充分、安全优先

---

## 📝 命名规范

### SQL 查询命名

**推荐格式**: `[模块]-[功能]-[类型]`

```markdown
✅ 好的命名：
- customer-active-list      # 客户-活跃列表
- order-payment-summary     # 订单-支付汇总
- product-inventory-query   # 产品-库存查询
- user-login-history        # 用户-登录历史

❌ 避免的命名：
- query1, query2            # 无意义的序号
- test, temp                # 临时性名称
- abc, xyz                  # 无含义缩写
- 数据查询                  # 中文命名（不推荐）
```

### BFF 脚本命名

**推荐格式**: `[动词]-[模块]-[功能]`

```markdown
✅ 好的命名：
- get-customer-list         # 获取客户列表
- update-order-status       # 更新订单状态
- sync-product-inventory    # 同步产品库存
- validate-user-permission  # 验证用户权限

❌ 避免的命名：
- api1, handler2            # 无意义序号
- do-something              # 模糊的动词
- test-api                  # 测试性名称
```

### 版本命名

```markdown
当需要创建新版本时：

✅ 推荐方式：
- customer-query-v2         # 明确的版本号
- customer-query-optimized  # 说明优化点
- customer-query-with-join  # 说明新增功能

❌ 不推荐：
- customer-query-new        # 模糊的描述
- customer-query-2          # 缺少说明
```

---

## 💬 沟通协议

### 修改他人代码前

**必须沟通的情况**:

1. **生产环境资源**（标记为 #生产使用）
2. **最近修改过**（<7天）
3. **核心业务逻辑**（订单、支付、库存等）
4. **多人共同维护**（有多个作者记录）

**沟通模板**:

```markdown
@同事姓名

我注意到你最近修改了 [资源名称]（最后修改: X天前）

我计划进行以下修改：
- [具体修改内容]

原因：
- [修改原因]

预计影响：
- [可能的影响范围]

请问是否方便？或者有其他建议？

时间: [期望修改时间]
```

**沟通渠道**:
- 即时通讯工具（Slack/钉钉/企业微信）
- 邮件（重要修改）
- 代码注释（在资源描述中更新）

### 修改后的通知

**通知模板**:

```markdown
已更新 [资源名称]

修改内容：
- [具体修改]

影响范围：
- [影响的功能/页面]

测试情况：
- [测试结果]

如有问题请联系我 @你的名字
```

---

## 🔒 安全检查清单

### SQL 发布前检查

```markdown
[ ] 1. 语法验证
    - 调用 validate_sql_content MCP 工具验证通过
    - 字段名拼写正确
    - 表名正确

[ ] 2. 性能检查
    - 避免 SELECT *
    - 添加合适的 WHERE 条件
    - 添加 LIMIT 限制（如需要）
    - 有合适的索引支持

[ ] 3. 参数化查询
    - 使用 #{param} 而非字符串拼接
    - 避免SQL注入风险

[ ] 4. 权限控制
    - 确认查询的数据权限
    - 不暴露敏感字段（密码、密钥等）

[ ] 5. 命名和描述
    - 名称符合规范
    - 描述清晰说明用途
    - 添加必要的注释

[ ] 6. 冲突检查
    - 调用 save_or_update_custom_sql 时检测冲突（toolbox 自动完成）
    - 处理了冲突响应（如有）
```

### BFF 发布前检查

```markdown
[ ] 1. 代码质量
    - 检查 TypeScript 类型正确
    - 无明显的逻辑错误

[ ] 2. 参数验证
    - 验证必填参数
    - 验证参数类型
    - 处理边界情况

[ ] 3. 错误处理
    - 使用 try-catch 捕获异常
    - 返回友好的错误消息
    - 记录错误日志

[ ] 4. 数据安全
    - 敏感数据已脱敏（手机号、邮箱等）
    - 不返回密码、密钥等机密信息
    - 遵循最小权限原则

[ ] 5. 权限检查（如需要）
    - 验证用户身份
    - 验证操作权限
    - 记录审计日志

[ ] 6. 性能优化
    - 避免N+1查询
    - 添加合适的分页
    - 避免死循环

[ ] 7. 命名和描述
    - 名称符合规范
    - 描述清晰说明API功能
    - 提供使用示例

[ ] 8. 冲突检查
    - 调用 save_or_update_bff_script 时检测冲突（toolbox 自动完成）
    - 处理了冲突响应（如有）
```

---

## 📊 版本控制建议

### 资源描述标记

在 SQL/BFF 的描述字段中使用标记：

```markdown
#生产使用 - 生产环境关键资源，修改需谨慎
#测试中 - 测试阶段，可能频繁修改
#待废弃 - 计划废弃，不要在新功能中使用
#多人维护 - 多人共同维护，修改前需沟通
```

**示例**:

```markdown
客户活跃度分析查询 #生产使用 #多人维护

用途：客户管理主页面的活跃客户展示
作者：zhangsan@company.com
依赖：客户列表页面 /dashboard/customer
```

### 变更记录

在资源描述中维护简单的变更记录：

```markdown
# 变更记录
- 2024-03-20: 增加状态筛选 (zhangsan)
- 2024-03-15: 初始版本 (lisi)
```

---

## 🎯 冲突处理策略

### 策略矩阵

| 场景 | 推荐策略 | 原因 |
|------|---------|------|
| 自己的旧代码 | 增量修改 | 了解上下文，风险低 |
| 他人的旧代码（>30天） | 协调后修改 | 可能仍在使用 |
| 他人的新代码（<30天） | 另起新名字 | 避免冲突 |
| 生产环境资源 | 强制协调 | 影响重大 |
| 测试环境资源 | 可适度灵活 | 影响较小 |

### 何时可以强制覆盖

**仅当满足以下所有条件时**:

1. ✅ 确认不会影响他人工作
2. ✅ 已与原作者沟通并获得同意
3. ✅ 资源不是生产环境关键资源
4. ✅ 有充分的测试覆盖
5. ✅ 已做好回滚准备

**强制覆盖后的必做事项**:

1. 立即通知相关人员
2. 更新资源描述（修改原因、修改人）
3. 验证功能正常
4. 保留旧版本备份（如可能）

---

## 🚨 高危操作识别

### 触发强制二次确认

**SQL 场景**:

```markdown
🔴 高危 SQL：
- 涉及 DELETE、TRUNCATE、DROP 操作
- 修改核心业务表（订单、支付、库存）
- 影响大量数据（>1000行）
- 无 LIMIT 限制的 UPDATE
- 生产环境标记的资源
```

**BFF 场景**:

```markdown
🔴 高危 BFF：
- 修改认证/授权逻辑
- 修改支付相关API
- 修改数据删除相关API
- 修改权限检查逻辑
- 涉及大量数据操作
- 生产环境标记的资源
```

### 危险操作示例

```typescript
// 🔴 高危：无限制的 UPDATE
UPDATE customer SET status = 'inactive';

// ✅ 安全：有限制的 UPDATE
UPDATE customer
SET status = 'inactive'
WHERE last_login < #{cutoffDate}
LIMIT 1000;

// 🔴 高危：无验证的参数
export default async function(params: any) {
  return await client.models.customer.delete(params.id);
}

// ✅ 安全：充分验证
export default async function(params: any, context: any) {
  // 1. 参数验证
  if (!params.id || typeof params.id !== 'number') {
    return { success: false, error: '无效的客户ID' };
  }

  // 2. 权限检查
  if (!context.user.permissions.includes('customer:delete')) {
    return { success: false, error: '无删除权限' };
  }

  // 3. 业务检查
  const customer = await client.models.customer.getOne(params.id);
  if (customer.orders > 0) {
    return { success: false, error: '该客户有订单，无法删除' };
  }

  // 4. 审计日志
  console.log(`[DELETE] customer ${params.id} by ${context.user.email}`);

  // 5. 执行删除
  return await client.models.customer.delete(params.id);
}
```

---

## 📚 团队培训建议

### 新成员入职培训

**Day 1 - 基础培训**:
1. 介绍 Lovrabet 平台基础概念
2. 讲解 SQL 和 BFF 的区别
3. 演示创建流程（无冲突场景）
4. 练习：创建一个简单的 SQL 查询

**Day 2 - 冲突处理培训**:
1. 讲解 4 级冲突检测机制
2. 演示冲突处理流程
3. 模拟场景：修改他人代码
4. 练习：处理一个冲突场景

**Day 3 - 安全培训**:
1. 讲解安全检查清单
2. 演示高危操作识别
3. 强调数据脱敏和权限检查
4. 练习：审查一段 BFF 代码

### 定期团队培训

**每月一次的团队培训**:
1. 分享本月最佳实践案例
2. 分析本月出现的问题
3. 更新团队协作规范
4. 答疑和讨论

---

## 🛠️ 工具使用建议

### 日常开发流程

```
# 1. 创建新资源（无冲突）
AI 生成代码 → validate_sql_content 验证 → save_or_update_custom_sql 保存

# 2. 修改现有资源（可能有冲突）
AI 生成代码 → validate_sql_content 验证 → save_or_update_custom_sql
如果返回 `confirmationRequired: true`，说明 HIGH 冲突阻断了保存，根据 `nextAction` 决定是否带 `forceUpdate: true` 重试或另起名字；如果返回 `nextAction: 'suggest_rename'`，说明是 MEDIUM 冲突，建议改名但不强制

# 3. 查看现有资源
MCP：`list_sql_queries`（列出已保存的自定义 SQL）、`execute_custom_sql`（执行已保存 SQL）
平台：在 Lovrabet 控制台「自定义 SQL」列表中查看名称、说明与最近修改（以你环境菜单为准）
```

### 定期维护

**建议每周一次**（无独立 CLI「按作者 / 按标签筛 SQL」能力时，用下面方式即可）：

* 在平台「自定义 SQL」列表中核对：命名是否符合团队约定、是否仍有「临时 / 测试」类条目需要合并或下线。
* 或在 MCP 中调用 `list_sql_queries`，结合你们在 SQL 名称、注释里的约定（例如前缀、说明字段）筛出待清理项。
* 删除或重命名以**平台侧能力**为准；高风险变更前仍建议走 `09-conflict-detection.md` 里的冲突与沟通流程。

---

## 📖 相关文档

- **SQL 工作流**: `07-sql-creation-workflow.md`
- **BFF 工作流**: `08-bff-creation-workflow.md`
- **冲突检测**: `09-conflict-detection.md`

---

**协作的核心是沟通和尊重！** 🤝
