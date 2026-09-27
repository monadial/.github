# monadial/.github reusable workflows

This repo hosts the org's reusable GitHub Actions workflows. It is **not**
the org profile — that lives in [`profile/README.md`](profile/README.md)
and is untouched by this document. This file documents the workflows
under [`.github/workflows/`](.github/workflows/): what they do, the
contract a calling repo must honor, and the action allowlist an org admin
needs to paste into **Settings → Actions → General → Allow select actions
and reusable workflows**.

## `build.yml` — build, scan, sign, write back

```yaml
jobs:
  build:
    # Pin the exact patch tag, NOT @v1 — see "Tag policy" below.
    uses: monadial/.github/.github/workflows/build.yml@v1.0.7
    with:
      image: rg.nl-ams.scw.cloud/monadial/hello
      context: .
      dockerfile: Dockerfile
      writeback_repo: monadial/hello
      writeback_path: deploy/values.yaml
      writeback_regex: 'rg\.nl-ams\.scw\.cloud/monadial/hello@sha256:[a-f0-9]{64}'
      # platforms defaults to linux/amd64; omit unless you need something else.
    secrets:
      OP_SERVICE_ACCOUNT_TOKEN: ${{ secrets.OP_SERVICE_ACCOUNT_TOKEN }}
```

### Required inputs

