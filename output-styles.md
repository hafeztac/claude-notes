#   Output Styles

Output style changes how Claude responds without changing its skills to set:
- the tone
- the role
- the format

Output style is chosen via `/config -> Output style` or settings.local.json. Changes take effect after `/clear` or a new session because output style is part of the system prompt.

Available built-in styles are:
1. Default
1. Proactive
1. Explanatory
1. Learning

##  Default

Default style is the standard software engineering assistant.

## Proactive

Proactive style executes commands more often, and it pauses to ask the user questions less often. It makes more assumptions.

##  Explanatory

Explanatory style adds educational hints between coding steps.

##  Learning

Learning style is a collaborative coding mode which marks strategic features with `TODO(human)` for the user to implement them himself.

##  Custom

Custom output styles can be defined globally (`~/.claude/output-styles/`) or on the project level (`.claude/output-styles/`)

