# pstack in prime-agent

pstack skills use Cursor names. In [prime-agent](https://github.com/PrimeIntellect-ai/prime-agent), read this file once per session before you follow a pstack skill, then map each Cursor name with the tables below. `<pstack>` means the folder that holds `skills/` and `docs/`. If a Cursor field has no row, drop it and say so.

## Install

prime-agent has no plugin installer, so pstack runs from a checkout.

1. Clone the repository and branch you intend to run. For this fork: `git clone --filter=blob:none --sparse --branch pstack-prime-agent-adapter https://github.com/technicalpickles/cursor-plugins.git ~/.prime/agent/vendor/cursor-plugins`, then `git -C ~/.prime/agent/vendor/cursor-plugins sparse-checkout set pstack`. For upstream, use `https://github.com/cursor/plugins` and its default branch once it includes this adapter. An existing checkout or worktree also works; set `<pstack>` to its `pstack/` folder. Check `git remote -v` and `git branch --show-current` in that checkout before linking.
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

Before every pstack spawn, including ad-hoc helpers, critics, experiments, and delegates of delegates:

1. Read `~/.prime/agent/pstack-models.md` and resolve the role before searching the catalog. Use the calling skill's role; otherwise use `swarm workers` for code or experiments, `judgment and prose` for critics or reviews, and `hardest tasks` for the hardest changes. Pass the resolved selector as `model` and its effort as `thinking`. Aliases omit both. Include this adapter and the role in delegate briefs so nested spawns follow the same policy.
2. Keep the configured model, adjusting only reasoning per poteto-mode. For a requested model comparison, name each arm explicitly before launching. Announce any other model substitution and its reason before spawning; catalog order is not a reason to change a role's model.
3. Validate a real selector with an exact match in `await rlm.find_models(selector, limit=20)`. Search results are capped matches, not a newest-first ranking or a complete catalog. A full page does not establish that later versions are unavailable. When the user asks to update models, use targeted version queries or ask which version they want, then confirm the role changes through setup-pstack. Keep existing choices until confirmed.
4. If a resolved selector is unavailable or a spawn rejects its reasoning level, report the failure and ask for a replacement model or effort. This overrides the skills' closest-model and parent-model fallbacks. Never silently downgrade a version or erase a panel's family diversity.

| Cursor | prime-agent |
| --- | --- |
| `claude-<family>-<version>-<effort>[-fast]`, such as `claude-opus-5-5-xhigh` | `model="anthropic/claude-opus-5-5", thinking="xhigh"`. Drop `-fast`. `thinking` takes `off`, `minimal`, `low`, `medium`, `high`, `xhigh`, or `max`. |
| `gpt-<version>-<effort>`, such as `gpt-5.5-xhigh` | `model="openai/gpt-5.5", thinking="xhigh"`. |
| A `grok-*` slug | Use the configured role first. For an unconfigured default, discover and confirm an exact Grok selector. If none is available, ask for a replacement through setup-pstack. |
| `auto` or `inherit-parent` | Omit `model` and `thinking`. |
| The `pstack-models.mdc` rule, `~/.cursor/rules/pstack-models.mdc` | `~/.prime/agent/pstack-models.md`. Read it before any spawn. Each role value is `<selector> <thinking>`, such as `openai/gpt-5.5 xhigh`. A missing file or role line means the skill default, mapped with this table. |

## setup-pstack

Follow its steps with these changes.

- Detect. Use `await rlm.find_models(query, limit=20)` to discover selectors. Start with the configured and default selectors, then family queries (`claude`, `gpt`, `grok`) for alternatives. Use targeted queries when a page fills; the result order does not rank models by age or quality. Require an exact selector match before offering a real model as available.
- Budget and validate (replaces the slug-membership checks in steps 3(b), 3(c), and 4; keep the role table and user confirmation). Split each real role entry into `<selector> <thinking>`, such as `openai/gpt-5.5 xhigh`. Validate only the selector against discovery. Apply the selected budget to the separate `thinking` field: `unlimited` → `max`, `large` → `xhigh`, `medium` → `high`, `small` → `medium`. Preserve existing model selectors and panel lists; aliases remain unchanged. RLM accepts the effort tokens `off`, `minimal`, `low`, `medium`, `high`, `xhigh`, and `max`, but discovery does not report which a model supports. Do not infer effort support or choose an older model from the discovery list. If model-specific support is unknown, say so at confirmation; handle a later rejection as described under Models. A combined value such as `openai/gpt-5.5 xhigh` is never compared with the selector-only detected set.
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
