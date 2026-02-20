---
name: relay
description: |
  Call Codex from Claude Code and return a structured result. Triggers on
  "ask codex", "have codex", "send to codex", "get codex to", "delegate to
  codex", "second opinion", "relay". Invoke with /relay.
allowed-tools: Read, Write, Bash(~/.claude/skills/relay/scripts/relay:*), Bash(codex exec:*), Bash(find:*), Bash(printf:*)
user-invocable: true
---

# Relay

Call Codex like a function:

```
relay(task, session?) → {status, verify, body}
```

Use the relay script at `scripts/relay` (inside this skill directory) to generate request/response files. Do not manually construct frontmatter.

## One-Shot Call

```bash
RELAY=~/.claude/skills/relay/scripts/relay
REQ=$($RELAY req --from claude --to codex --name auth-review "Review src/auth.py for security issues. Run pytest to verify.")
codex exec --full-auto "Read and execute $REQ"
```

Read the response:

```bash
RES="${REQ%.req.md}.res.md"
```

## Session Call

Sessions keep turn history so the receiver sees full context from both agents.

```bash
RELAY=~/.claude/skills/relay/scripts/relay
REQ=$($RELAY req --from claude --to codex --session auth-refactor "Fix the issues from my review. Run pytest to verify.")
codex exec --full-auto "Read and execute $REQ"
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
