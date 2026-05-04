# Account Opening E2E Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 React 重构项目里搭建一套 Playwright + opencode + gpt-5.4 驱动的 E2E 测试体系，2 周内交付 15 个 Account Opening 核心 case，并建立 Hook 闭环让 AI 能自动跑测试 → 失败 → 修代码 → 再跑。

**Architecture:** Hybrid Cross-Check（QA 录制 + AI 扫老代码 → spec → PO 双版本测试）。文本证据（DOM excerpt / console / network）替代 screenshot 适配公司 LLM 无多模态约束。Hook 闭环 max=5 次防死循环。

**Tech Stack:** Playwright 1.x (TypeScript) / opencode + gpt-5.4 / Node 20+ / 新加坡 React 项目子目录 `xx-sg-hbsp/`

**前置约定**:
- **路径前缀**: 项目按国家组织，新加坡子目录是 `xx-sg-hbsp/`。本 plan 中所有 `e2e/...` 与 `.opencode/...` 路径都相对于 `xx-sg-hbsp/`。例如 plan 写 `e2e/playwright.config.ts`，实际路径是 `xx-sg-hbsp/e2e/playwright.config.ts`。
- **命令在 `xx-sg-hbsp/` 下执行**（每个 Task 起手默认 `cd xx-sg-hbsp` 一次；后续命令不重复 cd）。
- **`.opencode/` 在 `xx-sg-hbsp/.opencode/`**，**不在 `e2e/` 内** —— opencode 默认从工作目录向上查找配置，放在项目根级让无论 e2e 还是业务代码上下文都被识别。
- **Hook 模式**: focus.json 含 `mode` 字段（`auto`/`debug`/`off`）。`auto` 时 hook 闭环开启；`debug`/`off` 时 hook 跳过，人工接管调试。
- Pilot 1 候选: Account Opening 1.1 入口选择页（即 S-1）
- Pilot 2 候选: E2E-1（个人 / Non-F2F / Sole 主路径）
- 设计文档参照: `docs/superpowers/specs/2026-05-03-e2e-test-strategy-design.md`（在 cc-chat 仓库；engineer 应同时打开此文档）
- 涉及"老 Angular 项目"的工作，假设 engineer 已 clone 老项目到本地（参照附录 D.1）

---

## File Structure

实施后将在 **`xx-sg-hbsp/`** 子目录下产生：

```
xx-sg-hbsp/                                    # 新加坡 React 项目根
├── package.json                               # 加 e2e:* scripts
├── src/                                       # 业务代码（已存在）
├── .gitignore                                 # 追加忽略项
│
├── .opencode/                                 # opencode 配置（项目级，非 e2e/ 内）
│   ├── agent/
│   │   ├── spec-extractor.md
│   │   ├── spec-merger.md
│   │   ├── test-generator.md
│   │   └── test-runner.md
│   ├── focus.json                             # 当前关注 case + mode 开关
│   └── hook-config.json                       # completion hook 配置
│
└── e2e/                                       # 测试代码
    ├── playwright.config.ts                   # Playwright 主配置
    ├── tsconfig.json                          # 测试代码独立 tsconfig
    ├── .env.example                           # 复制为 .env.local
    │
    ├── fixtures/                              # 老沙箱 Oracle 数据
    │   └── (各 flow 的 .json，verify-on-old 后落地)
    │
    ├── data/
    │   ├── test-data-factory.ts               # 唯一测试数据生成
    │   └── accounts.ts                        # 测试账号映射表
    │
    ├── pages/                                 # PO 类（业务命名）
    │   ├── AccountOpeningEntryPage.ts         # 1.1 入口选择
    │   ├── AccountOpeningTypePage.ts          # 1.2 账户类型
    │   ├── AccountOpeningCustomerSearchPage.ts # 1.3 search 入口
    │   ├── AccountOpeningFlow.ts              # 1.4 多步主流程
    │   ├── AccountOpeningSummaryPage.ts       # 1.5 summary
    │   └── MakerDashboardPage.ts              # menu 2 仅 resume 入口
    │
    ├── helpers/
    │   ├── text-evidence.ts                   # 文本证据自动 dump fixture
    │   └── login.ts                           # 共享登录 helper
    │
    ├── tests/
    │   ├── verify-on-old/                     # 一次性 verify（mode=old）
    │   └── new/                               # 主战场（mode=new，长期跑）
    │
    ├── recorded/                              # codegen 原料，不进 CI
    │   └── <flow>/{raw.ts, network.json, dom-snapshot.html, meta.json}
    │
    ├── specs/                                 # AI 合并后的业务规格
    │
    └── scripts/
        ├── record-with-network.mjs            # 包 codegen + 抓网络
        ├── run-case.mjs                       # npm run e2e:case 入口
        ├── run-by-spec.mjs                    # 按业务名跑
        ├── set-mode.mjs                       # 切 focus.mode (auto/debug/off)
        ├── set-focus.mjs                      # 写当前关注 case
        └── run-case-with-fallback.sh          # Hook 降级版 shell wrapper
```

`xx-sg-hbsp/` 还会修改:
- `package.json`: 加 e2e:* + e2e:debug:on/off + e2e:focus 系列 scripts
- `.gitignore`: 加 `e2e/recorded/`, `test-results/`, `e2e/.env.local`

---

## Task 0: Pre-flight Checklist（D1 启动前 1 小时）

**Files:** N/A（这是行政/调研任务）

**目标**: 走完 spec 附录 D.9 的 6 项 checklist，确保 D1 一开始就能动手。

- [ ] **Step 1: 读 spec 附录 D（D.1-D.10）**
  - 打开 `docs/superpowers/specs/2026-05-03-e2e-test-strategy-design.md` 末尾附录 D 全部
  - 列出哪些 `[TBD]` 由你自己填、哪些要找别的团队/同事
- [ ] **Step 2: 把 D.1-D.4 + D.6 + D.7 的 `[TBD]` 全填上**
  - 老 Angular 项目本地路径
  - 新 React 项目本地路径
  - 老 UAT URL / 新 SIT URL / 新本地 dev URL
  - 老沙箱测试账号 + 新 SIT 测试账号
  - 测试 customer (default + with-apps) customer number
  - opencode 当前版本及 hook/subagent 配置位置
  - 老/新 API 域名前缀

- [ ] **Step 3: 老 UAT 手工验证**
  - 用测试账号登录老 UAT
  - 用测试 customer (with-apps) 在老 UAT 上 search 一次，确认列表能出来
  - 至少创建一个测试草稿（点 Save 退出），下次能 Resume

- [ ] **Step 4: 新本地 dev 环境验证**
  - 在新 React 项目根目录启动 dev server
  - 浏览器打开三个 menu，确认 Account Opening 第一步、Maker Dashboard 列表都能加载

- [ ] **Step 5: opencode hook 文档查清**
  - 查 opencode 当前版本文档，确认 completion hook 怎么配置（字段名/位置）
  - 写在 `.opencode/hook-config.json` 注释里
  - 如果文档不清晰或当前版本不支持 → 标记"D2 直接走降级路径"

- [ ] **Step 6: 启动 sprint 前 commit 一次"D-1 baseline"**

```bash
git checkout -b feature/e2e-sprint
git commit --allow-empty -m "chore: e2e sprint baseline (D-1 pre-flight done)"
```

---

## Task 1: 项目脚手架与目录结构（D1 上午 Track B，30 分钟）

**Files:**
- Create: `e2e/playwright.config.ts`
- Create: `e2e/tsconfig.json`
- Modify: `package.json`（添加 deps + scripts）
- Modify: `.gitignore`（追加忽略项）

- [ ] **Step 1: 安装依赖**

```bash
cd xx-sg-hbsp                # 所有后续命令都在这里跑（每个 task 起手都先 cd）
npm install -D @playwright/test@latest typescript@latest
npx playwright install chromium
```

- [ ] **Step 2: 创建目录树**

```bash
mkdir -p e2e/{pages,data,helpers,tests/verify-on-old,tests/new,recorded,specs,fixtures,scripts}
mkdir -p .opencode/agent      # 注意：.opencode 在项目根，不在 e2e/ 内
```

- [ ] **Step 3: 写 `e2e/playwright.config.ts`**

```ts
import { defineConfig, devices } from '@playwright/test';

const isHookLoop = !!process.env.HOOK_LOOP;
const isCI = !!process.env.CI;

export default defineConfig({
  testDir: './tests',
  outputDir: './test-results',
  retries: isHookLoop ? 0 : (isCI ? 1 : 0),
  reporter: [
    ['json', { outputFile: 'test-results/results.json' }],
    ['html', { outputFolder: 'test-results/html', open: 'never' }],
    ['list'],
  ],
  use: {
    baseURL: process.env.PW_BASE ?? 'http://localhost:3000',
    trace: 'retain-on-failure',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    { name: 'main',  testIgnore: /.*@flaky.*/, use: { ...devices['Desktop Chrome'] } },
    { name: 'flaky', testMatch: /.*@flaky.*/, retries: 3, use: { ...devices['Desktop Chrome'] } },
  ],
});
```

- [ ] **Step 4: 写 `e2e/tsconfig.json`**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "strict": true,
    "baseUrl": ".",
    "paths": { "@e2e/*": ["./*"] },
    "resolveJsonModule": true,
    "skipLibCheck": true
  },
  "include": ["**/*.ts"]
}
```

- [ ] **Step 5: 修改 `.gitignore` 追加**

```
# E2E
e2e/recorded/
e2e/.env.local
test-results/
playwright-report/
```

- [ ] **Step 6: 验证 + commit**

```bash
npx playwright test --list   # 应输出"No tests found"，配置无语法错误
git add e2e/playwright.config.ts e2e/tsconfig.json package.json package-lock.json .gitignore
git commit -m "chore(e2e): scaffold Playwright project + tsconfig"
```

---

## Task 2: package.json scripts + .env.local 模板（D1 上午 Track B，15 分钟）

**Files:**
- Modify: `package.json`
- Create: `e2e/.env.example`

- [ ] **Step 1: 在 `package.json` 的 `scripts` 里加**

```json
"e2e:smoke":     "playwright test --config=e2e/playwright.config.ts tests/new",
"e2e:case":      "node e2e/scripts/run-case.mjs",
"e2e:by-spec":   "node e2e/scripts/run-by-spec.mjs",
"e2e:debug":     "PWDEBUG=1 node e2e/scripts/run-case.mjs --headed --slow",
"e2e:record":    "node e2e/scripts/record-with-network.mjs",
"e2e:verify":    "PW_BASE=$OLD_SANDBOX_URL playwright test --config=e2e/playwright.config.ts tests/verify-on-old",
"e2e:debug:on":  "node e2e/scripts/set-mode.mjs debug",
"e2e:debug:off": "node e2e/scripts/set-mode.mjs auto",
"e2e:focus":     "node e2e/scripts/set-focus.mjs"
```

- [ ] **Step 2: 写 `e2e/.env.example`（提交进仓，开发者复制为 .env.local）**

```env
# 复制为 e2e/.env.local，本地填值，不入仓
OLD_SANDBOX_URL=https://uat.old-account-opening.example.com
NEW_SIT_URL=https://sit.new-account-opening.example.com
NEW_LOCAL_URL=http://localhost:3000

