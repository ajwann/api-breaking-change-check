# Surface-diff tools by surface type

Use the tool if the project already has it or it's cheap to run. Otherwise
use the manual fallback. Check each command's `--help` against the installed
version before relying on a flag (see the `claim-verification` skill).

## HTTP APIs

| Spec | Tool | Command |
|---|---|---|
| OpenAPI 3.x | oasdiff | `oasdiff breaking origin/main:openapi.yaml openapi.yaml` (reads `<git-ref>:<path>` directly). Add `--fail-on ERR` for CI. Use `oasdiff changelog …` for all changes, breaking and not |
| Protobuf / gRPC | buf | `buf breaking --against '.git#branch=main'`. Rule categories FILE, PACKAGE, WIRE_JSON and WIRE separate source breaks from wire breaks |
| GraphQL | graphql-inspector | `graphql-inspector diff old.graphql new.graphql` |
| No spec | manual | Diff the route table: grep the router registrations (`@app.get`, `router.get(`, `@GetMapping`, `HandleFunc`) on base vs. HEAD, then read the request and response types of each changed handler |

If the spec is generated from code (FastAPI `/openapi.json`, NestJS swagger,
springdoc), generate it on both refs, using `git worktree add` for the base,
and diff the outputs.

## Libraries

| Language | Tool | Command |
|---|---|---|
| Python | griffe | `griffe check <package> -a <base-ref>` (`-s src` for a src layout) |
| Rust | cargo-semver-checks | `cargo semver-checks` (baseline = latest crates.io release), or `--baseline-rev origin/main` |
| Go | gorelease / apidiff | `gorelease -base=<last-tag>` (golang.org/x/exp/cmd/gorelease) suggests the next version; `apidiff` for two package builds |
| TypeScript | api-extractor, or a `.d.ts` diff | With no api-extractor config: run `tsc --declaration --emitDeclarationOnly --outDir /tmp/dts-<ref>` on both refs and `diff -r` them |
| Java/Kotlin | japicmp, revapi | `japicmp` on old and new jars. Also check the Kotlin `binary-compatibility-validator` `.api` dump if present |
| C/C++ | abi-compliance-checker / libabigail (`abidiff`) | Compare built `.so` files for ABI; headers for API |

## Database schema

Diff the migrations added on the branch (`git diff --name-only origin/main...HEAD -- '**/migrations/**'`)
and read each one for drops, renames, type narrowing, new NOT NULL columns without defaults, and changed constraints or indexes.
For Postgres, `migra` can diff two live schemas.

## CLI

Diff `--help` output on both refs. Build or install each ref in a worktree,
run `<cli> --help` and `<cli> <sub> --help` for every subcommand, and
`diff` the results. Also grep for exit-code constants and for stdout formats
that scripts might parse.

## Config, env vars, and events

- Config keys: diff the config schema, defaults file, or the typed settings
  class (pydantic `BaseSettings`, `viper` bindings, `@ConfigurationProperties`).
- Env vars: `git diff origin/main...HEAD | grep -E '^[-+].*(getenv|os\.environ|process\.env|System\.getenv|env::var)'`.
- Event or queue payloads: diff the schema registry (Avro or JSON Schema
  compatibility mode), or the payload types the producer serializes.

## Getting the base ref built side by side

```bash
git worktree add /tmp/base origin/main     # untouched copy of the base
# build or run the tool there, compare against the working tree
git worktree remove /tmp/base
```
