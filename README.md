# Multi-Agents sample for WebUI debugging

This repository contains a reusable `multi_agents/` workflow for multi-agent WebUI debugging with Codex.

## How to use this sample in another project

1. Put the `multi_agents/` directory in the root of the project you want to debug. Put playwright-visible-browser in the codex skills directory (`~/.codex/skills/` or `/path/to/project/.codex/skills/`)
2. `cd` to that project root. Run the scripts from the project root, not from inside `multi_agents/`.
3. **(Important)** Review and adapt the prompt files in `./multi_agents/` for the target project before the first run. In particular, update any project-specific assumptions such as:
   - WebUI URLs and port (`http://127.0.0.1:15373` by default)
   - repo-specific skills such as `playwright-visible-browser`
   - test locations such as `./tests/integration/web_debug/`
   - source-code hints
4. Start the project under development so the WebUI is reachable by the workflow.
5. Run the agents manually with `./multi_agents/start_agent` or use `./multi_agents/auto_agents` for full loop. (Refer to [Manual workflow](#manual-workflow) and [Automatic workflow](#automatic-workflow) sections)

## What `multi_agents/` contains

- `./multi_agents/start_agent` - starts one Codex role in a tmux-backed session
- `./multi_agents/auto_agents` - runs the full finder -> identifier -> judgment -> reflector -> fixer loop
- `./multi_agents/bug_finder.md` - instructions for the Finder agent
- `./multi_agents/bug_identifier.md` - instructions for the Identifier agent
- `./multi_agents/bug_fixer.md` - instructions for the Fixer agent
- `./multi_agents/reflector.md` - instructions for the Reflector agent
- `./multi_agents/bugs/` - shared handoff directory for generated case files

## Prerequisites

- Environment: MacOS/Linux. Do not modify this auto flow to use in Windows which is dangerous!
- `codex` is installed and available in `PATH`
- `tmux` is installed
- `git` is available
- Codex skill `playwright-interactive` is install
- the target WebUI can be started locally
- for the default sample prompts, the target WebUI is available at `http://127.0.0.1:15373`(You can modify instructions to change ports)
- if you want to use `./multi_agents/auto_agents`, set:
  - `DEEPSEEK_API_KEY`
  - `DEEPSEEK_BASE_URL`

## Manual workflow

Use `./multi_agents/start_agent` when you want to drive each stage yourself.

```bash
./multi_agents/start_agent finder [bug_focus_prompt]
./multi_agents/start_agent identifier
./multi_agents/start_agent reflector
./multi_agents/start_agent fixer <branch_name>
```

Notes:

- `finder` accepts an optional natural-language prompt to narrow the bug hunt.
- `fixer` requires a branch name; the script checks out that branch first and creates it if it does not exist.
- agents communicate only through files in `./multi_agents/bugs/`.

Example:

```bash
./multi_agents/start_agent finder "Check the execution details page for UI regressions"
./multi_agents/start_agent identifier
./multi_agents/start_agent reflector
./multi_agents/start_agent fixer fix/execution-details-page
```

## Automatic workflow

Use `./multi_agents/auto_agents` when you want the sample to keep iterating for you.

```bash
./multi_agents/auto_agents "Please fix the execution details page bug" [rounds]
```

Examples:

```bash
./multi_agents/auto_agents "Please fix Execution details page bug"
./multi_agents/auto_agents "Please fix Execution details page bug" 5
```

Behavior:

1. runs Finder
2. runs Identifier
3. asks DeepSeek whether the case should be solved now or skipped
4. if DeepSeek returns `SOLVE`, runs Reflector
5. if DeepSeek returns `SOLVE`, runs Fixer on an auto-generated branch name

If `rounds` is omitted, the script keeps running until you stop it.

## Output files

After one round of agents bug fix, you can use `git log` to see commit message on what has been changed. Or you can see detailed reports in `./multi_agents/bugs/YYYYMMDD_HHMMSS_*.md`.

Each bug case uses a shared timestamp case ID: `YYYYMMDD_HHMMSS`.

The workflow writes reports:

- `./multi_agents/bugs/YYYYMMDD_HHMMSS_finder.md`
- `./multi_agents/bugs/YYYYMMDD_HHMMSS_identifier.md`
- `./multi_agents/bugs/YYYYMMDD_HHMMSS_repair.md`
- `./multi_agents/bugs/YYYYMMDD_HHMMSS_reflector.md`

Files with the same timestamp prefix belong to the same case.

## Important notes

- `./multi_agents/start_agent` launches Codex in a tmux session named `codexrun`.
- the launcher starts Codex from the repository root, not from `./multi_agents/`.
- the current `start_agent` script launches `codex --sandbox danger-full-access`; review that before using it in another repo.
- the default sample assumes a WebUI debugging workflow and port `15373`.
- if your repo does not have the sample's referenced skills or test paths, update the prompt files before using it.
- agent completion is coordinated with the temporary file `./multi_agents/signal`.
- Currently, codex does not support using playwright with direct command. So we use tmux instead.
