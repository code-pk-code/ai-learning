# Prompt Engineering for Developers

> A practical reference guide covering core techniques, real examples, and production patterns for developers integrating AI into their daily workflow.

---

## Table of Contents

1. [The Developer's Prompt Formula](#the-developers-prompt-formula)
2. [Zero-Shot Prompting](#1-zero-shot-prompting)
3. [Few-Shot Prompting](#2-few-shot-prompting)
4. [Chain-of-Thought (CoT)](#3-chain-of-thought-cot)
5. [System Prompts](#4-system-prompts)
6. [XML Structuring](#5-xml-structuring)
7. [Iterative Refinement](#6-iterative-refinement)
8. [Real-World Example: JSDoc Generator](#real-world-example-jsdoc-generator)
9. [Quick Reference Cheatsheet](#quick-reference-cheatsheet)
10. [Learning Roadmap](#learning-roadmap)

---

## The Developer's Prompt Formula

Most weak prompts are just a vague task — "Review my code", "Write a test", "Explain this". Strong prompts fill in all five slots:

```
[Role/Context] + [Task with specifics] + [Examples if needed] + [Output format] + [Constraints]
```

**The "what + how + format" pattern** works for virtually every dev task:
- **What** — what to do and what to focus on
- **How** — the approach or method to use
- **Format** — exactly what shape the output should take

> **Key habit:** Treat every prompt like a unit test. When output is wrong, *debug the prompt* — add specificity, add examples, decompose the task. Don't just retry hoping for different results.

---

## 1. Zero-Shot Prompting

Ask directly with no examples. Quality depends entirely on how specific your instruction is.

### Weak vs Strong

| Weak | Strong |
|------|--------|
| `Review my code` | `Review this Python function for: 1) security vulnerabilities 2) performance issues 3) PEP 8 violations. For each: state the line number, the problem, and a fix. Reply in markdown with severity labels (critical/medium/low).` |
| `Write tests` | `Write pytest unit tests for this function. Cover: happy path, edge cases (empty input, None, boundary values), and one error case. Use fixtures for any shared setup.` |
| `Explain this` | `Explain the non-obvious parts of this code to a junior developer. Skip anything self-explanatory. Focus on: why this approach was chosen over alternatives, and what breaks if this is changed.` |

### Why the strong version works

The strong version specifies:
- **What** to look for (security, performance, style)
- **What** to include in each finding (line number, problem, fix)
- **Format** (markdown, severity labels)

### Template

```
[Task]: [specific action]
Focus on: [concern 1], [concern 2], [concern 3]
For each [item]: [what to include]
Format: [output structure]
```

---

## 2. Few-Shot Prompting

Show the model input/output examples before your actual request. Sets tone, format, and style precisely.

### How it works

```
[Optional: system context]

// Example 1
Input: [example input]
Output: [exact desired output]

// Example 2
Input: [example input]
Output: [exact desired output]

// Real request
Input: [your actual input]
Output:
```

### Real example: Git commit message generator

```
You write git commit messages following the Conventional Commits spec.

Diff: Added null check before calling user.getProfile()
Commit: fix(auth): prevent crash when user profile is null

Diff: Added Redis caching layer to /api/products endpoint
Commit: perf(api): cache product list with 5-min TTL via Redis

Diff: Removed legacy OAuth 1.0 token handling code
Commit: refactor(auth): remove deprecated OAuth 1.0 support

Diff: Updated Dockerfile base image from node:16 to node:20-alpine, removed unused ENV vars
Commit:
```

**Expected output:**
```
build(docker): upgrade base image to node:20-alpine, clean unused ENV vars
```

### Rules for few-shot examples

- **3 examples** is the sweet spot — more gives diminishing returns at higher token cost
- Examples should cover **edge cases** you care about, not just the happy path
- Make sure examples are **consistent** in style — one off-style example poisons the pattern
- Choose examples that **span the variation** in your real inputs

---

## 3. Chain-of-Thought (CoT)

Ask the model to reason step-by-step before answering. Dramatically improves accuracy on complex tasks like architecture decisions, debugging, and security analysis.

### Without vs With CoT

**Without CoT:**
```
Which database index should I add to speed up this query?

SELECT * FROM orders WHERE user_id = ? AND status = 'pending' ORDER BY created_at DESC
```

**With CoT:**
```
Which database index should I add to speed up this query?

SELECT * FROM orders WHERE user_id = ? AND status = 'pending' ORDER BY created_at DESC

Think step by step:
1. Identify which columns are in the WHERE clause
2. Identify which column is used for ORDER BY
3. Consider index cardinality for each column
4. Decide: composite index or multiple single-column indexes?
5. Then give your final recommendation with the CREATE INDEX statement.
```

### Multi-step debugging workflow with CoT

```
Step 1 — Understand the error:
"Read this stack trace. Explain in plain English what failed and why,
before suggesting any fix."

Step 2 — Hypothesis generation:
"List 3 possible root causes, ranked by likelihood.
For each: what evidence would confirm it?"

Step 3 — Fix + verification:
"For the most likely cause, write the fix AND a test that would
have caught this bug."
```

### When to use CoT

Use CoT when the **reasoning IS the value**:
- Architecture and design decisions
- Debugging complex issues
- Security analysis
- Performance optimization
- Any task where a quick wrong answer is worse than a slower right one

---

## 4. System Prompts

Set a persistent persona, rules, and constraints that apply to the entire conversation. The foundation of any AI feature you build.

### Anatomy of a strong system prompt

```
[Role + expertise level]

## Behaviour
- [How to act]
- [Tone and style]
- [Priority order]

## Output format
[Exact structure to use]

## What NOT to do
- [Explicit exclusions]
- [Anti-patterns to avoid]
```

### Real example: Senior code reviewer

```
You are a senior software engineer doing code review for a TypeScript/React codebase.

## Your behaviour
- Be direct and precise. No filler phrases.
- Focus on correctness, security, and maintainability — in that order.
- Suggest the fix, don't just identify the problem.
- If something is a style preference (not a bug), label it [style].

## Output format
Always respond in this structure:
CRITICAL: (issues that must be fixed before merge)
MEDIUM: (should be addressed)
LOW / STYLE: (nice to have)
VERDICT: approve | request-changes | needs-discussion

## What you don't do
- Don't rewrite working code just to show alternatives.
- Don't comment on things outside the diff.
- Don't add motivational language.
```

### Key ingredient: negative constraints

**Negative constraints are underused and powerful.** They prevent the model's default tendencies that don't fit your use case.

| Default tendency | Negative constraint |
|-----------------|---------------------|
| Adds caveats and disclaimers | `Don't add disclaimers unless something is genuinely unsafe.` |
| Offers alternatives unprompted | `Don't suggest alternative approaches unless asked.` |
| Over-explains obvious things | `Skip anything a senior developer would already know.` |
| Uses motivational language | `No phrases like 'Great question!' or 'Certainly!'.` |

---

## 5. XML Structuring

Use XML-like tags to separate instructions from content. Prevents the model from confusing your format template with data to summarize — critical when your prompt contains code, markdown, or user-generated text.

### When to use XML tags

Use tags whenever you're mixing:
- Instructions + user data
- Instructions + examples
- Multiple distinct sections
- Any code or markdown in the prompt body

### Real example: PR description generator

```xml
<task>
Generate a pull request description for the following code changes.
</task>

<format>
## What this PR does
(1-2 sentences, non-technical)

## Changes
- bullet list of key changes

## Testing done
- how it was tested

## Breaking changes
(none / list them)
</format>

<context>
Repo: e-commerce backend (Node.js/PostgreSQL)
Team convention: link Jira ticket if mentioned in commits
</context>

<diff>
[paste git diff here]
</diff>
```

### Common tag patterns

```xml
<task>...</task>           <!-- What to do -->
<context>...</context>     <!-- Background info -->
<format>...</format>       <!-- Output structure -->
<example>...</example>     <!-- Few-shot examples -->
<input>...</input>         <!-- The actual data to process -->
<constraints>...</constraints>  <!-- Rules and limits -->
```

> **Note:** Tag names don't need to match real XML schemas. Just be consistent within a prompt.

---

## 6. Iterative Refinement

Treat prompts like code: write, test, measure, improve. Most prompts need 3–5 iterations to reach production quality.

### Debugging playbook

| Problem | Fix |
|---------|-----|
| Output too long | Add: `"Reply in under 150 words."` |
| Wrong format | Add 1–2 examples of exactly the right output (few-shot) |
| Misses the point | Add: `"Focus specifically on X. Ignore Y."` |
| Too generic | Add: `"Give concrete examples for a [your stack] codebase."` |
| Task too complex | Decompose: split into sequential sub-prompts |
| Inconsistent quality | Add self-critique: `"Before finalising, check your answer against: [criteria]. Revise if needed."` |

### Real iteration log: code explanation prompt

```
v1 — too vague
"Explain this code"
→ Output: 3 paragraphs, too high-level, no examples

v2 — added audience + format
"Explain this code to a junior developer. Use bullet points."
→ Output: better, but explains obvious things, skips the tricky parts

v3 — added focus + negative constraint
"Explain the non-obvious parts of this code to a junior developer.
Skip anything self-explanatory. Focus on: why this approach was chosen
over alternatives, and what breaks if this is changed."
→ Output: exactly what a good code reviewer would highlight
```

### Self-critique pattern

Append to any prompt where quality needs to be consistent:

```
Before finalising your response, review it against these criteria:
- [ ] Does it answer the specific question asked?
- [ ] Is every recommendation actionable?
- [ ] Is there anything obvious that was missed?
Revise your response if any criterion fails.
```

---

## Real-World Example: JSDoc Generator

A complete few-shot prompt ready for production use. Demonstrates all techniques combined: system prompt, few-shot examples, negative constraints, and format specification.

### The prompt

```
You are a TypeScript documentation expert. Generate JSDoc comments for TypeScript functions.

Rules:
- Use @param with type and description for every parameter
- Use @returns with type and description
- Add @throws if the function can throw
- Add @example with a realistic usage snippet
- Keep descriptions concise but informative — explain WHY, not just WHAT
- Never restate the type in the description (the type tag already has it)

---
FUNCTION:
async function fetchUser(userId: string, options?: { timeout?: number; retry?: boolean }): Promise<User | null>

JSDOC:
/**
 * Fetches a user record from the database by ID.
 *
 * Returns null if the user does not exist rather than throwing,
 * so callers can distinguish "not found" from network errors.
 *
 * @param userId - The UUID of the user to retrieve.
 * @param options - Optional request configuration.
 * @param options.timeout - Request timeout in milliseconds. Defaults to 5000.
 * @param options.retry - Whether to retry once on transient failure. Defaults to false.
 * @returns The matching user record, or null if not found.
 * @throws {NetworkError} If the database is unreachable after all retries.
 *
 * @example
 * const user = await fetchUser('abc-123', { timeout: 3000 });
 * if (!user) {
 *   return res.status(404).json({ error: 'User not found' });
 * }
 */

---
FUNCTION:
function parseConfig<T extends Record<string, unknown>>(
  raw: string,
  schema: ZodSchema<T>
): Result<T, ConfigError>

JSDOC:
/**
 * Parses and validates a raw JSON config string against a Zod schema.
 *
 * Returns a Result type instead of throwing, so callers can handle
 * validation errors without try/catch at every call site.
 *
 * @param raw - The raw JSON string to parse. Must be valid JSON.
 * @param schema - The Zod schema to validate the parsed object against.
 * @returns `{ ok: true, value: T }` on success, or `{ ok: false, error: ConfigError }` on failure.
 *
 * @example
 * const result = parseConfig(process.env.APP_CONFIG!, AppConfigSchema);
 * if (!result.ok) {
 *   logger.error('Invalid config', result.error);
 *   process.exit(1);
 * }
 */

---
Now generate a JSDoc comment for this function:
[PASTE YOUR FUNCTION SIGNATURE HERE]
```

### Using it via the Claude API

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

const SYSTEM_PROMPT = `You are a TypeScript documentation expert...`; // full prompt above

async function generateJSDoc(functionSignature: string): Promise<string> {
  const msg = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 1024,
    system: SYSTEM_PROMPT,
    messages: [{ role: "user", content: functionSignature }],
  });
  return (msg.content[0] as { text: string }).text;
}
```

> **Tip:** Put everything up to "Now generate" in `system`. Send only the function signature as the `user` message. This keeps your system prompt fixed and minimizes per-call token cost.

---

## Quick Reference Cheatsheet

### Technique selector

```
Is your task well-defined and clear?
  YES → Zero-shot with strong format spec
  NO  → Add few-shot examples

Does quality matter a lot?
  YES → Add chain-of-thought reasoning steps
  NO  → Zero-shot is fine

Are you building a reusable AI feature?
  YES → Write a system prompt
  NO  → User-turn prompt is enough

Does your prompt mix instructions + code/data?
  YES → Use XML tags to separate them
  NO  → Plain prose is fine

Is output quality inconsistent?
  YES → Add self-critique step or iterate
  NO  → Ship it
```

### Copy-paste snippets

**Force a specific format:**
```
Reply only in this JSON format, no explanation:
{"issue": "...", "severity": "critical|medium|low", "fix": "..."}
```

**Add chain-of-thought:**
```
Think through this step by step before giving your final answer.
```

**Add self-critique:**
```
Before finalising, review your answer against: [criteria]. Revise if needed.
```

**Negative constraint block:**
```
Do not:
- Add disclaimers or caveats unless genuinely necessary
- Suggest alternatives unless asked
- Repeat information already stated in the question
```

**Few-shot opener:**
```
Here are examples of exactly the output I want:

Input: [example]
Output: [example]

Input: [example]
Output: [example]

Now do the same for:
Input: [your actual input]
Output:
```

---

## Learning Roadmap

Recommended order for maximum immediate value:

| Priority | Technique | Time to value |
|----------|-----------|--------------|
| 1 | Zero-shot + format control | Immediate — works on first try |
| 2 | System prompts | Day 1 of building any AI feature |
| 3 | XML structuring | When prompts get complex or include code |
| 4 | Few-shot examples | When style/format needs to be exact |
| 5 | Chain-of-thought | For complex reasoning tasks |
| 6 | Iterative refinement | Ongoing — applied to everything |

### What to practice first

Take one task you do every day (code review, writing commit messages, explaining code, writing tests) and:

1. Write a v1 prompt using the formula: `[Role] + [Task] + [Format]`
2. Run it 3 times on real input
3. Note what's wrong or inconsistent
4. Apply one technique from this guide to fix it
5. Repeat until the output is consistently what you'd write yourself

That process — not memorizing techniques — is how prompt engineering skill is actually built.

---

## Further Reading

- [Anthropic Prompt Engineering Docs](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
- [OpenAI Prompt Engineering Guide](https://platform.openai.com/docs/guides/prompt-engineering)
- [Learn Prompting](https://learnprompting.org/)
- [Anthropic Claude API Reference](https://docs.anthropic.com/en/api/getting-started)

---

*Guide compiled from hands-on AI SDLC and prompt engineering sessions. Part of the AI Skills for Developers learning path.*
