# api-breaking-change-check

An [Agent Skill](https://agentskills.io): **api breaking-change check**.

Mechanically diff a project's public surface between the base branch (or last release) and HEAD, and classify every change as a semver major, minor, or patch bump. Covers library exports and signatures, HTTP routes and OpenAPI/proto/GraphQL schemas, CLI flags and exit codes, config keys and env vars, DB schema, and event payloads. Flags behavior changes that callers may depend on (Hyrum's Law) and proposes a deprecation path for each break.

## Install

```bash
npx skills add ajwann/api-breaking-change-check
```

Or copy this directory to `~/.claude/skills/api-breaking-change-check/` (Claude Code) or your agent's skills folder.

## Contents

- `SKILL.md`: the instructions the agent loads (the frontmatter `description` says when it triggers)
- `references/tools.md`

## License

MIT
