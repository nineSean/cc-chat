# E2E 测试策略设计 — Account Opening (Angular → React 重构)

> 日期: 2026-05-03
> 工具栈: Playwright + opencode + gpt-5.4
> 目标: 利用 AI + 老系统 + 老代码，2 周内交付 15 个核心 E2E case，并建立可持续维护的测试基础设施

---

## 0. 背景与动机

### 项目情境
- **业务**: Account Opening（开户）流程，金融行业典型多步带分支带草稿场景
- **重构**: Angular → React，由另一个 team 起头开发，代码质量"一堆问题"，团队中途接手
- **老项目**: 有线上地址，**有沙箱环境**
- **新项目**: 本地 + SIT 可运行，但很多业务流程跑不通
- **痛点**: 业务规则因团队转手而模糊；新项目实现散漫；缺少质量护栏

### 测试体系的双重使命
1. **倒逼新项目跑通**: 测试驱动开发，case 不通即新项目要修
2. **业务规格固化**: 团队转手后，老系统行为是唯一的 oracle，测试是固化业务知识的载体

### AI 工作流定位
利用 AI 加速以下四件事：
1. 逆向提炼业务规格（从老 Angular 代码 + 老系统沙箱行为）
2. 清洗/改写录制脚本（codegen 原料 → PO 风格成品）
3. 从老系统抽取断言（真实接口响应作 oracle）
4. 失败诊断辅助（test-runner 给 hypothesis，**不自愈**）

### 明确不做（已锁定排除）
- ❌ GitHub Actions / Headless CI 自动跑测试
- ❌ Agent SDK 闭环自愈 + 自动 PR
- ❌ 老系统持续测试（老系统只是录制工厂，一次性）

### 业务结构概览

项目主导航有 **3 个 menu**：

```
┌─────────────────────────────────────────────────────────────┐
│  1. Account Opening   2. Maker Dashboard   3. Checker Dash  │
└─────────────────────────────────────────────────────────────┘
```

#### Menu 1: Account Opening（开户主流程）

```
1.1 选择类型: [Non-Face-to-Face] | [Face-to-Face]
            ↓
1.2 账户类型: [Sole] | [Joint]
            ↓
1.3 入口分流:
    ├─ Search 现有 customer
    │    ├─ 1.3.1.1 输入 customer number → search
    │    └─ 1.3.1.2 列表显示该 customer 所有 applications
    │              点击未完成的 → resume 到之前步骤/状态
    └─ 新建 application
         ├─ 1.3.2.1 直接创建：所有数据重新填
         └─ 1.3.2.2 search 后基于 customer 创建：用户信息 prefill
            ↓
1.4 多步表单流程（4-7 步，步数随类型而变）
    每步含：
      - 多个 subTabs（页内 tab 切换）
      - 底部按钮: [Exit] [Save] [Back] [Next]
            ↓
1.5 提交 → Summary 页（生成 account）
```

**关键复杂度**：
- 流程长度可变（4-7 步）
- 中途可 Save 草稿，Resume 时回到之前的步骤 + 状态
- Prefill 分支：基于已有 customer 创建时多个字段预填
- subTab 切换：单步内有多 tab，状态独立

#### Menu 2: Maker Dashboard

- 显示 maker 视角的所有 application 记录
- 未完成的 application 可在此 resume（替代路径，不必从 menu 1 走 search）

#### Menu 3: Checker Dashboard

- Checker 视角查看 application
- 查看详情、审核状态等

### 业务结构对测试设计的影响

| 业务特性 | 测试设计取舍 |
|---|---|
| 流程长度可变（4-7 步） | E2E case 选 1 个最长（7 步）+ 1 个最短（4 步），覆盖步数变化 |
| Save / Resume 草稿 | 单步聚焦测试用 `seedToStep` 时优先复用 Save → Resume 而不是手工点完前 N 步 |
| Prefill 分支 | 单独一个 case 验证"基于 customer 创建后字段确实 prefill" |
| subTabs | PO 类要建模 `tabIndex` 和 subTab 切换；selector 命名带 tab 标识 |
| 4 种底部按钮 | PO 提供统一方法 `clickExit()/clickSave()/clickBack()/clickNext()`，测试不直接点 |
| 3 个 menu 各有视角 | maker dashboard 列表页是 **Pilot 1 最佳候选**（只读、无多步、最易跑通工具链） |

---

## 1. 决策约束（已锁定）

| 维度 | 选择 |
|------|------|
| 老系统定位 | 录制工厂（一次性） |
| 新老 UI 一致性 | 中等（50-80%） |
| 覆盖范围 | 15 个核心 case（金字塔 3+8+4）—— **数量定，具体清单待附录 D 探索后确定** |
| AI 介入点 | 提炼规格 / 改写脚本 / 抽断言 / 失败诊断 |
| Selector 策略 | testid 优先，role/label 兜底 |
| 老系统访问 | 有沙箱 |
| 流程结构 | 4-7 步多步骤、有 subTabs、Save/Resume 草稿、Prefill 分支（详见 §0 业务概览） |
| 测试运行环境 | opencode + gpt-5.4 |
| 测试目标环境 | **本地 dev 为主**（迭代快），SIT 为辅（部署后验证）；新项目三个 menu 已全部上 SIT |
| 时间预算 | 2 周（10 工作日） |

---

## 2. 整体架构（方案 C: Hybrid Cross-Check）

### 2.1 流水线总览

```
┌──────────────────────────────────────────────────────────────┐
│                老 Angular 系统（沙箱 + 仓库代码）                │
└──────────┬─────────────────────────────────────┬─────────────┘
           │                                     │
   Track-1: QA codegen 录制              Track-2: AI 扫老代码
   （recorded/<flow>/...）                 （spec-extractor）
           │                                     │
           └──────────────────┬──────────────────┘
                              ▼
                  ┌────────────────────────┐
                  │ spec-merger (gpt-5.4)   │
                  │  → specs/<flow>.spec.md │
                  └───────────┬────────────┘
                              ▼
                       [ 人工 review ]
                              ▼
                  ┌────────────────────────┐
                  │ test-generator (gpt-5.4)│
                  │  生成 PO + 双版本脚本    │
                  └───────────┬────────────┘
            ┌─────────────────┴─────────────────┐
            ▼                                   ▼
   verify-on-old/<flow>.ts               new/<flow>.ts
   （老沙箱手动跑一次，                    （新 React SIT 上跑）
     落 fixtures/<flow>.json）             ↑
                                           │
                                  人显式触发 / Hook 自动触发
                                           ▼
                              ┌────────────────────────┐
                              │ test-runner (gpt-5.4)   │
                              │  context 隔离           │
                              │  跑 Playwright          │
                              │  过滤日志，回结构化 JSON  │
                              └───────────┬────────────┘
                                          ▼
                              ┌────────────────────────┐
                              │  人工分诊                │
                              │  - 业务代码 bug ✅       │
                              │  - 脚本/选择器变了 🔧    │
                              │  - 测试数据/老 spec 错 ⚠️│
                              └────────────────────────┘
```

