# avocado-linux/actions

Reusable GitHub Actions workflows and composite actions for the Avocado Linux
ecosystem. Point your extension, BSP, or (later) runtime/image repo at these so
CI is one source of truth instead of copy-pasted YAML — for our own repos and for
anyone publishing their own Avocado content.

## Reusable workflows

Two **single-combo** helpers. Each runs one `(target, distro-release, distro-channel
[, repo-url])` combination — the **caller owns the matrix**: put `strategy.matrix` on
the job that calls the helper and pass each combo through `with:`. This keeps the
combination set in the consuming repo's workflow, not in `avocado.yaml`.

### `extension-test.yml` — "does it build" (PR mode)

Builds the extension for one combo to prove it assembles. Never packages or publishes.
Use `repo-url` to build against a non-default feed when the extension depends on
upstream packages that live there.

```yaml
# .github/workflows/test.yml in your extension/BSP repo
name: Test
on: pull_request
jobs:
  build:
    strategy:
      fail-fast: false
      matrix:
        include:
          - { target: qemux86-64, distro-release: "2026", distro-channel: edge }
          - { target: qemuarm64,  distro-release: "2026", distro-channel: edge }
          # build against a different feed because we depend on an upstream pkg there:
          - { target: qemux86-64, distro-release: "2026", distro-channel: edge,
              repo-url: "https://repo.staging.example.com" }
    uses: avocado-linux/actions/.github/workflows/extension-test.yml@v1
    with:
      target:         ${{ matrix.target }}
      distro-release: ${{ matrix.distro-release }}
      distro-channel: ${{ matrix.distro-channel }}
      repo-url:       ${{ matrix.repo-url }}      # empty = default feed
```

Inputs: `target` (required), `distro-release` (`2026`), `distro-channel` (`edge`),
`repo-url` (default feed), `cli-version` (`latest`).

### `extension-release.yml` — package + publish (tag mode)

Packages the extension into a noarch RPM and publishes it to avocado-connect. The
publish feed is **derived from `distro-release`/`distro-channel`** — the same config
it's built against, so the feed can't diverge from the build (there is no separate
`target-release`/`target-channel`). Run it on a tag push; the tag must equal the
extension's `version` in `avocado.yaml` (a guard fails the publish otherwise).

```yaml
# .github/workflows/release.yml in your extension/BSP repo
name: Release
on:
  push:
    tags: ['[0-9]+.[0-9]+.[0-9]+', '[0-9]+.[0-9]+.[0-9]+-*']
jobs:
  publish:
    strategy:
      fail-fast: false
      matrix:
        include:
          - { target: qemux86-64, distro-release: "2026", distro-channel: edge }
          - { target: qemuarm64,  distro-release: "2026", distro-channel: edge }
    uses: avocado-linux/actions/.github/workflows/extension-release.yml@v1
    with:
      target:         ${{ matrix.target }}
      distro-release: ${{ matrix.distro-release }}
      distro-channel: ${{ matrix.distro-channel }}
    secrets:
      connect-token: ${{ secrets.AVOCADO_CONNECT_TOKEN }}   # PAT path (optional)
```

Inputs: `target` (required), `distro-release` (`2026`), `distro-channel` (`edge`),
`repo-url` (default feed), `connect-url`, `oidc-audience`, `cli-version` (`latest`).
Secret: `connect-token` (optional — see below).

### Publish auth: PAT or keyless OIDC

`extension-release.yml` authenticates one of two ways, **PAT first**:

1. **PAT (recommended today).** Set a connect API token as the `connect-token`
   secret (store it as e.g. an `AVOCADO_CONNECT_TOKEN` org secret and pass it through
   `secrets:` as above). It is used directly — no server-side trust setup, so it works
   before the OIDC trust plane is provisioned. The `id-token: write` permission is
   then unused.
2. **Keyless OIDC (fallback).** If no `connect-token` secret is set, the job falls
   back to exchanging its GitHub OIDC id-token for a short-lived connect token (see
   below). This needs a publish trust configured on connect.

## Composite actions (also usable standalone)

### `setup-avocado-cli`
Installs the `avocado` CLI from GitHub Releases onto the runner PATH.
```yaml
- uses: avocado-linux/actions/setup-avocado-cli@v1
  with: { version: latest }   # or a tag like 0.41.0
```

### `oidc-login`
Keyless connect auth: exchanges the job's GitHub OIDC id-token for a short-lived
connect token and logs the CLI in. No secret to store. Requires
`permissions: id-token: write` on the job and `setup-avocado-cli` to have run.
```yaml
- uses: avocado-linux/actions/setup-avocado-cli@v1
- uses: avocado-linux/actions/oidc-login@v1
  with: { connect-url: https://connect.peridio.com }
- run: avocado connect ext publish ./out/my-ext-1.0.0-r0.noarch.rpm --targets qemux86-64
```

## How keyless (OIDC) publishing is authorized

This is the fallback path, used when no `connect-token` secret is set. With it
there is **no long-lived token in a secret**. On a tag build the job mints a
GitHub-signed OIDC id-token and exchanges it at
`POST <connect>/auth/github-actions/exchange`. Connect verifies the token against
GitHub's JWKS and matches a **publish-trust policy** (which `owner/repo` may
publish, as whom, to which org) before returning a short-lived token. To grant a
repo publish access, an Avocado admin adds a trust for it — see
avocado-connect `docs/ci-oidc-publish.md`. For first-party `avocado-linux/*`
repos this is a single owner-wildcard trust.

## Versioning

Reference workflows/actions at `@v1`. The reusable workflows reference the composite
actions in this repo at `@v1` as well, so keep the `v1` tag moving forward within the
v1 major; cut `@v2` for breaking changes.

**Before `v1` exists / while iterating:** reusable workflows and actions also resolve
by branch or SHA, so a caller can pin to a development branch
(`...@jschneck/ext-ci-helpers`) to exercise changes before a release is tagged. During
that phase the workflows' internal action refs point at the same dev branch; the
release step is to repoint them to `@v1` and create the `v1` tag.
