---
title: "Vercel 发布 eve：开源 Agent 框架 + Vercel Connect，Agent 的 Next.js 时刻来了"
date: 2026-06-18
excerpt: "Vercel Ship 2026 一口气发布了 eve（开源 Agent 框架）、Vercel Connect（运行时凭证交换服务）以及完整的 Agent Stack。这意味着什么？怎么上手？"
tags: [Vercel, eve, Agent 框架, AI Agent, Vercel Connect, Agent Stack]
---

6 月 17 日，Vercel 在伦敦 Ship 2026 上发布了一系列重要更新，其中最引人注目的是 `eve` - 一个开源的 Agent 框架。同时亮相的还有 `Vercel Connect`（运行时凭证交换服务）和完整的 `Agent Stack`（AI SDK + AI Gateway + Workflow SDK + Vercel Sandbox + Chat SDK + Vercel Connect）。

这篇不是新闻综述。我会用代码逐层拆解 `eve` 怎么用、`Vercel Connect` 怎么集成、整套 Agent Stack 怎么落地，最后部署一个真实的 Agent 到生产环境。期望对大家有所帮助。

## Vercel 为什么做 Agent 框架

先说背景。Vercel 内部已经跑了上百个 Agent，其中最有名的是 `v0`（那个用自然语言生成 UI 的 coding agent）。但这些 Agent 有一个共同问题：每个团队都在重复造轮子。

- 每个 Agent 都需要持久化执行（会话可能跑几分钟甚至几天）
- 每个 Agent 都需要沙箱执行代码（模型写的代码不可信）
- 每个 Agent 都需要人工审批（有些操作不能自动放行）
- 每个 Agent 都需要对接多个消息渠道（Slack、Discord、Telegram......）
- 每个 Agent 都需要跟踪和评测

这些问题每个团队都遇到，每个团队都自己解决一遍。Vercel 的观察是：**Agent 有一个固定形状。**

就像 Next.js 把「路由」抽象成文件夹结构、让开发者不再操心路由配置一样，`eve` 把 Agent 的能力项抽象成文件目录。一个文件就是一个能力，文件系统就是 Agent 的定义界面。

这个类比很关键：Next.js 让 Web 开发不用再拼路由，`eve` 让 Agent 开发不用再拼管道。

## eve：文件系统即 Agent

### 初始化

```bash
npx eve@latest init my-agent
cd my-agent
npm run dev
```

三行命令启动一个能对话的 Agent。

### 目录结构

初始化的项目长这样：

```
my-agent/
├── package.json
└── agent/
    ├── agent.ts            # 模型和运行时配置
    ├── instructions.md     # 系统提示词（必选）
    ├── tools/              # 工具函数
    ├── skills/             # 领域知识
    ├── channels/           # 消息渠道
    ├── connections/        # 外部 MCP 服务
    ├── subagents/          # 子 Agent
    ├── schedules/          # 定时任务
    ├── sandbox/            # 沙箱配置
    └── hooks/              # 生命周期钩子
```

每个目录对应 Agent 的一个能力维度。读目录树就能知道这个 Agent 能做什么，以及它需要什么基础设施支持。

### 最小 Agent

**agent/agent.ts：**

```typescript
import { defineAgent } from "eve";
export default defineAgent({
  model: "anthropic/claude-sonnet-4.6",
});
```

**agent/instructions.md：**

```markdown
你是一个天气助手。如果用户问的是真实城市，直接返回模拟天气数据。
不要解释这是模拟数据，除非用户明确问。
```

**agent/tools/get_weather.ts：**

```typescript
import { defineTool } from "eve/tools";
import { z } from "zod";

export default defineTool({
  description: "返回某个城市的模拟天气数据",
  inputSchema: z.object({
    city: z.string().min(1).describe("城市名称"),
  }),
  async execute({ city }) {
    return {
      city,
      condition: "Sunny",
      temperatureF: 72,
      humidity: "45%",
    };
  },
});
```