### 2.2 Hook 闭环（核心生产力机制）

```
opencode 主 agent 改业务代码 or 改脚本
        ↓
   主 agent 准备结束本轮（"我做完了"）
        ↓
   ┌─── opencode completion hook ───┐
   │   1. 读 .opencode/focus.json    │
   │   2. invoke test-runner subagent│
   │   3. 跑 focus 指定的 case        │
   │      （无 focus → 跑全量 smoke） │
   │   4. 收 JSON 报告                │
   └────────────────┬────────────────┘
                    ▼
              全部 pass？
              ├─ 否 → JSON 报告（含 hypothesis）回喂主 agent
              │       主 agent 继续修代码 → 回到顶部
              └─ 是 → 真正结束
              
              max_iterations: 5（防死循环）
```

### 2.3 组件清单

| # | 组件 | 类型 | 责任 |
|---|------|------|------|
| 1 | `playwright codegen` | 外部工具 | 录原始脚本 |
| 2 | `record-with-network.ts` | 本地脚本 | 包 codegen，捕获网络 + DOM 快照 |
| 3 | `spec-extractor` | opencode subagent | 扫老 Angular 代码，提炼隐性业务规则 |
| 4 | `spec-merger` | opencode subagent | 合并录制 + 代码扫描 → spec.md |
| 5 | `test-generator` | opencode subagent | 按 spec 生成 PO + 双版本测试 |
| 6 | `pages/` | 代码 | 业务命名的 Page Object |
| 7 | `fixtures/` | 数据 | 老沙箱真实接口响应快照 |
| 8 | `test-runner` | opencode subagent | 验证者：context 隔离跑测试，输出结构化 JSON |
| 9 | completion hook | 配置 | 结束节点拦截 → 调 #8 → 失败回喂 → 循环 |

### 2.4 仓库目录结构

```
e2e/
├── playwright.config.ts
├── pages/                      # PO 类（业务命名）
│   └── AccountOpeningFlow.ts
├── data/
│   ├── test-data-factory.ts    # 测试数据生成
│   └── accounts.ts             # 测试账号（按 case 分配）
├── tests/
│   ├── verify-on-old/          # 一次性，跑通后归档
│   └── new/                    # 主战场（CI 长期跑）
├── recorded/                   # codegen 原料，不进 CI
│   └── <flow>/
│       ├── raw.ts
│       ├── network.json
│       ├── dom-snapshot.html
│       └── meta.json
├── specs/                      # AI 合并后的业务规格
│   └── <flow>.spec.md
├── fixtures/                   # 老沙箱接口响应 oracle
│   └── <flow>.json
├── scripts/                    # 入口脚本
│   ├── record-with-network.mjs
│   ├── run-case.mjs
│   └── run-by-spec.mjs
└── .opencode/
    ├── agent/                  # 4 个 subagent 配置
    │   ├── spec-extractor.md
    │   ├── spec-merger.md
    │   ├── test-generator.md
    │   └── test-runner.md
    ├── focus.json              # 主 agent 写入"当前关注 case"
    └── hook-config.json        # completion hook 配置
```

### 2.5 测试入口（三层）

#### 命令行
```bash
npm run e2e:smoke                       # 全量
npm run e2e:case order-submit           # 单 case
npm run e2e:case order-submit login     # 多 case
npm run e2e:by-spec '下单流程'           # 按业务名
npm run e2e:debug order-submit          # 带 UI、慢动作
```

#### opencode subagent 主动调用
主 agent 处理某个流程时显式：
> "调用 test-runner 跑 order-submit"

#### Hook 自动触发
主 agent 结束节点 → hook 读 `focus.json` → 调 test-runner。

```jsonc
// .opencode/focus.json（主 agent 在开始任务时写入）
{
  "cases": ["order-submit"],
  "started_at": "2026-05-03T10:00:00Z",
  "owner": "ninesean"
}
```

#### Hook 触发的测试范围（默认策略）
- 有 focus → 只跑 focus 指定的 case（30s-2min）
- 无 focus → 跑全量 smoke

### 2.6 模型与 agent 配置

- 全部 subagent 统一使用 **gpt-5.4**
- "降噪" 不靠模型分层，靠两件事：
  - **opencode subagent context 隔离**：subagent 内部跑测试时几千行日志只在 subagent 自己的 context，结束按契约只回结构化 JSON
  - **强契约 prompt**：test-runner 输出格式锁死

---

## 3. 单流程端到端数据流（以"下单"为例）

跟踪一个流程从零到"在新项目跑通且 hook 闭环"的全过程。
（Account opening 的具体 pilot 见第 5 章。）

### Step 1 · QA 在老沙箱录制

```bash
npm run record -- order-submit --target=$OLD_SANDBOX_URL
```

`record-with-network.ts` 包一层 codegen，做三件事：
- 启动 Playwright codegen UI
- 用 `page.route()` 拦截所有 fetch/XHR → `network.json`
- 关闭时 dump 关键时刻 DOM 到 `dom-snapshot.html`

QA 操作：登录 → /orders/new → 填表 → 提交 → 看到成功

产出：
```
recorded/order-submit/
├── raw.ts              # codegen 原始脚本
├── network.json        # 真实接口响应（oracle 雏形）
├── dom-snapshot.html
└── meta.json
```

### Step 2 · spec-extractor 扫老代码

主 agent invoke：
> "spec-extractor 扫老 Angular 仓库，找 OrderSubmit 相关 route/component/service/guard"

产出 `recorded/order-submit/extracted-rules.md`：
```markdown
## 路由
- /orders/new → OrderNewComponent
- guards: AuthGuard, CompletedProfileGuard

## 表单规则（OrderFormService）
- amount: 必填，0 < amount < 1_000_000
- amount > 10_000 → 触发审批流（AdminApprovalDialog）
- US: zipCode 5 位；CN: 6 位

## 提交
- POST /api/orders → { id, status: PENDING | APPROVED }
- APPROVED → /orders/{id}/success
- PENDING  → /orders/{id}/approval-pending

## 错误
- INSUFFICIENT_BALANCE → modal
- OUT_OF_STOCK → toast
```

### Step 3 · spec-merger 合并双源

输入：录制（happy path）+ 扫码（含分支）
输出 `specs/order-submit.spec.md`：

```markdown
# Spec: order-submit

## 主流程（录制 ✓ 验证）
1. 登录 testuser
2. /orders/new
3. amount=500, address=..., country=US, zip=12345
4. 提交 → expect POST /api/orders 200
5. expect URL /orders/<id>/success
6. expect text "订单提交成功"

## 分支（代码 ⚠ 未录制）
- amount > 10_000 → 审批弹窗 → /approval-pending
- INSUFFICIENT_BALANCE → modal "余额不足"
- OUT_OF_STOCK → toast "已售罄"

## Oracle（来自 recorded/network.json）
- POST /api/orders: { id: ord_123, status: APPROVED }

## 字段映射（老 → 新）
| 业务字段 | 老 selector | 新 selector |
|---|---|---|
| amount | input[name=amount] | testid=order-amount-input |
| submit | button.btn-primary | testid=order-submit-btn |
```

