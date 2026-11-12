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
  id-token: write   # required for the keyless publish on tags
jobs:
  ci:
    uses: avocado-linux/actions/.github/workflows/extension.yml@v1
    # all inputs are optional — the build target is auto-derived from avocado.yaml
```

- **On PR / push:** `avocado install` + `avocado ext build` against the distro SDK
  (default release `2024`, channel `edge`) to prove the extension still assembles.
- **On tag:** `avocado ext package` → publish the RPM to avocado-connect.

The build target comes from `avocado.yaml`: `supported_targets: '*'` builds on
`qemux86-64`; a single-board BSP builds on its board. Override with the `target`
input. The tag must equal the extension's `version` in `avocado.yaml` (a guard
fails the publish otherwise).

Key inputs (all optional): `distro-release` (`2024`), `distro-channel` (`edge`),
`cli-version` (`latest`), `target` (auto), `connect-url`, `connect-org`, `oidc-audience`,
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
  with:
    organization: ${{ vars.AVOCADO_CONNECT_ORG }}   # the connect organization id
- run: avocado connect ext publish ./out/my-ext-1.0.0-r0.noarch.rpm --targets qemux86-64
```

## How CI is authorized (keyless)

There is **no long-lived token in a secret**. The job asks GitHub for an OIDC
id-token and exchanges it at `POST <connect>/auth/github-actions/exchange` for a
short-lived token in one connect organization. Connect verifies the id-token
against GitHub's JWKS and requires a **CI trust** in that organization for the
repository (or every repository of its owner), optionally limited to a ref
prefix such as `refs/tags/`. An org admin creates the trust; connect pins the
repository's immutable GitHub ids on first use, so a renamed or re-created
repository does not inherit it.

The token acts as the admin who created the trust and expires after an hour.

## Versioning

Reference workflows/actions at `@v1`. The `extension.yml` workflow references the
composite actions in this repo at `@v1` as well, so keep the `v1` tag moving
forward within the v1 major; cut `@v2` for breaking changes.