这就跑起来了。`eve dev` 启动 TUI，你在终端里和 Agent 对话，每步推理、每次工具调用都实时展示，并且每步都是 checkpointed 的——会话崩溃后可以原地恢复。

### 增加审批控制

真实场景里，有些操作不能自动执行。`eve` 的审批模型是一个字段的事：

```typescript
// agent/tools/deploy_to_prod.ts
export default defineTool({
  description: "部署到生产环境",
  inputSchema: z.object({
    branch: z.string(),
  }),
  needsApproval: ({ toolInput }) => toolInput.branch === "main",
  async execute({ branch }) {
    // 只有 main 分支的部署需要审批
  },
});
```

`needsApproval` 可以是布尔值，也可以是接收 toolInput 的函数——这样就能根据上下文动态决定是否需要审批。Agent 会在需要审批的地方暂停，等人确认后才继续。

### Shadbox：让 Agent 自己写代码

有些问题没有现成的工具能解决。`eve` 的沙箱给了 Agent 一个真实的 Linux 环境：

```text
> 帮我把上周的销售数据按天画成柱状图

⦿ write_file analysis/chart_sales.py
⦿ bash
  python analysis/chart_sales.py
数据已处理，图表已保存到 analysis/daily_sales.png
```

沙箱运行在隔离的 security context 里，本地用 Docker，生产用 Vercel Sandbox。模型写的代码不会污染宿主环境。

### Subagent：委派任务

当 Agent 需要多步推理或专精能力时，可以委派给子 Agent：

```
agent/subagents/investigator/
├── agent.ts          # 可以选用不同的模型
├── instructions.md   # 独立的系统提示
└── tools/            # 独立的工具集
```

父 Agent 像调用普通工具一样调用子 Agent，但子 Agent 有独立的上下文窗口和工具集。

### 多渠道

Agent 默认自带 HTTP API。加一个渠道文件就能扩展到 Slack：

```typescript
// agent/channels/slack.ts
import { slackRoute } from "eve/channels/slack";

export default slackRoute({
  credentials: {
    botToken: process.env.SLACK_BOT_TOKEN!,
    signingSecret: process.env.SLACK_SIGNING_SECRET!,
  },
});
```

支持 Slack、Discord、Teams、Telegram、Twilio、GitHub、Linear，以及 `defineChannel` 自定义渠道。一个 Agent 跑在所有地方，逻辑代码不重复。

### Eval：像测试代码一样测试 Agent

```typescript
// evals/revenue.eval.ts
import { defineEval } from "eve/evals";
import { includes } from "eve/evals/expect";

export default defineEval({
  description: "分析师按团队规则回答收入问题",
  async test(t) {
    await t.send("上周收入是多少？");
    t.completed();
    t.calledTool("run_sql");
    t.check(t.reply, includes("净退款"));
  },
});
```

`eve eval` 跑起来后，每次修改 prompt 或模型都能看到分数变化。可以在 CI 里阻止降级合并。

## Vercel Connect：没有 token 的 Agent 集成

Agent 不接入外部系统就没有价值。但传统的做法是把 token 写在环境变量里——全局、永久、不区分用户。

`Vercel Connect` 解决这个问题的方式是把「凭证存储」变成「凭证请求」。

### 核心流程

```
Agent → OIDC 证明身份 → Vercel Connect 验证 → 返回短生命周期 token → Agent 执行操作
```

1. 在 Vercel 上注册一个 Connector（一次注册，多项目复用）
2. 每次需要操作时，Agent 通过 OIDC 证明自己的身份
3. Vercel Connect 验证身份和权限，返回一个 scope 受限的短生命期 token
4. Agent 用 token 执行操作，用完后废弃

```bash
# 创建 Slack connector
vercel connect create slack --name mybot

# 需要时吊销 token
vercel connect revoke-tokens slack/mybot --all-tokens
```

### eve + Vercel Connect 的集成

`@vercel/connect/eve` 适配器让 eve Agent 直接使用 Connect：

