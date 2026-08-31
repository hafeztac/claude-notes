# Claude Memory

## Memory Files
In the following order with subsequent files overriding preceding ones:

1. Managed Policy:                                          Org-wide
1. User instructions:                                       ~/.claude/CLAUDE.md
1. Project instructions:                                    ./CLAUDE.md
1. Local project instructions. Should be gitignored:        ./CLAUDE.local.md

## Initialising project instructions:
Run `/init` to initialise project instructions.

## Imports

- Any line like "@README.md" inside an instructions file gets expanded and its contents loaded into context, costs context tokens.
- Max import depth = 4.
- Permission is asked at first import from outside project path.

## Auto Memory

Auto memory is a dir `~/.claude/projects/<project>/` where Claude writes its own notes during sessions on these self-reported matters:
- project-specific behaviours
- debugging insights
The first 200 lines/25KB of `~/.claude/projects/<project>/memory/MEMORY.md` load automatically at session start.
- Subagents have their own auto memory in `~/.claude/agents/`
- Can be disabled with claude command: `/memory`, or start a session with auto memory disabled: `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1 claude`

## .memignore
In large monorepos with many CLAUDE.md files, use `claudeMdExcludes` in settings to skip irrelevant ones:
```
{
  "claudeMdExcludes": ["packages/legacy-app/CLAUDE.md", "vendors/**/CLAUDE.md"]
}
```
in project ./.claude/settings.json
or project local ./.claude/settings.local.json
or global ~/.claude/settings.json

