---
name: relay
description: |
  The ONLY way to call Claude Code. Use this skill whenever the user wants to
  ask, delegate to, or get a second opinion from Claude. Do NOT invoke the
  claude CLI directly — always use this skill's relay call command.
  Triggers on "ask claude", "have claude", "send to claude", "get claude to",
  "delegate to claude", "second opinion", "relay". Invoke with /relay.
---

# Relay

Call Claude Code like a function. One command: generates the request, invokes Claude, prints the response.

```
relay call --name <slug> [--effort <level>] [--body-only] <<'BODY'
task
BODY
```

The script auto-detects caller/peer from its install path. Always invoke via its absolute path as shown in the examples below.

**All Claude interactions go through `relay call`.** Do not invoke the `claude` CLI directly, do not pass model flags (`--model`), and do not use `--dangerously-skip-permissions` yourself. The model and invocation method are hardcoded in the script.

## One-Shot Call

```bash
~/.codex/skills/relay/scripts/relay call --name auth-review <<'BODY'
Review src/auth.py for security issues. Run pytest to verify.
BODY
```

## Session Call (Multi-Turn)

Use sessions when continuing a prior exchange or planning multiple related calls.

```bash
~/.codex/skills/relay/scripts/relay call --session auth-refactor <<'BODY'
Fix the issues from my review. Run pytest to verify.
BODY
```

Rule of thumb: if the user says "continue", "follow up", or references a prior Claude exchange, use `--session`. Otherwise, use `--name`.

## Effort Levels

Choose `--effort` based on the task:

| Level | When to use |
|-------|-------------|
| `none` | No thinking needed: reformat, extract fields, find-and-replace |
| `low` | Light thinking: triage, classify, apply a well-defined migration |
| `medium` | **Default.** Code review, writing tests, fixing bugs |
| `high` | Deeper reasoning: security audit, complex refactoring |
| `xhigh` | Avoid unless necessary. Multi-file architectural redesign |

The `--effort` flag controls Codex's reasoning effort directly. When calling Claude (the peer direction), effort is recorded in the request metadata but has no effect on Claude's behavior — Claude does not expose a reasoning effort parameter.

Before raising effort, improve the prompt first — add output contracts, verification steps, completeness criteria.

## Prompting Claude

Be clear and direct. Use XML tags to separate concerns. Key patterns:

- `<context>` — background and motivation (why, not just what)
- `<instructions>` — numbered steps for multi-part tasks
- `<example>` — example output if format matters

Don't over-prompt — Claude Opus is proactive; avoid excessive MUSTs/NEVERs.

See `~/.codex/skills/relay/references/prompting-claude.md` for the full guide.

**Example:**

```bash
~/.codex/skills/relay/scripts/relay call --name auth-hardening <<'BODY'
<context>
We're hardening auth before a security audit. The auth module has had
significant changes in the last 6 months.
</context>

<instructions>
Review src/auth.py for OWASP Top 10 vulnerabilities, focusing on injection
and broken access control.

1. Read src/auth.py and identify all vulnerabilities
2. Fix each one in-place
3. Run pytest to verify all tests pass
4. Return a summary: one line per fix, with line number and what changed
</instructions>
BODY
```

## Output

The script prints the response file content to stdout. The response has YAML frontmatter followed by free-form markdown:

- **Frontmatter**: `relay`, `re`, `from`, `to`, `status` (`done` | `error`), `verify` (`pass` | `fail` | `skip`)
- **Body**: findings, changes, reasoning — free-form markdown below the frontmatter fence

Use `--body-only` to strip the frontmatter and get just the markdown body.

Request and response files are saved in `.relay/` (auto-gitignored). Peer stderr is logged to a `.log` sidecar file alongside the request.

If the response file is missing, the script reports failure. Do not blindly retry — inspect the request, response path, and `.log` sidecar first. If the transport failed before the peer started, a retry is safe. If the task already executed, prefer continuing in the same session after diagnosis.

## Async / Parallel

Codex supports concurrency via native parallel tool calls and subagents, but **not** via shell backgrounding (`&`/`disown`/`nohup`). Background child processes do not survive after the shell command returns in Codex's sandbox.

**Do not use `--bg`** — the script rejects it when called from Codex.

Recommended pattern when you have independent work alongside a relay call:

1. Start any independent local work in parallel tool calls.
2. Spawn a Codex subagent whose only job is to run the blocking relay call.
3. Continue local work in the main agent.
4. Wait for the relay subagent only when you need Claude's answer.

**Rule: Never serialize independent work. Use subagents to run relay calls concurrently with local work.**

## Prism / Parallax

When Relay is used as the Parallax transport inside Prism, the relay call receives the **same full question and same context** as every local reviewer — only the lens (weighing posture) differs. Do not narrow the prompt for the Parallax agent.

For Codex Prism runs, spawn a Codex subagent whose only job is to run the blocking relay call. Launch it concurrently with the local reviewer subagents via parallel tool calls.

## Timeout

For long-running tasks, use the script's `--timeout <seconds>` flag to limit the peer invocation (requires coreutils `timeout` or `gtimeout`).

## Utility Commands

`relay list` shows all protocol files:

```bash
~/.codex/skills/relay/scripts/relay list
~/.codex/skills/relay/scripts/relay list --session auth-refactor
```

`relay --help` and `relay --version` print usage and version info.

## Low-Level Commands

For manual orchestration, use `req` and `res` to generate files without invoking the peer. If auto-detection fails, pass `--from codex --to claude` explicitly to `call`.

```bash
# Generate request only (does not invoke claude)
REQ=$(~/.codex/skills/relay/scripts/relay req --from codex --to claude --name slug "task body")

# Read response
# Response path: ${REQ%.req.md}.res.md
```

Do not invoke the `claude` CLI directly — always use `relay call` for the full round-trip.
