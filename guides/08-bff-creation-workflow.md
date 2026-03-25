---
requireFlags:
  - enable-bff-save
---
# Backend Function 创建与同步工作流

> **目标**：掌握 Backend Function 的完整开发流程，从 AI 辅助生成到安全保存，避免多人开发冲突
>
> **前置阅读**：`05-backend-function.md` - Backend Function 脚本编写规范

---

## 🎯 与 SQL 工作流的区别

| 维度 | SQL | BFF |
|------|-----|-----|
| 代码类型 | SQL 查询 | TypeScript/JavaScript |
| 执行环境 | 数据库 | 服务器 Node.js |
| 返回格式 | execResult 数组 | 直接数组或对象 |
| 参数化 | MyBatis 语法 | TypeScript 参数 |
| 调试方式 | SQL 测试 | 日志输出 |
| **影响范围** | 数据查询 | **服务稳定性** |

**⚠️ BFF 特有风险**：BFF 直接在服务器执行，错误代码可能影响服务稳定性，保存前务必仔细测试。

---

## 为什么需要同步工作流？

Backend Function 脚本直接保存在 Lovrabet 平台上，**没有像 Git 那样的版本控制**。多人开发时可能遇到：

| 问题 | 场景 | 后果 |
|------|------|------|
| **同名覆盖** | 两人同时开发同名脚本 | 后保存者覆盖前者代码 |
| **盲目新建** | 不知道平台已有相关脚本 | 创建重复功能 |
| **版本混乱** | 不知道脚本是谁改的 | 无法追溯修改历史 |

**解决方案**：5 步标准工作流 + 版本记录规范 + 冲突检测

---

## 📋 6 步完整工作流

```
理解需求 → 验证数据集（强制） → 检查重名 → 生成代码 → 验证测试 → 保存到平台
```

## 本地文件优先（与 `05-backend-function.md` 一致）

* **根目录**：项目 **`src/backend-function/`**（见 `05-backend-function.md`「文件存放目录」）。
* **ENDPOINT**：独立端点脚本放在 **`src/backend-function/endpoint/`**，文件名格式 `endpoint_<scriptName>.ts`（或 `.js`），与「文件命名与函数规范」表一致。
* **HOOK**（Before/After）：放在 **`src/backend-function/<表名>/`**，按表组织。
* **流程**：新建或修改时**先编辑上述本地文件并保存**，再经 MCP 校验与 `save_or_update_bff_script` 同步；`scriptContent` 须与磁盘文件一致。禁止仅在对话中输出整段脚本而不在工作区落盘。
* **版本管理**：**默认纳入 Git**（`git add` / `commit`）；若不提交，须团队统一约定并配置 `.gitignore`。

## MCP 与 BFF 写入能力

若当前环境**看不到** `list_bff_scripts`、`get_bff_script_info`、`save_or_update_bff_script` 等 MCP 工具，说明数据集 MCP 未启用 BFF 写入能力。请在 MCP 配置的 `args` 中增加：

`--enable-bff-save`

旧名 `--dangerously-bff-save` 仍可用。配置后重启 MCP 客户端或重载 MCP。

示例（节选，包名与路径以你本地为准）：

```json
"args": ["-y", "@lovrabet/dataset-mcp-server@latest", "--enable-bff-save"]
```

通过 `npx skills add lovrabet/lovrabet-skill` 安装后，本 skill 会包含全部指南；即使尚未开启该参数，仍可阅读本指南，但无法用 MCP 工具保存 BFF 到平台。

### Step 1: 理解需求（AI 辅助）

**AI 的职责**:
```markdown
1. **询问业务逻辑**：
   - API 的功能是什么？
   - 输入参数有哪些？
   - 返回什么数据？
   - 需要哪些数据集？

2. **确认技术细节**：
   - 是否需要数据验证？
   - 是否需要权限检查？
   - 是否需要数据脱敏？
   - 错误如何处理？

3. **明确用途**：
   - BFF 的名称
   - 使用场景
   - 调用方是谁？
```

### Step 2: 获取并验证数据集与字段（强制校验！）

**目标**：确保代码中使用的所有数据集和字段在平台上真实存在，拒绝 AI 幻觉编造。

**AI 必须在生成代码前做真实的校验**：
1. **查找相关数据集**：
   使用 `list_datasets`（或 `search_datasets`）搜索相关的业务对象，拿到确切的 `datasetCode`。
