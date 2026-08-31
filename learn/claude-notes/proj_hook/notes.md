# Claude Code Hooks

Hooks are shell commands defined in settings.json (user, project, or local) that run on specific events.

## Config location
1. `~/.claude/settings.json`
1. `.claude/settings.json`
1. `.claude/settings.local.json`

## Structure
```json
{
  "hooks": {
    "<EventName>": [
      {
        "matcher": "ToolNamePattern",
        "hooks": [
          { "type": "command", "command": "your-shell-command" }
        ]
      }
    ]
  }
}
```
- `matcher` is optional/only used on tool-related events (regex or exact tool name, e.g. `"Bash"`, `"Edit|Write"`, `"*"` for all).
- Hook receives JSON on stdin with event context; can exit non-zero to block the action (for events that support blocking).

## Events

Per official docs (code.claude.com/docs/en/hooks), there are 31 hook events (not 30 — noting the discrepancy from memory):

| # | Event | Description |
|---|-------|-------------|
| 1 | SessionStart | When a session begins or resumes |
| 2 | Setup | When you start Claude Code with `--init-only`, or with `--init`/`--maintenance` in `-p` mode |
| 3 | UserPromptSubmit | When you submit a prompt, before Claude processes it |
| 4 | UserPromptExpansion | When a user-typed command expands into a prompt, before it reaches Claude |
| 5 | PreToolUse | Before a tool call executes |
| 6 | PermissionRequest | When a tool call needs a permission decision |
| 7 | PermissionDenied | When auto mode denies a tool call |
| 8 | PostToolUse | After a tool call succeeds |
| 9 | PostToolUseFailure | After a tool call fails |
| 10 | PostToolBatch | After a full batch of parallel tool calls resolves |
| 11 | Notification | When Claude Code sends a notification |
| 12 | MessageDisplay | While assistant message text is displayed |
| 13 | SubagentStart | When a subagent is spawned |
| 14 | SubagentStop | When a subagent finishes |
| 15 | TaskCreated | When a task is being created via `TaskCreate` |
| 16 | TaskCompleted | When a task is being marked as completed |
| 17 | Stop | When Claude finishes responding |
| 18 | StopFailure | When the turn ends due to an API error |
| 19 | TeammateIdle | When an agent team teammate is about to go idle |
| 20 | InstructionsLoaded | When a CLAUDE.md or `.claude/rules/*.md` file is loaded into context |
| 21 | ConfigChange | When a configuration file changes during a session |
| 22 | CwdChanged | When the working directory changes |
| 23 | DirectoryAdded | When a working directory is added mid-session |
| 24 | FileChanged | When a watched file changes on disk |
| 25 | WorktreeCreate | When a worktree is being created |
| 26 | WorktreeRemove | When a worktree is being removed |
| 27 | PreCompact | Before context compaction |
| 28 | PostCompact | After context compaction completes |
| 29 | Elicitation | When an MCP server requests user input during a tool call |
| 30 | ElicitationResult | After a user responds to an MCP elicitation |
| 31 | SessionEnd | When a session terminates |

Source: https://code.claude.com/docs/en/hooks (fetched live, not from memory).

## Example: block writes to .env files
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          { "type": "command", "command": "scripts/block-env-writes.sh" }
        ]
      }
    ]
  }
}
```

## Example: log every bash command
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "jq -r '.tool_input.command' >> ~/.claude/bash-history.log" }
        ]
      }
    ]
  }
}
```

## Example: notify on stop
```json
{
  "hooks": {
    "Stop": [
      { "hooks": [ { "type": "command", "command": "notify-send 'Claude finished'" } ] }
    ]
  }
}
```