> **人工 review 是必经关卡**——补缺失分支、纠错映射。

### Step 4 · test-generator 生成代码

#### `pages/OrderSubmitPage.ts`
```ts
export class OrderSubmitPage {
  constructor(public page: Page, private mode: 'old' | 'new') {}

  amount = () => this.mode === 'new'
    ? this.page.getByTestId('order-amount-input')
    : this.page.locator('input[name=amount]');

  submit = () => this.mode === 'new'
    ? this.page.getByTestId('order-submit-btn')
    : this.page.getByRole('button', { name: '提交' });

  async fill(d: { amount: number; address: string; country?: string }) {
    await this.amount().fill(String(d.amount));
    // ... 其余字段
  }

  async expectSuccess() {
    await expect(this.page).toHaveURL(/\/orders\/[^/]+\/success/);
    await expect(this.page.getByText('订单提交成功')).toBeVisible();
  }
}
```

#### `tests/new/order-submit.spec.ts`
```ts
test('happy path: 普通金额下单', async ({ page }) => {
  const po = new OrderSubmitPage(page, 'new');
  await login(page, 'testuser', 'testpw');
  await page.goto('/orders/new');
  await po.fill({ amount: 500, address: '123 Main St', country: 'US' });

  const [resp] = await Promise.all([
    page.waitForResponse('**/api/orders'),
    po.submit().click(),
  ]);
  expect(resp.status()).toBe(200);
  expect(await resp.json()).toMatchObject({ status: 'APPROVED' });
  await po.expectSuccess();
});
```

#### `tests/verify-on-old/order-submit.spec.ts`
- 同逻辑，`mode='old'`，base URL 指向沙箱

### Step 5 · 老沙箱 verify（强制护栏）

```bash
PW_BASE=$OLD_SANDBOX_URL npx playwright test tests/verify-on-old/order-submit.spec.ts
```

- ✅ pass → spec 正确，`fixtures/order-submit.json` 落地，verify 文件归档
- ❌ fail → spec 错了，回 Step 3 修

> **关键约束**：未经 verify 的 spec 不允许进 `tests/new/`。否则后面失败时分不清是"新项目错"还是"业务理解错"。

### Step 6 · 新项目跑 + Hook 闭环

主 agent 开始处理 order-submit：
```bash
echo '{ "cases": ["order-submit"] }' > .opencode/focus.json
```

主 agent 改新项目代码 → 准备结束 → completion hook 触发：

```
hook
 → invoke test-runner subagent
 → 跑: npm run e2e:case order-submit
 → 收结构化 JSON：
   {
     "summary": { "passed": 0, "failed": 1 },
     "failures": [{
       "test": "happy path",
       "step": 'getByTestId("order-submit-btn").click',
       "kind": "selector_missing",
       "error_message": "Locator not found: getByTestId('order-submit-btn')",
       "evidence": {
         // ↓ AI 主用（全是文本）
         "dom_excerpt": "<form>...<button class='primary'>提交</button>...</form>",
         "console_logs": ["[error] React: hydration mismatch"],
         "network_summary": [{ "url": "/api/orders", "method": "POST", "status": 0 }],
         // ↓ 仅人工，AI 忽略
         "screenshot_path": "test-results/order-submit/failed.png",
         "trace_path":      "test-results/order-submit/trace.zip"
       },
       "hypothesis": "DOM 里有 button class='primary' 但缺 data-testid=order-submit-btn"
     }]
   }
 → 报告回喂主 agent
 → 主 agent 加 testid → 再次结束 → hook 再触发
 → ... 直到全绿 → hook 放行 → 真正完成
```

---

## 4. 错误处理 / 重试 / 数据隔离

### 4.1 失败分类与处置（test-runner 输出 `kind`）

| `kind` | 含义 | 主 agent 该做什么 |
|---|---|---|
| `selector_missing` | 元素找不到 | 新项目缺 testid → 加 |
| `selector_changed` | 元素变了 | 改 PO 的 selector 实现，**测试代码不动** |
| `assertion` | 断言失败（值/URL/文本不符） | 业务逻辑没接通 → 修代码 |
| `network` | 接口 4xx/5xx/超时 | 后端联调 / API mock |
| `timeout` | 等待超时 | 加 wait / 检查异步竞态 |
| `oracle_drift` | 真实响应与 fixture 不符 | 重 verify-on-old，更新 oracle |
| `infra` | 浏览器崩溃 / 端口占用 | 不修脚本，重试一次 |
| `flaky` | 同测试时通时不通 | 打 `@flaky` 标签，进隔离队列 |

### 4.2 重试策略

| 场景 | retries | 备注 |
|---|---|---|
| Hook 闭环（本地循环） | **0** | 失败立刻报，AI 修 |
| 命令行 / CI | **1** | 只对 `network / infra / timeout` 重试 |
| `@flaky` 隔离队列 | **3** | 单独跑，不阻塞主流 |

> **断言失败永不自动重试**——它代表"业务真错了"，重试只是浪费时间。

```ts
// playwright.config.ts
export default defineConfig({
  retries: process.env.HOOK_LOOP ? 0 : (process.env.CI ? 1 : 0),
  projects: [
    { name: 'main',  testIgnore: /.*@flaky.*/ },
    { name: 'flaky', testMatch: /.*@flaky.*/, retries: 3 },
  ],
});
```

### 4.3 数据隔离（Account Opening 特化）

**核心洞察**: 每个 case = 一个新 application。**测试天然幂等，不需要任何清理机制**。问题简化为"生成保证不冲突的 application 测试数据"。

> **本项目无需处理**: KYC / OCR / 活体识别 / 反欺诈 / AML —— 这些不在测试范围内（业务流程不涉及，或对 e2e 测试不需要 mock）。
> **本项目无需做**: 测试数据清理 —— 沙箱 DB 数据堆积可接受，运维不介入。

#### Test Data Factory

```ts
// e2e/data/test-data-factory.ts
import { randomUUID } from 'crypto';

export interface TestApplicationData {
  runId: string;
  caseId: string;
  // application 业务字段：根据老代码探索后补充（D1 探索）
  // 例如：applicationName, contactEmail, address 等
  uniqueTag: string;   // 用于在 UAT 上反查"这条是 e2e-xxx 跑出来的"
}

export function createTestData(caseId: string): TestApplicationData {
  const runId = process.env.RUN_ID ?? randomUUID().slice(0, 8);
  const seq   = Date.now().toString().slice(-7);
  return {
    runId,
    caseId,
    uniqueTag: `e2e-${caseId}-${runId}-${seq}`,
  };
}

// 已知测试 customer（用于 search 现有 customer 的场景）
// 由沙箱团队预置；从环境变量读
export const KNOWN_TEST_CUSTOMERS = {
  default:  process.env.TEST_CUSTOMER_DEFAULT  ?? '[TBD]',  // 用于 1.3.1 search 场景
  withApps: process.env.TEST_CUSTOMER_WITH_APPS ?? '[TBD]', // 已有多条 applications，用于列表测试
};
```