| Input | Required | Description |
| --- | --- | --- |
| `image` | yes | Fully-qualified image repository, no tag/digest — e.g. `rg.nl-ams.scw.cloud/monadial/hello`. **Must not contain `#`**, for the same reason as `writeback_regex` below (v1.0.7). |
| `context` | yes | Docker build context, relative to the calling repo root. |
| `dockerfile` | yes | Path to the Dockerfile, relative to the calling repo root. |
| `writeback_repo` | yes | `owner/repo` of the deploy-manifest repository to pin the digest into — e.g. `monadial/hello`. |
| `writeback_path` | yes | Path (within `writeback_repo`) to the file whose image reference gets updated. |
| `writeback_regex` | yes | Extended-regex (`grep -E`/`sed -E` compatible) matching the exact `image@sha256:...` reference to replace. **The write-back job fails on purpose if this matches zero lines** — a silently-no-op write-back is treated as a bug, not a success. **Must not contain `#`** (v1.0.7): it is the delimiter of the `sed` s-expression this value is interpolated into, so a `#` could terminate that expression and inject sed flags/commands — GNU sed's `s///e` executes shell, in the job holding the write-back App token. The check is a raw-substring test, so `[#]` is refused too. |
| `platforms` | no (default `linux/amd64`) | `linux/amd64`, `linux/arm64`, or both. Each builds on its own native runner (`ubuntu-latest`, `ubuntu-24.04-arm`); two produce an index signed alongside its per-arch images (v1.1.0). |
| `tag` | no (default: the caller's commit SHA) | Tag used for the build/scan/pre-push steps. The image that ends up signed and written back is always referenced **by digest**, never by this tag. |
| `build_args` | no (default empty) | Newline-separated `KEY=VALUE` list passed to the image build as `--build-arg` (v1.0.10) — e.g. provenance stamps `GIT_SHA=${{ github.sha }}`. Build args persist in image history: **never pass secrets here.** |
| `writeback` | no (default `true`) | `false` skips the write-back job (dry run). |
| `cache` | no (default `false`) | `true` gives each build leg a BuildKit layer cache in the calling repo's Actions cache, scope `<image name>-<arch>` (v1.1.1). See "Layer cache" below. |

### Required secret

| Secret | Required | Description |
| --- | --- | --- |
| `OP_SERVICE_ACCOUNT_TOKEN` | yes | Org secret — a 1Password service-account token (read-only, vault `monadial-cloud`). The build legs, `publish` and `writeback` authenticate to 1Password with it; **every job fails fast with an explicit `::error::`** if it is missing or empty, rather than a cryptic 1Password-CLI auth failure. |

App credentials for the write-back job (`github-app-ci-writeback` — App ID,
private key) are **not** GitHub secrets at all: they are read from
1Password (item `github-app-ci-writeback`, vault `monadial-cloud`) inside
the job, using the same `OP_SERVICE_ACCOUNT_TOKEN`. The App private key
never touches GitHub's secret store.

### Outputs

| Output | Description |
| --- | --- |
| `digest` | `sha256:...` digest: the index digest when more than one platform is built, else the image digest; signed either way. |

### Pipeline shape

1. **`plan` job** (`permissions: {}`): maps each `platforms` entry to a native runner (`linux/amd64` → `ubuntu-latest`, `linux/arm64` → `ubuntu-24.04-arm`) and emits the build matrix; an unsupported, repeated or empty platform list fails here. Nothing is ever emulated.
2. **`build` job** (a matrix, one leg per platform; `permissions: {contents: read}`, no `id-token`): each leg, on its own architecture: checkout → verify `OP_SERVICE_ACCOUNT_TOKEN` present → load Scaleway registry push credentials from 1Password (`op://monadial-cloud/scw-registry-push-ci/credentials/*`) → `docker login` → `docker buildx build` locally (`push: false, load: true`; with `cache: true`, layers read from and written to the calling repo's Actions cache, scope `<image name>-<arch>`, v1.1.1) → **Trivy scan gate** (`--severity HIGH,CRITICAL --exit-code 1`, fixable findings only; the leg stops here, nothing is ever pushed unscanned) → `docker push` of the loaded tag (the exact scanned bytes) → resolve the pushed digest (`docker pull` + `docker inspect` RepoDigests, not the tag) into a per-arch output (`digest_amd64`, `digest_arm64`). A multi-arch leg pushes `<tag>-<arch>`; a single-platform leg pushes `<tag>` itself.
3. **`publish` job** (`permissions: {id-token: write}`, the only job that signs): verify `OP_SERVICE_ACCOUNT_TOKEN` present → load registry credentials → `docker login` → with one platform, take that leg's digest; with two, `docker buildx imagetools create` the index `<tag>` from the legs' digests and **refuse it unless it holds exactly the scanned leg digests** (an extra manifest, such as an attestation, would otherwise be signed unscanned) → install cosign → **`cosign sign --yes <image>@<digest>`** for every digest (the index and each leg), keyless (identity = this job's OIDC token), Rekor bundle included by default so admission verification is offline-capable → offline verify → the same sign and verify again with cosign v2.5.3 (legacy format, Kyverno-vendored-cosign compatibility) → check each legacy `.sig` tag's shape.
4. **`writeback` job** (`needs: publish`; skipped when `writeback` is false; `permissions: {contents: read}`; see note below): verify `OP_SERVICE_ACCOUNT_TOKEN` present → load the GitHub App credentials from 1Password → mint a short-lived App installation token scoped to `writeback_repo` only (`actions/create-github-app-token`) → checkout `writeback_repo` with that token → `sed -E -i` the digest line matched by `writeback_regex` (refusing to proceed if the regex matched nothing) → commit `ci: pin <image> to <digest>` (skipped as a no-op if the digest didn't actually change) → push.

**Why `writeback` runs with `contents: read`, not `contents: write`:** the
job's ambient `GITHUB_TOKEN` permission only ever governs the *calling*
repository — the one this reusable workflow is invoked from. It is
irrelevant to the commit above, which targets a **different** repository
(`writeback_repo`). All write authority for that commit comes from the
GitHub App installation token minted inside the job, which the App's own
installation scopes to `writeback_repo` alone. Granting `contents: write`
on the ambient token here would only widen the blast radius of a bug in
this job (accidental write access to the calling repo) for zero benefit —
so it stays at the conservative non-empty default, `read`.

### Layer cache (v1.1.1, opt-in)

With `cache: true`, each build leg reads and writes BuildKit layers in the **calling** repository's GitHub Actions cache (`type=gha`, `mode=max`), one scope per image and architecture: the image name's last path segment plus the arch, for example `aktum-backend-arm64`. A cache export that fails never fails the build (`ignore-error=true`). A cached layer is loaded, scanned and pushed like a built one, so the Trivy gate is unchanged. Without `cache`, nothing is read or written.

Layers of the image that `publish` signs can come from the caller's default-branch Actions cache, which any workflow running on that branch can write; BuildKit trusts an imported cache and Trivy catches only known CVEs, so enable `cache` only in repos where every workflow that runs on the default branch is trusted.

Layers that patch the OS (`apt-get upgrade`, `apk upgrade`) are cached too, until the base image's digest changes. A caller that patches the OS declares a build argument that changes weekly directly before that layer and passes it through `build_args`. Aktum declares `ARG SECURITY_REFRESH` and passes `SECURITY_REFRESH=$(date -u +%G-W%V)`. A new value misses the cache there, so the upgrade reruns at least once a week.

When Trivy blocks a build within the week on a fix the upgrade would now install, clear the cache and build again:

    gh cache list --repo <owner>/<repo>
    gh cache delete --all --repo <owner>/<repo>

### ⚠️ Loop-guard requirement (every caller MUST set this)

The write-back commit is pushed with an **App installation token**, not
`GITHUB_TOKEN` — GitHub's built-in "actions pushing with GITHUB_TOKEN don't
re-trigger workflows" recursion suppression **does not apply** to App-token
pushes. Without a guard, the write-back commit re-triggers your `push`
workflow, which builds again, writes back again (a no-op, since the digest
is unchanged — the `writeback` job's no-op-commit check stops the actual
commit), but still burns a full CI run every time, forever.

Every caller's `push` trigger **must** set `paths-ignore` on the exact
`writeback_path` it passed to this workflow:

```yaml
on:
  push:
    branches: [main]
    paths-ignore:
      - deploy/values.yaml   # MUST match this workflow's `writeback_path` input exactly
```

If your repo has more than one `writeback_path` across multiple images,
list all of them under `paths-ignore`.

### Tag policy

Callers pin to a **major-version tag family**, not a full SHA:

```yaml
uses: monadial/.github/.github/workflows/build.yml@v1.0.7
```

`v1` is intended to move forward to the latest compatible `v1.x.y`
release; a breaking change to this workflow's contract ships as `v2`.

**Status of the `v*` tag ruleset: LIVE.** The `protect-v-tags` repository
ruleset on `monadial/.github` is `enforcement: active`, `target: tag`,
covering `refs/tags/v*` with `creation`/`update`/`deletion`/
`non_fast_forward` rules and an `OrganizationAdmin` bypass (verified via
`gh api repos/monadial/.github/rulesets`). So `v1` and every `v1.0.x` are
immovable by CI, and creating a new version tag is an org-admin act. This
matters because Kyverno's `verifyImages` attestor in `monadial-cloud`
trusts this workflow by its tag family via `subjectRegExp` — a movable tag
would make that trust meaningless.

**`v1` currently points at `d627f2d` (the pre-fix v1.0.0 commit) and has
NOT been moved.** Callers therefore pin the exact patch tag, not `v1`:
`...build.yml@v1.0.7`. Moving `v1` forward is an outstanding owner release
act; until it happens, `@v1` is not just stale but wrong.

## `rescan.yml` — weekly re-scan of deployed digests

Runs every Monday 06:00 UTC (`workflow_dispatch` also available for a
manual run). No inputs — v1 scope is fixed: it reads the pinned
`image@sha256` references directly out of `monadial-cloud`'s platform
manifests (`gitops/platform/ingress/admin-ingresses.yaml`,
`gitops/platform/docs/docs.yaml`, via a sparse checkout using the same App
installation token pattern as the write-back job, read-only) and re-scans
each one with Trivy (`--severity HIGH,CRITICAL --exit-code 1`). `hello`'s
`deploy/values.yaml` joins the scan list once that repo exists and has a
deployed digest to scan.

This catches vulnerabilities disclosed **after** the original build-time
Trivy gate passed, against images that are still running. A failing run
does not notify Slack itself — that's the GitHub Slack app's job
(Prerequisite 7: `/github subscribe monadial/.github
workflows:{event:"workflow_run" branch:"main"}`), which is org
configuration, not workflow code.

Same `OP_SERVICE_ACCOUNT_TOKEN` org secret requirement and fail-fast
preflight check as `build.yml`.

## Action allowlist (Prerequisite 4)

Paste the following into **Settings → Actions → General → Allow select
actions and reusable workflows** (one `owner/repo@SHA` line per action;
GitHub matches on the exact commit SHA, which is why every `uses:` in
these workflows is pinned to one). Re-derive with `gh api
repos/<owner>/<repo>/commits/<tag> --jq '.sha'` (dereferences annotated
tags to the commit automatically) whenever a pin is bumped.

```
actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1,
docker/setup-buildx-action@37fe631027851001ddb9b187196cc803df7f5f0e,
docker/build-push-action@53b7df96c91f9c12dcc8a07bcb9ccacbed38856a,
aquasecurity/trivy-action@ed142fd0673e97e23eac54620cfb913e5ce36c25,
sigstore/cosign-installer@6f9f17788090df1f26f669e9d70d6ae9567deba6,
1password/load-secrets-action@70062d7a876d3eb6334754fa26efd2fbd90c32f2,
actions/create-github-app-token@bcd2ba49218906704ab6c1aa796996da409d3eb1,
```

| Action | Pinned tag | SHA |
| --- | --- | --- |
| `actions/checkout` | v7.0.1 | `3d3c42e5aac5ba805825da76410c181273ba90b1` |
| `docker/setup-buildx-action` | v4.3.0 | `37fe631027851001ddb9b187196cc803df7f5f0e` |
| `docker/build-push-action` | v7.3.0 | `53b7df96c91f9c12dcc8a07bcb9ccacbed38856a` |
| `aquasecurity/trivy-action` | v0.36.0 | `ed142fd0673e97e23eac54620cfb913e5ce36c25` |
| `sigstore/cosign-installer` | v4.1.2 | `6f9f17788090df1f26f669e9d70d6ae9567deba6` |
| `1password/load-secrets-action` | v5.0.1 | `70062d7a876d3eb6334754fa26efd2fbd90c32f2` |
| `actions/create-github-app-token` | v3.2.0 | `bcd2ba49218906704ab6c1aa796996da409d3eb1` |

v1.1.0 adds no action: multi-arch reuses the pins above, and digests cross jobs as outputs, not artifacts.
v1.1.1 adds no action either: the layer cache is an input of docker/build-push-action.

**Transitive-action note:** `aquasecurity/trivy-action` internally calls a
SHA-pinned `aquasecurity/setup-trivy` step, and that step uses
`actions/cache` internally too. GitHub's "allow select actions" policy is
enforced transitively (it checks every action actually invoked in a run,
not just the top-level ones your workflow's `uses:` lines name). If the
org policy rejects runs over those nested actions, add
`aquasecurity/setup-trivy@<sha>` and `actions/cache@<sha>` to the allowlist
too — re-check `aquasecurity/trivy-action`'s `action.yaml` at the pinned
SHA above for their exact current pins, since those are trivy-action's
dependencies to track, not ours.

## Prerequisites this repo assumes are already done

See the plan document's Prerequisites section for the full list (App
install, 1Password items, org secret, org Actions policy, tag-protection
ruleset, `monadial/hello` repo, GitHub Slack app subscription). This repo
cannot self-verify any of them — they are owner/org-admin actions.