OLD_TEST_USER=
OLD_TEST_PASSWORD=
NEW_TEST_USER=
NEW_TEST_PASSWORD=

TEST_CUSTOMER_DEFAULT=
TEST_CUSTOMER_WITH_APPS=

# 默认 base URL（本地迭代时为 LOCAL，定期跑 SIT 时为 SIT）
PW_BASE=http://localhost:3000
```

- [ ] **Step 3: 验证 + commit**

```bash
npm run e2e:smoke   # 期望"No tests found"，scripts 配置无误
git add package.json e2e/.env.example
git commit -m "chore(e2e): add npm scripts + .env example"
```

---

## Task 3: 测试数据 factory + accounts + login helper（D1 上午 Track B，30 分钟）

**Files:**
- Create: `e2e/data/test-data-factory.ts`
- Create: `e2e/data/accounts.ts`
- Create: `e2e/helpers/login.ts`

- [ ] **Step 1: 写 `e2e/data/test-data-factory.ts`**

```ts
import { randomUUID } from 'crypto';

export interface TestApplicationData {
  runId: string;
  caseId: string;
  uniqueTag: string;   // 嵌入 application 名/备注，用于反查
  /**
   * 业务字段在 D1 探索后由 spec-merger 推荐，按需扩展。
   * 例: applicationName, contactEmail, address, idNumber, ...
   */
  [key: string]: string;
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

/** 已知预置测试 customer。沙箱团队需提前预置（见附录 D.4）。 */
export const KNOWN_TEST_CUSTOMERS = {
  default:  process.env.TEST_CUSTOMER_DEFAULT  ?? '__TBD__',
  withApps: process.env.TEST_CUSTOMER_WITH_APPS ?? '__TBD__',
};
```

- [ ] **Step 2: 写 `e2e/data/accounts.ts`**

```ts
export interface Account {
  user: string;
  pwd: string;
}

/** 沙箱测试账号 */
export function oldSandboxAccount(): Account {
  return {
    user: process.env.OLD_TEST_USER ?? '',
    pwd:  process.env.OLD_TEST_PASSWORD ?? '',
  };
}

/** 新项目（local / SIT）测试账号 */
export function newProjectAccount(): Account {
  return {
    user: process.env.NEW_TEST_USER ?? '',
    pwd:  process.env.NEW_TEST_PASSWORD ?? '',
  };
}
```

- [ ] **Step 3: 写 `e2e/helpers/login.ts`**

```ts
import { Page } from '@playwright/test';
import { Account } from '../data/accounts';

/**
 * 通用登录。具体 selector 在 D1 探索后填到下方常量。
 * 这里假设登录是表单形式，user/pwd/submit 按钮可定位。
 */
const LOGIN_SELECTORS = {
  user:   'input[name=username], input[type=email]',
  pwd:    'input[name=password], input[type=password]',
  submit: 'button[type=submit]',
};

export async function login(page: Page, acc: Account, baseUrl?: string) {
  const target = baseUrl ?? process.env.PW_BASE ?? '';
  await page.goto(`${target}/login`);
  await page.locator(LOGIN_SELECTORS.user).fill(acc.user);
  await page.locator(LOGIN_SELECTORS.pwd).fill(acc.pwd);
  await page.locator(LOGIN_SELECTORS.submit).click();
  await page.waitForURL(/\/(home|dashboard|account-opening)/, { timeout: 15_000 });
}
```

- [ ] **Step 4: 验证 TS 编译**

```bash
npx tsc --noEmit -p e2e/tsconfig.json
```

期望: 无报错。

- [ ] **Step 5: commit**

```bash
git add e2e/data e2e/helpers
git commit -m "chore(e2e): add test-data-factory, accounts, login helper"
```

---

## Task 4: text-evidence fixture（D1 上午 Track B，30 分钟）

**Files:**
- Create: `e2e/helpers/text-evidence.ts`

text-evidence fixture 是 AI 失败诊断的核心——在每个测试运行时自动捕获 console 和 network，失败时把 DOM/console/network 三类文本落到 outputDir。

- [ ] **Step 1: 写 `e2e/helpers/text-evidence.ts`**

```ts
import { test as base } from '@playwright/test';
import * as fs from 'fs/promises';
import * as path from 'path';

export type TextEvidence = {
  consoleLogs: string[];
  networkLog: Array<{
    url: string;
    method: string;
    status: number;
    timestamp: number;
  }>;
};

export const test = base.extend<{ textEvidence: TextEvidence }>({
  textEvidence: async ({ page }, use, info) => {
    const evidence: TextEvidence = { consoleLogs: [], networkLog: [] };

    page.on('console', (msg) => {
      evidence.consoleLogs.push(`[${msg.type()}] ${msg.text()}`);
    });
    page.on('response', (resp) => {
      evidence.networkLog.push({
        url: resp.url(),
        method: resp.request().method(),
        status: resp.status(),
        timestamp: Date.now(),
      });
    });

    await use(evidence);

    // 失败时 dump 三类文本证据
    if (info.status !== 'passed') {
      const outDir = info.outputDir;
      await fs.mkdir(outDir, { recursive: true });

      try {
        const html = await page.content();
        await fs.writeFile(path.join(outDir, 'dom.html'), html);
      } catch (e) {
        // page 可能已关闭，忽略
      }
      await fs.writeFile(
        path.join(outDir, 'console.log'),
        evidence.consoleLogs.slice(-50).join('\n'),
      );
      await fs.writeFile(
        path.join(outDir, 'network.json'),
        JSON.stringify(evidence.networkLog.slice(-30), null, 2),
      );
    }
  },
});

export { expect } from '@playwright/test';
```

- [ ] **Step 2: 写一个最简自验证测试 `e2e/tests/new/_smoke-self-test.spec.ts`**

```ts
import { test, expect } from '@e2e/helpers/text-evidence';

test('@flaky self-test: text-evidence fixture works on failure', async ({ page, textEvidence }) => {
  await page.goto('about:blank');
  // 故意失败以触发 dump
  await expect(page.locator('#non-existent')).toBeVisible({ timeout: 100 });
});
```

> 该测试打了 `@flaky` 标签，会进 flaky project 跑 3 次，最后失败但不阻塞 main project。

- [ ] **Step 3: 跑一次确认 dump 文件出现**

```bash
npx playwright test --config=e2e/playwright.config.ts \
  --project=flaky tests/new/_smoke-self-test.spec.ts || true

ls test-results/*self-test*/
# 期望看到 dom.html / console.log / network.json
```

- [ ] **Step 4: 自验证完成后删除该测试**

```bash
rm e2e/tests/new/_smoke-self-test.spec.ts
```

- [ ] **Step 5: commit**

```bash
git add e2e/helpers/text-evidence.ts
git commit -m "chore(e2e): add text-evidence fixture (dom/console/network dump on failure)"
```

---

## Task 5: 4 个 opencode subagent prompts（D1 上午 Track B，45 分钟）

**Files:**
- Create: `.opencode/agent/spec-extractor.md`
- Create: `.opencode/agent/spec-merger.md`
- Create: `.opencode/agent/test-generator.md`
- Create: `.opencode/agent/test-runner.md`

- [ ] **Step 1: `.opencode/agent/spec-extractor.md`**

```markdown
---
model: gpt-5.4
description: 扫老 Angular 仓库提炼业务规则
---

你是老 Angular 代码扫描器。给定一个流程名（如 account-opening-entry / order-submit），
你需要扫描已 clone 的老 Angular 仓库，找到相关代码并提炼业务规则。

输入参数: 流程名 + 老 Angular 仓库路径

工作流程:
1. 在老仓库找：
   - routes 配置中匹配该流程的路径
   - 对应 component 的 form 校验规则
   - 路由 guards / resolvers
   - service 中相关的 API 调用、错误处理
   - 业务规则常量（最大金额、状态枚举、字段长度等）

2. 输出 markdown 文件: recorded/<flow>/extracted-rules.md
   分四节: "## 路由" / "## 表单规则" / "## 提交动作" / "## 错误处理"

3. 每节内容要点：
   - 不输出代码片段，只输出业务规则文字描述
   - 路由要写 guards 名字
   - 表单字段要写校验范围（required, length, pattern, conditional）
   - API 要写 endpoint / method / 状态码分支

不要尝试修复或解释代码，只提炼规则。
```

- [ ] **Step 2: `.opencode/agent/spec-merger.md`**

```markdown
---
model: gpt-5.4
description: 合并录制脚本 + AI 扫码 → 业务规格 spec.md
---

你是 spec 合并器。

输入文件:
- recorded/<flow>/raw.ts            （Playwright codegen 录制脚本）
- recorded/<flow>/network.json      （真实接口响应）
- recorded/<flow>/extracted-rules.md（spec-extractor 产出）

输出: specs/<flow>.spec.md

格式约束:
- "## 主流程（录制 ✓ 验证）" 节列出录制覆盖的步骤
- "## 分支（代码 ⚠ 未录制）" 节列出代码暗示但录制未走的分支
- "## Oracle" 节列出 network.json 里关键接口的真实响应
- "## 字段映射（老 → 新）" 节给出每个业务字段的 selector 候选

每条 expect 必须标号（#1, #2, ...），便于后续 test-generator 在测试代码里写 `// from spec #N` 注释。

字段命名遵循业务语义（如 amount/address/customerNumber），不要直接复用录制里的 selector 名。
```

- [ ] **Step 3: `.opencode/agent/test-generator.md`**

```markdown
---
model: gpt-5.4
description: 按 spec 生成 PO + 双版本测试
---

你是测试生成器。

输入: specs/<flow>.spec.md

输出三个文件:
1. pages/<Flow>Page.ts —— PO 类，按 mode='old'|'new' 切 selector
2. tests/verify-on-old/<flow>.spec.ts —— mode='old'，跑老沙箱
3. tests/new/<flow>.spec.ts —— mode='new'，跑新项目

约束:
- import test/expect 必须从 '@e2e/helpers/text-evidence'，不要从 '@playwright/test' 直接导入
- 每个 expect 加注释 `// from spec #N`，对应 spec.md 里的标号
- 业务字段（amount/address 等）封装到 PO 方法 fill()/expectXxx()，测试代码不直接调 page.fill
- selector 优先级: getByTestId → getByRole → getByLabel → locator(css)
- 共享数据用 e2e/data/test-data-factory.ts 的 createTestData(caseId)
- 测试 customer number 用 KNOWN_TEST_CUSTOMERS.default / .withApps
- 不要 hardcode 沙箱 URL / 测试账号；走 process.env.PW_BASE / accounts.ts

PO 类骨架范例:
\`\`\`ts
export class XxxPage {
  constructor(public page: Page, private mode: 'old' | 'new') {}

  someField = () => this.mode === 'new'
    ? this.page.getByTestId('xxx-yyy')
    : this.page.locator('input[name=xxx]');

  async fillAll(data: { a: string; b: string }) { ... }
  async expectSuccess() { ... }
}
\`\`\`
```

- [ ] **Step 4: `.opencode/agent/test-runner.md`**

```markdown
---
model: gpt-5.4
description: 隔离跑 Playwright，输出文本-only JSON 报告（验证者）
---

你是测试运行验证者。输入：要跑的 case 名（或 'all'）。

工作流程:
0. **预检 focus.mode（关键）**: 读 .opencode/focus.json
   - 若 mode === "debug" 或 "off"：立即返回 `{ "skipped": true, "reason": "focus.mode=<mode>" }`，**不跑测试、不修代码**
   - 若 process.env.HOOK_DISABLED === "1"：同上
   - 否则继续
1. 读 .opencode/focus.json 确定目标 case
2. 跑 npm run e2e:case <names> 或 npm run e2e:smoke
3. 解析 test-results/results.json
4. 失败时读取每个失败 test 的 outputDir:
   - 读 dom.html → 按规则截取 dom_excerpt（2-5KB）
   - 读 console.log → 取最后 50 行作 console_logs
   - 读 network.json → 取最近 30 条作 network_summary
5. 输出严格符合下面契约的 JSON

【硬约束】公司 LLM 仅支持文本输入：
- 不要尝试读取 *.png / *.webm / *.zip 文件
- hypothesis 必须仅基于 dom_excerpt / console_logs / network_summary / error_message / stack
- 禁止在 hypothesis 中出现"截图显示""从图片看""视频中可以看到"等措辞
- screenshot_path / video_path / trace_path 字段只保留路径供人工查看，不读其内容

【硬约束】尊重 debug 模式：
- 当 focus.mode 是 debug/off 时，**绝不触发**任何修代码动作
- 仅返回 skipped 报告，让主 agent 知道 hook 没跑测试

dom_excerpt 截取规则:
- 找失败 step 的目标 selector（如 testid='xxx'）
- 在 dom.html 全文搜索该 selector
  - 命中 → 截取该元素 + 父级 3 层 + 兄弟节点 ~100 行
  - 未命中 → 搜近似元素（同 tag 或同 role）；都没有就截 body 前 5KB + 后 5KB

输出 JSON 契约:
\`\`\`json
{
  "summary": { "passed": 0, "failed": 0, "skipped": 0, "duration_ms": 0 },
  "failures": [{
    "test": "...",
    "file": "...",
    "step": "...",
    "kind": "selector_missing|selector_changed|assertion|network|timeout|oracle_drift|infra|flaky",
    "error_message": "...",
    "stack": "...",
    "evidence": {
      "dom_excerpt": "...",
      "console_logs": ["..."],
      "network_summary": [{ "url": "...", "method": "...", "status": 200 }],
      "screenshot_path": "...",
      "video_path": "...",
      "trace_path": "..."
    },
    "hypothesis": "1-3 句话基于文本证据的初步诊断"
  }],
  "iteration": 1,
  "max_iteration": 5
}
\`\`\`

不输出原始 Playwright 日志、stack 全文、stdout。
hypothesis 是初步诊断，不能包含修复指令。
```

- [ ] **Step 5: commit**

```bash
git add .opencode/agent/
git commit -m "chore(e2e): add 4 opencode subagent prompts (extractor/merger/generator/runner)"
```

---

## Task 6: completion hook 配置 + 降级 shell wrapper（D1 上午 Track B，30 分钟）

**Files:**
- Create: `.opencode/hook-config.json`
- Create: `.opencode/focus.json`
- Create: `e2e/scripts/run-case.mjs`
- Create: `e2e/scripts/set-mode.mjs`
- Create: `e2e/scripts/set-focus.mjs`
- Create: `e2e/scripts/run-case-with-fallback.sh`

- [ ] **Step 1: 写 `.opencode/focus.json` 初始（含 mode 字段）**

```json
{
  "cases": [],
  "mode": "auto",
  "started_at": null,
  "owner": null
}
```

`mode` 字段语义：
- `"auto"` — 默认；hook 触发时正常调 test-runner，失败回喂主 agent
- `"debug"` — hook 跳过，不自动跑测试；用于人工调试某 case
- `"off"` — 等同 debug，语义上"我现在不在做这个 case"

> 主 agent 在开始处理某个 flow 前会写入 cases + mode='auto'；test-runner subagent 读这个文件决定跑什么 + 是否跳过。

- [ ] **Step 2: 写 `.opencode/hook-config.json`**（具体字段以 Task 0 Step 5 查到的 opencode 版本为准）

```jsonc
{
  // 注意：以下字段名假设 opencode 当前版本支持。
  // Task 0 已查清若不支持，则不启用此 hook，直接走 fallback shell。
  "completion_hook": {
    "enabled": true,
    "max_iterations": 5,
    "on_max_reached": "stop_with_message",
    "message": "已尝试 5 次仍有失败，请人工介入。最近报告见 test-results/results.json。",
    "invoke_subagent": "test-runner",
    "subagent_args": {
      "focus_file": ".opencode/focus.json"
    },
    // 调试模式跳过条件
    "skip_when_focus_mode_in": ["debug", "off"],
    "skip_when_env_set": "HOOK_DISABLED"
  }
}
```

> 如果 opencode 当前版本 hook 字段不支持 `skip_when_focus_mode_in` / `skip_when_env_set`：等价做法已写在 `.opencode/agent/test-runner.md`（见 Task 5 Step 4 工作流程的 Step 0 预检）—— subagent prompt 内自检 mode，遇 debug/off 立即 return `{ "skipped": true }` 不跑测试。功能等价。

- [ ] **Step 3: 写 `e2e/scripts/set-mode.mjs`（切换 focus.mode）**

```js
#!/usr/bin/env node
// Usage:
//   node e2e/scripts/set-mode.mjs auto    # 默认
//   node e2e/scripts/set-mode.mjs debug   # 进入调试模式
//   node e2e/scripts/set-mode.mjs off     # 完全关
import * as fs from 'fs';

const FOCUS = '.opencode/focus.json';
const VALID = ['auto', 'debug', 'off'];

const mode = process.argv[2];
if (!VALID.includes(mode)) {
  console.error(`mode must be one of ${VALID.join('|')}, got: ${mode}`);
  process.exit(2);
}

let f;
try {
  f = JSON.parse(fs.readFileSync(FOCUS, 'utf-8'));
} catch {
  f = { cases: [], mode: 'auto', started_at: null, owner: null };
}
f.mode = mode;
fs.writeFileSync(FOCUS, JSON.stringify(f, null, 2));
console.log(`focus mode set to: ${mode}`);
```

- [ ] **Step 4: 写 `e2e/scripts/set-focus.mjs`（写当前关注 case）**

```js
#!/usr/bin/env node
// Usage:
//   node e2e/scripts/set-focus.mjs <case-name> [<case-name>...]
// 写入 .opencode/focus.json 的 cases 字段，并将 mode 设为 auto
import * as fs from 'fs';

const FOCUS = '.opencode/focus.json';
const cases = process.argv.slice(2);

if (cases.length === 0) {
  console.error('Usage: e2e:focus <case-name> [<case-name>...]');
  process.exit(2);
}

let f;
try {
  f = JSON.parse(fs.readFileSync(FOCUS, 'utf-8'));
} catch {
  f = {};
}
f.cases = cases;
f.mode = 'auto';
f.started_at = new Date().toISOString();
f.owner = process.env.USER ?? 'unknown';

fs.writeFileSync(FOCUS, JSON.stringify(f, null, 2));
console.log(`focus set to: ${cases.join(', ')} (mode=auto)`);
```

- [ ] **Step 5: 写 `e2e/scripts/run-case.mjs`**

```js
#!/usr/bin/env node
// Usage: node e2e/scripts/run-case.mjs <case-name> [<case-name>...]
// 直接转发给 playwright，跑 tests/new/<case-name>.spec.ts

import { spawnSync } from 'child_process';

const cases = process.argv.slice(2).filter(a => !a.startsWith('--'));
const flags = process.argv.slice(2).filter(a => a.startsWith('--'));

if (cases.length === 0) {
  console.error('Usage: e2e:case <name> [<name>...]');
  process.exit(2);
}

const args = [
  'playwright', 'test',
  '--config=e2e/playwright.config.ts',
  ...cases.map(n => `tests/new/${n}.spec.ts`),
  ...flags,
];

const r = spawnSync('npx', args, { stdio: 'inherit', env: process.env });
process.exit(r.status ?? 1);
```

- [ ] **Step 6: 写 `e2e/scripts/run-case-with-fallback.sh`（hook 降级版）**

```bash
#!/bin/bash
# Hook 降级路径（D2 18:00 检查点未通过时启用）
# 主 agent 显式调用此脚本，失败时返回非零；
# 主 agent 自行决定是否修代码 + 再次调用，max=5 由 prompt 自管。

set -e
HOOK_LOOP=1 node e2e/scripts/run-case.mjs "$@"
```

- [ ] **Step 7: chmod + 验证脚本不立即报错**

```bash
chmod +x e2e/scripts/run-case-with-fallback.sh e2e/scripts/run-case.mjs
chmod +x e2e/scripts/set-mode.mjs e2e/scripts/set-focus.mjs

# 测试 set-mode
npm run e2e:debug:on
cat .opencode/focus.json    # 期望 mode: "debug"
npm run e2e:debug:off
cat .opencode/focus.json    # 期望 mode: "auto"

# 测试 run-case
node e2e/scripts/run-case.mjs nonexistent-case 2>&1 | head -5
# 期望: playwright 提示找不到测试文件，但 node script 本身没语法错
```

- [ ] **Step 8: commit**

```bash
git add e2e/scripts .opencode/hook-config.json .opencode/focus.json
git commit -m "chore(e2e): add hook config + run-case/set-mode/set-focus scripts (with fallback)"
```

---

## Task 7: 业务 case inventory 探索（D1 上午 Track A，3-4 小时，与 Task 1-6 并行）

**Files:**
- Modify: `docs/superpowers/specs/2026-05-03-e2e-test-strategy-design.md` 附录 D.5 表格

> 此 task 是调研性质，没有代码。**应在 Task 1-6 同时进行**（团队 A 做调研、团队 B 做脚手架）。如果只有 1 人，把这个任务拆到 D1 整天而不是上午半天，Pilot 1 推到 D2。

- [ ] **Step 1: 自动扫老代码列 routes**

在老 Angular 仓库根目录执行：
```bash
grep -rE "path:\s*'[^']+'" src/ --include="*.ts" -l | head -30
grep -rE "RouterModule.forChild|RouterModule.forRoot" src/ -l
```

把 account-opening 相关的 route + component 整理成清单。

- [ ] **Step 2: 手工浏览 UAT 列可达页面**

用 Account Opening menu 走完一遍每条主路径（Non-F2F+Sole / F2F+Joint / search→resume），记录每个步骤标题、路由变化、关键字段。

- [ ] **Step 3: 与 spec 附录 D.5 候选清单 cross-check**

打开 spec 附录 D.5，逐条核对：
- E2E-1 / E2E-2 / E2E-3 是否合理（路由能否走通）
- S-1 ~ S-8 是否每个都有真实场景对应
- B-1 ~ B-4 是否分支跳转都能复现

把每条的"老路由"、"新路由"、"关键步骤名"填进表格，把"候选"改为"已确认"或"已替换为 X"。

- [ ] **Step 4: 标 Pilot 1 / Pilot 2 在哪个 case**

确认:
- Pilot 1 = S-1（1.1 入口选择 Non-F2F vs F2F）
- Pilot 2 = E2E-1（个人/Non-F2F/Sole 主路径）

- [ ] **Step 5: 跨条目检查"必经字段"**

把 E2E-1 / E2E-2 步骤涉及的所有字段（applicationName / address / contactEmail / 受益人信息 / Joint 第二申请人 / Prefill 来源字段...）列成总表。这个表会传递给 spec-merger，让它生成时把字段映射统一。

- [ ] **Step 6: commit 附录 D.5 更新**

```bash
cd <cc-chat-repo>
git add docs/superpowers/specs/2026-05-03-e2e-test-strategy-design.md
git commit -m "spec: confirm 15 cases with real routes after D1 exploration"
```

---

## Task 8: Pilot 1 — Account Opening 1.1 入口选择页全流程（D1 下午，3-4 小时）

**目标**: 第一次走完整 7 步流水线（录制 → 扫码 → spec 合并 → review → 生成 → verify → 在新项目跑通），验证 Task 1-6 搭的工具链能用。

**Files:**
- Create: `e2e/recorded/account-opening-entry/{raw.ts, network.json, dom-snapshot.html, meta.json}`
- Create: `e2e/recorded/account-opening-entry/extracted-rules.md`
- Create: `e2e/specs/account-opening-entry.spec.md`
- Create: `e2e/pages/AccountOpeningEntryPage.ts`
- Create: `e2e/tests/verify-on-old/account-opening-entry.spec.ts`
- Create: `e2e/tests/new/account-opening-entry.spec.ts`
- Create: `e2e/fixtures/account-opening-entry.json`

- [ ] **Step 1: [Manual] QA 在老 UAT 上录制**

```bash
npm run e2e:record -- account-opening-entry --target=$OLD_SANDBOX_URL
```

操作: 登录 → Menu Account Opening → 1.1 选 Non-F2F → 看到 1.2 页面加载

期望产出: `e2e/recorded/account-opening-entry/raw.ts`、`network.json`、`dom-snapshot.html`、`meta.json` 都生成。

- [ ] **Step 2: 调 spec-extractor**

在 opencode 里:
> "spec-extractor 扫老 Angular 仓库，找 Account Opening 入口页（1.1）相关 route/component/service"

期望产出: `e2e/recorded/account-opening-entry/extracted-rules.md`，至少包含路由、入口按钮的业务规则。

- [ ] **Step 3: 调 spec-merger**

在 opencode 里:
> "spec-merger 合并 recorded/account-opening-entry/raw.ts + extracted-rules.md → specs/account-opening-entry.spec.md"

期望产出: `e2e/specs/account-opening-entry.spec.md`，含主流程标号 expect、字段映射表、Oracle。

- [ ] **Step 4: [Manual] 人工 review spec**

打开 `e2e/specs/account-opening-entry.spec.md` 通读：
- 主流程是否漏步骤？
- 字段映射的"新 selector"候选是否合理？
- 有没有把"Non-F2F vs F2F"分支单独标出？

修订后 commit:
```bash
git add e2e/recorded/account-opening-entry e2e/specs/account-opening-entry.spec.md
git commit -m "feat(e2e): record + spec for account-opening-entry (pilot 1)"
```

- [ ] **Step 5: 调 test-generator**

在 opencode 里:
> "test-generator 按 specs/account-opening-entry.spec.md 生成 PO + verify-on-old + new"

期望产出三个文件。骨架应类似（具体字段以 spec 为准）：

```ts
// e2e/pages/AccountOpeningEntryPage.ts
import { Page, expect } from '@playwright/test';

export class AccountOpeningEntryPage {
  constructor(public page: Page, private mode: 'old' | 'new') {}

  menuAccountOpening = () => this.mode === 'new'
    ? this.page.getByTestId('menu-account-opening')
    : this.page.getByRole('link', { name: /Account Opening/i });

  optionNonF2F = () => this.mode === 'new'
    ? this.page.getByTestId('ao-entry-non-f2f')
    : this.page.getByRole('button', { name: /Non.?Face.?to.?Face/i });

  optionF2F = () => this.mode === 'new'
    ? this.page.getByTestId('ao-entry-f2f')
    : this.page.getByRole('button', { name: /Face.?to.?Face/i });

  async goToEntry() { await this.menuAccountOpening().click(); }
  async selectNonF2F() { await this.optionNonF2F().click(); }
  async expectOnTypePage() {
    // from spec #N
    await expect(this.page).toHaveURL(/account-opening\/(non-f2f|type)/);
  }
}
```

- [ ] **Step 6: TS 编译检查**

```bash
npx tsc --noEmit -p e2e/tsconfig.json
```

如失败，让 test-generator 修。

- [ ] **Step 7: 跑 verify-on-old**

```bash
npm run e2e:verify -- tests/verify-on-old/account-opening-entry.spec.ts
```

期望: PASS。如失败，说明 spec 写错了，回 Step 4 修。

- [ ] **Step 8: 落 fixture**

verify-on-old 跑通后，把 `recorded/account-opening-entry/network.json` 中关键接口响应整理为 `e2e/fixtures/account-opening-entry.json`。

- [ ] **Step 9: 在新项目跑（先看失败什么样）**

```bash
PW_BASE=$NEW_LOCAL_URL npm run e2e:case account-opening-entry
```

期望: 此时大概率失败（新项目没有 testid 或 selector 不对应）。**这是 Hook 闭环要解决的问题，不是缺陷**。

- [ ] **Step 10: 在新项目里加 testid（手工先加一次，验证通路）**

打开新 React 项目对应组件文件，加上 `data-testid="ao-entry-non-f2f"`、`data-testid="ao-entry-f2f"`、`data-testid="menu-account-opening"`。

- [ ] **Step 11: 再跑一次新项目**

```bash
PW_BASE=$NEW_LOCAL_URL npm run e2e:case account-opening-entry
```

期望: PASS（如果 spec 正确且新项目业务通）。

- [ ] **Step 12: commit Pilot 1 完成**

```bash
git add e2e/pages e2e/tests e2e/fixtures e2e/specs
git add <new-project-component-files-with-testid>
git commit -m "feat(e2e): pilot 1 done - account-opening-entry passes both old verify and new"
```

---

## Task 9: Pilot 2 — Account Opening E2E-1 主路径 + Hook 闭环验证（D2 整天）

**目标**: 第二次走完整流水线，但这次是 4-7 步多步骤，**重点验证 Hook 闭环能跑**（AI 改代码 → hook 触发 → 失败 → AI 修 → 再触发）。

**Files:**
- Create: `e2e/recorded/account-opening-e2e-1/{raw.ts, network.json, dom-snapshot.html, meta.json}`
- Create: `e2e/recorded/account-opening-e2e-1/extracted-rules.md`
- Create: `e2e/specs/account-opening-e2e-1.spec.md`
- Create: `e2e/pages/AccountOpeningFlow.ts`（多步流程主 PO）
- Create: `e2e/pages/AccountOpeningSummaryPage.ts`
- Create: `e2e/tests/verify-on-old/account-opening-e2e-1.spec.ts`
- Create: `e2e/tests/new/account-opening-e2e-1.spec.ts`
- Create: `e2e/fixtures/account-opening-e2e-1.json`

- [ ] **Step 1: [Manual] QA 在老 UAT 录制完整路径**

```bash
npm run e2e:record -- account-opening-e2e-1 --target=$OLD_SANDBOX_URL
```

操作: 登录 → Account Opening → Non-F2F → Sole → 直接创建 → 走完 4-7 步（每步填字段、点 Next） → 提交 → Summary

> 录制时**别点 Save**，全程用 Next 推进，因为 Pilot 2 验的是 happy path。

- [ ] **Step 2: 调 spec-extractor + spec-merger**

> "spec-extractor 扫 Account Opening 多步流程相关代码"
> "spec-merger 合并 → specs/account-opening-e2e-1.spec.md"

- [ ] **Step 3: [Manual] 人工 review spec**

特别检查:
- 4-7 步每步是否标号清楚？
- subTabs 在哪几步出现？
- Summary 页关键断言是否覆盖（生成 account number）？

- [ ] **Step 4: 调 test-generator，要求生成 AccountOpeningFlow 多步 PO**

PO 必须包含:

```ts
// e2e/pages/AccountOpeningFlow.ts 骨架
export class AccountOpeningFlow {
  constructor(public page: Page, private mode: 'old' | 'new') {}

  // 底部按钮（PO 提供统一方法）
  btnExit = () => this.mode === 'new' ? this.page.getByTestId('ao-btn-exit') : this.page.getByRole('button', { name: /exit/i });
  btnSave = () => this.mode === 'new' ? this.page.getByTestId('ao-btn-save') : this.page.getByRole('button', { name: /save/i });
  btnBack = () => this.mode === 'new' ? this.page.getByTestId('ao-btn-back') : this.page.getByRole('button', { name: /back/i });
  btnNext = () => this.mode === 'new' ? this.page.getByTestId('ao-btn-next') : this.page.getByRole('button', { name: /next/i });

  async clickExit() { await this.btnExit().click(); }
  async clickSave() { await this.btnSave().click(); }   // 显式 Save，不前进
  async clickBack() { await this.btnBack().click(); }
  async clickNext() { await this.btnNext().click(); }   // Save + 前进

  // 单步操作（每一步一个方法）
  async fillStep1(data: { ... }) { ... }
  async fillStep2(data: { ... }) { ... }
  // ...
  async submit() { ... }    // 最终提交
  async expectOnSummary() { ... }

  // seedToStep: UI 走完前 N 步以快速跳到目标，给单步聚焦测试用
  async seedToStep(target: number, data: TestApplicationData) {
    if (target <= 1) return;
    await this.fillStep1(data);
    await this.clickNext();
    if (target <= 2) return;
    await this.fillStep2(data);
    await this.clickNext();
    // ...
  }
}
```

- [ ] **Step 5: TS 编译 + verify-on-old 跑通**

```bash
npx tsc --noEmit -p e2e/tsconfig.json
npm run e2e:verify -- tests/verify-on-old/account-opening-e2e-1.spec.ts
```

期望: PASS。

- [ ] **Step 6: 落 fixture**

把 `recorded/account-opening-e2e-1/network.json` 整理后写到 `e2e/fixtures/account-opening-e2e-1.json`。

- [ ] **Step 7: 写 focus.json**

```bash
# 推荐用 npm script（自动写入 mode=auto）
npm run e2e:focus account-opening-e2e-1

# 或手动写
cat > .opencode/focus.json <<'EOF'
{
  "cases": ["account-opening-e2e-1"],
  "mode": "auto",
  "started_at": "2026-05-04T09:00:00Z",
  "owner": "<your-name>"
}
EOF
```

- [ ] **Step 8: 测 Hook 闭环（关键验证）**

故意破坏新项目的某个 selector（如把 `data-testid="ao-btn-next"` 删掉），然后让 opencode 主 agent 处理这个 case。

期望流程:
```
主 agent 看到任务 → 改某行代码 → 准备结束 → completion hook 触发
hook → invoke test-runner subagent → 跑 e2e:case account-opening-e2e-1
test-runner 返回 JSON: { failures: [{ kind: "selector_missing", hypothesis: "..." }] }
主 agent 收到报告 → 加回 testid → 再次结束 → hook 再触发 → PASS → 真正完成
```

- [ ] **Step 9: D2 18:00 检查点**

如果 Step 8 hook 闭环跑不通：
- 检查 opencode hook 配置（hook-config.json 字段名是否正确）
- 如查 opencode 文档发现当前版本不支持原生 hook → **立刻切换到降级路径**:
  - 主 agent 工作流改为"显式调用 `bash e2e/scripts/run-case-with-fallback.sh <case-name>`"
  - 不再尝试 opencode 原生 hook，**避免阻塞 D3+**
  - 在 `.opencode/hook-config.json` 注释顶部加上 `"// FALLBACK MODE - opencode native hook not supported in current version"`

如果 Step 8 跑通：保持原生 hook 模式。

- [ ] **Step 10: commit Pilot 2 完成**

```bash
git add e2e/pages e2e/tests e2e/fixtures e2e/specs e2e/recorded .opencode
git commit -m "feat(e2e): pilot 2 done - account-opening-e2e-1 + hook loop verified"
```

---

## Task 10-21: 12 个剩余 case（D3-D7，每个 case 一个 task）

接下来 12 个 case 流水线一致（每个 7 步：录制 → 扫码 → 合并 → review → 生成 → verify → 接入新项目）。**步骤不重复列**，每个 task 只列差异化要点（录制目标 / 关键 selector / 断言点 / 特殊处理）。

每个 task 通用步骤模板:
1. `npm run e2e:record -- <case-name> --target=$OLD_SANDBOX_URL`，QA 录制
2. opencode 调 spec-extractor → spec-merger
3. 人工 review spec
4. opencode 调 test-generator
5. `npx tsc --noEmit -p e2e/tsconfig.json` 编译过
6. `npm run e2e:verify -- tests/verify-on-old/<case-name>.spec.ts` 跑通
7. 落 fixture 到 `e2e/fixtures/<case-name>.json`
8. `npm run e2e:focus <case-name>`（或手动写 `.opencode/focus.json`，含 `mode: "auto"`）
9. 跑新项目: `PW_BASE=$NEW_LOCAL_URL npm run e2e:case <case-name>`，进入 hook 循环让 AI 修
10. PASS → commit

每个 task 退出条件: case 在 `tests/verify-on-old/` PASS + 在新项目本地 PASS。

---

### Task 10: E2E-2 — 基于 customer 创建 / F2F / Joint（含 Prefill 验证）

**Case 难点**: 这是最复杂的 E2E，涉及 search 现有 customer、基于其创建、F2F 7 步流程、Joint 第二申请人字段、Prefill 验证。

**Files:**
- Create: `e2e/recorded/account-opening-e2e-2/{raw.ts, ...}`
- Create: `e2e/specs/account-opening-e2e-2.spec.md`
- Modify: `e2e/pages/AccountOpeningFlow.ts`（加 Joint 字段方法 + prefilled 检查方法）
- Create: `e2e/pages/AccountOpeningCustomerSearchPage.ts`（如果 Pilot 1/2 没建过）
- Create: `e2e/tests/verify-on-old/account-opening-e2e-2.spec.ts`
- Create: `e2e/tests/new/account-opening-e2e-2.spec.ts`
- Create: `e2e/fixtures/account-opening-e2e-2.json`

- [ ] **Step 1-7: 走通用流水线（参照 Task 10-21 头部模板）**

录制目标: 登录 → Account Opening → F2F → Joint → Search 1.3.1 → 输入 KNOWN_TEST_CUSTOMERS.default 的 customer number → search → 1.3.2.2 基于该 customer 新建 → 验证 Step 1 字段已 prefill → 走完 7 步 → Summary

- [ ] **Step 8: 关键断言（PO 必须支持）**

新增 `AccountOpeningFlow` 方法:
```ts
async expectFieldsPrefilledFromCustomer(expectedFields: string[]) {
  // from spec #N: 基于 customer 创建后，以下字段应有非空值
  for (const field of expectedFields) {
    const v = await this.page.getByTestId(`ao-field-${field}`).inputValue();
    expect(v).not.toBe('');
  }
}

async fillJointSecondApplicant(data: { ... }) {
  // Joint 模式特有的第二申请人字段
}
```

- [ ] **Step 9: 测试代码示例**

```ts
test('E2E-2: 基于 customer / F2F / Joint 完整开户', async ({ page, textEvidence }) => {
  const data = createTestData('e2e-2');
  const search = new AccountOpeningCustomerSearchPage(page, 'new');
  const flow = new AccountOpeningFlow(page, 'new');

  await login(page, newProjectAccount());
  await page.goto('/account-opening');
  await flow.selectF2F();          // 1.1
  await flow.selectJoint();        // 1.2
  await search.searchByNumber(KNOWN_TEST_CUSTOMERS.default);  // 1.3.1
  await search.createApplicationFromCustomer();               // 1.3.2.2
  await flow.expectFieldsPrefilledFromCustomer(['firstName', 'lastName', 'address']); // from spec #1
  await flow.fillStep1Joint(data);
  await flow.fillJointSecondApplicant({ ... });
  await flow.clickNext();
  // ... 7 步走完
  await flow.submit();
  await flow.expectOnSummary();
});
```

- [ ] **Step 10: commit**

```bash
git add e2e/pages e2e/tests e2e/fixtures e2e/specs e2e/recorded
git commit -m "feat(e2e): E2E-2 done - based on customer / F2F / Joint with prefill"
```

---

### Task 11: E2E-3 — Resume 草稿（从 Maker Dashboard）

**Case 难点**: 需要先 Save 一次留草稿，再从 Maker Dashboard 列表 Resume。

**Files:**
- Create: `e2e/recorded/account-opening-e2e-3/...`
- Create: `e2e/specs/account-opening-e2e-3.spec.md`
- Create: `e2e/pages/MakerDashboardPage.ts`
- Modify: `e2e/pages/AccountOpeningFlow.ts`（加 expectResumedAtStep 方法）
- Create: `e2e/tests/verify-on-old/account-opening-e2e-3.spec.ts`
- Create: `e2e/tests/new/account-opening-e2e-3.spec.ts`
- Create: `e2e/fixtures/account-opening-e2e-3.json`

- [ ] **Step 1-7: 通用流水线**

录制目标:
- Phase A: 登录 → Account Opening → Non-F2F → Sole → 直接创建 → 填 Step 1 → 点 Save → 退出（不要点 Next）→ 记下 application id（注意 URL 或 toast）
- Phase B: 切到 Maker Dashboard → 列表里找到该草稿 → 点击 → Resume 到 Step 1 → 字段保留 → 继续填完 → Submit → Summary

- [ ] **Step 8: PO 新方法**

```ts
// MakerDashboardPage.ts
export class MakerDashboardPage {
  constructor(public page: Page, private mode: 'old' | 'new') {}

  applicationRow = (id: string) => this.mode === 'new'
    ? this.page.getByTestId(`maker-app-row-${id}`)
    : this.page.locator(`tr[data-app-id="${id}"]`);

  async clickApplicationToResume(id: string) {
    await this.applicationRow(id).click();
  }
}

// AccountOpeningFlow.ts 追加
async expectResumedAtStep(step: number) {
  // from spec #N: Resume 后应在 Step <step>，且字段保留
  await expect(this.page).toHaveURL(new RegExp(`step=${step}`));
}
```

- [ ] **Step 9: 测试代码**

```ts
test('E2E-3: Save 草稿 → Maker Dashboard Resume → 完成', async ({ page, textEvidence }) => {
  const data = createTestData('e2e-3');
  const flow = new AccountOpeningFlow(page, 'new');
  const maker = new MakerDashboardPage(page, 'new');

  // Phase A: Save 草稿
  await login(page, newProjectAccount());
  await page.goto('/account-opening');
  await flow.selectNonF2F();
  await flow.selectSole();
  await flow.createNewDirect();
  await flow.fillStep1(data);
  await flow.clickSave();   // 显式 Save，不前进 // from spec #1
  const appId = await flow.getCurrentApplicationId();
  await flow.clickExit();

  // Phase B: Maker Dashboard Resume
  await page.goto('/maker-dashboard');
  await maker.clickApplicationToResume(appId);
  await flow.expectResumedAtStep(1);   // from spec #2
  await flow.fillStep2(data);
  await flow.clickNext();
  // ... 完成剩余步骤
  await flow.submit();
  await flow.expectOnSummary();
});
```

- [ ] **Step 10: commit**

```bash
git commit -m "feat(e2e): E2E-3 done - resume draft from Maker Dashboard"
```

---

### Task 12: S-2 — 1.2 账户类型选择（Sole / Joint 子流程切换）

**Case 难点**: 验证选择不同账户类型后流程结构差异。

**Files:**
- Create: `e2e/specs/account-opening-s-2.spec.md`
- Modify: `e2e/pages/AccountOpeningFlow.ts`
- Create: `e2e/tests/verify-on-old/account-opening-s-2.spec.ts`
- Create: `e2e/tests/new/account-opening-s-2.spec.ts`

- [ ] **Step 1-7: 通用流水线**

录制目标: 进入 Account Opening 1.1 → 选 Non-F2F → 1.2 页 → 分别测两种点击：(a) 选 Sole → 进入 Sole 第 1 步；(b) 选 Joint → 进入 Joint 第 1 步（含第二申请人入口）

- [ ] **Step 8: 测试代码（两条 test 一组）**

```ts
test('S-2a: 选 Sole 进入 Sole 流程', async ({ page }) => {
  const flow = new AccountOpeningFlow(page, 'new');
  await login(page, newProjectAccount());
  await page.goto('/account-opening');
  await flow.selectNonF2F();
  await flow.selectSole();
  await flow.expectOnStep(1, 'sole');  // from spec #1
  await expect(flow.btnSecondApplicantTab()).not.toBeVisible(); // from spec #2
});

test('S-2b: 选 Joint 进入 Joint 流程', async ({ page }) => {
  const flow = new AccountOpeningFlow(page, 'new');
  await login(page, newProjectAccount());
  await page.goto('/account-opening');
  await flow.selectNonF2F();
  await flow.selectJoint();
  await flow.expectOnStep(1, 'joint'); // from spec #3
  await expect(flow.btnSecondApplicantTab()).toBeVisible(); // from spec #4
});
```

- [ ] **Step 9: commit**

```bash
git commit -m "feat(e2e): S-2 done - account type Sole/Joint sub-flow"
```

---

### Task 13: S-3 — 1.3.1 customer search 输入

**Case 难点**: 验证 search 表单输入 + 行为正确性（带 customer number 进入列表）。

**Files:**
- Create: `e2e/specs/account-opening-s-3.spec.md`
- Modify: `e2e/pages/AccountOpeningCustomerSearchPage.ts`
- Create: `e2e/tests/verify-on-old/account-opening-s-3.spec.ts`
- Create: `e2e/tests/new/account-opening-s-3.spec.ts`

- [ ] **Step 1-7: 通用流水线**

录制目标: 1.1 选完 → 1.2 选完 → 1.3 选 Search → 输入 `KNOWN_TEST_CUSTOMERS.default` 的 customer number → 点 Search → 列表加载

- [ ] **Step 8: 测试代码**

```ts
test('S-3: customer search 输入合法 customer number 后跳列表', async ({ page }) => {
  const search = new AccountOpeningCustomerSearchPage(page, 'new');
  await login(page, newProjectAccount());
  await page.goto('/account-opening');
  // 假设 1.1/1.2 已选
  await flow.selectNonF2F();
  await flow.selectSole();
  await flow.gotoSearchEntry();
  await search.inputCustomerNumber(KNOWN_TEST_CUSTOMERS.default);
  await search.clickSearch();
  await search.expectOnApplicationsList();   // from spec #1
});
```

- [ ] **Step 9: commit**

```bash
git commit -m "feat(e2e): S-3 done - customer search by number"
```

---

### Task 14: S-4 — 1.3.1.2 customer applications 列表 → 点击 resume

**Case 难点**: 列表加载、行选择、跳转到草稿。

**Files:**
- Create: `e2e/specs/account-opening-s-4.spec.md`
- Modify: `e2e/pages/AccountOpeningCustomerSearchPage.ts`
- Create: `e2e/tests/verify-on-old/account-opening-s-4.spec.ts`
- Create: `e2e/tests/new/account-opening-s-4.spec.ts`

- [ ] **Step 1-7: 通用流水线**

录制目标: search by `KNOWN_TEST_CUSTOMERS.withApps` → 列表显示该 customer 的 applications（≥1 条预置草稿）→ 点击未完成的 → 跳到草稿对应步骤

> 该 case 依赖 `KNOWN_TEST_CUSTOMERS.withApps` 在沙箱已有至少 1 条草稿。Task 0 应已确认。

- [ ] **Step 8: 测试代码**

```ts
test('S-4: 列表点击未完成 application 跳到对应步骤 resume', async ({ page }) => {
  const search = new AccountOpeningCustomerSearchPage(page, 'new');
  await login(page, newProjectAccount());
  await page.goto('/account-opening');
  await flow.selectNonF2F();
  await flow.selectSole();
  await flow.gotoSearchEntry();
  await search.inputCustomerNumber(KNOWN_TEST_CUSTOMERS.withApps);
  await search.clickSearch();
  const rows = await search.applicationRowCount();
  expect(rows).toBeGreaterThan(0);   // from spec #1
  await search.clickFirstIncompleteApplication();
  await flow.expectResumedAtSomeStep();   // from spec #2: URL 含 step= 参数
});
```

- [ ] **Step 9: commit**

```bash
git commit -m "feat(e2e): S-4 done - applications list click to resume"
```

---

### Task 15: S-5 — 1.3.2.2 Prefill 验证（基于 customer 新建后字段预填）

**Case 难点**: 单独验证 Prefill 行为（区别于 E2E-2 的完整流程）。

**Files:**
- Create: `e2e/specs/account-opening-s-5.spec.md`
- Modify: `e2e/pages/AccountOpeningFlow.ts`
- Create: `e2e/tests/verify-on-old/account-opening-s-5.spec.ts`
- Create: `e2e/tests/new/account-opening-s-5.spec.ts`

- [ ] **Step 1-7: 通用流水线**

录制目标: search by `KNOWN_TEST_CUSTOMERS.default` → 基于此 customer 新建（不点列表里已有的，点"Create new"）→ 立即停在 Step 1，**不填任何东西**直接看哪些字段已 prefill

- [ ] **Step 8: 测试代码**

```ts
test('S-5: 基于 customer 新建后 Step 1 关键字段 prefill', async ({ page }) => {
  const flow = new AccountOpeningFlow(page, 'new');
  const search = new AccountOpeningCustomerSearchPage(page, 'new');

  await login(page, newProjectAccount());
  await page.goto('/account-opening');
  await flow.selectNonF2F();
  await flow.selectSole();
  await flow.gotoSearchEntry();
  await search.inputCustomerNumber(KNOWN_TEST_CUSTOMERS.default);
  await search.clickSearch();
  await search.createApplicationFromCustomer();

  // 不填字段，直接验证 prefill
  // 具体字段名以 spec 为准（D1 探索 + spec-merger 确定）
  await flow.expectFieldsPrefilledFromCustomer([
    'firstName', 'lastName', 'address', 'contactEmail',
  ]);  // from spec #1-4
});
```

- [ ] **Step 9: commit**

```bash
git commit -m "feat(e2e): S-5 done - prefill from customer verification"
```

---

### Task 16: S-6 — 中间步骤 subTabs 切换状态保留

**Case 难点**: 单步内 subTab 切换不丢字段。

**Files:**
- Create: `e2e/specs/account-opening-s-6.spec.md`
- Modify: `e2e/pages/AccountOpeningFlow.ts`（加 subTab 方法）
- Create: `e2e/tests/verify-on-old/account-opening-s-6.spec.ts`
- Create: `e2e/tests/new/account-opening-s-6.spec.ts`

- [ ] **Step 1-7: 通用流水线**

录制目标: 用 seedToStep 跳到一个有 subTabs 的步骤（D1 探索时确定哪一步） → 在 Tab A 填字段 → 切到 Tab B → 切回 Tab A → 验证字段还在 → 切到 Tab B 填字段 → 切回 → Tab B 字段还在

- [ ] **Step 8: PO 新方法**

```ts
// AccountOpeningFlow.ts 追加
subTab = (n: number) => this.mode === 'new'
  ? this.page.getByTestId(`ao-subtab-${n}`)
  : this.page.getByRole('tab').nth(n);

async clickSubTab(n: number) { await this.subTab(n).click(); }
```

- [ ] **Step 9: 测试代码**

```ts
test('S-6: subTabs 切换状态保留', async ({ page }) => {
  const flow = new AccountOpeningFlow(page, 'new');
  const data = createTestData('s-6');
  await login(page, newProjectAccount());
  await page.goto('/account-opening');
  // 假设有 subTabs 的是 Step 3（D1 探索确认）
  await flow.seedToStep(3, data);

  await flow.clickSubTab(0);
  await flow.fillSubTabField('aFieldOnTab0', 'value-A');
  await flow.clickSubTab(1);
  await flow.fillSubTabField('bFieldOnTab1', 'value-B');
  await flow.clickSubTab(0);
  await expect(flow.fieldByName('aFieldOnTab0')).toHaveValue('value-A'); // from spec #1
  await flow.clickSubTab(1);
  await expect(flow.fieldByName('bFieldOnTab1')).toHaveValue('value-B'); // from spec #2
});
```

- [ ] **Step 10: commit**

```bash
git commit -m "feat(e2e): S-6 done - subTabs state retention"
```

---

### Task 17: S-7 — 显式 Save vs Next 隐式 Save 区别

**Case 难点**: 明确区分两种 save 行为（业务约定 Next 自带 Save）。

**Files:**
- Create: `e2e/specs/account-opening-s-7.spec.md`
- Create: `e2e/tests/verify-on-old/account-opening-s-7.spec.ts`
- Create: `e2e/tests/new/account-opening-s-7.spec.ts`

- [ ] **Step 1-7: 通用流水线**

录制目标:
- Test a: 中间步骤填字段 → 点 Save → Exit → 列表里草稿存在且仍在当前步
- Test b: 中间步骤填字段 → 点 Next → 跳下一步 → Exit → 列表里草稿在下一步（说明 Next 隐式 saved）

- [ ] **Step 8: 测试代码**

```ts
test('S-7a: 显式 Save 不前进步骤', async ({ page }) => {
  const flow = new AccountOpeningFlow(page, 'new');
  const maker = new MakerDashboardPage(page, 'new');
  const data = createTestData('s-7a');

  await login(page, newProjectAccount());
  await page.goto('/account-opening');
  await flow.selectNonF2F();
  await flow.selectSole();
  await flow.createNewDirect();
  await flow.fillStep1(data);
  await flow.clickSave();          // from spec #1: Save 显式
  const appId = await flow.getCurrentApplicationId();
  await flow.clickExit();
  await flow.confirmExit();

  await page.goto('/maker-dashboard');
  await maker.clickApplicationToResume(appId);
  await flow.expectResumedAtStep(1);  // from spec #2: 仍在 Step 1
});

test('S-7b: 点 Next 自动 Save 并前进', async ({ page }) => {
  const flow = new AccountOpeningFlow(page, 'new');
  const maker = new MakerDashboardPage(page, 'new');
  const data = createTestData('s-7b');

  await login(page, newProjectAccount());
  await page.goto('/account-opening');
  await flow.selectNonF2F();
  await flow.selectSole();
  await flow.createNewDirect();
  await flow.fillStep1(data);
  await flow.clickNext();          // from spec #3: Next 隐式 save + 前进
  const appId = await flow.getCurrentApplicationId();
  await flow.clickExit();
  await flow.confirmExit();

  await page.goto('/maker-dashboard');
  await maker.clickApplicationToResume(appId);
  await flow.expectResumedAtStep(2);  // from spec #4: 在 Step 2
});
```

- [ ] **Step 9: commit**

```bash
git commit -m "feat(e2e): S-7 done - explicit Save vs Next implicit save"
```

---

### Task 18: S-8 — 底部 Back 按钮（返回上一步字段保留）

**Case 难点**: Back 不丢失上一步已填字段。

**Files:**
- Create: `e2e/specs/account-opening-s-8.spec.md`
- Create: `e2e/tests/verify-on-old/account-opening-s-8.spec.ts`
- Create: `e2e/tests/new/account-opening-s-8.spec.ts`

- [ ] **Step 1-7: 通用流水线**

录制目标: 填 Step 1 → Next 到 Step 2 → 填一些 Step 2 字段 → 点 Back 回 Step 1 → 验证 Step 1 字段还在 → 再点 Next 到 Step 2 → 验证 Step 2 字段也还在

- [ ] **Step 8: 测试代码**

```ts
test('S-8: Back 返回不丢字段', async ({ page }) => {
  const flow = new AccountOpeningFlow(page, 'new');
  const data = createTestData('s-8');

  await login(page, newProjectAccount());
  await page.goto('/account-opening');
  await flow.selectNonF2F();
  await flow.selectSole();
  await flow.createNewDirect();

  await flow.fillStep1(data);
  await flow.clickNext();
  await flow.fillStep2Partial(data);  // 填一部分

  await flow.clickBack();
  await flow.expectOnStep(1);
  await flow.expectStep1FieldsRetained(data);   // from spec #1

  await flow.clickNext();
  await flow.expectOnStep(2);
  await flow.expectStep2FieldsRetained(data);   // from spec #2
});
```

- [ ] **Step 9: commit**

```bash
git commit -m "feat(e2e): S-8 done - Back button preserves fields"
```

---

### Task 19: B-1 — Non-F2F vs F2F 步数差异

**Case 难点**: 选择不同入口后流程总步数不同（4 vs 7）。

**Files:**
- Create: `e2e/specs/account-opening-b-1.spec.md`
- Create: `e2e/tests/verify-on-old/account-opening-b-1.spec.ts`
- Create: `e2e/tests/new/account-opening-b-1.spec.ts`

- [ ] **Step 1-7: 通用流水线**

录制目标: 两种点击各录一次：
- a: 选 Non-F2F → 看 stepper 显示总步数（页面通常有"Step 1 / 4"或类似）
- b: 选 F2F → 看 stepper 显示总步数

- [ ] **Step 8: 测试代码**

```ts
test('B-1: Non-F2F 4 步流程', async ({ page }) => {
  const flow = new AccountOpeningFlow(page, 'new');
  await login(page, newProjectAccount());
  await page.goto('/account-opening');
  await flow.selectNonF2F();
  await flow.selectSole();
  await flow.createNewDirect();
  await expect(flow.stepperTotal()).toHaveText('4'); // from spec #1
});

test('B-1: F2F 7 步流程', async ({ page }) => {
  const flow = new AccountOpeningFlow(page, 'new');
  await login(page, newProjectAccount());
  await page.goto('/account-opening');
  await flow.selectF2F();
  await flow.selectSole();
  await flow.createNewDirect();
  await expect(flow.stepperTotal()).toHaveText('7'); // from spec #2
});
```

> 具体 4/7 数字以 D1 探索的真实步数为准。

- [ ] **Step 9: commit**

```bash
git commit -m "feat(e2e): B-1 done - Non-F2F vs F2F step count branching"
```

---

### Task 20: B-2 — Sole vs Joint 子流程差异

**Files:**
- Create: `e2e/specs/account-opening-b-2.spec.md`
- Create: `e2e/tests/verify-on-old/account-opening-b-2.spec.ts`
- Create: `e2e/tests/new/account-opening-b-2.spec.ts`

- [ ] **Step 1-7: 通用流水线**

录制目标: 1.1 任选 → 选 Sole → Step 1 看是否有"第二申请人"入口 (应无)；返回再选 Joint → Step 1 看是否有"第二申请人"入口 (应有)。

- [ ] **Step 8: 测试代码**

```ts
test('B-2: Sole 流程不含第二申请人', async ({ page }) => {
  const flow = new AccountOpeningFlow(page, 'new');
  await login(page, newProjectAccount());
  await page.goto('/account-opening');
  await flow.selectNonF2F();
  await flow.selectSole();
  await flow.createNewDirect();
  await expect(flow.btnSecondApplicantTab()).not.toBeVisible(); // from spec #1
});

test('B-2: Joint 流程含第二申请人入口', async ({ page }) => {
  const flow = new AccountOpeningFlow(page, 'new');
  await login(page, newProjectAccount());
  await page.goto('/account-opening');
  await flow.selectNonF2F();
  await flow.selectJoint();
  await flow.createNewDirect();
  await expect(flow.btnSecondApplicantTab()).toBeVisible(); // from spec #2
});
```

- [ ] **Step 9: commit**

```bash
git commit -m "feat(e2e): B-2 done - Sole vs Joint branching"
```

---

### Task 21: B-3 — search vs 新建分支

**Files:**
- Create: `e2e/specs/account-opening-b-3.spec.md`
- Create: `e2e/tests/verify-on-old/account-opening-b-3.spec.ts`
- Create: `e2e/tests/new/account-opening-b-3.spec.ts`

- [ ] **Step 1-7: 通用流水线**

录制目标:
- a: 1.3 选 Search → 进入 customer search 页（有 customer number 输入框）
- b: 1.3 选 New → 直接进入 Step 1

- [ ] **Step 8: 测试代码**

```ts
test('B-3: 1.3 选 Search 进入 customer search 页', async ({ page }) => {
  const flow = new AccountOpeningFlow(page, 'new');
  const search = new AccountOpeningCustomerSearchPage(page, 'new');
  await login(page, newProjectAccount());
  await page.goto('/account-opening');
  await flow.selectNonF2F();
  await flow.selectSole();
  await flow.gotoSearchEntry();
  await expect(search.inputCustomerNumberLocator()).toBeVisible(); // from spec #1
});

test('B-3: 1.3 选 New 直接进入 Step 1', async ({ page }) => {
  const flow = new AccountOpeningFlow(page, 'new');
  await login(page, newProjectAccount());
  await page.goto('/account-opening');
  await flow.selectNonF2F();
  await flow.selectSole();
  await flow.createNewDirect();
  await flow.expectOnStep(1);  // from spec #2
});
```

- [ ] **Step 9: commit**

```bash
git commit -m "feat(e2e): B-3 done - search vs new entry branching"
```

---

### Task 22: B-4 — Exit 退出确认

**Case 难点**: Exit 触发对话框 + 取消/确认两路。

**Files:**
- Create: `e2e/specs/account-opening-b-4.spec.md`
- Modify: `e2e/pages/AccountOpeningFlow.ts`（加 confirmExit / cancelExit）
- Create: `e2e/tests/verify-on-old/account-opening-b-4.spec.ts`
- Create: `e2e/tests/new/account-opening-b-4.spec.ts`

- [ ] **Step 1-7: 通用流水线**

录制目标:
- a: 中间步骤点 Exit → 弹确认对话框 → 点 Cancel → 留在原页
- b: 中间步骤点 Exit → 弹确认对话框 → 点 Confirm → 回 menu

- [ ] **Step 8: PO 方法**

```ts
// AccountOpeningFlow.ts
exitConfirmDialog = () => this.mode === 'new'
  ? this.page.getByTestId('ao-exit-confirm-dialog')
  : this.page.getByRole('dialog');

async cancelExit() {
  await this.exitConfirmDialog().getByRole('button', { name: /cancel/i }).click();
}
async confirmExit() {
  await this.exitConfirmDialog().getByRole('button', { name: /confirm|yes/i }).click();
}
```

- [ ] **Step 9: 测试代码**

```ts
test('B-4: Exit cancel 留在原页', async ({ page }) => {
  const flow = new AccountOpeningFlow(page, 'new');
  const data = createTestData('b-4-cancel');
  await login(page, newProjectAccount());
  await page.goto('/account-opening');
  await flow.selectNonF2F();
  await flow.selectSole();
  await flow.createNewDirect();
  await flow.fillStep1(data);
  await flow.clickExit();
  await expect(flow.exitConfirmDialog()).toBeVisible(); // from spec #1
  await flow.cancelExit();
  await flow.expectOnStep(1);                            // from spec #2
});

test('B-4: Exit confirm 离开页面', async ({ page }) => {
  const flow = new AccountOpeningFlow(page, 'new');
  await login(page, newProjectAccount());
  await page.goto('/account-opening');
  await flow.selectNonF2F();
  await flow.selectSole();
  await flow.createNewDirect();
  await flow.clickExit();
  await flow.confirmExit();
  await expect(page).not.toHaveURL(/account-opening\/.+\/step/);  // from spec #3
});
```

- [ ] **Step 10: commit**

```bash
git commit -m "feat(e2e): B-4 done - Exit confirmation dialog branches"
```

---

## Task 23: Flaky 整理 + 全量 smoke 跑稳定性（D8，半天）

**Files:**
- Modify: 各 case 测试文件加 `@flaky` 标签（如有需要）
- Create: `e2e/.flaky-log.md`

- [ ] **Step 1: 跑全量 smoke 三次**

```bash
PW_BASE=$NEW_LOCAL_URL npm run e2e:smoke
PW_BASE=$NEW_LOCAL_URL npm run e2e:smoke
PW_BASE=$NEW_LOCAL_URL npm run e2e:smoke
```

- [ ] **Step 2: 记录"通过率不稳"的 test**

汇总 3 次结果，把"3 次中失败 ≥1 次"的 test 列到 `e2e/.flaky-log.md`：
```markdown
# Flaky log

## 2026-05-12

- `S-4 列表点击未完成 application 跳到对应步骤 resume`: 3 次失败 1 次（疑似列表加载竞态）
- `E2E-3 Resume 草稿`: 3 次失败 1 次（疑似 Maker Dashboard 初次进入慢）
```

- [ ] **Step 3: 把 flaky test 加上 `@flaky` 标签**

修改对应 test 标题:
```ts
test('S-4 ... @flaky', async ({ page }) => { ... });
```

- [ ] **Step 4: 跑两次确认 main project 通过率到 100%**

```bash
PW_BASE=$NEW_LOCAL_URL npm run e2e:smoke
PW_BASE=$NEW_LOCAL_URL npm run e2e:smoke
```

期望: main project 全 PASS（flaky 项目独立跑，可能仍有失败但不阻塞）。

- [ ] **Step 5: 跑 SIT 一次确认 SIT 也能跑**

```bash
PW_BASE=$NEW_SIT_URL npm run e2e:smoke
```

期望: 主要 case PASS（SIT 与本地差异在于网络/部署，发现差异另立 task 修）。

- [ ] **Step 6: commit**

```bash
git add e2e/tests e2e/.flaky-log.md
git commit -m "chore(e2e): tag flaky tests after stability run"
```

---

## Task 24: 团队上手 SOP（D9，半天）

**Files:**
- Create: `e2e/README.md`
- Create: `e2e/docs/triage.md`

- [ ] **Step 1: `e2e/README.md`（≤2 页）**

````markdown
# E2E Tests for Account Opening (xx-sg-hbsp)

> 路径约定: 所有 `e2e/...` 与 `.opencode/...` 都在 `xx-sg-hbsp/` 下。命令在 `xx-sg-hbsp/` 目录跑。

## 快速开始

1. 复制 `e2e/.env.example` 为 `e2e/.env.local`，填值
2. `npm install`
3. 跑测试:
   - 全量: `npm run e2e:smoke`
   - 单 case: `npm run e2e:case account-opening-e2e-1`
   - debug headed: `npm run e2e:debug account-opening-e2e-1`

## 添加新 case

1. 录制: `npm run e2e:record -- <case-name>`
2. 调 opencode subagent: spec-extractor → spec-merger → test-generator
3. 跑 verify-on-old: `npm run e2e:verify -- tests/verify-on-old/<case-name>.spec.ts`
4. 跑 new: `npm run e2e:case <case-name>`

## Hook 闭环（自动模式）

主 agent 处理某 case 时:
```bash
npm run e2e:focus <case-name>     # 自动写 mode=auto
```
然后开始改代码。结束时 hook 自动跑测试。失败回喂主 agent 继续修，直到 PASS 或 max=5 次。

## 调试模式（避免 AI 自动改代码）⭐

当你想自己手动调试某 case，不希望 hook 触发主 agent 自动改业务代码时：

```bash
# 进入调试模式
npm run e2e:debug:on
# focus.json 的 mode 变成 "debug"
# hook 触发时 test-runner 立刻 return skipped，不跑测试也不修代码

# 自己手动跑、headed 调试
npm run e2e:debug -- account-opening-s-7
npx playwright show-trace test-results/.../trace.zip

# 改 PO 反复试，主 agent 不会动业务代码
# ...

# 调通后退出调试模式
npm run e2e:debug:off
# focus.json mode 回到 "auto"，hook 重新接管
```

更暴力的 kill switch（整个 session 完全跳过 hook）:
```bash
HOOK_DISABLED=1 opencode
```

什么时候用哪种:
| 场景 | 用 |
|---|---|
| 调试某 case，不想 AI 自动改业务 | `npm run e2e:debug:on` |
| 临时用 opencode 写别的代码 | `HOOK_DISABLED=1 opencode` |
| 主 agent 在做 case，正常循环 | mode='auto'（默认） |

## 失败排查

参见 `e2e/docs/triage.md`。
````

- [ ] **Step 2: `e2e/docs/triage.md`（失败分诊单页）**

```markdown
# 失败分诊指南

test-runner subagent 输出 JSON 中 `kind` 字段是 8 类之一。每类的处置：

| kind | 含义 | 你该做什么 |
|---|---|---|
| selector_missing | 元素找不到 | 新项目缺 testid → 加 |
| selector_changed | 元素变了 | 改 PO selector 实现，**测试代码不动** |
| assertion | 断言失败（值/URL/文本不符） | 业务逻辑没接通 → 修业务代码 |
| network | 接口 4xx/5xx/超时 | 后端联调 / API mock |
| timeout | 等待超时 | 加 wait / 检查异步竞态 |
| oracle_drift | 真实响应与 fixture 不符 | 重 verify-on-old，更新 oracle |
| infra | 浏览器崩溃 / 端口占用 | 重试一次，仍不行查环境 |
| flaky | 时通时不通 | 加 `@flaky` tag |

**断言失败永不自动重试**——它代表业务真错了。

**截图/视频/trace.zip 仅供人查看**，AI 看不懂。AI 用 dom_excerpt / console_logs / network_summary 做诊断。
```

- [ ] **Step 3: commit**

```bash
git add e2e/README.md e2e/docs
git commit -m "docs(e2e): add team SOP and triage guide"
```

---

## Task 25: 整体验收 + buffer（D10）

**Files:** N/A（验收任务）

- [ ] **Step 1: 完整 checklist 走一遍**

参照 spec 附录 C 核心交付清单：

工具链:
- [ ] e2e/ 项目骨架 ✓
- [ ] record-with-network 脚本 ✓
- [ ] test-data-factory + accounts ✓
- [ ] 4 个 subagent prompt ✓
- [ ] completion hook 配置（或降级 shell） ✓

测试用例:
- [ ] 3 个 E2E 全链路（Pilot 1 + E2E-1 / E2E-2 / E2E-3）✓
  - 注意: Pilot 1 = S-1，Pilot 2 = E2E-1
- [ ] 8 个单步聚焦（S-1 ~ S-8）✓
- [ ] 4 个分支跳转（B-1 ~ B-4）✓
- [ ] 全部 verify-on-old 通过 ✓
- [ ] fixtures/ 全部落地 ✓

文档:
- [ ] 团队上手 SOP（README.md） ✓
- [ ] 失败分诊单页（triage.md） ✓

- [ ] **Step 2: 跑一次完整 smoke + 一次 SIT**

```bash
PW_BASE=$NEW_LOCAL_URL npm run e2e:smoke
PW_BASE=$NEW_SIT_URL    npm run e2e:smoke
```

main project 应全绿。

- [ ] **Step 3: 写一段交付报告（在 cc-chat 里）**

```bash
cd <cc-chat-repo>
cat > docs/superpowers/plans/2026-05-13-delivery-report.md <<EOF
# E2E Sprint Delivery Report (2026-05-04 to 2026-05-13)

## 已交付
- 工具链: 全部齐全（含 hook 或 shell 降级）
- Cases: 15 个全部通过 verify-on-old 和新项目本地
- 文档: README + triage 各 1 份

## 决策记录
- D2 hook 验证结果: <原生 / 降级>
- 砍掉/推迟的 case: <无 / 哪几个>
- Flaky 数量: <N 个>，列在 .flaky-log.md

## 维护期 backlog
- Maker / Checker Dashboard 自身功能（搜索、筛选、详情、审核）
- 高金额审批等可能漏的边角分支
- Pairwise 表自动化工具

EOF
git add docs/superpowers/plans/2026-05-13-delivery-report.md
git commit -m "docs: e2e sprint delivery report"
```

- [ ] **Step 4: 用剩余时间处理 buffer 任务**

按优先级:
1. flaky 队列里能稳定下来的尝试修
2. 单步 case 里有遗漏的小行为补 1-2 个
3. PO 类 refactor（如有重复代码）
4. 提一个跨 sprint 的 follow-up issue 列表

---

## 自我审查（写完计划后）

### Spec 覆盖

| Spec 章节 | 实施任务 |
|---|---|
| §0 业务概览 | Task 7 探索 + 各 case task 都参考 |
| §1 决策约束 | Task 0 Step 1-2 全部对齐 |
| §2 整体架构 | Task 1-6 (脚手架 + agents + hook) |
| §3 单流程数据流 | Task 8 / 9 (Pilot 1/2) 完整演练 |
| §4.1 失败分类 8 类 | Task 24 triage.md + Task 5 test-runner prompt |
| §4.2 重试策略 | Task 1 playwright.config.ts |
| §4.3 数据隔离（无 KYC 简化版） | Task 3 test-data-factory.ts |
| §4.4 取证配置 + 文本证据 | Task 4 text-evidence fixture |
| §4.5 老沙箱 / oracle drift | 维护期 SOP（Task 24 triage） |
| §4.6 AI 失误兜底（含 text-only） | Task 5 test-runner prompt 硬约束 |
| §4.7 测试金字塔 3+8+4 | Task 8-22 全部 case |
| §5 阶段化交付 2 周 | Task 0-25 整体节奏 |
| §6 test-runner 输出契约 | Task 5 spec 内嵌契约 |
| §7 维护期 SOP | Task 24 + Task 25 报告 |
| 附录 A subagent prompts | Task 5 全部 4 个 |
| 附录 B package.json scripts | Task 2 |
| 附录 C 核心交付清单 | Task 25 验收 |
| 附录 D 配置项 | Task 0 全部填 |
| 附录 D.10 模型能力假设 | Task 5 test-runner.md 硬约束 |

### Type 一致性
- `mode: 'old' | 'new'` 在所有 PO 类一致
- `clickExit/Save/Back/Next` 命名贯穿（Task 9 + 17 + 18 + 22）
- `seedToStep(target, data)` 签名一致（Task 9 定义，Task 16 使用）
- `KNOWN_TEST_CUSTOMERS.default / .withApps` 在 Task 13 / 14 / 15 一致引用

### 占位符扫描
- `[TBD]` 仅在 Task 0 / 7 / 5（test-data-factory `[key: string]`）—— 这些是有意标的，因为依赖 D1 探索，**不是 plan 的盲区**
- 没有 "implement later" / "similar to Task N" 等占位

