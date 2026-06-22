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
jobs:
  build:
    uses: Taegost/shared-workflows/.github/workflows/docker-build-push.yml@v1.0.0
    with:
      event_name: ${{ github.event_name }}
    secrets:
      DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
      DOCKERHUB_IMAGENAME: ${{ secrets.DOCKERHUB_IMAGENAME }}
```

The example references `@v1.0.0`. Use whatever version tag fits your needs — `@main` for the latest, or a specific release tag for stability.

### Supported Triggers

The **caller workflow** (shown above) triggers on:

- **Push tags** (`v*.*.*`) — builds and pushes with full semver tags
- **Pull request to `main`** — builds only (no push) to validate the Dockerfile
- **Weekly schedule** — rebuilds to pick up base image security patches
- **Manual dispatch** — allows on-demand builds from the Actions tab

The **shared workflow** itself triggers on `workflow_call` (called by consumer repos) and `workflow_dispatch` (for manual testing from the shared repo).

## License

[MIT](LICENSE)