特性：
- **唯一**（同 runId 内不冲突）：`Date.now()` + 自增 seq
- **可追溯**（出问题能反查）：`uniqueTag` 嵌 `caseId-runId-seq`
- **可识别**（运维/业务方一眼认出）：`e2e-` 前缀
- **可重放**（debug 时复现）：显式传 `RUN_ID=xxx` 重跑产出可预测数据

#### beforeEach（极简）

```ts
test.beforeEach(async ({ page }, info) => {
  // 登录测试账号（PO 提供 login helper）
  await login(page, process.env.SANDBOX_USER!, process.env.SANDBOX_PWD!);
});
```

无 sandbox bypass、无 KYC 配置、无 reset 调用。

#### 测试 customer 怎么用

| 场景 | 数据来源 |
|---|---|
| 1.3.2.1 直接创建 application | 用 `createTestData(caseId)` 生成的字段填表 |
| 1.3.2.2 基于 customer 创建（验证 prefill） | 用 `KNOWN_TEST_CUSTOMERS.default` 做 search，然后基于该 customer 新建 |
| 1.3.1 customer search → applications 列表 | 用 `KNOWN_TEST_CUSTOMERS.withApps`，沙箱预置至少 1 条已存在 application |
| Resume 草稿（E2E-3） | 测试中先 Save 一次草稿留下 applicationId，下一段用同 id Resume |

#### 数据堆积

每跑一次会留下若干 application 在沙箱 DB。本项目**接受堆积**，无清理机制。

如果未来沙箱性能因数据量出问题，再补（不进 2 周 sprint 范围）。

### 4.4 取证配置

> **关键设计约束**: 公司 LLM 仅支持文本输入。**screenshot/video/trace.zip 只供人工查看**，AI 必须仅基于文本证据（DOM excerpt / console / network / error message）做诊断。test-runner 必须在 JSON 输出里产出文本字段，不能假设 AI 能解读图像或二进制。

#### 基础配置（playwright.config.ts）

```ts
use: {
  trace:      'retain-on-failure',  // 仅人工
  screenshot: 'only-on-failure',    // 仅人工
  video:      'retain-on-failure',  // 仅人工
},
reporter: [
  ['json', { outputFile: 'test-results/results.json' }],   // test-runner 解析
  ['html', { outputFolder: 'test-results/html' }],
  ['list'],
],
```

#### 文本证据自动 dump（fixture / setup）

```ts
// e2e/fixtures/text-evidence.ts
import { test as base, Page } from '@playwright/test';
import * as fs from 'fs/promises';

type TextEvidence = {
  consoleLogs: string[];
  networkLog: Array<{ url: string; method: string; status: number; timestamp: number }>;
};

export const test = base.extend<{ textEvidence: TextEvidence }>({
  textEvidence: async ({ page }, use, info) => {
    const evidence: TextEvidence = { consoleLogs: [], networkLog: [] };

    page.on('console', msg => evidence.consoleLogs.push(`[${msg.type()}] ${msg.text()}`));
    page.on('response', resp => evidence.networkLog.push({
      url: resp.url(), method: resp.request().method(),
      status: resp.status(), timestamp: Date.now(),
    }));

    await use(evidence);

    // 失败时 dump 三份文本
    if (info.status !== 'passed') {
      await fs.mkdir(info.outputDir, { recursive: true });
      await fs.writeFile(`${info.outputDir}/dom.html`, await page.content());
      await fs.writeFile(`${info.outputDir}/console.log`, evidence.consoleLogs.join('\n'));
      await fs.writeFile(`${info.outputDir}/network.json`, JSON.stringify(evidence.networkLog, null, 2));
    }
  },
});
```

每个 case 文件改用这个增强版 `test`：

```ts
import { test, expect } from '@/fixtures/text-evidence';

test('happy path', async ({ page }) => {
  // 自动捕获 console/network，失败时自动 dump dom.html
});
```

#### test-runner subagent 内部的 DOM excerpt 截取

test-runner 拿到 `dom.html`（可能几 MB），不能整段塞 JSON：

```
1. 读 results.json 找失败 step 的目标 selector（如 testid='order-submit-btn'）
2. 在 dom.html 里全文搜索该 selector
   - 命中 → 截取该元素 + 父级 3 层 + 兄弟节点 ~100 行
   - 未命中 → 在 DOM 里搜近似元素（同标签名/同 role）；都没有就截 body 前 5KB + 后 5KB
3. 截取结果作为 evidence.dom_excerpt 字段写入 JSON
```

输出大小目标: `dom_excerpt` 控制在 **2-5KB**（约 50-150 行 HTML），既给 AI 足够上下文又不爆 context。

### 4.5 老沙箱 / Oracle 风险

| 问题 | 影响 | 处置 |
|---|---|---|
| 老沙箱挂了 | 不能验新 spec | `tests/new/` 不依赖（fixtures 已落地）；只阻塞写新 spec |
| Oracle 漂移 | fixture 过期 → `oracle_drift` | 月度 `npm run e2e:verify-all-on-old`，diff fixtures，业务方 review |

### 4.6 AI agent 自身失误兜底

| 失误 | 处置 |
|---|---|
| 生成 TS 编译失败 | tsc 直接退，主 agent 看错误信息再生成 |
| 假断言（pass 但跟 spec 不符） | spec 中 expect 标号 + 测试代码 `// from spec #N` 注释，人工 review 时映射检查 |
| Hook 死循环 | `max_iterations: 5`，超过停下让人介入 |
| **AI 试图"读图"做判断**（gpt-5.4 不支持多模态） | test-runner prompt 显式禁用图像引用；hypothesis 必须基于 dom_excerpt / console / network 文本证据；评审时若发现 hypothesis 含"截图显示..."类措辞，视为 prompt 漏写要修补 |

```jsonc
// .opencode/hook-config.json（具体字段名以 opencode 实际版本为准）
{
  "completion_hook": {
    "max_iterations": 5,
    "on_max_reached": "stop_with_message",
    "message": "已尝试 5 次仍有失败，请人工介入。最近报告见 test-results/last.json"
  }
}
```

### 4.7 测试金字塔（Account Opening 多步分支草稿场景）

```
            ┌──────────────────┐
            │  E2E 全链路 (3)   │  每个 3-5 分钟
            │  抓步骤间状态丢失   │
            └──────────────────┘
        ┌──────────────────────────┐
        │  单步聚焦 (8)              │  每个 30s-1min
        │  seedToStep 快跳后断言    │
        └──────────────────────────┘
    ┌──────────────────────────────────┐
    │  分支跳转 (4)                      │  每个 <30s
    │  只验"选 A → 跳到 X 页"            │
    └──────────────────────────────────┘
```

