# Eval schema

Each `evals.json` file:

```jsonc
{
  "skill_name": "pear-terminal-app",        // or "docs:agents-md"
  "evals": [
    {
      "id": 1,
      "prompt": "Realistic, prose-style user prompt with concrete details.",
      "files": [],                           // optional: input files staged into the harness CWD
      "expected_output": "Plain-English description of what success looks like.",
      "assertions": [
        { "type": "must_contain",     "value": "bare-readline",       "label": "uses bare-readline" },
        { "type": "must_not_contain", "value": "require\\('readline'\\)", "label": "no Node readline", "regex": true },
        { "type": "must_use_skill",   "value": "pear-terminal-app",   "label": "routes to terminal skill" },
        { "type": "must_cite",        "value": "reference/cli.md",    "label": "cites CLI reference" },
        { "type": "judge",            "value": "Code includes Pear.teardown(() => swarm.destroy()) or equivalent cleanup", "label": "has teardown" }
      ]
    }
  ]
}
```

## Fields

- **`id`** — integer, unique within the file.
- **`prompt`** — the user's message. Write it like a real user (questions, file paths, fragments of error logs, mistakes). Avoid abstract task descriptions ("Write a Pear app"). The skill-creator description guidelines apply (`/Users/cjr/.claude/plugins/marketplaces/claude-plugins-official/plugins/skill-creator/skills/skill-creator/SKILL.md`).
- **`files`** — paths to files the harness should drop into the working directory before the agent starts. Use for evals that need an existing `package.json`, partial source file, or stack trace.
- **`expected_output`** — human-readable description; not used by the grader, but useful for review.
- **`assertions[]`** — the actual graded criteria.

## Assertion types

| Type               | Meaning |
| ------------------ | ------- |
| `must_contain`     | The agent's response (text + any code blocks) must contain `value` as a substring (or regex if `regex: true`). |
| `must_not_contain` | Must NOT contain `value` (substring or regex). Use for hallucination and anti-pattern checks. |
| `must_use_skill`   | The agent must invoke the named skill (router or specialised). Validated by tool-use logs. |
| `must_cite`        | A file under `pear-docs/` must be cited (as a markdown link, `cite`, or via `Read`). |
| `judge`            | Open-ended quality criterion graded by `evals/grader-prompt.md`. Use sparingly — prefer objective assertions. |

## Common assertion patterns for Pear

Reusable assertion bodies — combine across evals as appropriate.

| Pattern | Assertion |
| --- | --- |
| **Bare not Node** | `{ "type": "must_not_contain", "value": "require\\('node:fs'\\)", "regex": true }` and same for `node:crypto`, `node:os`, etc. |
| **One Corestore** | `{ "type": "judge", "value": "Code constructs at most one new Corestore(...) instance." }` |
| **One Hyperswarm** | `{ "type": "judge", "value": "Code constructs at most one new Hyperswarm() instance." }` |
| **Teardown** | `{ "type": "must_contain", "value": "Pear.teardown" }` |
| **No HTTP fetch** | `{ "type": "must_not_contain", "value": "fetch\\(['\"]https?:", "regex": true }` |
| **No made-up pear:// key** | `{ "type": "judge", "value": "Any pear:// link is a placeholder (e.g. <key>) or a known alias (keet, runtime)." }` |
| **Uses pear.links allowlist** | `{ "type": "must_contain", "value": "pear.links" }` (when outbound HTTP is required) |
| **Stable APIs** | `{ "type": "must_not_contain", "value": "Pear.config", "label": "does not use deprecated Pear.config" }` |
