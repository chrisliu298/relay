# Relay

**一个让 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 和 [Codex CLI](https://github.com/openai/codex) 进行跨模型协作的 skill。**

[English](README.md) | 中文

Relay 让一个 agent 像调用函数一样调用另一个 agent。写任务、调用对端、读结果。极简协议、自然语言通信、完全可审计。

```bash
relay call --name <slug> --effort <level> <<'BODY'
task
BODY
```

由 Claude Code 和 Codex 共同开发。

## 目录

- [为什么要用 Relay](#为什么要用-relay)
- [设计哲学](#设计哲学)
- [由 Agent 打造，也为 Agent 打造](#由-agent-打造也为-agent-打造)
- [工作流程](#工作流程)
- [安装](#安装)
- [使用方式](#使用方式)
- [接口](#接口)
- [异步 / 并行](#异步--并行)
- [安全](#安全)
- [仓库结构](#仓库结构)
- [贡献者](#贡献者)

---

## 为什么要用 Relay

使用单一 agent 时，你只能得到单一模型的视角。Relay 让你组合两个模型的能力：

- **任务委派：** 一个 agent 把任务交给另一个执行
- **第二意见：** 交叉审查，降低同模型盲区
- **跨模型流水线：** 一个实现，一个验证

### 为什么不直接用 subagent

Subagent 生成的是同一模型的副本。Relay 调用不同模型 — 不同训练数据、不同推理模式、不同盲区。跨模型审查比同模型审查发现更多问题。

---

## 设计哲学

Relay 融合了 Anthropic 与 OpenAI 在 agent 设计上的实践经验，并将其压缩为一个极简协议。

- **协议淡出，任务凸显。** Frontmatter 负责路由；任务正文保持自然语言。[^1]
- **请求自包含，上下文引用优先。** 请求包含任务与响应模板；上下文以文件引用为主，不粘贴大段内容。[^2]
- **验证是一等信号。** 响应在 frontmatter 中携带 `verify: pass | fail | skip`；验证命令和证据放在正文。[^3]
- **引导而非强制。** Relay 推荐正文结构，但不施加僵硬 schema。[^4]

这些选择减少格式错误，把协议规则集中在请求文件中，并让调用方无需解析正文即可按验证结果分支。

---

## 由 Agent 打造，也为 Agent 打造

Relay 是 Claude Code 与 Codex 通过 Relay 协议本身协作构建的：双方分别研究各自生态原则，在多轮 relay 往返中讨论取舍，交叉审查彼此改动，并端到端验证最终结果。

这个 skill 也被设计为可修改。`SKILL.md` 是纯 markdown，团队可以按工作流快速调整：

- 调整正文模式以匹配团队约定
- 增加领域特定的验证命令
- 修改响应 footer 模板以适配不同输出格式
- 如果接入其他 agent，替换对端名称

Relay 刻意保持简洁：没有锁死的 schema，只有一个 agent 与人类都能读懂、可扩展的协议。

---

## 工作流程

```mermaid
sequenceDiagram
    participant I as 发起方
    participant R as 接收方

    I->>I: 1. relay call --name ... <<'BODY'
    Note left of I: 生成 .relay/{id}.req.md
    Note left of I: 调用对端 agent
    R->>R: 2. 读取请求
    R->>R: 3. 执行任务
    R->>R: 4. 运行验证（如果有）
    R->>I: 5. 写入 .relay/{id}.res.md
    I->>I: 6. 输出响应内容
```

`call` 子命令封装了完整的往返流程：生成请求文件、调用对端 agent、将响应内容输出到 stdout。脚本从安装路径自动检测调用方和对端。

---

## 安装

### 快速安装（npx）

```bash
npx skills add chrisliu298/relay
```

使用 [skills CLI](https://github.com/vercel-labs/skills) 为所有支持的 agent（Claude Code、Codex）安装 skill。

### 手动安装（curl）

每个 skill 自带 `scripts/relay` 生成脚本，无需共享二进制文件。

**Claude Code skill：**

```bash
mkdir -p ~/.claude/skills/relay/scripts ~/.claude/skills/relay/references
curl -sL https://raw.githubusercontent.com/chrisliu298/relay/main/claude/skills/relay/SKILL.md \
  -o ~/.claude/skills/relay/SKILL.md
curl -sL https://raw.githubusercontent.com/chrisliu298/relay/main/claude/skills/relay/scripts/relay \
  -o ~/.claude/skills/relay/scripts/relay && chmod +x ~/.claude/skills/relay/scripts/relay
curl -sL https://raw.githubusercontent.com/chrisliu298/relay/main/claude/skills/relay/references/prompting-codex.md \
  -o ~/.claude/skills/relay/references/prompting-codex.md
```

**Codex CLI skill：**

```bash
mkdir -p ~/.codex/skills/relay/scripts ~/.codex/skills/relay/references
curl -sL https://raw.githubusercontent.com/chrisliu298/relay/main/codex/skills/relay/SKILL.md \
  -o ~/.codex/skills/relay/SKILL.md
curl -sL https://raw.githubusercontent.com/chrisliu298/relay/main/codex/skills/relay/scripts/relay \
  -o ~/.codex/skills/relay/scripts/relay && chmod +x ~/.codex/skills/relay/scripts/relay
curl -sL https://raw.githubusercontent.com/chrisliu298/relay/main/codex/skills/relay/references/prompting-claude.md \
  -o ~/.codex/skills/relay/references/prompting-claude.md
```

**重要：** 两个 skill 必须一起安装并同步更新，且保持同一 Relay 版本。请求/响应格式必须匹配；版本不一致会导致任一侧解析失败。

---

## 使用方式

直接自然语言触发委派：

> "让 Codex 审一下 `src/auth.py` 的中间件"

> "把这个实现发给 Claude 做第二意见"

也可以直接输入 `/relay`。

---

## 接口

### 模型

每个方向指定了最优模型，**请勿**自行替换，否则可能导致调用失败。

| 方向 | 模型参数 | 推理力度 | 说明 |
|---|---|---|---|
| Claude Code → Codex | `--model gpt-5.4` | 动态（`none`–`xhigh`） | Claude 根据任务选择推理力度 |
| Codex → Claude Code | `--model opus` | 不适用 | Claude CLI 无推理力度参数 |

### 单次调用

一条命令完成完整往返：生成请求、调用对端、输出响应。

**Claude Code → Codex：**

```bash
~/.claude/skills/relay/scripts/relay call --name auth-review --effort medium <<'BODY'
检查 src/auth.py 的安全问题。运行 pytest 验证。
BODY
```

**Codex → Claude Code：**

```bash
~/.codex/skills/relay/scripts/relay call --name auth-review <<'BODY'
检查 src/auth.py 的安全问题。运行 pytest 验证。
BODY
```

`--name` 提供可读的短名称；脚本自动添加时间戳前缀。`--effort` 控制 Codex 的推理力度（默认 `medium`，调用 Claude 时忽略）。

生成的请求文件 `.relay/20260219-1630-auth-review.req.md`：

```markdown
---
relay: 4
id: 20260219-1630-auth-review
from: claude
to: codex
effort: medium
---

检查 src/auth.py 的安全问题。运行 pytest 验证。

---
Reply: .relay/20260219-1630-auth-review.res.md
Format:
  ---
  relay: 4
  re: 20260219-1630-auth-review
  from: codex
  to: claude
  status: done | error
  verify: pass | fail | skip
  ---
  {your response}
```

### 会话调用

会话保留完整轮次历史，接收方读取所有先前交换作为上下文：

```text
.relay/
  auth-refactor/         # 会话目录
    01.req.md            # 第 1 轮请求
    01.res.md            # 第 1 轮响应
    02.req.md            # 第 2 轮请求（可简短 — 上下文在先前轮次中）
    02.res.md            # 第 2 轮响应
```

**Claude Code → Codex：**

```bash
~/.claude/skills/relay/scripts/relay call --session auth-refactor --effort medium <<'BODY'
修复问题并添加测试。运行 pytest 验证。
BODY
```

**Codex → Claude Code：**

```bash
~/.codex/skills/relay/scripts/relay call --session auth-refactor <<'BODY'
修复问题并添加测试。运行 pytest 验证。
BODY
```

会话名必须是 slug（`[a-z0-9-]+`）。会话按顺序执行 — 同一时间只有一个写入者。

### 输出

`call` 子命令将响应文件内容输出到 stdout。响应文件（单次或会话）：

```markdown
---
relay: 4
re: 20260219-1630-auth-review
from: codex
to: claude
status: done
verify: pass
---

发现 src/auth.py 中 2 个问题：
1. 第 45 行会话令牌未验证 — 已添加 hmac 校验
2. 第 52 行缺少输入净化 — 已改用参数化查询

修改后 12 个测试全部通过。
```

- **status**：`done` | `error`
- **verify**：`pass` | `fail` | `skip`
- **body**：发现、变更、推理过程 — 自由格式 markdown

若调用后响应文件不存在，对端失败或超时。

### 底层命令

如需自定义工作流或手动编排，可直接使用 `req` 和 `res` 子命令。正文可通过参数或 stdin 管道传入。

```bash
# 仅生成请求
REQ=$(~/.claude/skills/relay/scripts/relay req --from claude --to codex --name slug "task body")

# 手动调用对端
codex exec --model gpt-5.4 -c 'model_reasoning_effort="medium"' --full-auto "Read and execute $REQ"

# 响应文件路径
echo "${REQ%.req.md}.res.md"
```

---

## 异步 / 并行

默认情况下，`relay call` 会阻塞直到对端完成。当你有独立的工作需要与 relay 调用并行执行时，使用平台原生的并发机制。

### Claude Code

Claude Code 支持 Bash 工具的 `run_in_background: true` 和脚本的 `--bg` 标志：

```bash
# 方式 1：Bash 工具的 run_in_background（agent 原生）
# relay 调用在后台运行，subagent 同时执行其他工作

# 方式 2：--bg 标志（脚本原生）
# 在后台 fork 对端调用，立即返回响应文件路径
RES=$(~/.claude/skills/relay/scripts/relay call --bg --name auth-review --effort medium <<'BODY'
检查 src/auth.py 的安全问题。
BODY
)
# RES 是预期的响应文件路径 — 轮询：[ -f "$RES" ] && cat "$RES"
```

### Codex

Codex 通过原生并行工具调用和子 agent 支持并发，但**不支持** shell 后台化（`&`/`disown`/`nohup` — 子进程在 shell 命令返回后不会存活）。

**不要从 Codex 使用 `--bg`。** 改为生成一个 Codex 子 agent 来运行阻塞的 relay 调用，同时主 agent 继续本地工作：

1. 通过并行工具调用启动独立的本地工作。
2. 生成一个子 agent，其唯一任务是运行 relay 调用。
3. 主 agent 继续本地工作。
4. 仅在需要结果时等待 relay 子 agent。

---

## 安全

- `.relay/` 已加入 `.gitignore` — 脚本自动处理
- **Codex** 默认 `--full-auto`（`workspace-write` 沙盒）
- **Claude** 非交互模式使用 `--dangerously-skip-permissions` — 仅限可信仓库
- 清理：`rm .relay/*.md`（单次）或 `rm -rf .relay/{session}/`（会话）

---

## 仓库结构

```text
relay/
├── claude/skills/relay/
│   ├── SKILL.md
│   ├── references/
│   │   └── prompting-codex.md   # 如何有效提示 Codex
│   └── scripts/relay            # 请求/响应生成脚本
└── codex/skills/relay/
    ├── SKILL.md
    ├── references/
    │   └── prompting-claude.md  # 如何有效提示 Claude
    └── scripts/relay            # 相同副本
```

---

## 贡献者

- [@chrisliu298](https://github.com/chrisliu298)
- **Claude Code** — 协议设计
- **Codex** — 执行契约与 CLI 集成

[^1]: Anthropic — [Building effective agents](https://www.anthropic.com/research/building-effective-agents)、[Writing tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents)；OpenAI — [A practical guide to building agents](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/)、[Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/)
[^2]: Anthropic — [Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)；OpenAI — [Conversation state](https://developers.openai.com/api/docs/guides/conversation-state)、[Compaction](https://developers.openai.com/api/docs/guides/compaction)
[^3]: Anthropic — [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)；OpenAI — [Agent evals](https://developers.openai.com/api/docs/guides/agent-evals)
[^4]: Anthropic — [Building effective agents](https://www.anthropic.com/research/building-effective-agents)、[Writing tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents)；OpenAI — [Function calling](https://developers.openai.com/api/docs/guides/function-calling)
