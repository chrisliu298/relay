---
name: relay
description: |
  Call Claude Code from Codex and return a structured result. Triggers on
  "ask claude", "have claude", "send to claude", "get claude to",
  "delegate to claude", "second opinion", "relay". Invoke with /relay.
user_invocable: true
---

# Relay

Call Claude Code like a function. One command: generates the request, invokes Claude, prints the response.

```
relay call --name <slug> <<'BODY'
task
BODY
```

The script auto-detects caller/peer from its install path. Always invoke via its absolute path as shown in the examples below.

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

## Effort Level

The `--effort` flag is accepted but currently has no effect when calling Claude (Claude does not expose a reasoning effort parameter). The script defaults to `medium`. If a future Claude release supports effort tuning, the flag will be wired through automatically.

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

The script prints the response file content to stdout. The response includes YAML frontmatter:

- **status**: `done` | `error`
- **verify**: `pass` | `fail` | `skip`
- **body**: findings, changes, reasoning — free-form markdown

Request and response files are saved in `.relay/` (auto-gitignored).

If the response file is missing, the script reports failure — do not retry.

## Async / Parallel

Codex supports concurrency via native parallel tool calls and subagents, but **not** via shell backgrounding (`&`/`disown`/`nohup`). Background child processes do not survive after the shell command returns in Codex's sandbox.

**Do not use `--bg`** — it relies on shell backgrounding which is unreliable from Codex.

Recommended pattern when you have independent work alongside a relay call:

1. Start any independent local work in parallel tool calls.
2. Spawn a Codex subagent whose only job is to run the blocking relay call.
3. Continue local work in the main agent.
4. Wait for the relay subagent only when you need Claude's answer.

**Rule: Never serialize independent work. Use subagents to run relay calls concurrently with local work.**

## Low-Level Commands

For custom workflows or manual orchestration, use `req` and `res` directly. Body can be passed as an argument or piped via stdin. If auto-detection fails, pass `--from codex --to claude` explicitly to `call`.

```bash
# Generate request only
REQ=$(~/.codex/skills/relay/scripts/relay req --from codex --to claude --name slug "task body")

# Then invoke claude manually (--dangerously-skip-permissions is required
# because Claude runs non-interactively and cannot prompt for approval)
env -u CLAUDECODE claude --model opus -p --dangerously-skip-permissions "Read and execute $REQ"

# Read response
# Response path: ${REQ%.req.md}.res.md
```
