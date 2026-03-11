---
name: relay
description: |
  The ONLY way to call Codex. Use this skill whenever the user wants to
  ask, delegate to, or get a second opinion from Codex. Do NOT spawn agents
  to run the codex CLI directly — always use this skill's relay call command.
  Triggers on "ask codex", "have codex", "send to codex", "get codex to",
  "delegate to codex", "second opinion", "relay". Invoke with /relay.
allowed-tools: Read, Write, Bash(~/.claude/skills/relay/scripts/relay:*), Bash(find:*), Bash(printf:*)
user-invocable: true
---

# Relay

Call Codex like a function. One command: generates the request, invokes Codex, prints the response.

```
relay call --name <slug> [--effort <level>] [--bg] [--timeout <sec>] [--body-only] <<'BODY'
task
BODY
```

The script auto-detects caller/peer from its install path. Use `scripts/relay` inside this skill directory.

**All Codex interactions go through `relay call`.** Do not invoke `codex exec` directly, do not spawn agents to run the codex CLI, and do not pass model flags (`-m`, `--model`). The model and invocation method are hardcoded in the script.

## One-Shot Call

```bash
~/.claude/skills/relay/scripts/relay call --name auth-review --effort medium <<'BODY'
Review src/auth.py for security issues. Run pytest to verify.
BODY
```

## Session Call (Multi-Turn)

Use sessions when continuing a prior exchange or planning multiple related calls.

```bash
~/.claude/skills/relay/scripts/relay call --session auth-refactor --effort medium <<'BODY'
Fix the issues from my review. Run pytest to verify.
BODY
```

Rule of thumb: if the user says "continue", "follow up", or references a prior Codex exchange, use `--session`. Otherwise, use `--name`.

## Effort Levels

Choose `--effort` based on the task:

| Level | When to use |
|-------|-------------|
| `none` | No thinking needed: reformat, extract fields, find-and-replace |
| `low` | Light thinking: triage, classify, apply a well-defined migration |
| `medium` | **Default.** Code review, writing tests, fixing bugs |
| `high` | Deeper reasoning: security audit, complex refactoring |
| `xhigh` | Avoid unless necessary. Multi-file architectural redesign |

Before raising effort, improve the prompt first — add output contracts, verification steps, completeness criteria.

## Prompting Codex

Use XML tags for structure. Key patterns:

- `<output_contract>` — exact format and structure expected
- `<completeness_contract>` — what "done" means explicitly
- `<verification_loop>` — check correctness before finalizing

See `~/.claude/skills/relay/references/prompting-codex.md` for the full guide.

**Example:**

```bash
~/.claude/skills/relay/scripts/relay call --name pool-refactor --effort medium <<'BODY'
Refactor src/db/pool.py to add connection timeouts.

1. Add timeout_seconds param to ConnectionPool.__init__
2. Implement auto-reconnection for stale connections
3. Add reclaim_stale() method
4. Keep backward compatibility

<output_contract>
Summary of changes, one per line, with file path and description.
</output_contract>

<verification_loop>
Run pytest tests/test_pool.py — all tests must pass.
</verification_loop>

<completeness_contract>
Done means: all 4 requirements implemented, tests pass, no new lint errors.
</completeness_contract>
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

When you have independent subagent work alongside a relay call, **never block on relay while subagents wait (or vice versa)**. Run everything concurrently:

1. **Background the Bash call**: Use `run_in_background: true` on the Bash tool so the relay call runs concurrently with your subagents.
2. **Or use `--bg`**: The script forks the peer invocation and returns the response path immediately for polling.

```bash
# --bg returns immediately with the response path
RES=$(~/.claude/skills/relay/scripts/relay call --bg --name auth-review --effort medium <<'BODY'
Review src/auth.py for security issues.
BODY
)
# Poll for completion, then read with the Read tool
# [ -f "$RES" ] && echo "ready"
```

**Rule: Launch relay calls and subagents concurrently. Never serialize independent work.**

## Prism / Parallax

When Relay is used as the Parallax transport inside Prism, the relay call receives the **same full question and same context** as every local reviewer — only the lens (weighing posture) differs. Do not narrow the prompt for the Parallax agent.

Launch the relay Bash call with `run_in_background: true` in the same parallel dispatch step as the local reviewer subagents. Do not wrap Relay itself in another subagent layer.

## Timeout

For complex tasks, set a longer Bash timeout (default is 2 minutes, max 10 minutes):

```
Bash(timeout: 600000)
```

Or use the script's `--timeout <seconds>` flag to limit the peer invocation directly (requires coreutils `timeout` or `gtimeout`).

## Utility Commands

`relay list` shows all protocol files:

```bash
~/.claude/skills/relay/scripts/relay list
~/.claude/skills/relay/scripts/relay list --session auth-refactor
```

`relay --help` and `relay --version` print usage and version info.

## Low-Level Commands

For manual orchestration, use `req` and `res` to generate files without invoking the peer. If auto-detection fails, pass `--from claude --to codex` explicitly to `call`.

```bash
# Generate request only (does not invoke codex)
REQ=$(~/.claude/skills/relay/scripts/relay req --from claude --to codex --name slug "task body")

# Read response (use the Read tool, not cat)
# Response path: ${REQ%.req.md}.res.md
```

Do not invoke `codex exec` directly — always use `relay call` for the full round-trip.