#### E2E 全链路（3 个，最有代表性的主路径）
- 个人/本地/年轻/进取（Pilot 候选）
- 企业/本地（企业流程独立）
- 个人/外籍（外籍流程分支）

#### 单步聚焦（8 个，主战场）
关键能力 `seedToStep(n)`：UI 层走完前 N 步，用于在 hook 循环里"修哪一步只跑哪一步"。

```ts
async seedToStep(targetStep: number, customer: TestCustomer) {
  await this.startNew();
  if (targetStep <= 1) return;
  await this.fillBasic(customer);
  if (targetStep <= 2) return;
  await this.uploadIdDocs(customer);
  if (targetStep <= 3) return;
  await this.fillContact(customer);
  // ...
}
```

#### 分支跳转（4 个，最快）
只断言"分支选择 → 跳到正确页"，用 Pairwise 表压缩组合：

| # | 类型 | 国籍 | 年龄 | 风险 | 备注 |
|---|------|------|------|------|------|
| 1 | 个人 | 本地 | 年轻 | 进取 | E2E-1 主路径 |
| 2 | 个人 | 外籍 | 中年 | 平衡 | 单步覆盖 |
| 3 | 企业 | 本地 | - | - | E2E-2 |
| 4 | 个人 | 本地 | 未成年 | - | 监护人流程分支 |
| 5-8 | 其他单步 + 分支断言 | | | | |

---

## 5. 阶段化交付（2 周冲刺）

### 5.1 时间线

```
W1                          W2
D1-D2       D3-D7            D8-D10
基建打底     横向拉            收尾稳定
2 天        5 天             3 天
工具链      12-15 case       Flaky + 文档
+ Pilot     上线
```

### 5.2 Day-by-Day

> **前置条件（D1 启动前必须 ready）**: 附录 D 的所有配置项收集完毕。如果 D1 当天才能凑齐配置，整体 sprint 起步推迟 0.5-1 天，相应砍一两个尾部 case 或缩短 D10 buffer。

| 天 | 工作 | 产出 |
|---|------|------|
| **D1 上午** | (并行) **A: 读老代码 + 浏览 UAT，列 15 个 case 清单 → 写入附录 D**；B: 项目脚手架 / record-with-network / test-data-factory / 4 个 subagent prompt v0 | case 清单确定 + 工具链可用 |
| **D1 下午** | Pilot 1（最简非开户流程，验证工具链）整条流水线走一次 | 工具链验证完毕 |
| **D2** | Pilot 2（个人/本地/年轻/进取 开户主路径）走通；completion hook 配置 + 真实闭环跑一次 | 1 个开户 case 在 hook 循环跑通 |
| **D3-D4** | 2 个 E2E（企业本地、个人外籍）+ 2 个单步 | 4 个 case |
| **D5-D6** | 4 个单步（subTabs、Save/Resume、Prefill、Back/Exit 等） | 4 个 case |
| **D7** | 4 个分支跳转 + 2 个剩余单步 | 6 个 case |
| **D8** | 补缺 + Flaky 标记 + 全部 case 在 verify-on-old 通过 | 15 个 case 全产 |
| **D9** | 团队上手文档（2 页 SOP）+ 失败分诊单页 | 文档交付 |
| **D10** | 整体复查 + buffer + 第一次跑全量 smoke 验稳定性 | 缓冲日 |

合计 **3 E2E + 8 单步 + 4 分支 = 15 个 case**

### 5.3 Pilot 策略（分两阶段）

#### Pilot 1: 最简非开户流程（D1 下午）
- 目标: 验证整条工具链能跑通
- **强烈推荐: Maker Dashboard 列表页 / Checker Dashboard 列表页** —— 只读、无多步、无草稿，最易跑通
- 备选: Account Opening 入口的 1.1 选择页（Non-F2F / F2F 两选项的简单选择）
- 选择标准: 步骤 ≤3 步、无外部依赖、能在 5 分钟内手工走通
- 走完: 录制 → spec-extractor → spec-merger → test-generator → verify-on-old → 在新项目 SIT 跑通

#### Pilot 2: 开户主路径（D2 整天）
- 目标: 验证 hook 闭环 + 多步流程在工具链里能跑
- 选"个人/本地/年轻/进取"
- 必须验证: hook 触发 → 主 agent 收报告 → 修代码 → 再 hook → 通过

#### D1 结束就有交付物，避免 D2 末才发现工具链不工作

### 5.4 取舍清单

#### 必须保留
| 项 | 不能砍的原因 |
|---|---|
| Hook 闭环 | AI 生产力的核心 |
| verify-on-old | 失败时能区分"新项目错"vs"业务理解错"，调试时间不爆炸 |
| 双源对照（录制 + spec-extractor） | spec 质量保证 |
| 测试金字塔 | hook 循环跑得动的前提 |
| Test Data Factory | 不可绕过 |

#### 砍 / 推迟（接受功能性损失）
| 项 | 推迟方案 | 风险 |
|---|---|---|
| 13 个 case (28→15) | 上线后按月迭代加 | 覆盖率打 53% |
| Pairwise 表自动化工具 | 手工列 yaml | 加 case 慢 30 分钟 |
| Oracle drift 月度任务 | 第二个月再做 | 老沙箱悄悄改了不会主动发现 |
| 失败分诊看板 | 直接看 test-runner JSON | 多花眼力 |
| Prompt 调优回灌机制 | 两周内手工迭代 | 经验不沉淀 |
| 全 28 case 在新项目 SIT 全绿 | 验收门改为"15 个全绿连续 1 天" | 稳定性观察期短 |

### 5.5 关键风险与止损点

| 风险 | 概率 | 影响 | 止损 |
|---|---|---|------|
| **D2 Hook 闭环跑不通** | 中 | 高 | **D2 18:00 检查点**：不达预期立刻降级为 npm script + shell wrapper（功能等价）；不再尝试 opencode 原生 hook，避免阻塞 D3+ |
| spec-merger 输出质量差 | 中 | 中 | Phase 1 前 3 个 case 持续调 prompt；超过 3 个还不行 → 退化为人工写 spec，AI 只做 generator |
| AI 生成的脚本反复假断言 | 中 | 中 | spec 中 expect 标号 + 测试代码注释（4.6 节方案） |
| Pilot 2 选错（太复杂） | 低 | 中 | D2 中午还没跑通 → 立刻换更简单的 case |

#### Hook 降级路径（D2 18:00 检查点未通过时启用）

降级版核心区别：**hook 是"主 agent 想结束时被动触发"，shell wrapper 只能"主 agent 显式调用时跑"**。两者功能等价但触发时机不同。

