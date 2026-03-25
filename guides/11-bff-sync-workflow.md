---
name: bff-sync-workflow
description: Backend Function 同步工作流，包含拉取、检查、编写、保存平台脚本的完整流程，解决多人开发协作问题。
---

# Backend Function 同步工作流

> **目标**：掌握 Backend Function 脚本的正确同步流程，避免多人开发冲突
>
> **前置阅读**：`05-backend-function.md` - Backend Function 脚本编写规范

---

## 为什么需要同步工作流？

Backend Function 脚本直接保存在 Lovrabet 平台上，**没有像 Git 那样的版本控制**。多人开发时可能遇到：

| 问题 | 场景 | 后果 |
|------|------|------|
| **同名覆盖** | 两人同时开发同名脚本 | 后保存者覆盖前者代码 |
| **盲目新建** | 不知道平台已有相关脚本 | 创建重复功能 |
| **版本混乱** | 不知道脚本是谁改的 | 无法追溯修改历史 |

**解决方案**：强制工作流 + 代码注释规范

---

## MCP 与 BFF 写入能力

若当前环境**看不到** `list_bff_scripts`、`get_bff_script_info`、`save_or_update_bff_script` 等 MCP 工具，说明数据集 MCP 未启用 BFF 写入能力。请在 MCP 配置的 `args` 中增加：

`--enable-bff-save`

该标志为数据集 MCP 的启动参数名；旧名 `--dangerously-bff-save` 仍可用。配置后重启 MCP 客户端或重载 MCP。

示例（节选，包名与路径以你本地为准）：

```json
"args": ["-y", "@lovrabet/dataset-mcp-server@latest", "--enable-bff-save"]
```

通过 `npx skills add lovrabet/lovrabet-skill` 等方式安装后，本 skill 会包含全部指南；未加该参数时仍可阅读本指南，但需按上述方式开启 MCP 后才能用工具保存 BFF。

### Backend Function 获取与保存速查

```
拉取列表 → 检查重名 → 编写脚本 → 保存到平台
```

1. `list_bff_scripts` - 列出平台所有 Backend Function（含 id、functionName、description、scriptContent）
2. `get_bff_script_info(id)` - 按 id 获取单个脚本详情（修改前必看）
3. `save_or_update_bff_script` - 保存或更新：**新建**不传 id，**修改**传 id + description + scriptContent

---

## 4 步同步工作流

```
拉取列表 → 检查重名 → 编写脚本 → 保存到平台
```

### 第 1 步：拉取平台脚本列表

**目标**：了解平台上已有的脚本，避免重复开发。

使用 MCP 工具 `list_bff_scripts` 获取所有 ENDPOINT 类型脚本：

```typescript
// 拉取平台上的所有 Backend Function 脚本
const result = await list_bff_scripts();

if (!result.success) {
  console.error("拉取失败:", result);
  return;
}

console.log("平台脚本列表:");
result.data?.forEach(script => {
  console.log(`  - ${script.description} (ID: ${script.id})`);
});
```

**输出示例**：

```
平台脚本列表:
  - createOrder - 创建订单并扣减库存 (ID: 123)
  - batchDelete - 批量删除订单 (ID: 124)
  - calculatePrice - 计算订单价格 (ID: 125)
```

### 第 2 步：检查同名函数

**目标**：确认是要新建还是修改现有脚本。

```typescript
const SCRIPT_NAME = "createOrder"; // 你想写的脚本名

// 查找是否已存在同名或相似脚本
const result = await list_bff_scripts();
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

### 第 3 步：编写脚本

**目标**：按照规范编写脚本内容。

**前置要求**：
- 已阅读 `05-backend-function` 编写规范
- 已使用 `get_dataset_detail` 获取数据集字段信息
- 已确认字段名、类型、必填项

**脚本模板**（ENDPOINT 类型）：

```javascript
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
  // 数据集编码映射表
  const TABLES = {
    products: "dataset_XXXXXXXXXX", // 数据集: 商品 | 数据表: products
    orders: "dataset_YYYYYYYYYYYY", // 数据集: 订单 | 数据表: orders
  };
  const models = context.client.models;

  // 业务逻辑...
  return { orderId: xxx, payAmount: xxx };
}
```

**版本记录规范**：

| 格式 | 说明 | 示例 |
|------|------|------|
| `YYYY-MM-DD \| 姓名 \| 操作` | 每次修改都追加一行 | `2025-01-30 \| 张三 \| 修复库存扣减bug` |

### 第 4 步：保存到平台

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
updatedContent = updatedContent.replace(
  /\[版本记录\]([\s\S]*?)(?=\n \* @param|\* @param|$)/,
  `[版本记录]\n * 2025-01-30 | 张三 | 修复库存扣减bug\n$1`
);

// 3. 保存更新
await save_or_update_bff_script({
  id: 123,  // 传入 ID，表示更新
  appCode: "app-12345678",
  description: "创建订单并扣减库存",
  scriptContent: updatedContent,
  scriptType: "ENDPOINT",
});

console.log("✅ 更新脚本成功");
```

---

## 完整示例：新建脚本工作流

