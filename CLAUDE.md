# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

`common-ci` is a private collection of shared **GitHub composite actions** for
the owner's personal bun/TypeScript repos, both Biome-based ones and SvelteKit
ones that use Prettier + ESLint. There is no application code here —
every unit of work is a self-contained composite action under `actions/`,
consumed by other repos as:

```yaml
uses: mhrvatin/common-ci/actions/<name>@v1
```

Consumers pin to a major tag (`@v1`), never `@main` — a break in a shared
action here breaks every consumer at once, so backwards-incompatible changes
require cutting a new major tag rather than editing behavior in place. Move
the floating major tag forward (`git tag -f v1 && git push -f origin v1`)
only for non-breaking changes.

## Commands

- `bun install` — install deps (only `@biomejs/biome` as a devDependency)
- `bun run check` — runs `biome check .` (lint + format check) across the repo

There is no unit test framework. Actions are validated by dogfooding: each
push/PR to `main` runs `.github/workflows/self-test.yml`, which invokes every
action in this repo (via local `uses: ./actions/<name>`) against the tiny
`fixture/hello` bun workspace that exists solely to give the actions
something real to check out, lint, and build. To run a single self-test job
locally, use `act` (runs in Docker); the README's "Self-test" section has
the exact command and its known limitations (post steps and JavaScript
actions after `setup-bun`/`setup-ruby` fail with `node: executable file not
found`; git-history actions fail from a worktree). Judge an act run by its
main steps. Otherwise, open a PR and let self-test run, or read
`self-test.yml` to see the exact inputs each action is expected to handle.

## Architecture

Each folder in `actions/` is one composite action (`action.yml`) with typed
inputs/outputs and no shared runtime code between them — every action embeds
its own `checkout` + `setup-bun` + logic. `.github/workflows/self-test.yml`
is the best single file to read to see how they compose into a real
pipeline. The actions and the pattern they form:

- **`docs-only-skip`** — diffs the PR against its base and classifies it as
  docs-only or not, emitting `code` (`'true'`/`'false'`) and
  `changed-files`. Pushes always report `code=true`.
- **`biome-lint`** / **`bun-build`** — the actual code-check jobs. `biome-lint`
  can be scoped to `docs-only-skip`'s `changed-files` output instead of
  linting the whole repo. `bun-build` runs `bun run --filter <pkg> build`
  per package in a space-separated list (with an optional scope prefix for
  scoped workspace names, e.g. `@acme/`). Both restore the bun install
  cache before `setup-bun`.
- **`bun-run`** — the tool-agnostic code-check job for repos that don't use
  Biome (e.g. SvelteKit repos with Prettier + ESLint). Runs a
  space-separated list of root `package.json` scripts in order (e.g.
  `lint`, `check`, `build`, `db:migrate test`), stopping at the first
  failure.
- **`setup-kamal`** — prepares a deploy job: checkout, SSH agent, pinned
  `known_hosts` (exactly one of `known-hosts` or `known-hosts-file`; no
  `ssh-keyscan` fallback by design), Buildx, GHCR login, Ruby, and Kamal.
  It does not run Kamal; each consumer runs its own `kamal setup` or
  `kamal deploy` step with its own secrets.
- **`ci-ok-gate`** — the single required status check meant to be the *only*
  job branch protection points at. It takes two JSON maps of
  `job-name -> result` — `always-required` (checked unconditionally, e.g.
  `committer-check`) and `skippable` (only checked when `skip` isn't
  `'true'`) — and fails if anything in the relevant map didn't succeed. Wire
  `docs-only-skip`'s `code` output into `ci-ok-gate`'s `skip` input so
  docs-only PRs skip lint/build/test without touching branch protection
  settings. Adding, removing, or reordering jobs in a consumer's workflow
  never requires updating branch protection, since it only ever watches
  `ci-ok-gate`.
- **`committer-check`** — fails if any commit in the push/PR wasn't authored
  by a fixed name+email. Note the push-event logic assumes consumers use the
  "Create a merge commit" PR merge strategy (it skips merge commits when
  diffing `before..after`); switching a consumer to squash/rebase merges
  would require revisiting this action, since GitHub's own commits would no
  longer be filtered out correctly.

Third-party actions (`actions/checkout`, `actions/cache`, `oven-sh/setup-bun`,
`ruby/setup-ruby`, `webfactory/ssh-agent`, the `docker/*` actions) are pinned
to a full commit SHA with a version comment, not a tag — keep that pattern
when bumping or adding third-party action references.

## Deliberately out of scope

Documented in the README as intentionally *not* shared here, kept
repo-specific instead:

- Monorepo affected-package / dependency-graph detection — too coupled to
  each repo's package graph to generalize from one real consumer.
- Coverage ratchet — depends on a hardcoded per-repo package list.
- Deploy commands and secrets — `setup-kamal` shares the toolchain, but each
  repo keeps its own `kamal setup`/`kamal deploy` step, secrets, and extras.
- Postgres-backed test jobs — composite actions can't declare `services:`,
  and ports, credentials, and init steps differ per repo.