```bash
# scripts/run-case-with-fallback.sh
# 替代 opencode hook 的 shell wrapper：
# 主 agent 写完代码后显式调用此脚本，失败时返回非零给主 agent，
# 主 agent 据此决定是否修复 + 再次调用。
#
# 注意: max_iterations=5 在 shell 模式下无法跨进程跟踪；
# 主 agent 自己负责"5 次循环还失败时停下找人"。

#!/bin/bash
set -e
npm run e2e:case "$@"
# Exit code 即 Playwright 退出码：
#   0  → 全 pass，主 agent 可以真正完成
#   非 0 → 失败，主 agent 解析 test-results/results.json 决定下一步
```

主 agent 工作流改为：写完代码后**显式调用 `bash scripts/run-case-with-fallback.sh <flow>`**（而非依赖 hook 自动触发），失败时主 agent 自行决定是否修复 + 重试。主 agent 在 prompt 里需要写明"重试不超过 5 次，超过停下汇报人"。

---

## 6. Test-runner 输出契约

为保证主 agent 收到的失败信息可被一致解析，且**只用文本证据**（公司 LLM 无多模态），test-runner subagent 必须输出如下 JSON：

```json
{
  "summary": {
    "passed": 8,
    "failed": 2,
    "skipped": 0,
    "duration_ms": 124300
  },
  "failures": [
    {
      "test": "order-submit > happy path",
      "file": "tests/new/order-submit.spec.ts",
      "step": "page.getByTestId(\"order-submit-btn\").click",
      "kind": "selector_missing | selector_changed | assertion | network | timeout | oracle_drift | infra | flaky",
      "error_message": "locator.click: Timeout 30000ms exceeded.",
      "stack": "at OrderSubmitPage.submit (e2e/pages/OrderSubmitPage.ts:42:18)",
      "evidence": {
        "dom_excerpt": "<form id='order-form'>\n  <input name='amount' value='500'/>\n  <button class='primary'>提交</button>\n</form>",
        "console_logs": [
          "[error] React: hydration mismatch at OrderForm",
          "[warn]  deprecated API used"
        ],
        "network_summary": [
          { "url": "/api/orders", "method": "POST", "status": 0, "note": "request never sent" },
          { "url": "/api/me",     "method": "GET",  "status": 200 }
        ],
        "old_fixture_diff": "fixtures/order-submit.json vs actual: status field missing in actual",
        "screenshot_path": "test-results/order-submit/failed.png",
        "video_path":      "test-results/order-submit/video.webm",
        "trace_path":      "test-results/order-submit/trace.zip"
      },
      "hypothesis": "DOM 里有 button 但没有 testid='order-submit-btn'，文本是'提交'用了 class='primary'。建议在新项目 OrderForm 给 button 补 data-testid='order-submit-btn'。"
    }
  ],
  "iteration": 2,
  "max_iteration": 5
}
```

### 字段语义

#### AI 必须看（文本证据）
| 字段 | 内容 | 大小约束 |
|---|---|---|
| `error_message` | Playwright 抛出的错误消息原文 | 单行，<500 字符 |
| `stack` | 错误 stack trace（裁剪到测试代码层） | 5-15 行 |
| `evidence.dom_excerpt` | 失败时刻 DOM 的关键片段（4.4 节截取规则） | 2-5KB |
| `evidence.console_logs` | 浏览器 console 输出（最近 50 条） | 数组，每条 <500 字符 |
| `evidence.network_summary` | 失败前后的网络请求记录（仅 url/method/status） | 数组，最多 30 条 |
| `evidence.old_fixture_diff` | 仅 `kind=oracle_drift` 时填，oracle 与实际响应的 diff | <2KB |
| `hypothesis` | 基于上面文本证据的初步诊断 | 1-3 句话 |

#### 仅供人工查看（AI 必须忽略）
| 字段 | 用途 |
|---|---|
| `evidence.screenshot_path` | 人工排查时打开看 |
| `evidence.video_path` | 人工看交互过程 |
| `evidence.trace_path` | 人工用 `npx playwright show-trace` 打开 |

### 关键约束

- `kind` 必须是 8 个枚举之一
- `hypothesis` **必须仅基于文本证据**——禁止出现"截图显示...""从图片看..."类措辞
- `hypothesis` 是初步诊断，**不修代码**
- `iteration` 由 hook 注入，便于主 agent 知道循环到第几次
- 不允许返回原始 Playwright 日志（context 隔离的意义）
- screenshot/video/trace 字段保留路径供人事后查，但**不要把图像内容塞进 JSON**

---

## 7. 后续维护期 SOP

进入维护期的标志：新项目核心流程跑通，team 工作重心从"修跑不通的"转向"加新业务"。

| 触发 | 应做 |
|---|------|
| 新增业务流程 | 重走完整 8 步流水线（录制 → 扫码 → spec → 生成 → verify → 接入） |
| 老业务规则变了（业务方告知） | 改 spec → 重 verify-on-old → 让生成器更新测试 |
| 老业务规则悄悄变了（oracle drift 暴露） | 月度任务捕获 → 业务方 review → 决定是 spec 错还是接受新行为 |
| 测试连续失败 | 按 8 类 `kind` 分诊，never 自动改业务代码 |
| 沙箱挂了 | tests/new 不受影响（fixtures 已落地）；只阻塞写新 case |

---

## 8. 后续步骤

1. 用户复核本设计文档（spec review gate）
2. 进入 `superpowers:writing-plans`，按本设计撰写 day-by-day 实施计划
3. 实施计划由后续 agent 按 `superpowers:executing-plans` 执行

---

## 附录 A: 4 个 subagent 的 prompt 骨架

### A.1 spec-extractor.md
```markdown
你是老 Angular 代码扫描器。给定一个流程名（如 order-submit），扫仓库找：
- routes 配置中匹配的路径
- 对应 component 的 form 校验规则、guard、resolver
- service 中相关的 API 调用、错误处理
- 任何业务规则常量（最大金额、状态枚举等）

输出格式: markdown，按"路由/表单规则/提交动作/错误处理"分节。
不输出任何代码片段，只输出业务规则文字描述。
```

### A.2 spec-merger.md
```markdown
你是 spec 合并器。输入：
- recorded/<flow>/raw.ts（codegen 录制脚本）
- recorded/<flow>/network.json（真实接口响应）
- recorded/<flow>/extracted-rules.md（spec-extractor 产出）

输出 specs/<flow>.spec.md，包含：
- 主流程（标"录制 ✓ 验证"）
- 分支（标"代码 ⚠ 未录制"）
- Oracle（来自 network.json）
- 字段映射表（老 selector → 新 selector 候选）

每条 expect 必须标号（#1, #2, ...）便于后续映射检查。
```