2. **验证字段与关联关系**：
   使用 `get_dataset_detail` 获取每个涉及数据集的详细结构，并仔细核对：
   - **验证字段是否存在**：不要假设有 `user_id` 或 `status`，必须在返回的 `properties` 里找到确切的 `code`。
   - **验证字段类型**：确认是 `NUMBER`、`TEXT` 还是 `DATE`，不要把字符串存进数字类型。
   - **验证枚举含义**：如果涉及状态判断（如 `status === 'active'`），必须查看字段的 `options` 确认有效的枚举值，不能凭空猜测。
   - **验证关联关系**：如果需要查询关联数据，检查是否有关联字段。

> ⛔ **严禁猜测表结构**：如果尚未调用 `get_dataset_detail`，AI 绝不能开始写带有具体字段的 BFF 业务逻辑！

### Step 3: 检查重名（避免冲突）

**目标**：确认是要新建还是修改现有脚本。

使用 MCP 工具 `list_bff_scripts` 获取所有 ENDPOINT 类型脚本：

```typescript
const SCRIPT_NAME = "createOrder"; // 你想写的脚本名

// 拉取平台上的所有 Backend Function 脚本
const result = await list_bff_scripts();
if (!result.success) {
  console.error("拉取失败:", result);
  return;
}

// 查找是否已存在同名或相似脚本
const existing = result.data?.find(
  s => s.description?.includes(SCRIPT_NAME) ||
       s.scriptContent?.includes(`function ${SCRIPT_NAME}`)
);

if (existing) {
  console.log(`⚠️ 发现已存在脚本: ${existing.description} (ID: ${existing.id})`);
  console.log("请确认：");
  console.log("  1. 是要修改现有脚本？→ 使用 get_bff_script_info + save_or_update_bff_script");
  console.log("  2. 还是要新建不同功能的脚本？→ 使用不同的脚本名");

  // 查看现有脚本内容
  const detail = await get_bff_script_info({ id: existing.id });
  console.log("现有脚本内容:", detail.data?.scriptContent);

  return; // 停止，等待用户确认
}
```

**决策树**：

```
发现同名脚本？
├── 是 → 要修改功能？
│   ├── 是 → 进入"修改现有脚本"流程
│   └── 否 → 换个脚本名
└── 否 → 进入"新建脚本"流程
```

### Step 4: 生成 BFF 代码（AI 辅助）

**本地落盘（ENDPOINT）**：在 `src/backend-function/endpoint/endpoint_<scriptName>.ts`（或 `.js`）中创建或打开文件，写入或修改完整脚本（含文件头注释与 `export default`），保存后再进入验证与保存步骤。HOOK 类型则放在 `src/backend-function/<表名>/`，见 `05-backend-function.md`。

**标准模板**（ENDPOINT 类型）：

```typescript
/**
 * {脚本功能描述}
 *
 * [接口路径] POST /api/{appCode}/endpoint/{scriptName}
 * [平台配置] https://app.lovrabet.com/app/{appCode}/data/backend-function
 *
 * [版本记录]
 * 2025-01-30 | tangshuang | 新建：创建订单并扣减库存
 *
 * [HTTP 请求体参数]
 * { "productId": "商品ID", "quantity": "数量" }
 *
 * [返回数据结构]
 * { "orderId": "订单ID", "payAmount": "支付金额" }
 *
 * @param {Object} params - 请求参数
 * @param {Object} context - 执行上下文（平台自动注入）
 * @returns {Object} 返回结果
 */
export default async function {scriptName}(params, context) {
  const { client } = context;

  try {
    // 1. 参数验证
    if (!params.productId || !params.quantity) {
      return {
        success: false,
        error: '缺少必填参数: productId 或 quantity',
      };
    }

    // 2. 数据集编码映射表
    const TABLES = {
      products: "dataset_XXXXXXXXXX", // 数据集: 商品 | 数据表: products
      orders: "dataset_YYYYYYYYYYYY", // 数据集: 订单 | 数据表: orders
    };
    const models = client.models;

    // 3. 业务逻辑
    const product = await models[TABLES.products].findOne({
      id: params.productId,
    });

    if (!product) {
      throw new Error("商品不存在");
    }

    if (product.stock < params.quantity) {
      throw new Error(`库存不足：当前库存 ${product.stock}，需求数量 ${params.quantity}`);
    }

    // 4. 创建订单
    const order = await models[TABLES.orders].create({
      productId: params.productId,
      quantity: params.quantity,
      payAmount: product.price * params.quantity,
      status: 'pending',
    });

    // 5. 扣减库存
    await models[TABLES.products].update({
      id: params.productId,
      stock: product.stock - params.quantity,
    });

    // 6. 返回结果
    return {
      success: true,
      data: {
        orderId: order.id,
        payAmount: order.payAmount,
      },
    };

  } catch (error) {
    // 7. 错误处理
    console.error('[BFF Error]', error);
    return {
      success: false,
      error: error.message || '操作失败',
    };
  }
}
```

