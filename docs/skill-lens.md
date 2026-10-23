# skill-lens eval

A repeatable with-and-without check for `planning-with-files`, run by
[skill-lens](https://github.com/EmadMokhtar/skill-evaluator). One case to start with, measured
on two agents: Claude Code (a CLI product) and PydanticAI (an agent SDK).

## Run it

```bash
uv tool install "skill-lens[pydantic-ai]==0.23.0"
skill-lens run skills/planning-with-files --baseline none --repeat 5 \
  --runner claude-code --runner pydantic-ai
```

- `--baseline none` runs every case a second time with no skill loaded; `--repeat 5` runs each
  arm five times.
- `claude-code` uses your Claude Code login. `pydantic-ai` and the judge use `OPENAI_API_KEY`.
- With only a Claude Code login, set `judge = "claude-code"` in `skill-lens.toml` and run
  `--runner claude-code` alone.
- The exit code is the CI contract: `0` the gate passed, `1` it failed, `2` a mistake in the
  eval files.

## What the case checks

`creates-the-three-files` is eval 1 (todo-cli) from [evals.md](evals.md), with the same seven
assertions plus two on the contents:

| Check | How |
| --- | --- |
| `task_plan.md`, `findings.md`, `progress.md` exist | file assertions |
| `## Goal`, `### Phase`, `**Status:**`, `## Errors Encountered` in `task_plan.md` | regex / contains |
| `findings.md` records something: a bullet with text, or a table row | regex |
| `progress.md` records a concrete action, not only template fields | LLM judge |

Every check accepts the files in either place the skill uses: the project root (legacy layout)
or one named plan directory, `.planning/<date>-<slug>/`. Exactly one plan must exist; a plan in
both places fails.

## Results

skill-lens 0.23.0, planning-with-files v3.25.0 (`de837a9`), Claude Code 2.1.289. Judge:
`openai:gpt-6-luna`. Five runs per arm; tokens, cost and latency are means per run.

| Runner and model | With skill | Without skill | Tokens | Cost | Latency |
| --- | --- | --- | --- | --- | --- |
| `claude-code`, `claude-sonnet-4-6` | 3/5 | 0/5 | 457k (153k without) | $0.33 ($0.14) | 63 s (39 s) |
| `pydantic-ai`, `openai:gpt-6-luna` | 5/5 | 0/5 | 248k (8k without) | $0.012 ($0.002) | 170 s (21 s) |

Total: $2.43 for the 20 runs, plus $0.004 for the judge.

What it shows:

- **The skill is what produces the files.** Without it, neither agent wrote a planning file in
  any of the 10 runs.
- **Claude Code sometimes creates the files and leaves them empty.** The two failed runs kept
  the empty `findings.md` (and in one, `progress.md`) that `init-session.sh` writes, and one
  plan had no `## Goal`. The same pattern showed up with Sonnet under PydanticAI, so it follows
  the model, not the runner. `gpt-6-luna` filled every file in every run.
- **Both layouts happen.** PydanticAI used the project root in 2 of 5 runs and a named plan
  directory in 3; Claude Code always used a named directory.

## How a run is isolated

- Every run gets its own new temporary directory, and the two arms never share one.
- Claude Code starts with `--setting-sources project --strict-mcp-config
  --no-session-persistence`, so the runner's own settings, hooks and MCP servers stay out and
  no session is saved. The skill's frontmatter hooks do run.
- **It is not a sandbox.** Claude Code runs with permission prompts off and the full
  environment, so it can read outside the directory. The skill's hooks fall back to
  `$HOME/.claude/skills/planning-with-files/` when present, so a personal install can affect a
  run.
- PydanticAI runs the skill's scripts through skill-lens's own sandbox (`sandbox-exec` on
  macOS, `bwrap` on Linux): no network, no writes outside the run's directory.

## Budgets

- A `budget:` block is checked after a run ends; it fails the check, it does not stop spending.
- Claude Code stops a run itself at `--max-budget-usd` (`1.00` in `skill-lens.toml`); such a run
  is reported as errored.
- A timeout (`timeout_seconds`, 600 here) also stops a run.

## Known limits

- A case is one session. A test where one session writes the files and a fresh session resumes
  from them needs multi-session cases
  ([skill-evaluator#95](https://github.com/EmadMokhtar/skill-evaluator/issues/95)).
- skill-lens delivers `scripts/`, `references/` and `assets/` only, so `templates/`,
  `reference.md` and `examples.md` do not reach the agent
  ([skill-evaluator#92](https://github.com/EmadMokhtar/skill-evaluator/issues/92)).
- The report cannot yet show which hooks fired
  ([skill-evaluator#96](https://github.com/EmadMokhtar/skill-evaluator/issues/96)).
- Only Claude Code can stop spending mid-run
  ([skill-evaluator#94](https://github.com/EmadMokhtar/skill-evaluator/issues/94)), and product
  runs are not sandboxed ([skill-evaluator#99](https://github.com/EmadMokhtar/skill-evaluator/issues/99)).