```typescript
// agent/connections/linear.ts
import { defineMcpClientConnection } from "eve/connections";
import { connect } from "@vercel/connect/eve";

export default defineMcpClientConnection({
  url: "https://mcp.linear.app/sse",
  auth: connect("linear/mybot"),
});
```

```typescript
// agent/channels/slack.ts
import { slackRoute } from "eve/channels/slack";
import { connectSlackCredentials } from "@vercel/connect/eve";

export default slackRoute({
  credentials: connectSlackCredentials("slack/mybot"),
});
```

环境变量里不再需要 `SLACK_BOT_TOKEN` 和 `SLACK_SIGNING_SECRET`。Agent 每次启动时通过 OIDC 向 Connect 请求凭证。

### 精确作用域

不只是替代 token——Connect 允许更细粒度的权限控制：

```typescript
const token = await getToken("github/mybot", {
  subject: { type: "app" },
  authorizationDetails: [
    {
      type: "github_app_installation",
      repositories: ["myorg/repo1"],  // 只访问一个仓库
      permissions: ["contents:read"], // 只读权限
    },
  ],
});
```

最小权限原则做到了请求级别：这次操作只需要这个仓库的读权限，那就只给这么多。

### 按用户授权

```typescript
const token = await getToken("linear/mybot", {
  subject: { type: "user", id: "user_123" },
});
```

Agent 可以以特定用户的身份操作，scope 受限于该用户的授权范围。审计日志也能追溯到具体用户。

## Agent Stack：全栈 Agent 基建

`eve` 是 Agent Stack 的「意见实现」。整个 Stack 从左到右解决 Agent 的三个核心需求：

| 层 | 产品 | 解决什么问题 |
|---|---|---|
| 模型接入 | AI SDK | 统一接口调用任意模型 |
| 模型路由 | AI Gateway | 单 endpoint 路由 + 自动故障转移 |
| 持久执行 | Workflow SDK | 每步 checkpoint，崩溃可恢复 |
| 安全执行 | Vercel Sandbox | 隔离微 VM，模型写的代码不会逃逸 |
| 凭证管理 | Vercel Connect | 运行时凭证交换，无存储的 token |
| 多渠道交付 | Chat SDK | 一次接入，分发到所有平台 |

`eve` 把所有这些预制件组装成一个目录结构。但你也可以单独使用每一层——比如只用 AI Gateway 做模型路由，或者只用 Workflow SDK 做后台 Job。

### AI SDK 示例

```typescript
import { generateText } from "ai";
const { text } = await generateText({
  model: "anthropic/claude-sonnet-4.6",
  prompt: "总结最近的部署",
});
```

### Vercel Sandbox 示例

```typescript
import { Sandbox } from "@vercel/sandbox";

const sandbox = await Sandbox.create({ runtime: "python3.13" });
await sandbox.writeFiles([
  { path: "agent.py", content: Buffer.from(code) },
]);
const result = await sandbox.runCommand("python", ["agent.py"]);
```

## 实战：一个完整的 eve + Vercel Connect 项目

我们从头搭建一个能查 GitHub Issue 并可以在 Slack 里对话的 Agent。

### 1. 初始化项目

```bash
npx eve@latest init support-agent
cd support-agent
```

### 2. 配置 Agent

```typescript
// agent/agent.ts
import { defineAgent } from "eve";
export default defineAgent({
  model: "anthropic/claude-sonnet-4.6",
  // 支持 provider 自动故障转移
  modelFallback: "openai/gpt-5.4",
});
```

```markdown
// agent/instructions.md
你是一个技术支持 Agent。
你的工作是帮助团队检索和追踪 GitHub Issue。

规则：
- 优先用工具查数据，不要猜。
- 返回结果时附上 Issue 编号和状态。
- 如果用户要求创建 Issue，先确认标题和描述后再操作。
```

### 3. 添加 GitHub 工具

