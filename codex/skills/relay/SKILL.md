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

The script auto-detects caller/peer from its install path. Use `scripts/relay` inside this skill directory.

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

## Prompting Claude

Be clear and direct. Use XML tags to separate concerns. Key patterns:

- `<context>` — background and motivation (why, not just what)
- `<instructions>` — numbered steps for multi-part tasks
- `<example>` — example output if format matters

Don't over-prompt — Claude Opus is proactive; avoid excessive MUSTs/NEVERs.

See `references/prompting-claude.md` for the full guide.

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

## Low-Level Commands

For custom workflows or manual orchestration, use `req` and `res` directly. Body can be passed as an argument or piped via stdin.

```bash
# Generate request only
REQ=$(~/.codex/skills/relay/scripts/relay req --from codex --to claude --name slug "task body")

# Then invoke claude manually
env -u CLAUDECODE claude --model opus -p --dangerously-skip-permissions "Read and execute $REQ"

# Read response
cat "${REQ%.req.md}.res.md"
```
