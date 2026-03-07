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

## Required Flags

Every `codex exec` call MUST include these flags — no exceptions:

- `--model gpt-5.4` — the only allowed model. Even if the user asks for a "faster" or "cheaper" model, use gpt-5.4. If the user explicitly requests a different model, explain that the relay protocol requires gpt-5.4 and proceed with it.
- `--full-auto` — required for non-interactive relay. Without it, Codex prompts for confirmation on each tool use, breaking the automated handoff.

## Reasoning Effort

Choose `-c 'model_reasoning_effort="LEVEL"'` based on the task you are delegating:

| Level          | When to use                                                              | Examples |
|----------------|--------------------------------------------------------------------------|----------|
| `none`         | Fast, cost/latency-sensitive tasks where the model does not need to think | Reformat a file, extract field values, simple find-and-replace |
| `low`          | Latency-sensitive tasks where a small amount of thinking helps, especially with complex instructions | Triage a bug report, classify code patterns, apply a well-defined migration |
| `medium`/`high`| Tasks that truly require stronger reasoning and can absorb the latency and cost. Choose between them based on how much the task benefits from additional reasoning. Default to `medium`. | Code review, security audit, refactoring, writing tests, fixing bugs |
| `xhigh`        | Avoid unless evals show clear benefit. Long agentic reasoning where max intelligence matters more than speed or cost | Multi-file architectural redesign with cross-cutting concerns |

Before raising effort, first try improving the prompt: add an output contract, a verification step, or completeness criteria. Better prompts at lower effort often outperform vague prompts at higher effort.

## Prompting Codex

When crafting the task body for Codex, apply these patterns for best results:

- **Use XML tags** for structure — `<output_contract>`, `<completeness_contract>`, `<verification_loop>`, `<dependency_checks>`
- Define an **output contract** — exact format, length, and structure expected
- Define a **completeness contract** — what "done" means explicitly
- Add a **verification loop** — check correctness against each requirement
- Use **flat formatting** — modular sections with headers, no nested bullets
- Include **dependency checks** — don't let it skip prerequisite steps

Read `references/prompting-codex.md` for the full guide and reasoning effort selection matrix.

**Example — well-structured task body:**

> Refactor src/db/pool.py to add connection timeouts. We're seeing connection leaks in production — the pool creates connections but never reclaims stale ones.
>
> 1. Add `timeout_seconds` param to `ConnectionPool.__init__`
> 2. Implement auto-reconnection for stale connections
> 3. Add `reclaim_stale()` method
> 4. Keep backward compatibility
>
> \<output_contract>
> Summary of changes, one per line, with file path and description.
> \</output_contract>
>
> \<verification_loop>
> Run `pytest tests/test_pool.py` — all tests must pass.
> \</verification_loop>
>
> \<completeness_contract>
> Done means: all 4 requirements implemented, tests pass, no new lint errors.
> \</completeness_contract>

## Choosing One-Shot vs Session

- **One-shot** (`--name`): default for standalone tasks with no prior context. Use when the task is self-contained.
- **Session** (`--session`): use when the task continues a prior exchange, or when you plan multiple related relay calls that should share context. The session directory accumulates turn history so each subsequent call sees all prior requests and responses.

Rule of thumb: if the user says "continue", "follow up", "next step", or references a prior Codex exchange, use a session. Otherwise, use one-shot.

## One-Shot Call

Run as a single chained command so shell variables persist:

```bash
REQ=$(~/.claude/skills/relay/scripts/relay req --from claude --to codex --name auth-review "Review src/auth.py for security issues. Run pytest to verify.") && codex exec --model gpt-5.4 -c 'model_reasoning_effort="medium"' --full-auto "Read and execute $REQ"
```

Read the response:

```bash
RES="${REQ%.req.md}.res.md"
```

## Session Call

Sessions keep turn history so the receiver sees full context from both agents.

```bash
REQ=$(~/.claude/skills/relay/scripts/relay req --from claude --to codex --session auth-refactor "Fix the issues from my review. Run pytest to verify.") && codex exec --model gpt-5.4 -c 'model_reasoning_effort="medium"' --full-auto "Read and execute $REQ"
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
