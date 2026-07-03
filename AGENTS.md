## Rules

- Default to writing no code comments. Only add one when the "why" is non-obvious — a hidden constraint, subtle invariant, bug workaround, or surprising behaviour. Don't restate what the code does, don't label sections, don't annotate every change
- Before starting edits, summarize intended changes and ask for confirmation unless the user clearly asked you to create or modify files

## Tool preferences

The following non-standard CLI tools are available; prefer them over the defaults:

- Prefer `rg` instead of `grep` (example: `rg "pattern"` instead of `grep -r "pattern"`)
- Prefer `fd` instead of `find` (example: `fd "filename"` instead of `find . -name "filename"`)
- Infer the package manager from the project lockfile; if no lockfile exists, fall back to system defaults (eg: npm)

## Commit preferences

- Follow a lightweight Conventional Commit pattern for commit messages: `<type>: <brief description>`
- Add a scope (ie: `<type>(scope):`) only if it adds meaningful clarity, eg: when the commit targets one package in a monorepo of many
- Add a body (ie: after a blank line) only if there is non-obvious context not inferable from the main message

## Testing preferences

- Tests must verify behaviour, not restate implementation. Before adding a test, be able to identify the real bug or regression it would catch; if you can't, skip it.
- Test public contracts and observable outcomes, not internals; skip trivial formatters, stdlib wrappers, and pass-through helpers unless they encode important behaviour. Assert mock/fake internals only when that interaction is itself the contract.
- Skip exhaustive parameter sweeps across one code path — pick representative edge cases and one happy path.
- Prioritise (in order): error/rejection paths, state invariants, data integrity, cross-component contracts, safety/security boundaries, and non-trivial heuristics.
- When touching tests, consolidate or remove nearby low-value tests instead of adding more noise.

## Code convention preferences

### Typescript

- Prefer `type` over `interface` unless features of interfaces (eg: declaration merging) are strictly required

### Ruby

- Use multiple lines for inheretence when defining modules
- Prefer no default arguments when defining functions. For things that we want to default when nil is passed in, use a ||= fallback
- Prefer stateless service/serializer objects: inject collaborators, pass runtime data to public methods (like `call`), and do not store per-call/request state
- Prefer project dependency helpers for injectable collaborators. Avoid generic `initialize(**dependencies)` or `initialize(depenndency:)` plumbing where possible
- After writing or editing Ruby code, lint with Rubocop (`bundle exec rubocop <files>`) as well as standard `rspec` etc validation
- When authoring Ruby tests, prefer doing test steup in a context block with only actions and assertions in the `it` block

## Context efficient bash

When the only signal you need from a command is whether it succeeded (eg: tests, lint/typecheck/format checks, builds, dependency installs), wrap it so a success output collapses to a short label and a non-zero exit prints its full output. Skip when you actually need the output (eg: git diff/status, watching long-running streams, test coverage numbers, etc)

```sh
out=$(<cmd> 2>&1) && echo "<passed label>" || { printf '%s\n' "$out"; exit 1; }

# eg
out=$(pnpm test 2>&1) && echo "tests pass" || { printf '%s\n' "$out"; exit 1; }
out=$(tsc --noEmit 2>&1) && echo "typecheck clean" || { printf '%s\n' "$out"; exit 1; }
```

## Subagents

- Use the Explore subagent for non-trivial codebase discovery: locating symbols, tracing usage, answering "where is X / what references Y / what calls Z", or mapping patterns across files. Reach for it before running your own multi-step or wide grep/find sweeps; it keeps the main context lean. Skip it when a targeted read or two would answer the question — a single known file, an obvious local lookup, or a tiny edit; don't escalate that into a full Explore
- Use the Review subagent as an independent second-pass reviewer for large, risky, or multi-file diffs during an active implementation session. Avoid delegating to Review in a fresh session where the user’s main request is already a review; you have no context to isolate from, so review the artifact directly
- Use General purpose subagents only for well-scoped parallel work with clear boundaries

## Devcontainers

- Some repos only have a working dev environment inside a devcontainer — Ruby/Bundler deps, the database, and services are provisioned there, not on the host.
- When environment-dependent commands fail in a way that suggests the environment isn't set up (missing gems/`bundle` failures, no DB connection, missing services/binaries), first determine whether you're on the host or inside the container before doing anything else. The host is macOS (`uname -s` = `Darwin`); the devcontainer is Linux (`uname -s` = `Linux`, and typically has `/.dockerenv`).
- On the host (Darwin): stop and flag it to the user. Do not attempt to self-heal. Never install dependencies to work around it. Host-level installs pollute the global environment and are not the fix; the fix is to run inside the container.
- Inside the container (Linux devcontainer): be self-directed. Attempting to repair the environment is safe and expected — run `bundle install`, install missing gems, run pending migrations / DB setup, or start required services as needed to unblock the task. The container is disposable, so global installs there are fine.
