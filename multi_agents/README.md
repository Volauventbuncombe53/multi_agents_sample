# Multi-Agent Bug Workflow

This directory defines a file-based collaboration workflow for four agents:

- `Bug Finder`
- `Bug Identifier`
- `Bug Fixer`
- `Reflector`

All cross-agent communication happens through `./multi_agents/bugs/`.

## WebUI Port Convention

When an agent needs to launch or inspect the WebUI, use port `15373`.

- Base URL: `http://127.0.0.1:15373`
- Editor URL: `http://127.0.0.1:15373/editor`

## Playwright-Interactive Headed Launch

When Finder uses the `playwright-interactive` skill in this environment, it should also use the repo skill `playwright-visible-browser`.

That skill codifies the verified launch recipe for this repo:

- load Playwright through `createRequire(...)` from the workspace `package.json`
- launch headed Chromium with an explicit display override

Use:

```javascript
browser ??= await chromium.launch({
  headless: false,
  env: { ...process.env, DISPLAY: ':0' }
});
```

Reason: in this repo's `js_repl` workflow, `await import('playwright')` can be less reliable than `createRequire(...)`, and the headed browser launch path may not propagate a usable `DISPLAY` automatically. Without those adjustments, Playwright may fail before the investigation even starts.

Do not mark the investigation as blocked until this explicit headed launch has been tried.

## Directory Layout

- `./multi_agents/bug_finder.md` — instructions for the Finder agent
- `./multi_agents/bug_identifier.md` — instructions for the Identifier agent
- `./multi_agents/bug_fixer.md` — instructions for the Fixer agent
- `./multi_agents/reflector.md` — instructions for the Reflector agent
- `./multi_agents/start_agent` — launcher for the workflow agents
- `./multi_agents/bugs/` — shared handoff directory for bug case files
- `./.claude/skills/playwright-visible-browser/` — repo skill for visible Playwright Chromium launch

## Shared Case ID Rule

Every bug case is identified by a timestamp-based case ID:

- `YYYYMMDD_HHMMSS`

This timestamp is the only reliable pairing key across agents.

Example case:

- `20260308_001530_finder.md`
- `20260308_001530_identifier.md`
- `20260308_001530_repair.md`
- `20260308_001530_reflector.md`

## Timestamp Ordering Rule

Report filenames encode the local **date and time** in the `YYYYMMDD_HHMMSS` prefix.

- `YYYY` = year
- `MM` = month
- `DD` = day
- `HHMMSS` = hour, minute, second
- the lexicographically largest eligible case ID is the **most recent** report
- the lexicographically smallest eligible case ID is the **oldest** report

When an instruction says to read the most recent Finder, Identifier, Reflector, or Repair report, determine that from the filename timestamp prefix, not from filesystem modification time.

## One Report Per Agent Run

Each agent run may create **at most one new report file**.

- do not create multiple new `*.md` report files in one run
- if several candidate issues or cases are available, pick the single case required by the stage rules
- if Finder discovers multiple new bugs in one run, write one Finder report for the strongest new case and stop

## Core Rule: Never Read By "Latest File" Alone

Old files are never deleted, so agents must never choose work by “the newest file in the directory” alone.

Every agent must:

1. list all filenames in `./multi_agents/bugs/`
2. group files by case ID
3. decide which case IDs are actionable for its stage
4. select the correct case ID using the stage-specific pending rules
5. read only files belonging to that selected case ID

A file is relevant only if its filename starts with the same case ID.

## Cleanup Rule

Agents must not leave temporary investigation tooling running after handoff.

- If an agent starts a temporary local server for its work, it must stop that server before ending its turn unless the user explicitly asked to keep it running.
- If Finder launches a headed Playwright browser, Finder must close it before ending the turn and kill all process which uses port 15373.
- If a `js_repl` timeout or reset orphaned a Playwright browser process, Finder must explicitly terminate the orphaned Playwright-owned browser process before ending the turn.
- Cleanup is required on every exit path, including success, duplicate detection, blocked tooling, and no-repro.
- After cleanup, the agent should verify that the temporary port/processes it created are no longer left behind.

## Stage-Specific Pending Rules

### Finder

The Finder does **not** process a pending case. Instead, it checks whether a newly observed issue is already covered by an old case.

Finder should:

- if a branch is provided, inspect the commits reachable from that branch but not from its base branch before scanning historical bug files
- treat those branch-only commits as high-priority suspicion areas for regressions
- scan filenames for existing cases
- read only old cases that look semantically similar
- avoid reading unrelated historical cases in full
- create a new case only when the issue is genuinely new

If a prior case already describes the same symptom, trigger path, and feature area, the Finder should treat that as the same case and avoid creating a duplicate file.

### Identifier

A case is `identifier-pending` if:

