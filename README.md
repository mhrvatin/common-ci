# common-ci

Shared GitHub composite actions for my personal bun/TS repos: Biome-based
repos and SvelteKit repos using Prettier and ESLint. Private repo —
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
    expected-name: Jane Doe
    expected-email: jane@example.com
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

Checks out, restores the bun install cache, installs with bun, and runs
Biome — the repo's `check` script in full, or scoped to a file list.

```yaml
- uses: mhrvatin/common-ci/actions/biome-lint@v1
  with:
    bun-version: 1.3.14
    files: ${{ needs.changes.outputs.changed-files }} # optional
```

Assumes: `check` script in `package.json` (e.g. `"check": "biome check ."`),
Biome as a devDependency, `bun.lock` committed.

### `bun-build`

Checks out, restores the bun install cache, installs with bun, and runs
`bun run --filter <name> build` for each package in a space-separated list.

```yaml
- uses: mhrvatin/common-ci/actions/bun-build@v1
  with:
    bun-version: 1.3.14
    packages: shared api web
    scope: '@acme/' # optional, prefixed onto each package name
```

### `bun-run`

Checks out, restores the bun install cache, installs with bun, and runs
package.json scripts in order, stopping at the first failure. It does not
care which tools the scripts call, so it covers Prettier + ESLint
(`lint`), `svelte-check` (`check`), `vite build` (`build`) and test runners
alike.

```yaml
lint:
  needs: changes
  if: needs.changes.outputs.code == 'true'
  runs-on: ubuntu-latest
  steps:
    - uses: mhrvatin/common-ci/actions/bun-run@v1
      with:
        bun-version: 1.3.14
        scripts: lint
```

`scripts` takes several names, e.g. `db:migrate test` for a test job that
migrates a Postgres service first.

Assumes: `bun.lock` committed, and each script exists in the root
`package.json`.

### `setup-kamal`

Prepares a job to deploy with Kamal: checks out, loads the deploy SSH key
into an agent, trusts the server's pinned host key, sets up Docker Buildx,
logs in to GHCR, and installs Ruby and Kamal. It does not run Kamal. Run
`kamal setup` (repos with accessories, since it also boots them) or
`kamal deploy` (repos without accessories, since it needs no sudo) in the
next step, with the repo's own secrets.

```yaml
deploy:
  runs-on: ubuntu-latest
  permissions:
    contents: read
    packages: write
  steps:
    - uses: mhrvatin/common-ci/actions/setup-kamal@v1
      with:
        ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
        known-hosts: 203.0.113.10 ssh-ed25519 AAAA...
    # Repo-specific steps (e.g. building a backup image) go here.
    - run: kamal setup
      env:
        KAMAL_REGISTRY_PASSWORD: ${{ secrets.GITHUB_TOKEN }}
        POSTGRES_PASSWORD: ${{ secrets.POSTGRES_PASSWORD }}
```

Set exactly one of `known-hosts` (the line itself) or `known-hosts-file` (a
path in the repo, e.g. `.github/ssh/known_hosts`). There is deliberately no
`ssh-keyscan` fallback: a pinned key makes a changed host key fail the
deploy instead of being trusted. Optional inputs: `kamal-version` (default
`2.12.0`), `ruby-version` (`3.3`), `fetch-depth` (`1`), `registry`
(`ghcr.io`), `registry-username` (repo owner), `registry-password`
(`github.token` for `ghcr.io`; required for any other registry, so the
token never goes to a third party).

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

To run a self-test job locally, install [act](https://github.com/nektos/act)
(it runs jobs in Docker, so nothing is installed on the host) and run:

```sh
act pull_request --job bun-run --container-architecture linux/amd64 \
  --platform ubuntu-latest=catthehacker/ubuntu:act-latest \
  --secret GITHUB_TOKEN="$(gh auth token)"
```

Known act limitations:

- Post steps of JavaScript actions fail with `node: executable file not
  found`, so act reports the job as failed. Judge the run by its main steps.
- JavaScript actions that run after `setup-bun` or `setup-ruby` in the same
  job hit the same error. That is why the bun cache is restored before
  `setup-bun`, and why each self-test job calls a bun action only once.
- `docs-only-skip` and `committer-check` read git history, which fails when
  run from a git worktree (its `.git` file points outside the container).

## Scope

Deliberately out of scope for now (stays repo-specific, not shared):

- Monorepo affected-package / dependency-graph detection — too tightly
  coupled to each repo's package graph to generalize usefully with one real
  consumer.
- Coverage ratchet — the one existing ratchet script hardcodes its package
  list; revisit sharing it if a second repo needs the same ratchet.
- Deploy commands and secrets — `setup-kamal` shares the toolchain, but each
  repo keeps its own `kamal setup`/`kamal deploy` step, secrets, and extras
  such as backup images and accessory reboots.
- Postgres-backed test jobs — composite actions can't declare `services:`,
  and ports, credentials, and init steps differ per repo.
