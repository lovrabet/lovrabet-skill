# 冲突检测与保存规则

**文档版本**: v3.0
**适用于**: SQL 和 BFF 资源保存

---

## 保存规则（三分支）

`save_or_update_custom_sql` 和 `save_or_update_bff_script` 均通过服务端 toolbox 执行保存，规则如下：

| 场景 | 行为 | AI 提示 |
|------|------|---------|
| 无同名资源 | 直接创建 | 告知创建成功 |
| 有同名资源，上次提交是本人 | 直接更新 | 告知更新成功 |
| 有同名资源，上次提交非本人 | **阻断，返回 `blocked: true`** | 告知用户手动操作 |

---

## blocked 场景处理

当 toolbox 返回 `blocked: true` 时，表示同名资源上次由其他人提交，AI 禁止自行绕过，必须告知用户：

```
该 SQL/BFF 上次由其他人提交。
如需更新，请前往 Lovrabet 平台手动操作，或联系上次提交人。
```

**禁止的行为**：
* 重试保存
* 尝试用其他参数绕过
* 替用户做决定

---

## 响应结构

### 成功（创建/更新）

```json
{ "success": true, "action": "created" | "updated", "data": { "id": 1234, "sqlName": "..." } }
```

### 阻断（非本人资源）

```json
{
  "success": false,
  "action": "blocked",
  "blocked": true,
  "message": "SQL「xxx」上次由其他人提交，请手动操作"
}
```

### 非 SELECT SQL（MCP 层拦截）

```json
{
  "success": false,
  "action": "blocked",
  "message": "SQL \"xxx\" save blocked: Detected INSERT statement ..."
}
```

---

## AI 的职责

**根据返回结果调整回复**：

| 结果 | 语气 | 行动 |
|------|------|------|
| `action: created` | 简洁直接 | 告知创建成功，给出 sqlCode |
| `action: updated` | 友好说明 | 告知更新成功 |
| `blocked: true` | 清楚说明原因 | 展示 message，引导用户手动操作，不重试 |
| 非 SELECT | 中性解释 | 说明无法自动保存，建议保存到本地文件 |

---

## 相关文档

* **SQL 工作流**: `07-sql-creation-workflow.md`
* **BFF 工作流**: `08-bff-creation-workflow.md`
* **最佳实践**: `10-best-practices.md`