- `YYYYMMDD_HHMMSS_finder.md` exists
- `YYYYMMDD_HHMMSS_identifier.md` does not exist

Identifier must:

- ignore all cases that already have `*_identifier.md`
- choose the most recent remaining pending case ID by selecting the highest `YYYYMMDD_HHMMSS` case ID
- read only files belonging to that selected case ID

### Fixer

A case is `fixer-eligible` if:

- `YYYYMMDD_HHMMSS_finder.md` exists
- `YYYYMMDD_HHMMSS_identifier.md` exists

Fixer must:

- select the **most recent** eligible case ID by choosing the highest `YYYYMMDD_HHMMSS` case ID
- read only files belonging to that selected case ID
- include `*_reflector.md` in context only if it already exists for the same case ID
- avoid overwriting an existing `*_repair.md` unless explicitly asked to revise that case
- add or update `web_debug` integration coverage and iterate until the focused test passes or the repair is reverted

### Reflector

A case is `reflector-pending` if:

- `YYYYMMDD_HHMMSS_finder.md` exists
- `YYYYMMDD_HHMMSS_identifier.md` exists
- `YYYYMMDD_HHMMSS_reflector.md` does not exist

Reflector must:

- ignore all cases that already have `*_reflector.md`
- choose the most recent remaining pending case ID by selecting the highest `YYYYMMDD_HHMMSS` case ID
- read only files belonging to that selected case ID

## Agent Responsibilities

### 1. Bug Finder

The Finder explores the WebUI and looks for bug candidates.

It must:

- use the `playwright-interactive` skill to investigate the WebUI
- use the `playwright-visible-browser` skill to bootstrap headed Chromium for Finder
- when using `playwright-interactive`, keep the browser visible so the user can watch the investigation
- when launching headed Chromium from `js_repl`, pass `env: { ...process.env, DISPLAY: ':0' }`
- if a branch is provided, inspect its branch-only commit history before opening the WebUI
- scan existing files in `./multi_agents/bugs/` before opening a new case
- avoid duplicating already reported cases
- create a new detailed finder report only for a genuinely new candidate bug

The Finder creates:

- `./multi_agents/bugs/YYYYMMDD_HHMMSS_finder.md`
- at most one new Finder report file in a single run

The Finder should provide:

- clear reproduction steps
- expected vs actual behavior
- evidence
- suspected fault regions
- similar existing cases checked
- open questions for the Identifier

### 2. Bug Identifier

The Identifier decides whether a Finder report is a real bug.

It must:

- compute the `identifier-pending` case list
- choose the most recent pending case ID by selecting the highest `YYYYMMDD_HHMMSS` case ID
- read only that case's files
- inspect the suggested fault regions
- verify whether the issue is truly a bug
- classify the bug type
- define trigger conditions
- identify the root cause
- assess the impact scope

The Identifier creates:

- `./multi_agents/bugs/YYYYMMDD_HHMMSS_identifier.md`
- exactly one Identifier report file for the selected case in a single run

The Identifier should output one of these statuses:

- `confirmed-bug`
- `not-a-bug`
- `needs-more-evidence`

### 3. Bug Fixer

The Fixer turns the most recent eligible confirmed bug into a tested repair.

It must:

- compute the `fixer-eligible` case list
- choose the most recent eligible case ID
- read the selected case files only
- modify the code to fix the confirmed root cause
- add or update focused tests under `./tests/integration/web_debug/`
- rerun the focused verification test until it passes or the repair is abandoned
- revert all code changes if the bug cannot be solved after multiple meaningful attempts
- create a git commit only when the user provided a branch before launch
- include WebUI verification steps in the commit message when committing
- always record the outcome in `./multi_agents/bugs/`

The Fixer creates:

- `./multi_agents/bugs/YYYYMMDD_HHMMSS_repair.md`
- exactly one Repair report file for the selected case in a single run

The Fixer should output one of these statuses:

- `fixed-tested`
- `fixed-tested-committed`
- `failed-reverted`
- `blocked`

### 4. Reflector

The Reflector reviews the full case lifecycle and extracts reusable lessons.

It must:

- compute the `reflector-pending` case list
- choose the most recent pending case ID by selecting the highest `YYYYMMDD_HHMMSS` case ID
- read only that case's files
- read the repair file if one exists for the same case ID
- analyze where the process was efficient or inefficient
- summarize lessons and process improvements
- propose better heuristics for future runs

The Reflector creates:

- `./multi_agents/bugs/YYYYMMDD_HHMMSS_reflector.md`
- exactly one Reflector report file for the selected case in a single run

## File Handoff Rules

### Finder read/create rules

Finder reads:

