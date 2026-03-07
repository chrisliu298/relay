# How to Prompt Codex (GPT-5.4) Effectively

Read this before composing complex relay tasks for Codex. These patterns come from
the official GPT-5.4 prompt guidance and improve reliability, completeness, and
token efficiency.

## Use XML Tags for Structure

GPT-5.4 responds best to modular, block-structured prompts using XML tags. Wrap
each type of guidance in its own tag so the model can parse instructions
unambiguously. Use consistent, descriptive tag names.

## Output Contracts

Define exactly what you want returned — format, length, structure. This is the
single most effective way to improve Codex output quality.

```xml
<output_contract>
- Return exactly the sections requested, in the requested order.
- If a format is required (JSON, Markdown, SQL), output only that format.
- Apply length limits only to the section they are intended for.
</output_contract>
```

**Example:**

```xml
<output_contract>
- Return a markdown list of issues.
- Each item: file path, line number, severity (high/medium/low), one-line
  description, and suggested fix.
</output_contract>
```

## Completeness Contracts

Define what "done" means explicitly. Codex is more reliable when the prompt
specifies the finish line.

```xml
<completeness_contract>
- Treat the task as incomplete until all requested items are covered or explicitly
  marked [blocked].
- Keep an internal checklist of required deliverables.
- For lists, batches, or paginated results: track processed items and confirm
  coverage before finalizing.
- If any item is blocked by missing data, mark it [blocked] and state exactly
  what is missing.
</completeness_contract>
```

## Verification Loops

Ask Codex to check its own work before finalizing. This catches errors that
higher reasoning effort alone would not.

```xml
<verification_loop>
Before finalizing:
- Check correctness: does the output satisfy every requirement?
- Check grounding: are claims backed by the provided context or tool outputs?
- Check formatting: does the output match the requested schema or style?
</verification_loop>
```

For programmatically checkable assertions, tell it to write and run a script
rather than eyeballing.

## Tool Persistence

Tell Codex to persist until the task is fully complete.

```xml
<tool_persistence_rules>
- Use tools whenever they materially improve correctness or completeness.
- Do not stop early when another tool call would improve the result.
- Keep calling tools until the task is complete and verification passes.
- If a tool returns empty or partial results, retry with a different strategy.
</tool_persistence_rules>
```

## Dependency Checks

Codex can skip prerequisite steps when the end state seems obvious. Prevent this.

```xml
<dependency_checks>
- Before taking an action, check whether prerequisite discovery or lookup steps
  are required.
- Do not skip prerequisite steps just because the intended final action seems
  obvious.
- If the task depends on the output of a prior step, resolve that dependency
  first.
</dependency_checks>
```

## Empty Result Recovery

When a lookup or search returns nothing, don't conclude no results exist.

```xml
<empty_result_recovery>
If a lookup returns empty, partial, or suspiciously narrow results:
- Do not immediately conclude that no results exist.
- Try at least 1-2 fallback strategies (alternate query wording, broader filters,
  prerequisite lookups, or alternate tools).
- Only report absence after documenting what was attempted.
</empty_result_recovery>
```

## Follow-Through Policy

Clarify when to proceed autonomously vs. stop and report.

```xml
<default_follow_through_policy>
- If the user's intent is clear and the next step is reversible and low-risk,
  proceed without asking.
- Ask permission only if the next step is irreversible, has external side effects,
  or requires missing information.
- If proceeding, briefly state what you did and what remains optional.
</default_follow_through_policy>
```

## Parallel vs Sequential Tool Calling

Structure tasks to enable Codex to parallelize independent steps while
sequencing dependent ones.

```xml
<parallel_tool_calling>
- When multiple retrieval or lookup steps are independent, prefer parallel tool
  calls to reduce wall-clock time.
- Do not parallelize steps that have prerequisite dependencies.
- After parallel retrieval, pause to synthesize results before making more calls.
</parallel_tool_calling>
```

## Prompt Structure

- Use modular sections with clear headers and XML tags
- Keep formatting flat: avoid deeply nested bullets, use `1. 2. 3.` for ordered steps
- Never use nested bullets — keep lists single-level
- If you need hierarchy, split into separate sections

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
improving the prompt itself — add an `<output_contract>`, `<completeness_contract>`,
or `<verification_loop>`. If the model still feels too literal or stops at the
first plausible answer, add a dig-deeper nudge before increasing effort:

```xml
<dig_deeper_nudge>
- Don't stop at the first plausible answer.
- Look for second-order issues, edge cases, and missing constraints.
- If the task is safety or accuracy critical, perform at least one verification
  step.
</dig_deeper_nudge>
```
