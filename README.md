# shared-workflows

Reusable GitHub Actions workflows shared across Taegost's projects.

## Docker Build and Publish

A reusable workflow that builds multi-arch Docker images and pushes them to Docker Hub. It handles:

- **Multi-arch builds** — `linux/amd64` + `linux/arm64` via QEMU and Buildx
- **Cosign signing** — images are signed against Sigstore/Fulcio for supply-chain verification
- **Docker Hub README sync** — keeps the Docker Hub repository description in sync with `README.md`
- **Semver tagging** — version tags (e.g., `v1.2.3`) produce `1.2.3`, `1.2`, `1`, and `latest`

### Required Secrets

| Secret | Description | Level |
|--------|-------------|-------|
| `DOCKERHUB_USERNAME` | Your Docker Hub username | Org-level |
| `DOCKERHUB_TOKEN` | Docker Hub access token (not your password). Generate at [hub.docker.com](https://hub.docker.com) → Account Settings → Security | Org-level |
| `DOCKERHUB_IMAGENAME` | Image name, e.g. `devops-toolbox` | Repo-level |

> **Note:** For `workflow_dispatch` (manual testing from the shared repo itself), these three secrets must be configured as repository secrets in `Taegost/shared-workflows`. The `secrets: inherit` mechanism only applies when the workflow is called via `workflow_call` from another repo.

### Optional Inputs

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `enable_from_cache` | boolean | `false` | Whether **schedule** and **manual (`workflow_dispatch`)** runs may read from the Docker layer cache. Defaults to `false` so these ad-hoc runs always pull fresh base image layers instead of risking a stale cached layer masking an upstream security patch. Tag-push and pull_request runs always use the cache regardless of this setting. |

Scheduled runs always pass `false` — there's no way to interact with a cron-triggered run, so caching there is a fixed, deliberate off. Manual runs are different: the caller template below exposes `enable_from_cache` as a `workflow_dispatch` input, so whoever clicks "Run workflow" in the Actions tab gets a checkbox and decides per run whether that build should read from cache — no caller YAML edits needed to exercise the choice.

> **⚠️ Breaking change:** prior versions of this workflow always read from the build cache on every trigger, including scheduled and manual runs. As of the version introducing `enable_from_cache`, scheduled runs skip the cache unconditionally, and manual runs skip it unless the checkbox is checked at dispatch time. Existing callers get this safer default automatically once they bump their pinned version tag — no `with:` block edits required. Repos that want scheduled builds specifically to keep reading the cache can hardcode `enable_from_cache: true` in their caller's `with:` block instead of forwarding the dispatch input, at the cost of losing the per-run manual toggle.

### Usage

Copy the following into your consumer repo at `.github/workflows/build-and-push.yml`:

```yaml
name: Build and Publish
on:
  push:
    tags:
      - 'v*.*.*'
  pull_request:
    branches:
      - main
  schedule:
    - cron: '0 4 * * 1'
  workflow_dispatch:
    inputs:
      enable_from_cache:
        description: 'Read from the Docker layer cache on this manual run (default: false, always pulls fresh base image layers)'
        required: false
        default: false
        type: boolean
jobs:
  build:
    permissions:
      id-token: write
      contents: read
    uses: Taegost/shared-workflows/.github/workflows/docker-build-push.yml@v2.0.0
    with:
      event_name: ${{ github.event_name }}
      enable_from_cache: ${{ inputs.enable_from_cache || false }}
    secrets:
      DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
      DOCKERHUB_IMAGENAME: ${{ secrets.DOCKERHUB_IMAGENAME }}
```

The example references `@v2.0.0` — first tag with breaking `enable_from_cache` default change. Use whatever version tag fits your needs — `@main` for latest, specific release tag for stability. Callers pinned below `v2.0.0` keep old always-cache behavior until bumped.

### Supported Triggers

The **caller workflow** (shown above) triggers on:

- **Push tags** (`v*.*.*`) — builds and pushes with full semver tags
- **Pull request to `main`** — builds only (no push) to validate the Dockerfile
- **Weekly schedule** — rebuilds to pick up base image security patches
- **Manual dispatch** — allows on-demand builds from the Actions tab

The **shared workflow** itself triggers on `workflow_call` (called by consumer repos) and `workflow_dispatch` (for manual testing from the shared repo).

## License

[MIT](LICENSE)