**关键要点**:
```markdown
✅ 必须使用 try-catch 错误处理
✅ 必须验证输入参数
✅ 必须返回标准格式: { success, data/error }
✅ 考虑数据脱敏（手机号、邮箱等）
✅ 考虑权限检查（如需要）
✅ 添加有意义的日志
```

**版本记录规范**：

| 格式 | 说明 | 示例 |
|------|------|------|
| `YYYY-MM-DD | 姓名 | 操作` | 每次修改都追加一行 | `2025-01-30 | 张三 | 修复库存扣减bug` |

### Step 5: 验证测试

**AI 必须提示**:

```markdown
**✅ BFF 代码已生成！**

接下来请按以下步骤验证和保存：

**1. 保存到本地文件**

BFF 脚本应保存到项目的 `src/backend-function/endpoint/` 目录：

```bash
# ✅ 正确：保存到项目目录
mkdir -p src/backend-function/endpoint
cat > src/backend-function/endpoint/endpoint_create-order.ts << 'EOF'
[TypeScript 代码]
EOF

# ❌ 错误：使用临时目录 /tmp/
# /tmp/ 目录的文件会被系统清理，不适合存放业务代码
```

**目录结构示例**：
```
src/backend-function/
  ├── endpoint/                    # ENDPOINT 脚本目录
  │   ├── endpoint_create-order.ts
  │   └── endpoint_sync-orders.ts
  └── {tablename}/                 # HOOK 脚本目录（按表名组织）
      ├── beforeInsert.js
      └── afterUpdate.js
```

**2. 验证代码**

使用 MCP 工具 `validate_sql_content` 的 BFF 模式，或直接查看语法和逻辑：
- 检查是否使用了正确的 SDK 方法
- 确认参数类型和返回值格式

**3. 保存到平台（自动检测冲突）**

使用 MCP 工具 `save_or_update_bff_script`，toolbox 会自动检测冲突：
- `scriptName`: 脚本名称（如 "create-order"）
- `scriptContent`: 完整脚本内容
- `scriptType`: "ENDPOINT" 或 "HOOK"

如果返回 `confirmationRequired: true`，说明是 HIGH 冲突，当前保存被阻断，需要查看 `existing` 信息后决定是否带 `forceUpdate: true` 重新调用。MEDIUM 冲突不阻断，但会返回 `nextAction: 'suggest_rename'` 提示考虑改名。

**⚠️ 如果存在冲突**：
- LOW（自己的代码）：可以直接更新
- MEDIUM/HIGH（他人代码）：建议另起新名字或与作者协商

---
```

### Step 6: 保存到平台

**场景 A：新建脚本**

```typescript
await save_or_update_bff_script({
  appCode: "app-12345678",
  description: "创建订单并扣减库存",
  scriptContent: scriptContent,  // 完整脚本内容
  scriptType: "ENDPOINT",
  // id 不传，表示新建
});

console.log("✅ 新建脚本成功");
```

**场景 B：修改现有脚本**

