---
name: relay
description: |
  Call Claude Code from Codex and return a structured result. Triggers on
  "ask claude", "have claude", "send to claude", "get claude to",
  "delegate to claude", "second opinion", "relay". Invoke with /relay.
user_invocable: true
---

# Relay

Call Claude Code like a function:

```
relay(task, session?) → {status, verify, body}
```

Use the relay script at `scripts/relay` (inside this skill directory) to generate request/response files. Do not manually construct frontmatter.

## Model (REQUIRED — do not change)

You MUST use exactly `--model opus` when calling `claude`. No other model is allowed. Do **not** substitute or omit the model flag under any circumstances.

## Prompting Claude

When crafting the task body for Claude, apply these patterns for best results:

- Be **clear and direct** — explicit instructions, numbered steps for multi-part tasks
- Add **context and motivation** — explain why, not just what
- Use **structured sections** — headers for complex tasks; place data first, query last
- Include **examples** if output format matters (3-5 diverse examples)
- Don't over-prompt — Claude Opus is proactive; avoid excessive MUSTs/NEVERs

Read `references/prompting-claude.md` for the full guide.

**Example — well-structured task body:**

> We're hardening auth before a security audit. Review src/auth.py for OWASP Top 10 vulnerabilities, focusing on injection and broken access control.
>
> 1. Read src/auth.py and identify all vulnerabilities
> 2. Fix each one in-place
> 3. Run pytest to verify all tests pass
> 4. Return a summary: one line per fix, with line number and what changed

## One-Shot Call

Run as a single chained command so shell variables persist:

```bash
REQ=$(~/.codex/skills/relay/scripts/relay req --from codex --to claude --name auth-review "Review src/auth.py for security issues. Run pytest to verify.") && env -u CLAUDECODE claude --model opus -p --dangerously-skip-permissions "Read and execute $REQ"
```

Read the response:

```bash
RES="${REQ%.req.md}.res.md"
```

## Session Call

Sessions keep turn history so the receiver sees full context from both agents.

```bash
REQ=$(~/.codex/skills/relay/scripts/relay req --from codex --to claude --session auth-refactor "Fix the issues from my review. Run pytest to verify.") && env -u CLAUDECODE claude --model opus -p --dangerously-skip-permissions "Read and execute $REQ"
```

Read the response:

```bash
RES="${REQ%.req.md}.res.md"
```

## Output

Read the response file:

- **status**: `done` | `error`
- **verify**: `pass` | `fail` | `skip`
- **body**: findings, changes, reasoning — free-form markdown

If the request includes a verify command, run it and set `verify: pass` or `verify: fail`; include the command and key result in the body. If no verify command is provided or verification is not feasible, set `verify: skip` and state why briefly.

If the response file is missing, report failure — do not retry.
