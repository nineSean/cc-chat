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

---

## 1. 决策约束（已锁定）

| 维度 | 选择 |
|------|------|
| 老系统定位 | 录制工厂（一次性） |
| 新老 UI 一致性 | 中等（50-80%） |
| 覆盖范围 | 15 个核心 case（金字塔 3+8+4） |
| AI 介入点 | 提炼规格 / 改写脚本 / 抽断言 / 失败诊断 |
| Selector 策略 | testid 优先，role/label 兜底 |
| 老系统访问 | 有沙箱 |
| KYC/OCR/反欺诈 | 沙箱有完整测试模式开关 |
| 测试身份证/手机号段 | 有专用测试段 |
| 数据清理 | DBA 月度 cron（DELETE WHERE email LIKE 'e2e-%'） |
| 流程结构 | 5-15 步、多步骤带分支、有草稿保存 |
| 测试运行环境 | opencode + gpt-5.4 |
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
│   ├── customer-factory.ts     # 测试数据生成
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
       "evidence": {
         "trace": "test-results/order-submit/trace.zip",
         "screenshot": "test-results/order-submit/failed.png"
       },
       "hypothesis": "新项目 OrderPage 没加 data-testid=order-submit-btn"
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

**核心洞察**: 每个 case = 一个新 customer。**测试天然幂等，不需要清理接口**。问题转变为"生成保证不冲突的 KYC 数据 + 防风控拦截"。

#### Customer Factory
```ts
// e2e/data/customer-factory.ts
import { randomUUID } from 'crypto';

export interface TestCustomer {
  runId: string;
  caseId: string;
  email: string;
  phone: string;
  idNumber: string;
  firstName: string;
  lastName: string;
  birthDate: string;
}

export function createTestCustomer(caseId: string): TestCustomer {
  const runId = process.env.RUN_ID ?? randomUUID().slice(0, 8);
  const seq   = Date.now().toString().slice(-7);

  return {
    runId,
    caseId,
    email:     `e2e-${caseId}-${runId}-${seq}@test.local`,
    phone:     `199${seq.slice(-8)}`,
    idNumber:  `99999${seq}1990010100`,
    firstName: 'E2E',
    lastName:  `${caseId}-${seq}`,
    birthDate: '1990-01-01',
  };
}
```

四个特性：
- **唯一**（同 runId 内不冲突）：`Date.now()` + 自增 seq
- **可追溯**（出问题能反查）：email/lastName 嵌 `caseId-runId-seq`
- **可识别**（DBA / 业务方一眼认出）：`e2e-` 前缀 + 测试号段 + `@test.local`
- **可重放**（debug 时复现）：显式传 `RUN_ID=xxx` 重跑产出可预测数据

#### Sandbox Bypass（已确认沙箱全部支持）

| 系统 | 沙箱绕法 |
|---|---|
| 公安身份证校验 | 测试身份证段（`99999*`）直接放过 |
| 短信验证码 | 测试号段固定验证码 / sandbox bypass |
| 邮箱验证 | `@test.local` 域跳过 |
| 反欺诈/风控 | 测试 IP / 测试账号绕过 |
| KYC OCR | 上传特定文件名 → mock |
| 活体识别 | sandbox skip 开关 |
| AML 反洗钱 | 测试身份段不命中名单 |

```ts
// e2e/setup.ts
test.beforeEach(async ({ page }) => {
  await page.context().addCookies([
    { name: 'sandbox-skip-kyc', value: 'true', url: process.env.PW_BASE! }
  ]);
});
```

#### KYC / OCR / 活体处理（按推荐度）
1. 沙箱总开关（`?skipKyc=true` / cookie / API 配置）
2. Playwright 上传预设文件
3. Playwright 路由 mock（最后手段）

#### 数据堆积兜底
**DBA 月度 cron**: `DELETE FROM customer WHERE email LIKE 'e2e-%'`
- 不依赖业务接口
- 第一周内必须跟 DBA 确认部署

### 4.4 取证配置

```ts
// playwright.config.ts
use: {
  trace:      'retain-on-failure',
  screenshot: 'only-on-failure',
  video:      'retain-on-failure',
},
reporter: [
  ['json', { outputFile: 'test-results/results.json' }],   // test-runner 解析
  ['html', { outputFolder: 'test-results/html' }],
  ['list'],
],
```

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
- 个人/外籍（外籍 KYC 分支）

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

| 天 | 工作 | 产出 |
|---|------|------|
| **D1 上午** | 项目脚手架 / record-with-network / customer-factory / sandbox bypass / 4 个 subagent prompt v0 / 找 DBA 沟通 cron | 工具链可用 |
| **D1 下午** | Pilot 1（最简非开户流程，验证工具链）整条流水线走一次 | 工具链验证完毕 |
| **D2** | Pilot 2（个人/本地/年轻/进取 开户主路径）走通；completion hook 配置 + 真实闭环跑一次 | 1 个开户 case 在 hook 循环跑通 |
| **D3-D4** | 2 个 E2E（企业本地、个人外籍）+ 2 个单步 | 4 个 case |
| **D5-D6** | 4 个单步（KYC、风险偏好、协议等） | 4 个 case |
| **D7** | 4 个分支跳转 + 2 个剩余单步 | 6 个 case |
| **D8** | 补缺 + Flaky 标记 + 全部 case 在 verify-on-old 通过 | 15 个 case 全产 |
| **D9** | 团队上手文档（2 页 SOP）+ 失败分诊单页 | 文档交付 |
| **D10** | DBA cron 验证 + 整体复查 + buffer | 缓冲日 |

合计 **3 E2E + 8 单步 + 4 分支 = 15 个 case**

### 5.3 Pilot 策略（分两阶段）

#### Pilot 1: 最简非开户流程（D1 下午）
- 目标: 验证整条工具链能跑通
- 候选: 系统中最简单的非开户页面（如登录页 / 系统首页 / 任意只读查询页 / 一个不涉及 KYC 的设置页面）—— 由项目实际情况决定
- 选择标准: 步骤 ≤3 步、无 KYC/OCR/反欺诈交互、能在 5 分钟内手工走通
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
| Customer Factory + Sandbox bypass | 不可绕过 |
| DBA cron | 沙箱会爆 |

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
| DBA cron 没落实 | 低 | 中 | D1 必须发起沟通，不拖 |

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

为保证主 agent 收到的失败信息可被一致解析，test-runner subagent 必须输出如下 JSON：

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
      "evidence": {
        "trace": "test-results/order-submit/trace.zip",
        "screenshot": "test-results/order-submit/failed.png",
        "old_fixture_diff": "fixtures/order-submit.json vs actual response: ..."
      },
      "hypothesis": "新项目 OrderPage 还没加 data-testid=order-submit-btn"
    }
  ],
  "iteration": 2,
  "max_iteration": 5
}
```

**关键约束**：
- `kind` 必须是 8 个枚举之一
- `hypothesis` 是初步诊断，**不修代码**
- `iteration` 由 hook 注入，便于主 agent 知道循环到第几次
- 不允许返回原始 Playwright 日志（context 隔离的意义）

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
4. 输出严格符合第 6 章契约的 JSON

不输出任何 Playwright 原始日志、stack trace、stdout。
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
- [ ] customer-factory.ts + accounts.ts
- [ ] sandbox bypass setup
- [ ] 4 个 subagent prompt
- [ ] completion hook 配置（或降级 shell wrapper）
- [ ] DBA cron 部署

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