```typescript
/**
 * 示例：创建新的 Backend Function 脚本
 */
async function createNewBffScript() {
  const SCRIPT_NAME = "calculatePrice";
  const APP_CODE = "app-12345678";

  // 第 1 步：拉取列表
  const listResult = await list_bff_scripts();
  if (!listResult.success) {
    console.error("拉取失败");
    return;
  }

  // 第 2 步：检查重名
  const existing = listResult.data?.find(
    s => s.description?.toLowerCase().includes(SCRIPT_NAME.toLowerCase())
  );

  if (existing) {
    console.log(`⚠️ 已存在相似脚本: ${existing.description}`);
    const detail = await get_bff_script_info({ id: existing.id });
    console.log("现有脚本:", detail.data?.scriptContent);
    console.log("请确认是否要修改现有脚本，或使用其他脚本名");
    return;
  }

  // 第 3 步：编写脚本（实际开发中由 AI 生成）
  const scriptContent = `
/**
 * 计算订单价格（含优惠）
 *
 * [接口路径] POST /api/${APP_CODE}/endpoint/${SCRIPT_NAME}
 * [平台配置] https://app.lovrabet.com/app/${APP_CODE}/data/backend-function
 *
 * [版本记录]
 * 2025-01-30 | tangshuang | 新建：基础价格计算
 *
 * @param {Object} params
 * @param {string} params.productId - 商品ID
 * @param {number} params.quantity - 数量
 * @param {string} params.couponCode - 优惠码（可选）
 * @param {Object} context
 * @returns {Object} { originalPrice: number, finalPrice: number, discount: number }
 */
export default async function ${SCRIPT_NAME}(params, context) {
  const TABLES = {
    products: "dataset_XXXXXXXXXX", // 数据集: 商品 | 数据表: products
    coupons: "dataset_YYYYYYYYYY",   // 数据集: 优惠券 | 数据表: coupons
  };
  const models = context.client.models;

  // 查询商品价格
  const product = await models[TABLES.products].findOne({
    id: params.productId,
  });

  if (!product) {
    throw new Error("商品不存在");
  }

  const originalPrice = product.price * params.quantity;
  let discount = 0;

  // 计算优惠
  if (params.couponCode) {
    const coupon = await models[TABLES.coupons].findOne({
      code: params.couponCode,
      status: "active",
    });

    if (coupon) {
      discount = originalPrice * (coupon.discountPercent / 100);
    }
  }

  return {
    originalPrice,
    finalPrice: originalPrice - discount,
    discount,
  };
}
`.trim();

  // 第 4 步：保存到平台
  const saveResult = await save_or_update_bff_script({
    appCode: APP_CODE,
    description: "计算订单价格（含优惠）",
    scriptContent,
    scriptType: "ENDPOINT",
  });

  if (saveResult.success) {
    console.log("✅ 脚本创建成功！");
  } else {
    console.error("❌ 保存失败:", saveResult);
  }
}
```

---

## 完整示例：修改现有脚本工作流

```typescript
/**
 * 示例：修改现有的 Backend Function 脚本
 */
async function updateExistingBffScript() {
  const SCRIPT_ID = 123;  // 现有脚本的 ID
  const APP_CODE = "app-12345678";

  // 第 1 步：获取现有脚本
  const existing = await get_bff_script_info({ id: SCRIPT_ID });
  if (!existing.success) {
    console.error("获取脚本失败");
    return;
  }

  console.log("现有脚本:", existing.data?.description);

  // 第 2 步：修改内容（追加版本记录）
  let content = existing.data?.scriptContent || "";
  const today = new Date().toISOString().split('T')[0];
  const versionLine = `\n * ${today} | tangshuang | 修复：库存不足时抛出明确错误`;

  // 在 [版本记录] 后追加新行
  if (content.includes('[版本记录]')) {
    content = content.replace(
      /(\[版本记录\][\s\S]*?)((?=\n \* @param|\n \*\/|\* @param))/,
      `$1${versionLine}$2`
    );
  } else {
    // 如果没有版本记录块，在注释中添加
    content = content.replace(
      /(\* [\s\S]*?\*\/)/,
      (match) => {
        const lines = match.split('\n');
        lines.splice(-1, 0, ` * [版本记录]${versionLine}`);
        return lines.join('\n');
      }
    );
  }

  // 修改业务逻辑（示例：在库存检查处抛出明确错误）
  content = content.replace(
    /if \(!product \|\| product\.stock < params\.quantity\) \{[\s\S]*?\}/,
    `if (!product || product.stock < params.quantity) {
  throw new Error(\`库存不足：当前库存 \${product?.stock || 0}，需求数量 \${params.quantity}\`);
}`
  );

  // 第 3 步：保存更新
  const saveResult = await save_or_update_bff_script({
    id: SCRIPT_ID,
    appCode: APP_CODE,
    description: existing.data?.description || "更新后的描述",
    scriptContent: content,
    scriptType: "ENDPOINT",
  });

  if (saveResult.success) {
    console.log("✅ 脚本更新成功！");
  } else {
    console.error("❌ 保存失败:", saveResult);
  }
}
```

---

## 团队协作规范

### 开发前检查清单

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

## 常见问题

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

---

## 相关指南

- **Backend Function 编写规范**：`05-backend-function.md`
- **数据接口访问规范**：`06-data-api-guidelines.md`

---

> 更新时间: 2025-01-30