```typescript
// agent/tools/list_issues.ts
import { defineTool } from "eve/tools";
import { z } from "zod";
import { getToken } from "@vercel/connect";

export default defineTool({
  description: "查询仓库的 GitHub Issue 列表",
  inputSchema: z.object({
    repo: z.string().describe("格式为 owner/repo"),
    state: z.enum(["open", "closed", "all"]).default("open"),
    limit: z.number().max(50).default(10),
  }),
  async execute({ repo, state, limit }) {
    const token = await getToken("github/support-bot", {
      subject: { type: "app" },
    });
    const res = await fetch(
      `https://api.github.com/repos/${repo}/issues?state=${state}&per_page=${limit}`,
      { headers: { Authorization: `Bearer ${token}` } },
    );
    return await res.json();
  },
});
```

```typescript
// agent/tools/create_issue.ts
import { defineTool } from "eve/tools";
import { z } from "zod";
import { getToken } from "@vercel/connect";

export default defineTool({
  description: "在指定仓库创建 GitHub Issue",
  inputSchema: z.object({
    repo: z.string(),
    title: z.string(),
    body: z.string().optional(),
  }),
  needsApproval: true, // 创建 Issue 需要人工审批
  async execute({ repo, title, body }) {
    const token = await getToken("github/support-bot", {
      subject: { type: "app" },
    });
    const res = await fetch(
      `https://api.github.com/repos/${repo}/issues`,
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${token}`,
          "Content-Type": "application/json",
        },
        body: JSON.stringify({ title, body }),
      },
    );
    return await res.json();
  },
});
```

### 4. 连接 Slack 渠道

```typescript
// agent/channels/slack.ts
import { slackRoute } from "eve/channels/slack";
import { connectSlackCredentials } from "@vercel/connect/eve";

export default slackRoute({
  credentials: connectSlackCredentials("slack/support-bot"),
});
```

### 5. 添加排期（每日汇总）

```typescript
// agent/schedules/daily-summary.ts
import { defineSchedule } from "eve/schedules";

export default defineSchedule({
  cron: "0 9 * * 1-5", // 工作日早上 9 点
  channel: "slack",
  async run(agent) {
    await agent.send(
      "查看团队仓库的 open Issue，按紧急程度列出今天的首要任务",
    );
  },
});
```

### 6. 部署

```bash
vercel deploy
```

好的，已经生产了。环境变量里没有任何第三方服务的 token，Agent 通过 OIDC + Vercel Connect 在运行时获得凭证。

## eve 对 Agent 生态的意义

回到开头那句话：Agent 有一个固定形状。`eve` 是 Vercel 把这个形状做成了框架。

把 `eve` 和 Next.js 放在一起看：

| | Next.js | eve |
|---|---|---|
| 核心理念 | 文件系统即路由 | 文件系统即 Agent |
| 免写的样板 | 路由配置、SSR/SSG 决策 | Agent 循环、持久化、沙箱、审批 |
| 扩展方式 | 文件 + 目录 | 文件 + 目录 |
| 学习路径 | 从 page.js 开始，逐步加 API、middleware | 从 instructions.md 开始，逐步加 tool、channel |
| 部署模型 | Vercel Platform | Vercel Platform |

这种类比不是巧合。Vercel 的底层逻辑一直是「开发者不需要关心他们不在乎的东西」。`eve` 是这个逻辑在 Agent 时代的延续。

当然，`eve` 还在 Public Beta。框架 API 可能还会变，支持的外部服务还在扩充，Vercel Connect 的 Trigger Forwarding 也还在 Beta。但方向是清楚的：Agent 开发正在从「手工拼管道」走向「声明式框架」。

如果你的 Agent 目前还是一个 Python 脚本 + 硬编码 API key + `while True` 循环，现在可能是动手试 `eve` 的好时机。

---

- 仓库：[github.com/vercel/eve](https://github.com/vercel/eve)
- 文档：[eve.dev/docs](https://eve.dev/docs)
- Vercel Connect：[vercel.com/connect](https://vercel.com/connect)
- Agent Stack：[vercel.com/blog/agent-stack](https://vercel.com/blog/agent-stack)
