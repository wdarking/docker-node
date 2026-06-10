# Plan: Make `kooldev/node` images actually refresh npm on rebuild

## Context

`kooldev/node:24` ships npm **11.9.0**, below the **11.13.0** threshold we need — even though the
Dockerfile already does `RUN npm i --location=global npm@latest` ([template/Dockerfile.blade.php:36](template/Dockerfile.blade.php#L36)).
`npm@latest` resolves at *build time*, so the published image is frozen at whatever npm was latest the
last time that layer was actually built.

There is already a weekly cron (`0 0 * * 0`) in [.github/workflows/ci-cd.yml](.github/workflows/ci-cd.yml#L6-L7)
that is *supposed* to keep images fresh. It doesn't, because of the buildx layer cache:

- The `Cache Docker layers` step keys on `docker-buildx-${version}-${github.sha}` with restore-key
  prefix `docker-buildx-${version}-` ([ci-cd.yml:24-29](.github/workflows/ci-cd.yml#L24-L29)).
- The build passes `--cache-from type=local` ([ci-cd.yml:33-38](.github/workflows/ci-cd.yml#L33-L38)).
- On the weekly cron, `master`'s sha is unchanged, so the prefix restore-key always hits the prior
  cache. Every layer — including `RUN npm i npm@latest` — is served from cache, so npm is never
  re-resolved. It stays frozen until a code change alters the sha.

**Goal (chosen approach): keep `npm@latest`, fix the cache** so the weekly cron does a genuine cold
rebuild and always re-resolves npm (plus the base image, pnpm, dockerize) — while intra-week pushes
still get fast incremental builds. No version pinning, zero manual upkeep.

## Change — `.github/workflows/ci-cd.yml` only

The template/version Dockerfiles stay exactly as-is (`npm@latest` is correct). The fix is entirely in CI.

**1. Add a step that computes an ISO week stamp** (before the cache step), so the cache rotates weekly:

```yaml
- name: Compute cache week
  id: cachekey
  run: echo "week=$(date -u +%Y-%V)" >> "$GITHUB_OUTPUT"
```

**2. Fold the week into the cache key and restore-key** ([ci-cd.yml:24-29](.github/workflows/ci-cd.yml#L24-L29)):

```yaml
- name: Cache Docker layers
  uses: actions/cache@v4
  with:
    path: /tmp/.buildx-cache
    key: docker-buildx-${{ matrix.version }}-${{ steps.cachekey.outputs.week }}-${{ github.sha }}
    restore-keys: docker-buildx-${{ matrix.version }}-${{ steps.cachekey.outputs.week }}-
```

Each new week → no restore-key match → cold build → `npm@latest` re-resolves → fresh npm.
Same-week pushes → prefix match → fast incremental builds (npm not re-resolved, which is fine —
freshness is weekly, matching the existing cron intent).

**3. Add `--pull` to the buildx build** ([ci-cd.yml:33-38](.github/workflows/ci-cd.yml#L33-L38)) so a cold
build also refreshes the base `node:${version}-alpine` digest, not just npm:

```yaml
docker buildx build \
  --pull \
  --cache-from type=local,src=/tmp/.buildx-cache/${{ matrix.version }} \
  ...
```

## Why this also fixes it *immediately*

Changing the cache key *format* (inserting the `${week}` segment) means the new restore-key prefix
`docker-buildx-${version}-${week}-` no longer matches any existing cache (named
`docker-buildx-${version}-<sha>`). So the very first build after this PR merges to `master` is cold
for all versions → npm is refreshed to current latest (>11.13) and pushed to DockerHub on that run.
No separate manual `workflow_dispatch` trigger required.

## Notes / caveats

- `24-nginx` is `FROM kooldev/node:24` ([24-nginx/Dockerfile:2](24-nginx/Dockerfile#L2)) and inherits
  npm from the published `:24`. Because the matrix builds `24` and `24-nginx` in parallel, `24-nginx`
  pulls the *previously* published `:24`, so it lags npm by exactly one build cycle (it catches up next
  run). This is pre-existing and out of scope for the npm fix — flagging only.
- Simpler fallback if we ever want bulletproof-but-slower: delete the cache step entirely and rely on
  cold builds every run. Rejected here because it makes intra-week pushes slow for no benefit.

## Verification

1. **Lint the workflow** locally: `actionlint .github/workflows/ci-cd.yml` (if available), or just
   confirm valid YAML.
2. **Confirm the cold-build trigger logic**: after merge, watch the CI run — the `Cache Docker layers`
   step should report a cache *miss* (no matching restore-key) for each version on the first run.
3. **Assert the shipped npm version** — the existing `Tests` step already runs `npm -v`
   ([ci-cd.yml:40-48](.github/workflows/ci-cd.yml#L40-L48)). After the run, confirm it prints ≥ 11.13.0.
   Optionally tighten the test to fail-fast on a stale npm:
   ```sh
   docker run kooldev/node:${version}-$(arch) sh -c \
     'npm -v | awk -F. "{ exit (\$1>11 || (\$1==11 && \$2>=13)) ? 0 : 1 }"' \
     || { echo "npm below 11.13"; exit 1; }
   ```
4. **End-to-end**: once `master` CI is green and DockerHub push + manifest amend complete, run
   `docker run --rm kooldev/node:24 npm -v` and confirm ≥ 11.13.0.
