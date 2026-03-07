# How to Prompt Codex (GPT-5.4) Effectively

Read this before composing complex relay tasks for Codex. These patterns come from
the official GPT-5.4 prompt guidance and improve reliability, completeness, and
token efficiency.

## Output Contracts

Define exactly what you want returned — format, length, structure. This is the
single most effective way to improve Codex output quality.

- Bad: "Review this file"
- Good: "Return a markdown list of issues. Each item: file path, line number,
  severity (high/medium/low), one-line description, and suggested fix."

## Completeness Contracts

Define what "done" means explicitly. Codex is more reliable when the prompt
specifies the finish line.

- "Done means: every function has a docstring, every public API has a type hint,
  and the test suite passes with no failures."
- For batch tasks, tell it to track processed items and confirm coverage before
  finalizing.

## Verification Loops

Ask Codex to check its own work before finalizing. This catches errors that
higher reasoning effort alone would not.

- "After making changes, re-read each modified file and verify every requirement
  from this task is satisfied. List each requirement and its pass/fail status."
- For programmatically checkable assertions, tell it to write and run a script
  rather than eyeballing.

## Tool Persistence

Tell Codex to persist until the task is fully complete.

- "If a search returns empty results, try 1-2 fallback strategies (broader glob,
  alternate wording, prerequisite lookups) before concluding nothing exists."
- "Do not stop at analysis or partial fixes. Carry changes through implementation,
  verification, and explanation."

## Dependency Checks

Codex can skip prerequisite steps when the end state seems obvious. Prevent this.

- "Before modifying any file, first read it to understand the current state."
- "Before running tests, ensure dependencies are installed."
- "Do not skip prerequisite steps just because the final action seems obvious."

## Follow-Through Policy

Clarify when to proceed autonomously vs. stop and report.

- "Proceed with the fix without asking for confirmation. If you encounter an
  ambiguous case, document your assumption and continue."
- "If a step is irreversible or has external side effects, describe what you
  intend to do and stop."

## Empty Result Recovery

When a lookup or search returns nothing, don't conclude no results exist. Try
1-2 fallback strategies before reporting absence.

- Try alternate query wording, broader filters, or prerequisite lookups
- Only report absence after documenting what was attempted
- "If grep returns no results, try a broader pattern or search in parent
  directories before concluding the code doesn't exist."

## Parallel vs Sequential Tool Calling

Structure tasks to enable Codex to parallelize independent steps while
sequencing dependent ones.

- Parallelize independent retrieval steps (reading multiple files, searching
  multiple terms)
- Sequence steps with dependencies (read file, then modify it)
- "Read all test files in parallel, then synthesize findings before making
  changes."

## Prompt Structure

- Use modular sections with clear headers — Codex follows structured prompts well
- Keep formatting flat: avoid deeply nested bullets, use `1. 2. 3.` for ordered steps
- For multi-step tasks, request intermediary status updates via the commentary channel

## Reasoning Effort Selection

Choose the reasoning effort level based on the task you are delegating.

| Level | When to use |
|-------|-------------|
| `none` | Fast, cost/latency-sensitive tasks where the model does not need to think. Best for execution-heavy work: workflow steps, field extraction, triage, short structured transforms. |
| `low` | Latency-sensitive tasks where a small amount of thinking can produce a meaningful accuracy gain, especially with complex instructions. |
| `medium` or `high` | Reserve for tasks that truly require stronger reasoning and can absorb the latency and cost tradeoff. Choose between them based on how much performance gain the task gets from additional reasoning. Start with `medium` for research-heavy work. |
| `xhigh` | Avoid as a default unless evals show clear benefits. Best suited for long, agentic, reasoning-heavy tasks where maximum intelligence matters more than speed or cost. |

**Escalation rule**: Reasoning effort is a last-mile tuning knob, not the
primary way to improve quality. Before raising the effort level, first try
improving the prompt itself — add an output contract, completeness contract, or
verification loop. If the model feels too literal or stops at the first
plausible answer, add a dig-deeper nudge ("Look for second-order issues, edge
cases, and missing constraints") before increasing effort.