```typescript
// 1. 先获取现有脚本内容
const existing = await get_bff_script_info({ id: 123 });

// 2. 修改内容（保留版本记录）
let updatedContent = existing.data?.scriptContent || "";
const today = new Date().toISOString().split('T')[0];
const versionLine = `\n * ${today} | tangshuang | 修复：库存不足时抛出明确错误`;

// 在 [版本记录] 后追加新行
if (updatedContent.includes('[版本记录]')) {
  updatedContent = updatedContent.replace(
    /(\[版本记录\][\s\S]*?)((?=\n \* @param|\n \*\/|\* @param))/,
    `$1${versionLine}$2`
  );
} else {
  // 如果没有版本记录块，在注释中添加
  updatedContent = updatedContent.replace(
    /(\* [\s\S]*?\*\/)/,
    (match) => {
      const lines = match.split('\n');
      lines.splice(-1, 0, ` * [版本记录]${versionLine}`);
      return lines.join('\n');
    }
  );
}

// 修改业务逻辑（示例：在库存检查处抛出明确错误）
updatedContent = updatedContent.replace(
  /if \(!product \|\| product\.stock < params\.quantity\) \{[\s\S]*?\}/,
  `if (!product || product.stock < params.quantity) {
  throw new Error(\`库存不足：当前库存 \${product?.stock || 0}，需求数量 \${params.quantity}\`);
}`
);

// 3. 保存更新
await save_or_update_bff_script({
  id: 123,  // 传入 ID，表示更新
  appCode: "app-12345678",
  description: existing.data?.description || "更新后的描述",
  scriptContent: updatedContent,
  scriptType: "ENDPOINT",
});

console.log("✅ 更新脚本成功");
```

---

## 🔄 平台同步与协作速查

### Backend Function 获取与保存速查

```
拉取列表 → 检查重名 → 本地修改 → 保存到平台
```

1. `list_bff_scripts`：列出平台已有脚本，开始开发前必查。
2. `get_bff_script_info(id)`：修改现有脚本前先拉最新内容，避免基于旧版本覆盖。
3. `save_or_update_bff_script`：新建时不传 `id`，更新时传 `id + description + scriptContent`。

### 推荐同步节奏

* **开始开发前**：先用 `list_bff_scripts` 看是否已有同名或相近脚本。
* **确认要修改现有脚本**：先 `get_bff_script_info` 拉最新代码，再把改动合并回本地文件。
* **完成开发后**：确认本地文件与 `scriptContent` 一致，再调用 `save_or_update_bff_script`。
* **保存遇到冲突**：若工具返回 `blocked`、`confirmationRequired` 或建议改名，不要盲目重试；按 `09-conflict-detection.md` 处理。

### 更新现有脚本的最小流程

```typescript
const existing = await get_bff_script_info({ id: 123 });
const currentContent = existing.data?.scriptContent || "";

// 先把平台最新内容合并回本地文件，再保存
await save_or_update_bff_script({
  id: 123,
  appCode: "app-12345678",
  description: existing.data?.description || "更新后的描述",
  scriptContent: currentContent,
  scriptType: "ENDPOINT",
});
```

---

## 🔐 BFF 特有注意事项

### 数据脱敏示例

```typescript
// ✅ 好的做法：返回前脱敏
const maskPhone = (phone: string) => {
  if (!phone) return phone;
  return phone.replace(/(\d{3})\d{4}(\d{4})/, '$1****$2');
};

const maskEmail = (email: string) => {
  if (!email) return email;
  const [name, domain] = email.split('@');
  return `${name.substring(0, 2)}***@${domain}`;
};

return {
  success: true,
  data: result.data.map(item => ({
    ...item,
    phone: maskPhone(item.phone),
    email: maskEmail(item.email),
  })),
};
```

### 权限检查示例

```typescript
// ✅ 添加权限验证
export default async function(params: any, context: any) {
  const { client, user } = context;

  // 检查用户权限
  if (!user || !user.permissions.includes('customer:read')) {
    return {
      success: false,
      error: '无权限访问客户数据',
    };
  }

  // 继续业务逻辑
  // ...
}
```

### SQL 执行差异（重要！）

```typescript
// ⚠️ 注意：BFF 中 SQL 返回的是数组，不是包装对象！

// ❌ 错误（前端 SDK 写法）
const data = await context.client.sql.execute({
  sqlCode: 'customer-query'
});
if (data.execSuccess) {  // BFF 中没有这个字段！
  const result = data.execResult;
}

// ✅ 正确（BFF 写法）
const rows = await context.client.sql.execute({
  sqlCode: 'customer-query'
});
// rows 已经是 T[] 数组，直接使用
return {
  success: true,
  data: rows
};
```

---

## 📊 使用示例

**AI 应该提供两种使用方式**:

```markdown
**✅ BFF 保存成功！**

**使用方式 1: 前端直接调用**
```typescript
// 在前端代码中
import { LovrabetClient } from '@lovrabet/sdk';

const client = new LovrabetClient({ appCode: 'your-app' });

const response = await client.bff.execute({
  name: 'create-order',
  params: {
    productId: '123',
    quantity: 2
  }
});

if (response.success) {
  console.log('订单创建成功:', response.data);
} else {
  console.error('错误:', response.error);
}
```

**使用方式 2: 作为 API 端点**
```bash
# 通过 HTTP 调用
curl -X POST https://api.lovrabet.com/bff/create-order \
  -H "Content-Type: application/json" \
  -d '{"productId":"123","quantity":2}'
```

---
```

---

## 🚨 BFF 特有风险与缓解

### 风险等级

| 风险 | 说明 | 缓解措施 |
|------|------|----------|
| 逻辑错误 | 影响所有调用方 | 充分测试 |
| 性能问题 | 慢查询影响服务 | 添加超时限制 |
| 数据泄露 | 返回敏感信息 | 数据脱敏 |
| 权限绕过 | 未授权访问 | 权限检查 |
| 无限循环 | 死循环占满 CPU | 添加超时和限制 |

### 安全检查清单

```markdown
保存前必须检查：

[ ] 1. 参数验证完整吗？
[ ] 2. 错误处理到位吗？
[ ] 3. 敏感数据脱敏了吗？
[ ] 4. 权限检查有了吗（如需要）？
[ ] 5. SQL 返回值处理正确吗？
[ ] 6. 有性能问题吗？
[ ] 7. 日志输出合理吗？
[ ] 8. 不会造成死循环吗？
```

---

## 👥 团队协作规范

### 开发前检查清单

- [ ] 必须先使用 `list_datasets` / `get_dataset_detail` 获取确切的数据集与字段（禁止凭空捏造）
- [ ] 使用 `list_bff_scripts` 拉取最新列表
- [ ] 检查是否有同名或功能相似的脚本
- [ ] 如有重名，与团队成员确认是修改还是新建

### 代码注释规范

| 必需项 | 说明 |
|--------|------|
| **版本记录块** | 每次修改必须追加一行 |
| **修改日期** | YYYY-MM-DD 格式 |
| **修改人** | 姓名/拼音 |
| **修改说明** | 简洁描述修改内容 |

### 同步时机建议

| 场景 | 操作 |
|------|------|
| 开始开发前 | 拉取最新列表 |
| 发现同名脚本 | 停止，与团队确认 |
| 完成开发后 | 立即保存到平台 |
| 发现平台脚本被改 | 拉取最新内容，合并后再保存 |

---

## ❓ 常见问题

### Q: 多人同时修改同一脚本怎么办？

**A**: 没有锁机制，后保存者会覆盖前者。建议：
1. 修改前先拉取最新内容
2. 修改后立即保存，缩短时间窗口
3. 重要修改提前在团队中沟通

### Q: 如何回滚到之前的版本？

**A**: 平台没有版本历史，建议：
1. 本地保留脚本备份
2. 重要脚本提交到 Git 仓库
3. 使用版本记录块追踪修改历史

### Q: 脚本保存后如何生效？

**A**:
1. 保存后立即生效，无需重启
2. 建议在测试环境先验证
3. 使用 curl 或 Postman 测试端点

### Q: 如何调试 BFF 脚本？

**A**:
1. 使用 `console.log` 输出调试信息（会在平台日志中显示）
2. 在本地测试核心逻辑
3. 使用 try-catch 捕获并返回详细错误信息

### Q: 为什么看不到 `list_bff_scripts` 或 `save_or_update_bff_script`？

**A**:
1. 检查数据集 MCP 启动参数是否加了 `--enable-bff-save`
2. 旧环境也可能仍使用 `--dangerously-bff-save`
3. 修改 MCP 配置后，重启 MCP 客户端或重载 MCP

---

## 🔗 相关指南

- **Backend Function 编写规范**：`05-backend-function.md`
- **数据接口访问规范**：`06-data-api-guidelines.md`
- **SQL 创建工作流**：`07-sql-creation-workflow.md`
- **团队协作最佳实践**：`10-best-practices.md`

---

**BFF 更强大，也更需要谨慎！** 🚀
