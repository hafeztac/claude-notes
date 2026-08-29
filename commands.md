# Claude Code

These notes are based on https://claude.nagdy.me, which was sent to my by _Eng. Mostafa_.

`-p` runs prompt non-interactively

Claude commands are grouped into the following categories:
1. **Configuration**
1. **Context management**
1. **Session tools**
1. **Pre-ship reviews**
1. **Diagnostics**

## Configuration
Configuration commands adjust Claude’s behavior mid-session.
1. `/model` — switches between available models such as Sonnet, Opus, Haiku, and other aliases like best or opusplan
1. `/effort` — sets reasoning depth: low, medium, high, xhigh, max, or ultracode; max and ultracode are session-only, and auto resets to the model default
1. `/permissions` — permissions by tool and allowed dirs.
1. `/config` — opens the settings menu
1. `/theme (new in v2.1.118)` — creates and switches between named custom themes. Themes are stored as JSON files in ~/.claude/themes/ and can also be hand-edited directly.
1. `/tui` — switches between classic and flicker-free fullscreen rendering mid-conversation. Also available as a tui setting.
1. `/focus` — toggles focus view for distraction-free input.
1. `/fewer-permission-prompts` — scans your recent transcripts for common read-only Bash and MCP tool calls, then proposes an allowlist to add to .claude/settings.json to reduce future permission prompts.

1. ext management controls how much of the conversation Claude can see.

1. `/context` — shows a visual grid of your context usage
1. `/compact` — compresses the conversation. Pass instructions to control what’s preserved: /compact keep the migration plan, drop the debugging
1. `/clear` — clears conversation context and history

Session tools let you manage and revisit work.

1. `/rename my-feature` — gives the session a readable name
1. `/resume` — picks up a previous session
1. `/branch` — creates a parallel conversation to explore an alternative without losing your current state
1. `/rewind` — rolls back to an earlier point. /undo is an alias for /rewind.
1. `/export` — saves the session to a file or clipboard
1. `/recap` — generates a one-line summary of what happened in the session. Also runs automatically when you return to the terminal after stepping away.

Pre-ship reviews check your work before it leaves your branch.

1. `/code-review` — reviews the current diff for correctness bugs and reports findings without editing files. Accepts an effort level (/code-review high returns broader coverage; /code-review low returns fewer, higher-confidence findings) and --comment to post findings as inline comments on the current GitHub PR. Pass a path or PR reference to review a specific target. This is the bundled skill (formerly /simplify before v2.1.147); /code-review --fix applies the findings to your working tree. From v2.1.154, /simplify is a separate cleanup-only skill that applies fixes without hunting for bugs — use /code-review to find bugs.
1. `/code-review ultra` — runs a deep, multi-agent code review in a cloud sandbox using parallel analysis and critique. With no arguments it reviews your current branch; pass a PR number (/code-review ultra 123) to fetch and review a GitHub PR. /ultrareview is an alias. Pro and Max include a few free runs, after which it draws on usage credits.
1. `/review` — fast single-pass, read-only review of a GitHub pull request (requires the gh CLI). As of v2.1.223 it’s a lighter alias-style entry point to /code-review; reach for /code-review when you want effort levels or fixes.
1. `/security-review` — security-focused review of pending changes

Diagnostics help when something isn’t working.

1. `/cost` — shows session cost, duration, code changes, and token usage
1. `/usage-credits (new in v2.1.144)` — view and manage usage credits on your account. Renamed from /extra-usage; the old name still works.
1. `/status` — shows version, model, and account info
1. `/doctor` — checks installation health
1. `/diff` — opens an interactive viewer for uncommitted changes, useful for reviewing what Claude has done before committing

## Additional commands
1. `/batch` Uses parallelism to order related changes across many files on a large scale; plans work, uses isolated git worktrees and can coordinate verification and PR-oriented follow-up
1. `loop` runs a prompt repeatedly on an interval. e.g: `/loop 5m check deploy status`
1. `/debug` enables verbose logging to diagnose issues with Claude behaviour or tool use.
1. `/fast` is faster at same quality for more tokens. Shows ↯ in prompt bar.
1. `/btw` ask a side question.
1. `/diff` show a nice Claude-made diff
1. `/insights`  generates a session analysis report with statistics on what was accomplished.
1. `/goal` Orders Claude to work autonomously until a completion condition is achieved. Limit turns with CLAUDE_CODE_MAX_TURNS 
