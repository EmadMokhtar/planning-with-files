# skill-lens eval

A repeatable with-and-without check for `planning-with-files`, run by
[skill-lens](https://github.com/EmadMokhtar/skill-evaluator). Two cases, measured on two
agents: Claude Code (a CLI product) and PydanticAI (an agent SDK).

## Run it

```bash
uv tool install "skill-lens[pydantic-ai]==0.25.0"
skill-lens run skills/planning-with-files --baseline none --repeat 5 \
  --runner claude-code --runner pydantic-ai
```

- `--baseline none` runs every case a second time with no skill loaded; `--repeat 5` runs each
  arm five times.
- `claude-code` runs with a temporary home directory (see
  [How a run is isolated](#how-a-run-is-isolated)), so the login saved in your own home
  directory or keychain does not reach it. Export `CLAUDE_CODE_OAUTH_TOKEN` (create one with
  `claude setup-token`) or `ANTHROPIC_API_KEY`. `pydantic-ai` and the judge use
  `OPENAI_API_KEY`.
- With only Claude Code, set `judge = "claude-code"` in `skill-lens.toml` and run
  `--runner claude-code` alone. Without a supported sandbox backend, also pass
  `--no-allow-scripts` and `--product-sandbox auto`. The first disables skill-lens's framework
  script tool; Claude Code still runs its own shell commands. The second runs Claude Code with
  a temporary home directory but no sandbox, and the report says so.
- Add `--json-output report.json` to keep every hook the skill ran (see
  [What the hooks did](#what-the-hooks-did)).
- The exit code is the CI contract: `0` the gate passed, `1` it failed, `2` a mistake in the
  eval files.

## What the cases check

### `creates-the-three-files`

`creates-the-three-files` is eval 1 (todo-cli) from [evals.md](evals.md), with the same seven
assertions plus two on the contents:

| Check | How |
| --- | --- |
| `task_plan.md`, `findings.md`, `progress.md` exist | file assertions |
| `## Goal`, `### Phase`, `**Status:**`, `## Errors Encountered` in `task_plan.md` | regex / contains |
| `findings.md` records something: a bullet with text, or a table row | regex |
| `progress.md` records a concrete action, not only template fields | LLM judge |

Every check accepts the files in either place the skill uses: the project root (legacy layout)
or a named plan directory, `.planning/<date>-<slug>/`. Each filename must have exactly one
match. skill-lens checks each filename on its own, so files split across different plan
directories can still pass. This case does not verify that all three files share one directory.

### `resumes-from-its-own-plan`

This case tests the skill's main claim: a fresh session resumes from the files an earlier
session wrote. It runs as two sessions against one workspace. Each session is a new agent
process with no conversation history; only the files carry over.

1. "Build a Python CLI todo tool with persistence, in three phases. Phase 1: design the
   storage format and the commands. Phase 2: implement add and list. Phase 3: implement done
   and delete. Do phase 1 only, then stop."
2. "Continue the todo tool from where the last session stopped. Do the next phase only, then
   stop."

Session 2 is not told which phase is next; it has to read the plan to know. Every check runs
after session 2, so each looks for what only session 2 could have done:

| Check | How |
| --- | --- |
| `task_plan.md` marks Phase 2 `complete` | regex on the `### Phase 2` section |
| `task_plan.md` leaves Phase 3 `pending` or `in_progress` | regex on the `### Phase 3` section |
| `progress.md` records concrete work on `add` and `list` | LLM judge |
| `progress.md` still records the phase 1 design work, before the `add` and `list` work | LLM judge |

Both files must have exactly one match, so a session 2 that starts a second plan instead of
continuing the first one fails.

## Results

Contributor-reported measurements using skill-lens 0.25.0, planning-with-files v3.25.1
(`99402ce`), Claude Code 2.1.289, with the configuration in this repository: both sandboxes
required, `sandbox-exec` on macOS. Judge: `openai:gpt-6-luna`. Five runs per arm; tokens,
cost and latency are means per run. Both cases ran in one run, which cost $6.03 in all.

### `creates-the-three-files`

| Runner and model | With skill | Without skill | Tokens | Cost | Latency |
| --- | --- | --- | --- | --- | --- |
| `claude-code`, `claude-sonnet-4-6` | 3/5 | 0/5 | 431k (103k without) | $0.34 ($0.12) | 66 s (28 s) |
| `pydantic-ai`, `openai:gpt-6-luna` | 5/5 | 0/5 | 259k (7k without) | $0.012 ($0.002) | 116 s (23 s) |

Total: $2.38 for the 20 runs, plus $0.004 for the judge.

The pass counts match the earlier run on skill-lens 0.23.0 and planning-with-files v3.25.0,
which had no product sandbox and did not deliver `templates/`, `reference.md` or
`examples.md` to the agent.

Observations from these runs:

- Without the skill, neither agent wrote a planning file in any of the 10 runs.
- **Claude Code sometimes creates the files and leaves them empty.** Both failed runs left
  `findings.md` as the empty template that `init-session.sh` writes, and one also left the
  actions in `progress.md` empty. The earlier run showed the same pattern, and the contributor
  also observed it with Sonnet under PydanticAI. This small sample does not isolate model,
  runner or environment effects. `gpt-6-luna` filled every file in the five runs with the
  skill.
- **Both layouts happen.** PydanticAI used the project root in 1 of 5 runs and a named plan
  directory in 4; Claude Code always used a named directory.
- **The hooks ran every time.** Each Claude Code run with the skill ran 14 to 26 hooks
  (`UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `Stop`), and none failed.

### `resumes-from-its-own-plan`

Tokens, cost and latency add up both sessions of a run.

| Runner and model | With skill | Without skill | Tokens | Cost | Latency |
| --- | --- | --- | --- | --- | --- |
| `claude-code`, `claude-sonnet-4-6` | 4/5 | 0/5 | 593k (201k without) | $0.52 ($0.19) | 92 s (39 s) |
| `pydantic-ai`, `openai:gpt-6-luna` | 1/5 | 0/5 | 266k (14k without) | $0.010 ($0.003) | 124 s (36 s) |

Total: $3.64 for the 20 runs, plus $0.005 for the judge.

Observations from these runs:

- Without the skill, no run wrote a plan, so session 2 had none to continue and every check
  failed.
- **Claude Code resumed in 4 of 5 runs.** Session 2 found the one named plan, marked Phase 2
  complete, left Phase 3 open, and added to session 1's log. The failed run did the same work
  but wrote Phase 3's status as `not_started`, which is not one of the three values the
  template allows (`pending`, `in_progress`, `complete`), so its regex check failed.
- **PydanticAI left Phase 3 open in every run, but passed only once.** In 3 runs session 2
  rewrote `progress.md` as a new session entry and dropped session 1's entries. Its
  `write_file` tool replaces a whole file, while Claude Code edits a file in place. In 2 runs
  session 2 left Phase 2 `in_progress` because it could not run the tests it wrote: a
  framework runner has no shell, and `run_script` runs only the skill's own scripts. One run
  had both problems.
- **The hooks ran in both sessions.** Each Claude Code run with the skill ran 26 to 31 hooks
  across its two sessions, with `UserPromptSubmit` and `Stop` once per session, and none
  failed.

## How a run is isolated

- Every run gets its own new temporary directory, and the two arms never share one.
- Claude Code starts with `--setting-sources project --strict-mcp-config
  --no-session-persistence`, so the runner's own settings, hooks and MCP servers stay out and
  no session is saved. The skill's frontmatter hooks do run.
- Claude Code runs with permission prompts off, inside skill-lens's product sandbox
  (`product_sandbox = "required"`): `sandbox-exec` on macOS, `bwrap` on Linux. Each case gets
  a temporary home directory, deleted when the case ends, and an environment built from an
  allowlist. Claude Code cannot change files outside its working directory and its own
  temporary directories, and cannot read other skill-lens temporary directories, such as the
  other arm's workspace.
- **It can still read the rest of your files.** The sandbox allows reads anywhere you can
  read, network access (Claude Code must reach its provider), and requests to system services
  that act outside it. Treat it as a guard against accidents, not against a hostile skill.
- The skill's hooks resolve `$HOME` to the temporary home. Their fallback to
  `$HOME/.claude/skills/planning-with-files/` finds nothing there, so a personal install no
  longer affects a run, and the `~/.cache/pwf-pretool` cache is deleted with the case.
- Without a working backend the run stops before it spends anything. `--product-sandbox auto`
  runs it anyway with only the temporary home directory and the allowlist. That keeps personal
  installs out, but it is not a sandbox: Claude Code can then write anywhere you can. The
  report's `claude-code` line says which of the two applied.
- PydanticAI runs the skill's scripts through skill-lens's own sandbox (`sandbox-exec` on
  macOS, `bwrap` on Linux). The configuration requires a working sandbox backend for script
  execution and refuses to fall back to unsandboxed scripts. This framework path needs a
  supported backend; it is unavailable on native Windows. With only Claude Code available,
  use the CLI-only runner and judge configuration above with `--no-allow-scripts`. Sandbox
  preflight otherwise runs even for a CLI-only run.

## What the hooks did

The skill's hooks (`UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `Stop`, `PreCompact`) are
half of how it works, and only Claude Code runs them. Under `claude-code`, skill-lens records
every hook that ran, in order: its name (`PreToolUse:Write`), its outcome, its exit code, and
its stdout and stderr.

- A case that does not pass lists the hooks that did not succeed under `failed hooks:`.
- `--json-output` keeps every hook, successful ones included, under `hooks`.
- Hooks are reported, never checked: a failed hook does not fail a case by itself. Exit code
  `2` is how a hook blocks on purpose, and Claude Code reports it as `error` too.
- PydanticAI does not run the hooks, so a PydanticAI result says nothing about them.

## Budgets

- A `budget:` block is checked after a run ends; it fails the check, it does not stop spending.
- Claude Code stops a run itself at `--max-budget-usd` (`1.00` in `skill-lens.toml`); such a run
  is reported as errored.
- The Claude Code runner's timeout (`timeout_seconds`, 600 here) also stops its run. This
  setting does not impose an overall timeout on the PydanticAI runner.
- `max_spend_usd` (`10.00` in `skill-lens.toml`, or `--max-spend`) caps the whole run. Once
  the recorded spend of every run and every judge call reaches it, no new case starts. Cases
  already running finish, the rest are reported as not run, and the gate fails.
- A PydanticAI run stops at its own default of 50 model requests. A `budget: max_tokens` in a
  case would stop it earlier, while it spends, and fail that case. Neither case sets one, so
  `creates-the-three-files` keeps the same checks as eval 1.

## Known limits

- `resumes-from-its-own-plan` checks the planning files, not the code: it does not test that
  `add` and `list` work. A first session that ignored "then stop" and finished phase 2 itself,
  followed by a second session that changed nothing, could still pass every check.
- Under PydanticAI the agent has no shell, so it cannot run the code it writes. A session that
  wants to test a phase before it marks the phase complete leaves it `in_progress`.
- Hooks are reported, not checked. No case asserts that a hook fired.
- The product sandbox does not stop reads outside the working directory or network access.
- Running the eval costs money (about $6.03 for the 40 runs above) and needs
  `CLAUDE_CODE_OAUTH_TOKEN` (or `ANTHROPIC_API_KEY`) and `OPENAI_API_KEY`.
