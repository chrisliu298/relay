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

Call Codex like a function. One command: generates the request, invokes Codex, prints the response.

```
relay call --name <slug> --effort <level> <<'BODY'
task
BODY
```

The script auto-detects caller/peer from its install path. Use `scripts/relay` inside this skill directory.

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

The script prints the response file content to stdout. The response includes YAML frontmatter:

- **status**: `done` | `error`
- **verify**: `pass` | `fail` | `skip`
- **body**: findings, changes, reasoning — free-form markdown

Request and response files are saved in `.relay/` (auto-gitignored).

If the response file is missing, the script reports failure — do not retry.

## Timeout

For complex tasks, set a longer Bash timeout (default is 2 minutes, max 10 minutes):

```
Bash(timeout: 600000)
```

## Low-Level Commands

For custom workflows or manual orchestration, use `req` and `res` directly. Body can be passed as an argument or piped via stdin. If auto-detection fails, pass `--from claude --to codex` explicitly to `call`.

```bash
# Generate request only
REQ=$(~/.claude/skills/relay/scripts/relay req --from claude --to codex --name slug "task body")

# Then invoke codex manually
codex exec --model gpt-5.4 -c 'model_reasoning_effort="medium"' --full-auto "Read and execute $REQ"

# Read response (use the Read tool, not cat)
# Response path: ${REQ%.req.md}.res.md
```
