# Claude Code Skills

Skills are packaged instructions invoked via `/skill-name` or auto-loaded when prompt matches description.

## Location
`~/.claude/skills/<skill-name>/SKILL.md` (user-level)
`.claude/skills/<skill-name>/SKILL.md` (project-level)

## Frontmatter fields
(All optional; only `description` recommended. Source: code.claude.com/docs/en/skills, fetched live.)

| Field | Description |
|---|---|
| `name` | Display name in skill listings. Defaults to directory name. |
| `description` | What the skill does and when to use it — used to decide auto-invocation. If omitted, uses first markdown paragraph. Combined with `when_to_use`, truncated at 1,536 chars. |
| `when_to_use` | Extra trigger phrases/examples appended to `description`. |
| `argument-hint` | Autocomplete hint for expected args, e.g. `[issue-number]`. |
| `arguments` | Named positional args for `$name` substitution. Space-separated string or YAML list. |
| `disable-model-invocation` | `true` → Claude can never auto-load it; only `/name` invokes it. Also blocks preloading into subagents and scheduled-task auto-run. Default `false`. |
| `user-invocable` | `false` → only Claude can invoke it; hidden from `/` menu, `/name` won't run it. For background-only knowledge. Default `true`. |
| `allowed-tools` | Tools pre-approved without asking, for the invoking turn only. |
| `disallowed-tools` | Tools removed from the pool while this skill is active. |
| `model` | Model override while this skill is active (or subagent model, with `context: fork`). |
| `effort` | Effort override (`low`/`medium`/`high`/`xhigh`/`max`) while active. |
| `context` | `fork` → runs in a forked subagent context. |
| `agent` | Subagent type to use when `context: fork`. |
| `background` | With `context: fork`, `false` waits for the result instead of running in background. Default `true`. |
| `hooks` | Hooks registered when the skill is invoked, active for rest of session. |
| `paths` | Glob patterns limiting when the skill auto-activates (by file being worked on). |
| `shell` | Shell for `!\`command\`` injection: `bash` (default) or `powershell`. |
| `metadata` | Free-form YAML map for your own tooling; Claude Code ignores it. |
| `license` | Agent Skills spec field; Claude Code accepts but doesn't act on it. |
| `compatibility` | Agent Skills spec field (env requirements); accepted, not acted on. |

Note: outside Claude Code (claude.ai uploads, Skills API) only 6 fields are valid: `name`, `description`, `license`, `compatibility`, `metadata`, `allowed-tools`.

## Mechanism to prevent direct user invocation
Two separate frontmatter fields control *who* can invoke a skill — they are opposite directions, easy to confuse:

- **`disable-model-invocation: true`** — Claude is blocked from auto-invoking; only the user can run it via `/name`. Use for side-effecting actions (`/deploy`, `/commit`) you want to control the timing of.
- **`user-invocable: false`** — the user is blocked from direct invocation; hidden from the `/` menu, `/name` won't run it. Only Claude can invoke it. Use for background reference knowledge that isn't a meaningful user action.

## Body
Plain markdown instructions Claude follows when the skill is invoked.

## Available string substitutions
| Variable | Description |
|---|---|
| `$ARGUMENTS` | All arguments passed at invocation. |
| `$ARGUMENTS[N]` | Specific argument by 0-based index. |
| `$N` | Shorthand for `$ARGUMENTS[N]`, e.g. `$0`, `$1`. |
| `$name` | Named argument declared in the `arguments` frontmatter list. |
| `${CLAUDE_SESSION_ID}` | Current session ID. |
| `${CLAUDE_EFFORT}` | Current effort level. |
| `${CLAUDE_SKILL_DIR}` | Directory containing this skill's SKILL.md. |
| `${CLAUDE_PROJECT_DIR}` | Project root directory. |
| `${CLAUDE_PLUGIN_ROOT}` | Plugin's install directory (plugin skills only). |
| `${CLAUDE_PLUGIN_DATA}` | Plugin's persistent data directory (plugin skills only). |

## Example: minimal skill
```markdown
---
name: hello
description: Print a friendly greeting
---

Say hello to the user in a friendly, brief way.
```

## Example: skill with arguments and tool restriction
```markdown
---
name: deploy
description: Deploy the current branch to staging
argument-hint: [environment]
allowed-tools: Bash
---

Deploy the current branch to the environment given in $1 (default: staging).
Run the deploy script: `./scripts/deploy.sh $1`
Report success/failure clearly.
```

## Example: skill that only runs on explicit invocation
```markdown
---
name: release-checklist
description: Run the pre-release checklist
disable-model-invocation: true
---

1. Check CHANGELOG.md is updated
2. Run full test suite
3. Confirm version bump in package.json
4. Tag the release
```