### A.3 test-generator.md
```markdown
你是测试生成器。输入 specs/<flow>.spec.md，输出三个文件：
1. pages/<Flow>Page.ts —— PO 类，按 mode='old'|'new' 切 selector
2. tests/verify-on-old/<flow>.spec.ts —— mode='old'
3. tests/new/<flow>.spec.ts —— mode='new'

约束：
- 每个 expect 加注释 `// from spec #N`
- 业务字段（amount, address 等）封装到 PO 方法 fill()
- selector 优先 testid，fallback role/label
```

### A.4 test-runner.md
```markdown
你是测试运行验证者。输入：要跑的 case 名（或 'all'）。

工作流程：
1. 读 .opencode/focus.json，确定目标
2. 跑 npm run e2e:case <names> 或 npm run e2e:smoke
3. 解析 test-results/results.json
4. 失败时读取每个失败 test 的 outputDir：
   - 读 dom.html → 按规则截取 dom_excerpt（2-5KB）
   - 读 console.log → 取最后 50 行作 console_logs
   - 读 network.json → 取最近 30 条作 network_summary
5. 输出严格符合第 6 章契约的 JSON

【硬约束】本项目 LLM 仅支持文本输入：
- 不要尝试读取 *.png / *.webm / *.zip 文件
- hypothesis 必须仅基于 dom_excerpt / console_logs / network_summary / error_message / stack
- 禁止在 hypothesis 中出现"截图显示""从图片看""视频中可以看到"等措辞
- screenshot_path / video_path / trace_path 字段只保留路径供人工查看，不读其内容

