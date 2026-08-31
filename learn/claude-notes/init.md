#   Setting up Claude Code for a Project

## `/init`
Scans your codebase for:
1. package.json
1. existing docs
1. directory structure
and generates a CLAUDE.md that captures your tech stack, key commands, and initial conventions.

## Settings

`env`               sets env vars
`model`             sets model
`agent`             sets custom default agent
`claudeMdExcludes`  Excludes MD files


## CLAUDE.md Best Practices
A good CLAUDE.md is 
1. concise
1. specific
1. (prefereably) under 200 lines per file.
1. relevant to nearly every session

if something only matters for one feature, put it in a path-scoped rules file instead.

The most valuable sections are: tech stack and versions, development commands (install, test, build, lint), naming conventions that aren’t obvious, and known gotchas that would trip up a new developer:

```md
# Project: Payment Service

## Stack

- Node.js 20, TypeScript 5, PostgreSQL 15
- Express for API, Prisma for ORM, Jest for tests

## Commands

- `npm run dev` — start with hot reload
- `npm test` — run test suite
- `npm run migrate` — apply pending migrations
- `npm run lint` — ESLint + Prettier check

## Conventions

- All monetary values stored as integers (cents)
- Use `Result<T, E>` pattern for error handling, never throw in service layer
- Database columns: snake_case; TypeScript: camelCase
```


