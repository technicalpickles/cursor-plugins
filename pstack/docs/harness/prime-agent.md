# pstack in prime-agent

pstack skills use Cursor names. In [prime-agent](https://github.com/PrimeIntellect-ai/prime-agent), read this file once per session before you follow a pstack skill, then map each Cursor name with the tables below. `<pstack>` means the folder that holds `skills/` and `docs/`. If a Cursor field has no row, drop it and say so.

## Install

prime-agent has no plugin installer, so pstack runs from a checkout.

1. Clone this repository. A sparse checkout of `pstack` is enough: `git clone --filter=blob:none --sparse https://github.com/cursor/plugins ~/.prime/agent/vendor/cursor-plugins`, then `git -C ~/.prime/agent/vendor/cursor-plugins sparse-checkout set pstack`.
2. Link each `<pstack>/skills/<name>` folder into `~/.prime/agent/skills/<name>`. Every skill but `setup-pstack` sets `disable-model-invocation`, so they stay out of the system prompt and run as `/skill:<name>`, such as `/skill:poteto-mode`.
3. Add a line to `~/.prime/agent/AGENTS.md`, which prime-agent loads into every session: "pstack lives at `<pstack>`. Before you follow a pstack skill, read `<pstack>/docs/harness/prime-agent.md`." Write the real path.
4. Run `/skill:setup-pstack` to write the model rule.

Update with `git pull` in the checkout, then link any new skill folders.

## Tools

| Cursor | prime-agent |
| --- | --- |
| `Task` | `await rlm.spawn(prompt, name=..., model=..., thinking=...)` in the Python REPL. Keep the handle. Results arrive only by `agent_message` or files, so the brief tells the child to reply with `await agent_message.send(..., receiver_role="parent")` or to write a file you name. |
| `AskQuestion` | Ask in plain text with numbered options, then end the turn. |
| Open a todolist | Keep the list in your reply and update it at each phase. |
| Run another pstack skill (`/name` or "the **name** skill") | Read `<pstack>/skills/<name>/SKILL.md` and follow it. pstack skills are hidden from the skill catalog, so a missing catalog entry does not mean pstack is absent. When you tell the user to type it, write `/skill:<name>`. |
| `Read`, `Grep`, `Glob`, `Shell` | Python file reads, `bash("rg ...")`, and `bash(...)`. |
| Wait for a background `Task` | End the turn and wait for the `agent_message` notification. `rlm.collect(...)` gives a status snapshot. |

## Subagent fields

| Cursor | prime-agent |
| --- | --- |
| `subagent_type: "poteto-agent"` | Start the brief with: "You are a poteto-mode delegate. Before any work, read `<pstack>/agents/poteto-agent.md`, `<pstack>/skills/poteto-mode/SKILL.md`, and `<pstack>/docs/harness/prime-agent.md` in full." Write the real paths. |
| `subagent_type: "comment-sicko"` | Start the brief with: "Read `<pstack>/agents/comment-sicko.md` and act as that agent." |
| `subagent_type: generalPurpose` | Plain `rlm.spawn`. |
| `readonly: true` | Add "Do not edit files." to the brief, and check `git status` afterward. |
| `environment: "cloud"` | Make a git worktree for the worker and pass its path in the brief. |
| `environment: "local"`, `run_in_background: true` | Omit. Subagents always run in the background. |
| `cloud_base_branch: <b>` | Tell the worker to check out `<b>` first. |
| `resume`, or a follow-up to a running agent | `await agent_message.send(msg, receiver_role="child", receiver_name=handle.name)`. A finished child stays reachable the same way. |
| `file_attachments` | List the paths in the brief. |

## Models

| Cursor | prime-agent |
| --- | --- |
| `claude-<family>-<version>-<effort>[-fast]`, such as `claude-opus-5-5-xhigh` | `model="anthropic/claude-opus-5-5", thinking="xhigh"`. Drop `-fast`. `thinking` takes `off`, `minimal`, `low`, `medium`, `high`, `xhigh`, or `max`. |
| `gpt-<version>-<effort>`, such as `gpt-5.5-xhigh` | `model="openai/gpt-5.5", thinking="xhigh"`. |
| A `grok-*` slug | Use the selector `rlm.find_models("grok")` returns. When it returns none, use the role's line in `pstack-models.md`. Without one, omit `model` and say so. |
| `auto` or `inherit-parent` | Omit `model` and `thinking`. |
| The `pstack-models.mdc` rule, `~/.cursor/rules/pstack-models.mdc` | `~/.prime/agent/pstack-models.md`. Read it before any spawn. Each role value is `<selector> <thinking>`, such as `openai/gpt-5.5 xhigh`. A missing file or role line means the skill default, mapped with this table. |

## setup-pstack

Follow its steps with these changes.

- Detect. The detected models are the selectors `await rlm.find_models(query, limit=20)` returns. Query by family (`claude`, `gpt-5`, `grok`), because results cap at 20. Write a real model as `<selector> <thinking>`, such as `anthropic/claude-opus-5-5 xhigh`. The effort is the last token, so the budget step works as written.
- Defaults. Map each default slug with the rows above. `claude-opus-5-5-xhigh` is `anthropic/claude-opus-5-5 xhigh`. When no grok is detected, a grok default needs a choice. A model from another family keeps panels diverse.
- Write. Write `~/.prime/agent/pstack-models.md` with no frontmatter, then the comment lines, the `# budget` line, and one line per role from step 5. Overwrite the whole file.

## Paths

| Cursor | prime-agent |
| --- | --- |
| `.cursor/skills/`, `~/.cursor/skills/` | `.prime/agent/skills/`, `~/.prime/agent/skills/` |
| `~/.cursor/plugins/`, `.cursor-plugin/plugin.json` | `<pstack>`, `<pstack>/plugin.json` |
| `.cursor/rules/*.mdc`, always-applied rules | `AGENTS.md` or `CLAUDE.md` in the project, or `~/.prime/agent/AGENTS.md` for every session. prime-agent never reads `.mdc` files. |
| `https://github.com/cursor/plugins/blob/main/pstack/<path>` | `<pstack>/<path>` when that file exists. Read the local copy. |
| Transcripts in `~/.cursor/projects/<slug>/agent-transcripts/` | `~/.prime/agent/sessions/*.jsonl`. The system prompt names the current session's log. Subagent logs live under `~/.prime/agent/session-artifacts/<session>/`. |

## Cursor-only features

prime-agent has no Cursor Automations, Bugbot, Cursor cloud agents, or built-in PR tool. When a step needs one, put one line in your reply, `Skipped in prime-agent: <step>. <reason>.`, then go on to the next step. Do not stop, ask, or retry.

| Cursor | prime-agent |
| --- | --- |
| Cursor Automations | Skip with the line. For a recurring wake inside a live session, `rlm_heartbeat.create` is the closest match. |
| Bugbot | Bugbot comments on a GitHub PR are ordinary review threads, so triage them with the forge command that the step gives. Skip a step that starts, waits for, or configures Bugbot. |
| Cursor cloud agents | `environment: "cloud"` maps to a worktree, as in Subagent fields. Skip a step that needs the cloud agent itself. |
| A built-in PR tool | Use the forge command that the step gives, such as `gh pr create`. That is not a skip, so write no line. |