不输出任何 Playwright 原始日志、stack trace 全文、stdout。
hypothesis 字段必须是初步诊断，不能包含修复指令。
```

---

## 附录 B: package.json scripts

```json
{
  "scripts": {
    "e2e:smoke":   "playwright test tests/new",
    "e2e:case":    "node scripts/run-case.mjs",
    "e2e:by-spec": "node scripts/run-by-spec.mjs",
    "e2e:debug":   "PWDEBUG=1 node scripts/run-case.mjs --headed --slow",
    "record":      "node scripts/record-with-network.mjs",
    "verify":      "PW_BASE=$OLD_SANDBOX_URL playwright test tests/verify-on-old"
  }
}
```

---

## 附录 C: 核心交付清单（2 周末必须有的）

### 工具链 (D1-D2)
- [ ] e2e/ 项目骨架
- [ ] record-with-network.mjs
- [ ] test-data-factory.ts + accounts.ts
- [ ] 4 个 subagent prompt
- [ ] completion hook 配置（或降级 shell wrapper）

### 测试用例 (D3-D8)
- [ ] 3 个 E2E 全链路 case
- [ ] 8 个单步聚焦 case
- [ ] 4 个分支跳转 case
- [ ] 全部 verify-on-old 通过
- [ ] fixtures/ 全部落地

### 文档 (D9-D10)
- [ ] 团队上手 SOP（2 页）
- [ ] 失败分诊单页（按 8 类 kind）
- [ ] 维护期 SOP（来自第 7 节）

---

## 附录 D: 实施前必填配置（D1 启动前 ready）

整套设计的实施依赖一系列具体配置项。本附录是 **占位符 + 来源说明**。所有 `[TBD]` 必须在 D1 启动前由对应负责人填完，否则起步推迟。

### D.1 项目路径

| 配置项 | 值 | 来源 |
|---|---|---|
| 老 Angular 项目本地路径 | `[TBD]` | 团队成员本地 clone |
| 新 React 项目本地路径 | `[TBD]` | 团队成员本地 clone |
| e2e 测试代码所在仓库 | `[TBD - 独立仓 / 新项目子目录]` | 决策点（推荐新项目仓 `e2e/` 子目录） |

### D.2 环境地址

| 配置项 | 值 | 来源 |
|---|---|---|
| 老 Angular UAT/沙箱 URL | `[TBD]` | 业务方 / 运维 |
| 老 Angular 线上 URL（参考用） | `[TBD]` | 公开 |
| 新 React SIT URL | `[TBD]` | 团队 |
| 新 React 本地 dev URL | `[默认 http://localhost:3000]` | 项目自带 |

### D.3 测试账号

| 配置项 | 值 | 来源 |
|---|---|---|
| 老沙箱测试账号（可登录、可发起开户） | `[TBD - 用户名 + 密码]` | 沙箱团队 |
| 新 SIT 测试账号 | `[TBD]` | 新项目团队 |
| 并发备用账号（至少 2-3 个） | `[TBD]` | 运维 |
| 登录步骤说明（如有特殊：双因子/验证码/SSO） | `[TBD - 步骤序列]` | 沙箱团队 |

### D.4 沙箱预置数据（仅需测试 customer，无 KYC 配置）

> 本项目**不需要** KYC / OCR / 活体 / 反欺诈 / AML 任何 bypass 配置。沙箱测试账号能登录、能创建 application 即可。

| 配置项 | 值 | 来源 |
|---|---|---|
| 测试 customer (default) | `[TBD - customer number]` | 沙箱团队预置 |
| 测试 customer (有多条 applications) | `[TBD - customer number]` | 沙箱团队预置 + 至少 1 条已存在 application 用于列表测试 |
| 测试 application 草稿（用于 Resume 场景） | `[TBD - 通过 e2e 自己 Save 留下，或沙箱团队预置]` | 沙箱团队 / e2e seed |

### D.5 业务流程清单（D1 上午探索后填）

#### 探索方法
1. 用 `spec-extractor` 自动扫老 Angular 仓库 → 列出所有 route + 主要 component
2. 团队成员手动浏览 UAT，按 "可达页面/可触发流程" 列 inventory
3. 两路汇总 → 按业务价值打分 → 选出 15 个

#### 流程清单（基于 §0 业务概览的建议候选，D1 探索后微调确认）

> 下表是基于已知业务结构的**预填候选**，每条还需 D1 上午探索时填入真实路由 / 关键步骤名 / 优先级最终确认。

| 类型 | # | 流程名 | 业务描述 | 老路由 | 新路由 | 优先级 | 状态 |
|---|---|---|---|---|---|---|---|
| E2E 全链路 | E2E-1 | 直接创建 / Non-F2F / Sole | 主路径，最简：1.1 选 Non-F2F → 1.2 Sole → 1.3.2.1 直接创建 → 4 步 → Summary | `[TBD]` | `[TBD]` | P0 | 候选 |
| E2E 全链路 | E2E-2 | 基于 customer 创建 / F2F / Joint | 最复杂：1.1 选 F2F → 1.2 Joint → 1.3 search customer → 1.3.2.2 基于 customer 创建（验证 prefill） → 7 步 → Summary | `[TBD]` | `[TBD]` | P0 | 候选 |
| E2E 全链路 | E2E-3 | Resume 草稿（从 Maker Dashboard） | Maker Dashboard 列表 → 点未完成 application → resume 到正确步骤 → 完成剩余步骤 → Summary | `[TBD]` | `[TBD]` | P0 | 候选 |
| 单步聚焦 | S-1 | 1.1 入口选择 | 验证 Non-F2F / F2F 两选项点击后路由正确 | `[TBD]` | `[TBD]` | P1 | 候选 |
| 单步聚焦 | S-2 | 1.2 账户类型选择 | 验证 Sole / Joint 两选项点击后流程切换正确 | `[TBD]` | `[TBD]` | P1 | 候选 |
| 单步聚焦 | S-3 | 1.3.1.1 customer search 输入 | 输入 customer number → search → 列表加载（数据正确） | `[TBD]` | `[TBD]` | P1 | 候选 |
| 单步聚焦 | S-4 | 1.3.1.2 customer applications 列表 | 显示该 customer 所有 applications，点击未完成项 resume | `[TBD]` | `[TBD]` | P1 | 候选 |
| 单步聚焦 | S-5 | 1.3.2.2 Prefill 验证 | 基于 customer 创建后，多个字段确实预填正确（这是 prefill 这一关键功能的回归保护） | `[TBD]` | `[TBD]` | P1 | 候选 |
| 单步聚焦 | S-6 | 任意中间步骤 subTabs 切换 | subTabs 切换状态保留、字段不丢失 | `[TBD]` | `[TBD]` | P1 | 候选 |
| 单步聚焦 | S-7 | 底部 Save 按钮（保存草稿） | 任意中间步骤点 Save → 退出 → 列表能找到该草稿 | `[TBD]` | `[TBD]` | P1 | 候选 |
| 单步聚焦 | S-8 | 底部 Back 按钮 | 中间步骤点 Back → 上一步、字段保留 | `[TBD]` | `[TBD]` | P1 | 候选 |
| 分支跳转 | B-1 | 1.1 选 Non-F2F vs F2F 步数差异 | 选择后流程步数不同（4 vs 7） | `[TBD]` | `[TBD]` | P2 | 候选 |
| 分支跳转 | B-2 | 1.2 Sole vs Joint 子流程差异 | Joint 多了第二申请人相关字段 | `[TBD]` | `[TBD]` | P2 | 候选 |
| 分支跳转 | B-3 | 1.3 search vs 新建 | 选 search 进入 customer 输入页；选新建直接进入步骤 1 | `[TBD]` | `[TBD]` | P2 | 候选 |
| 分支跳转 | B-4 | 底部 Exit 退出确认 | 点 Exit → 弹确认对话框 → 确认后回 menu / 取消后留在原页 | `[TBD]` | `[TBD]` | P2 | 候选 |

> **D1 上午需做的事**：把 `[TBD]` 路由填上，把"候选"状态改成"已确认"或"已替换为 X"，并验证每条 case 在老 UAT 上能手工走通。

#### Maker / Checker Dashboard 是否进 case 清单？

当前 15 个 case 全部围绕 Account Opening。Maker / Checker Dashboard 仅作为：
- **Pilot 1 候选**（D1 验证工具链）
- **E2E-3 的入口**（从 Maker Dashboard 列表 resume 草稿）

如果 Maker / Checker 自身的查看功能（搜索、筛选、详情）也需要 E2E 保护，需扩大总数到 20+。本 sprint 不覆盖，列入维护期 backlog。

### D.6 opencode 配置

| 配置项 | 值 | 来源 |
|---|---|---|
| opencode 版本 | `[TBD]` | 用户本地安装 |
| subagent 配置位置 | `[TBD - 例: .opencode/agent/ 或 opencode.json]` | opencode 文档 |
| completion hook 配置位置 + 字段名 | `[TBD]` | opencode 文档（决定 hook 能否原生支持，否则降级 shell） |
| 模型 | gpt-5.4 | 已定 |

### D.7 接口/路由信息（D1 探索同时收集）

| 配置项 | 值 | 来源 |
|---|---|---|
| 老 API 域名/前缀 | `[TBD]` | 老代码 + 网络抓包 |
| 新 API 域名/前缀 | `[TBD]` | 新项目配置 |
| 关键 API endpoint 列表 | `[TBD - 由 spec-extractor 提取]` | 老代码 |
| 登录接口（老/新） | `[TBD]` | 探 |
| 关键开户提交接口 | `[TBD]` | 探 |

### D.8 配置文件模板

D.1-D.7 都填完后，建议落到 `e2e/.env.local`（不入仓） + `e2e/config.yaml`（入仓）：

```env
# .env.local（含敏感信息，不入仓）
OLD_SANDBOX_URL=https://...
NEW_SIT_URL=https://...
NEW_LOCAL_URL=http://localhost:3000
OLD_TEST_USER=...
OLD_TEST_PASSWORD=...
NEW_TEST_USER=...
NEW_TEST_PASSWORD=...
TEST_CUSTOMER_DEFAULT=...
TEST_CUSTOMER_WITH_APPS=...

# 默认 base URL（本地测试时用 NEW_LOCAL_URL；上 SIT 验证时用 NEW_SIT_URL）
PW_BASE=$NEW_LOCAL_URL
```

```yaml
# config.yaml（入仓，无敏感信息）
projects:
  old_repo: /path/to/old-angular
  new_repo: /path/to/new-react
flows:
  e2e:
    - id: E2E-1
      name: 直接创建 / Non-F2F / Sole
      old_route: [TBD]
      new_route: [TBD]
  single_step:
    - id: S-1
      target_step: 1
      description: 1.1 入口选择 Non-F2F vs F2F
  branch:
    - id: B-1
      decision_point: 1.1
      branch_value: F2F
      expected_outcome: 进入 7 步流程
```

### D.9 配置就绪 checklist（D1 启动前最后一遍）

- [ ] 项目路径（老/新）已 clone 到本地
- [ ] 老 UAT URL 可访问，测试账号能登录
- [ ] 新项目本地 dev 能启起来（**测试主跑环境**），新 SIT URL 可访问（次要环境，定期跑）
- [ ] 沙箱预置的测试 customer（D.4）确实存在，能成功 search 出来
- [ ] 15 个 case 清单写在 D.5
- [ ] opencode 当前版本的 hook/subagent 配置方式已查清（决定 D2 是用原生 hook 还是降级 shell）

> **缺哪一项就停哪一项**，不要"先开始再说"。D1 起步前的 1 小时把这个 checklist 走一遍，能省掉后面 3 天的 debug 时间。

### D.10 模型能力假设（重要）

| 假设 | 影响 | 兜底 |
|---|---|---|
| **公司 LLM (gpt-5.4) 仅支持文本输入** | screenshot/video/trace.zip 对 AI 完全无效 | §4.4 文本证据自动 dump、§6 输出契约 text-only、A.4 prompt 显式禁用读图 |
| 模型上下文窗口（按 gpt-5 系列） | dom_excerpt 不能太大 | 截取规则 2-5KB，超过就只取头尾 |
| 如未来公司 LLM 升级支持多模态 | 设计仍向后兼容 | screenshot_path 字段一直保留，未来加 prompt 让 AI 也读图即可，不需要重构 |
