# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

`common-ci` is a private collection of shared **GitHub composite actions** for
Marcus's personal bun/TypeScript repos. There is no application code here —
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
something real to check out, lint, and build. There is no way to exercise a
single action locally short of running it inside an actual GitHub Actions
job (e.g. via `act`) — to validate a change, open a PR and let self-test run,
or read `self-test.yml` to see the exact inputs each action is expected to
handle.

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
  scoped workspace names, e.g. `@facit/`).
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

Pinned action versions (`actions/checkout`, `oven-sh/setup-bun`) are pinned
to a full commit SHA with a version comment, not a tag — keep that pattern
when bumping or adding third-party action references.

## Deliberately out of scope

Documented in the README as intentionally *not* shared here, kept
repo-specific instead:

- Monorepo affected-package / dependency-graph detection — too coupled to
  each repo's package graph to generalize from one real consumer.
- Coverage ratchet — depends on a hardcoded per-repo package list.
- Deploy steps — target platform and secrets vary per repo.
