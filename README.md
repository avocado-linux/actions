# avocado-linux/actions

Reusable GitHub Actions workflows and composite actions for the Avocado Linux
ecosystem. Point your extension, BSP, or (later) runtime/image repo at these so
CI is one source of truth instead of copy-pasted YAML — for our own repos and for
anyone publishing their own Avocado content.

## Reusable workflows

### `extension.yml` — build extensions on PR, publish on tag

```yaml
# .github/workflows/ci.yml in your extension/BSP repo
name: CI
on:
  pull_request:
  push:
    tags: ['[0-9]+.[0-9]+.[0-9]+', '[0-9]+.[0-9]+.[0-9]+-*']
permissions:
  contents: read
  id-token: write   # only needed for the keyless OIDC publish path
jobs:
  ci:
    uses: avocado-linux/actions/.github/workflows/extension.yml@v1
    secrets:
      connect-token: ${{ secrets.AVOCADO_CONNECT_TOKEN }}   # PAT path (optional)
    # all inputs are optional — the build target is auto-derived from avocado.yaml
```

- **On PR / push:** `avocado install` + `avocado ext build` against the distro SDK
  (default release `2024`, channel `edge`) to prove the extension still assembles.
- **On tag:** `avocado ext package` → publish the RPM to avocado-connect.

### Publish auth: PAT or keyless OIDC

The publish step authenticates one of two ways, **PAT first**:

1. **PAT (recommended today).** Set a connect API token as the `connect-token`
   secret (store it as e.g. `AVOCADO_CONNECT_TOKEN` in the repo/org and pass it
   through `secrets:` as above). It is used directly — no server-side trust setup,
   so it works before the OIDC trust plane is provisioned. The `id-token: write`
   permission is then unused.
2. **Keyless OIDC (fallback).** If no `connect-token` secret is set, the job falls
   back to exchanging its GitHub OIDC id-token for a short-lived connect token
   (see below). This needs a publish trust configured on connect **and**
   `permissions: id-token: write` on the caller job.

The build target comes from `avocado.yaml`: `supported_targets: '*'` builds on
`qemux86-64`; a single-board BSP builds on its board. Override with the `target`
input. The tag must equal the extension's `version` in `avocado.yaml` (a guard
fails the publish otherwise).

Key inputs (all optional): `distro-release` (`2024`), `distro-channel` (`edge`),
`cli-version` (`latest`), `target` (auto), `connect-url`, `oidc-audience`,
`target-release` (`2026`), `target-channel` (`edge`), `publish-targets`.

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

Reference workflows/actions at `@v1`. The `extension.yml` workflow references the
composite actions in this repo at `@v1` as well, so keep the `v1` tag moving
forward within the v1 major; cut `@v2` for breaking changes.
