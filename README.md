# common-ci

Shared GitHub composite actions for my personal bun/TS repos. Private repo —
consumers authenticate via their own `GITHUB_TOKEN`, which can read other
private repos owned by the same account by default.

Each action lives in its own folder under `actions/` and is referenced as:

```yaml
uses: mhrvatin/common-ci/actions/<name>@v1
```

Pin to a tag (`@v1`), not `@main` — a break in a shared action breaks every
consumer at once.

## Actions

### `committer-check`

Fails the job if any commit in the push/PR has a committer other than a
fixed name+email.

```yaml
- uses: mhrvatin/common-ci/actions/committer-check@v1
  with:
    expected-name: Marcus Hrvatin
    expected-email: github.com@hrvatin.se
```

### `docs-only-skip`

Detects whether a PR only touches docs/markdown. Outputs `code` (`'true'` /
`'false'`) and `changed-files` (space-separated, empty on push or docs-only).
Use `code` to gate downstream jobs, and `changed-files` to scope lint runs.

```yaml
jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      code: ${{ steps.detect.outputs.code }}
      changed-files: ${{ steps.detect.outputs.changed-files }}
    steps:
      - id: detect
        uses: mhrvatin/common-ci/actions/docs-only-skip@v1
```

Override `skip-pattern` (an extended regex of "not code" paths) if a repo's
docs live somewhere other than `docs/` + `*.md`.

### `biome-lint`

Checks out, installs with bun, and runs Biome — the repo's `check` script in
full, or scoped to a file list.

```yaml
- uses: mhrvatin/common-ci/actions/biome-lint@v1
  with:
    bun-version: 1.3.14
    files: ${{ needs.changes.outputs.changed-files }} # optional
```

Assumes: `check` script in `package.json` (e.g. `"check": "biome check ."`),
Biome as a devDependency, `bun.lock` committed.

### `bun-build`

Checks out, installs with bun, and runs `bun run --filter <name> build` for
each package in a space-separated list.

```yaml
- uses: mhrvatin/common-ci/actions/bun-build@v1
  with:
    bun-version: 1.3.14
    packages: shared api web
    scope: '@facit/' # optional, prefixed onto each package name
```

### `ci-ok-gate`

Single required-status-check job. Fails if any `always-required` job didn't
succeed, or — unless `skip` is `'true'` — if any `skippable` job didn't
succeed. Point branch protection at this job only, so adding/reordering jobs
never requires touching branch protection settings.

```yaml
ci-ok:
  if: always()
  needs: [committer-check, changes, lint, build, test]
  runs-on: ubuntu-latest
  steps:
    - uses: mhrvatin/common-ci/actions/ci-ok-gate@v1
      with:
        always-required: '{"committer-check":"${{ needs.committer-check.result }}","changes":"${{ needs.changes.result }}"}'
        skippable: '{"lint":"${{ needs.lint.result }}","build":"${{ needs.build.result }}","test":"${{ needs.test.result }}"}'
        skip: ${{ needs.changes.outputs.code == 'false' }}
```

## Versioning

Tag releases as `v1`, `v2`, etc. Move the major tag forward on non-breaking
changes (`git tag -f v1 && git push -f origin v1`); cut a new major tag for
breaking input/output changes so existing consumers don't silently break.

## Self-test

`.github/workflows/self-test.yml` runs every action in this repo against the
`fixture/` bun workspace on every push/PR, so a broken action fails here
instead of in a consumer repo.

## Scope

Deliberately out of scope for now (stays repo-specific, not shared):

- Monorepo affected-package / dependency-graph detection — too tightly
  coupled to each repo's package graph to generalize usefully with one real
  consumer.
- Coverage ratchet — facit's `scripts/check-coverage.ts` hardcodes its
  package list; revisit sharing it if a second repo needs the same ratchet.
- Deploy steps — target platform and secrets vary per repo.
