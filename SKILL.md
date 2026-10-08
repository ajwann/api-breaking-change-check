---
name: api-breaking-change-check
description: Mechanically diff a project's public surface between the base branch (or last release) and HEAD, and classify every change as a semver major, minor, or patch bump. Covers library exports and signatures, HTTP routes and OpenAPI/proto/GraphQL schemas, CLI flags and exit codes, config keys and env vars, DB schema, and event payloads. Flags behavior changes that callers may depend on (Hyrum's Law) and proposes a deprecation path for each break. Use before merging or releasing, when asked "is this a breaking change", "what version bump does this need", "will this break clients/consumers/users", "write release notes", or when reviewing a PR that renames, removes, retypes, or changes defaults of anything outside the module, even if the user only says "review my branch" for a library, SDK, CLI, or service with external callers.
---

# API breaking-change check

A design review asks "is this API good?". This skill asks a narrower,
mechanical question: **what did this change do to callers who exist
today?** Answer it by diffing the surface, not by reading the diff and hoping
to spot problems. A renamed kwarg in the middle of a 600-line refactor is
easy to miss by eye and trivial for a diff tool.

## 1. Decide what the public surface is

List which of these the project exposes. Each is a separate contract:

- **Library API:** exported modules, functions, classes, types, constants;
  parameter names (callers use kwargs), defaults, return types, raised
  errors.
- **HTTP/RPC:** routes, methods, request and response fields, status codes,
  auth requirements, pagination, error shapes.
- **CLI:** commands, flags, positional args, exit codes, stdout formats that
  scripts parse.
- **Config and env:** keys, env var names, defaults, file locations.
- **Data:** DB schema that other services read, migrations, files on disk,
  cache keys, serialized formats.
- **Events:** queue and topic names, payload schemas, webhook bodies.

Anything named `_private`, `internal/`, or `@internal` is excluded only if
callers really can't reach it. If it has been reachable for years, someone
probably depends on it.

## 2. Diff each surface against the base

Base = `origin/main` for a PR, or the last release tag for a release
(`git describe --tags --abbrev=0`). Use a real diff tool where one exists.
`references/tools.md` has commands per surface and ecosystem: oasdiff,
buf, griffe, cargo-semver-checks, gorelease, `.d.ts` diffing, `--help`
diffing, and migration review. Use `git worktree add` to build the base side
by side.

When no tool fits, do it by hand, but systematically: list the exported
symbols and signatures on both refs and diff the two lists. Don't read the
code diff and pick out changes from memory.

## 3. Classify every change

| Change | Bump |
|---|---|
| Remove or rename anything public (symbol, route, field, flag, env var, config key) | **major** |
| Add a required parameter, field or flag; make an optional one required | **major** |
| Narrow an accepted type or value set; widen a returned type or value set (e.g. a new enum value clients must handle) | **major** (the enum case is often treated as minor; say which and why) |
| Change a default, a unit, a format, the error raised, or a status code | **major** (behavioral) |
| Reorder positional parameters | **major** |
| DB: drop, rename, narrow a type, or add NOT NULL without a default | **major** for anyone reading the table |
| Add an optional parameter, field, route, flag or export | **minor** |
| Deprecate (still works, warns) | **minor** |
| Internal-only change, doc fix, perf with identical behavior | **patch** |

For 0.x versions, many ecosystems treat a minor bump as breaking. Say which
convention the project follows.

**Hyrum's Law check:** also list *behavior* changes that aren't part of
the declared contract but that callers can observe: ordering of results,
timing, log or message text that people grep, error message strings, side
effects, and performance cliffs. Mark these "possibly breaking" rather than
silently calling them patch-level.

## 4. Propose a path for each break

For every major item, suggest the least disruptive route:
- **Keep the old name as an alias** that emits a deprecation warning, and remove it in the next major.
- **Accept both** old and new fields or flags for one release.
- **Version the endpoint** (`/v2/`) or add a new method beside the old one.
- **Expand–contract for the DB:** add the new column, write to both, backfill,
  switch reads, then drop the old column in a later release.
- Or accept the break: bump major and write a migration note.

## 5. Report

```
Surface diff: <base> → HEAD  (tools: oasdiff, griffe; manual: CLI --help diff)
Required bump: MAJOR

MAJOR
  - lib: `Client.fetch(timeout=)` renamed to `timeout_s=`; callers using the kwarg break
      → keep `timeout` as a deprecated alias until 3.0
  - http: GET /orders response field `total` (string) → (number)
      → add `total_amount` (number) and keep `total` for one release
MINOR
  - cli: new `--json` flag on `export`
POSSIBLY BREAKING (behavioral)
  - `list_users()` now orders by created_at instead of id
PATCH
  - …
Not checked: <surfaces you couldn't diff, and why>
```

Always include "Not checked". A clean report that silently skipped the CLI
or the DB is worse than one that admits the gap.