- if a branch is provided, the commits reachable from that branch but not from its base branch
- filenames of all existing files in `./multi_agents/bugs/`
- full contents only for old cases that look similar to the current issue
- matching `*_identifier.md`, `*_repair.md`, and `*_reflector.md` only for those similar case IDs

Finder creates:

- one new `./multi_agents/bugs/YYYYMMDD_HHMMSS_finder.md` only when the issue is genuinely new
- never more than one new Finder report in the same run

### Identifier read/create rules

Identifier reads:

- only files for the selected `identifier-pending` case ID
- `./multi_agents/bugs/YYYYMMDD_HHMMSS_finder.md` is required
- `./multi_agents/bugs/YYYYMMDD_HHMMSS_repair.md` is optional
- `./multi_agents/bugs/YYYYMMDD_HHMMSS_reflector.md` is normally ignored unless explicitly needed

Identifier creates:

- one `./multi_agents/bugs/YYYYMMDD_HHMMSS_identifier.md`
- never more than one new Identifier report in the same run

Selection rule:

- build the pending list from case IDs, not from modification times
- choose the most recent case ID that has finder but no identifier by taking the highest `YYYYMMDD_HHMMSS` prefix
- ignore all completed identifier cases

### Fixer read/create rules

Fixer reads:

- only files for the selected most-recent `fixer-eligible` case ID
- `./multi_agents/bugs/YYYYMMDD_HHMMSS_finder.md`
- `./multi_agents/bugs/YYYYMMDD_HHMMSS_identifier.md` from that most recent eligible case
- `./multi_agents/bugs/YYYYMMDD_HHMMSS_reflector.md` if present for that same case ID
- `./multi_agents/bugs/YYYYMMDD_HHMMSS_repair.md` only if explicitly asked to continue or revise that same case

Fixer creates:

- one `./multi_agents/bugs/YYYYMMDD_HHMMSS_repair.md`
- never more than one new Repair report in the same run

Selection rule:

- build the eligible list from case IDs, not from modification times alone
- choose the most recent case ID that has finder and identifier by taking the highest `YYYYMMDD_HHMMSS` prefix
- ignore unrelated historical cases after the target case ID is selected
- do not overwrite an existing repair file unless explicitly asked to revise that same case

### Reflector read/create rules

Reflector reads:

- only files for the selected `reflector-pending` case ID
- `./multi_agents/bugs/YYYYMMDD_HHMMSS_finder.md`
- `./multi_agents/bugs/YYYYMMDD_HHMMSS_identifier.md`
- `./multi_agents/bugs/YYYYMMDD_HHMMSS_repair.md` if present

Reflector creates:

- one `./multi_agents/bugs/YYYYMMDD_HHMMSS_reflector.md`
- never more than one new Reflector report in the same run

Selection rule:

- build the pending list from case IDs, not from modification times
- choose the most recent case ID that has finder and identifier but no reflector by taking the highest `YYYYMMDD_HHMMSS` prefix
- ignore all completed reflection cases

## Suggested Lifecycle

1. Finder discovers a candidate issue and writes a finder report.
2. Identifier reads the most recent pending Finder case and confirms or rejects it.
3. Fixer reads the most recent eligible confirmed case, attempts a repair, adds verification, and writes `*_repair.md`.
4. Reflector reads the most recent pending reviewed case and writes lessons learned.

## Naming and Safety Rules

- Do not overwrite old case files.
- Keep one case per timestamp prefix.
- Keep all communication inside `./multi_agents/bugs/`.
- Treat observations, confirmation, repair, and reflection as separate stages.
- Use the same case ID across all files for the same bug.
- Never combine files from different case IDs into one analysis.
- Never pick a target file by “latest file overall” alone.

## Quick Start

- Start Finder with `./multi_agents/start_agent finder`
- Narrow Finder with `./multi_agents/start_agent finder 'Find bugs about Execution details page'`
- Continue Identifier with `./multi_agents/start_agent identifier`
- Run Fixer with `./multi_agents/start_agent fixer <branch_name>`
- Finish Reflector with `./multi_agents/start_agent reflector`

Fixer branch behavior:

- `fixer` requires a branch name
- if the branch exists locally, `start_agent` checks it out before Codex starts
- if the branch exists on `origin` only, `start_agent` creates a local tracking branch and checks it out
- if the branch does not exist, `start_agent` creates it and checks it out
- if no branch is provided, Fixer should not be started through `start_agent`

## Automation

- Run `./multi_agents/auto_agents "<bug description>"` to keep looping through finder → identifier → DeepSeek judgment → reflector → fixer.
- Run `./multi_agents/auto_agents "<bug description>" <rounds>` to stop after a fixed number of rounds.
- `auto_agents` uses `DEEPSEEK_API_KEY` and `DEEPSEEK_BASE_URL` to derive the fixer branch name and to judge whether the current case should proceed to repair.
