# ZCode tool mapping for pstack

pstack skills are written in Claude Code tool language (the `Skill` tool, the `Agent` tool, `AskUserQuestion`, `claude-*` model slugs). On ZCode the skills are the same files; only the resolution differs, and most of it is identity — ZCode ships those tool names itself. Read this when a pstack skill names a Claude tool, a Claude built-in skill, or a `claude-*` model. This file is ZCode-specific. Codex, Gemini CLI, opencode, Prime Agent, and other runtimes must use their own concrete tools, model names, and configuration paths.

## Tool actions

| pstack / Claude action | ZCode equivalent |
|------------------------|------------------|
| Read a file | `Read` |
| Create / edit / delete a file | `Write` (create or fully replace), `Edit` (exact string replace); delete via `Bash` (`rm`) |
| Run a shell command | `Bash` |
| Search file contents / find files | `Bash` (`rg`, `grep`, `find`, `ls`), preferring the dedicated file tools where one fits |
| Fetch a URL | `WebFetch` |
| Search the web | `WebSearch` |
| Invoke a skill (the `Skill` tool, `/command`) | The `Skill` tool, unchanged. User-invocable skills are also slash-invocable — `/tdd` routes to the Skill tool. |
| Dispatch a subagent (the `Agent`/`Task` tool) | `Agent`, unchanged |
| Dispatch N parallel subagents in one turn | N `Agent` calls in one response |
| Wait for a subagent result | The agent's final message returns as the tool result; with `run_in_background: true`, collect it via `TaskOutput` |
| Track tasks (the todolist / `TodoWrite`) | `TodoWrite`, unchanged |
| Ask the human a fixed-choice question (`AskUserQuestion`) | `AskUserQuestion`, unchanged |

One real difference: the ZCode `Agent` tool has no `model` parameter. Every dispatch runs on the session model; there is no per-subagent model selection (see Model names below).

## Subagent policy

poteto-mode's Subagents section sets `subagent_type: "poteto-agent"` and `run_in_background: true`. On ZCode:

- On a plugin install (marketplace or local directory), `poteto-agent` and `comment-sicko` register from the plugin's `agents/` directory, so `subagent_type: "poteto-agent"` works as written.
- On a skills-only install there are no registered subagent types. Dispatch `general-purpose` instead, pointing its instructions at the vendored copy in `poteto-mode/references/agents/poteto-agent.md` (tell it to read that file in full first). The **no-comments** skill's `comment-sicko` dispatch degrades the same way: a `general-purpose` subagent told to read `poteto-mode/references/agents/comment-sicko.md` in full first.
- `run_in_background: true` is supported; drain finished background agents with `TaskOutput`.
- ZCode runs every subagent locally on this machine, so the **swarm** skill's workers and the fan-out playbooks (`orchestrate`, `autopilot-full`, `autopilot-stack`) isolate writers with worktrees, exactly as on Claude Code.
- Keep the rest of the policy unchanged. Pass file pointers not inlined context, give each worker its own worktree or branch when they write, review every subagent's diff yourself.

## Model names

Skills name Claude defaults (a single-role default for code/prose/judgment plus a diverse-model panel for diverse-model panels; each model-consuming skill lists its own in a Models section). These slugs do not resolve on ZCode. Use your configured ZCode models:

- Single-model roles: your configured ZCode model (for example `glm-5.3`).
- Diverse-model panels (`arena`, `architect`, `interrogate`, `how` critics, `reflect`): the ZCode `Agent` tool takes no `model` parameter, so every panelist runs on the session model. Keep the diversity the prompts create (distinct briefs, adversarial angles) and note in the verdict that model diversity was reduced.

`/setup-pstack`'s override sheet is Claude-only; ZCode has no per-role model override mechanism. Put a standing model policy in your `AGENTS.md` instead.

## Claude built-in skills pstack references

Some triggers name skills that ship with Claude Code, not pstack. They do not exist on ZCode. Substitute the behavior:

| Claude built-in named in pstack | On ZCode |
|---------------------------------|----------|
| `run` (drive a CLI/TUI to see a change work) | Drive it yourself via `Bash` and observe the real output. |
| `verify` (drive a UI to confirm a fix) | The official `browser-use:web-gui-tester` plugin drives a real browser; without it, hand the user a concrete manual check. Do not claim done without observing the artifact. |
| `plugin-dev:skill-development` (Claude's SKILL.md authoring guidance) | The official `skill-creator:skill-creator` plugin, or your platform's skill-authoring guidance. Keep `name` + `description` frontmatter and progressive disclosure. |
| `loop` (recurring/self-paced re-invocation, used by `babysit`) | ZCode has no `loop` skill. Re-run the step yourself on a cadence, or schedule a fixed interval with a scheduled automation (`CronCreate`). |

## Vendored scripts

`skills/poteto-mode/scripts/` ships the `watch-pr` PR watcher, the `orch` store CLI, and `worktree-audit.sh`. They are plain bun and bash, so they run the same on ZCode; invoke them through `Bash`. They need `bun`, `gh`, (for stack work) `gt`, and (for `worktree-audit.sh`) `jq` and `rg`. `worktree-audit.sh` reads Claude Code transcripts under `~/.claude/projects/`; ZCode session data lives under `~/.zcode/cli/` (rollout files), so locate that tree at runtime and point the audit at it instead. For loading a prior session's context mid-task, ZCode's `ReadSessionContext` tool reads a past session directly.

## Instructions file

Where a pstack skill says "your instructions file", on ZCode that is `AGENTS.md` (`~/.zcode/AGENTS.md` user scope, `<repo>/AGENTS.md` workspace scope). On Claude Code it is `CLAUDE.md`.
