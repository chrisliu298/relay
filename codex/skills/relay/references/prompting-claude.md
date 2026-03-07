# How to Prompt Claude (Opus) Effectively

Read this before composing complex relay tasks for Claude. These patterns come
from the official Claude prompting best practices and improve accuracy,
consistency, and output quality.

## Be Clear and Direct

Claude responds well to explicit, specific instructions. Tell Claude what to DO
rather than what NOT to do. Use numbered steps for multi-part tasks.

- Bad: "Look at the auth code and maybe fix some stuff"
- Bad: "Don't use markdown formatting" (negative instruction)
- Good: "1. Read src/auth.py  2. Identify SQL injection vulnerabilities
  3. Fix each one  4. Run pytest to verify all tests pass"
- Good: "Write your response in flowing prose paragraphs" (positive instruction)

## Be Explicit About Action vs. Suggestion

Claude distinguishes between being asked to suggest and being asked to act. If
you want Claude to make changes, say so directly.

- Bad: "Can you suggest some improvements to the error handling?"
- Good: "Improve the error handling in src/api.py."
- Good: "Make these edits to the authentication flow."

If the task is ambiguous, Claude may default to suggesting rather than
implementing. In relay tasks you almost always want implementation, so be direct.

## Add Context and Motivation

Explain why the task matters, not just what to do. Claude generalizes better
when it understands the purpose behind an instruction.

- "We're hardening auth before a security audit. Focus on OWASP Top 10
  vulnerabilities, especially injection and broken access control."
- Instead of "NEVER use ellipses", say "The output will be read by a
  text-to-speech engine, so avoid ellipses since it can't pronounce them."

## Set a Role

A single sentence setting Claude's role focuses its behavior and tone. Even
brief context about what role to adopt makes a measurable difference.

- "You are a security auditor reviewing this codebase for vulnerabilities."
- "You are a performance engineer optimizing database queries."

## Use Structured Sections and XML Tags

For complex tasks, organize with clear headers or XML-style tags. Put long
context or data at the top, instructions and query at the bottom — this can
improve response quality significantly.

- Use descriptive section headers for different parts of the task
- Use XML tags like `<context>`, `<instructions>`, `<constraints>` to separate
  concerns unambiguously — Claude parses these reliably
- For multi-document input, label each document clearly with source and content
- Ask Claude to quote relevant parts of documents before analyzing — this helps
  it cut through noise in long inputs

## Provide Examples

When output format matters, include 3-5 examples showing input and expected
output. This is one of the most reliable ways to steer format, tone, and
structure.

- Wrap examples in `<example>` tags (multiple in `<examples>`) so Claude
  distinguishes them from instructions
- Cover edge cases, not just the happy path
- Vary examples enough that Claude doesn't pick up unintended patterns
- Include both input and expected output in each example

## Leverage Parallel Execution

Claude excels at parallel tool calls. Structure tasks to enable this — Claude
will naturally parallelize independent operations.

- "Read all three config files and compare their settings" — Claude will read
  them in parallel
- For independent subtasks, list them as separate items rather than a single
  sequential chain

## Ask Claude to Self-Check

For tasks where correctness matters, ask Claude to verify its own work before
finishing. This catches errors reliably, especially for coding and math.

- "Before you finish, verify your changes against each requirement in this task."
- "After implementing the fix, re-read the modified file and confirm the issue
  is resolved."

## Don't Over-Prompt

Claude Opus is proactive and fills in reasonable gaps. Heavy-handed prompting
that was needed for older models can cause overtriggering with current models.

- Avoid excessive MUST/NEVER/ALWAYS unless truly non-negotiable
- Trust the model to make reasonable choices within your constraints
- Instead of "CRITICAL: You MUST use this tool when...", use "Use this tool
  when..." — Claude follows normal-weight instructions reliably
- Reserve strong directives for genuinely safety-critical requirements
