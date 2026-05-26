# Pear agent evals

Two suites:

```
evals/
  docs/         Effectiveness of the agent-facing documentation (AGENTS.md, llms.txt, llms-full.txt).
  skills/       One eval per bundled skill in .claude/skills/.
```

Both follow a common schema (see `schema.md`). Each eval is a realistic user prompt with one or more `assertions` describing what a correct response must contain (or not contain). Assertions are categorised:

- `must_contain`     — substring/regex that must appear in the response.
- `must_not_contain` — substring/regex that must not appear.
- `must_use_skill`   — for skill evals; the named skill must be invoked.
- `must_cite`        — file path that must be referenced.
- `judge`            — open-ended quality criterion graded by a separate model.

## Running

Evals are designed to be run by the [skill-creator](.claude/skills/skill-creator/) workflow already vendored into this repo:

```sh
# evaluate one skill, comparing with-skill vs no-skill baseline runs
python -m skill_creator.scripts.run_evals \
  --skill .claude/skills/pear-terminal-app \
  --evals evals/skills/pear-terminal-app/evals.json
```

For doc evals, the baseline is "no AGENTS.md / no llms.txt in context"; the treatment is "AGENTS.md + llms.txt available". The harness measures whether the agent picks the correct skill and produces correct code.

## Grading

`judge`-type assertions use the prompt template at `evals/grader-prompt.md`. The grader returns `{passed: bool, evidence: string}` per assertion. Aggregate into `pass_rate` per skill / per doc.

## Doc-effectiveness metric

Beyond per-assertion pass rate, the doc eval reports three aggregate metrics:

- **Misroute rate** — % of skill-routing evals where the wrong skill is invoked.
- **Hallucination rate** — % of evals containing a `must_not_contain` violation (typically `require\('fs'\)`, `process.cwd`, made-up `pear://` keys, etc.).
- **Anti-pattern rate** — % of evals that produce code violating `reference/recommended-practices.md` (multiple Corestores, multiple Hyperswarms, HTTP/HTTPS load, no teardown).

Target after one optimisation round: misroute < 10%, hallucination < 5%, anti-pattern < 5%.
