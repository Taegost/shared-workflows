---
title: "feat: Shared Docker build-and-push reusable workflow"
type: feat
status: completed
date: 2026-06-21
---

# feat: Shared Docker build-and-push reusable workflow

## Summary

Create a reusable GitHub Actions workflow in `Taegost/shared-workflows` that consolidates the Docker build-and-push pipeline currently duplicated across four repos (`DevOps-Toolbox`, `aws-ddns-docker`, `hermes-sandbox`, `claude-code-env`). Each consumer repo replaces its copy with a thin caller workflow that delegates to the shared workflow.

## Problem Frame

The same ~130-line Docker build-and-push workflow exists in three repositories (DevOps-Toolbox, aws-ddns-docker, claude-code-env), with a fourth (hermes-sandbox) planned but not yet added. Changes to the pipeline (action version bumps, signing config, cache strategy) require identical edits in each repo. A reusable `workflow_call` workflow eliminates this duplication.

The key technical challenge: `github.event_name` in a reusable workflow called via `workflow_call` is always `"workflow_call"`, not the original trigger event. All conditionals based on `github.event_name` must be rewritten to use `github.ref`, which IS correctly inherited from the caller.

## Requirements

**Workflow behavior**

- R1. The shared workflow triggers on `workflow_call` (with an optional `schedule_trigger` boolean input and three required secrets: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `DOCKERHUB_IMAGENAME`) and `workflow_dispatch`.
- R2. All `github.event_name`-based conditions are replaced with `github.ref`-based equivalents per the transformation rules below.
- R3. All action versions match the source workflow exactly — no upgrades.
- R4. All existing comments from the source workflow are preserved; only the conditions and header block change.
- R5. The header comment block reflects that this is a called workflow, not a standalone workflow.

**Caller workflow**

- R6. The caller workflow template triggers on `push: tags: 'v*.*.*'`, `pull_request: branches: main`, `schedule`, and `workflow_dispatch`.
- R7. The caller template passes secrets via `secrets: inherit`.

**Repo hygiene**

- R8. The LICENSE file already exists (MIT). Confirm it is correct; no changes needed.
- R9. The README documents usage: what the workflow does, required secrets, and how to add it to a consumer repo.

## Key Technical Decisions

**KTD1: `github.ref` instead of `github.event_name` for all conditionals.**
Rationale: `github.event_name` is always `"workflow_call"` in a reusable workflow. `github.ref` is inherited from the caller and reflects the actual trigger context.

**KTD2: Secrets passed via `secrets: inherit`; one optional `input` for schedule context.**
Rationale: The caller uses `secrets: inherit` to forward all repository/org secrets. One `input` is needed: `schedule_trigger` (boolean) so the shared workflow can distinguish schedule events from other triggers and apply `latest`-only tagging. This is required because `github.event_name` in a reusable workflow is always `"workflow_call"`, not the original trigger event.

**KTD3: Reference tag `v1.0.0` in the caller workflow.**
Rationale: The caller references `Taegost/shared-workflows/.github/workflows/docker-build-push.yml@v1.0.0`. After the shared repo is created and the workflow merged, a `v1.0.0` tag must be pushed.

---

## Implementation Units

### U1. Create the shared reusable workflow

**Goal:** Create `.github/workflows/docker-build-push.yml` in the shared repo — the reusable workflow with all `github.event_name` conditions replaced.

**Files:** `.github/workflows/docker-build-push.yml`

**Approach:** The shared workflow consumes secrets as direct references (`secrets.DOCKERHUB_USERNAME`, `secrets.DOCKERHUB_TOKEN`, `secrets.DOCKERHUB_IMAGENAME`), matching the source workflow's pattern. Copy the source workflow and apply these transformations:

| Original condition | Replacement |
|---|---|
| `if: github.event_name == 'push' && startsWith(github.ref, 'refs/tags/')` | `if: startsWith(github.ref, 'refs/tags/')` |
| `if: github.event_name != 'pull_request'` | `if: ${{ !startsWith(github.ref, 'refs/pull/') }}` |
| `push: ${{ github.event_name != 'pull_request' }}` (in build-push-action) | `push: ${{ !startsWith(github.ref, 'refs/pull/') }}` |

**Schedule trigger handling:** The weekly schedule trigger should run the full pipeline as if it's a semver tag release, but only update the `latest` tag (not a versioned tag). Since `github.event_name` in a reusable workflow is always `"workflow_call"`, the schedule context is passed via an input parameter. The caller sets `schedule_trigger: true` when the event is `schedule`, and the shared workflow uses this input to apply `latest`-only tagging instead of versioned tagging. The transformation table must be updated to add a conditional for the tagging step that checks this input.

Replace the `on:` block with:

```yaml
on:
  workflow_call:
    inputs:
      schedule_trigger:
        description: 'Set to true when caller is triggered by schedule (applies latest-only tagging)'
        required: false
        default: false
        type: boolean
    secrets:
      DOCKERHUB_USERNAME:
        required: true
      DOCKERHUB_TOKEN:
        required: true
      DOCKERHUB_IMAGENAME:
        required: true
  workflow_dispatch:
```

Update the header comment block: change "Build and Publish Pipeline" to note this is a reusable/called workflow, and remove the "TRIGGERS" section (triggers are now defined by the caller). Keep the "REQUIRED SECRETS" and "TAGGING STRATEGY" sections.

**Test expectation:** none — YAML workflow files are validated by GitHub Actions at run time, not by local tests.

**Verification:** Before applying transformations, grep the source workflow for all occurrences of `github.event_name`. Verify the transformation table covers every occurrence. If any pattern is not covered, flag it before proceeding. The YAML must be syntactically valid and all transformation rules applied to every affected line.

**Note:** For `workflow_dispatch` (manual testing from the shared repo), the three Docker Hub secrets must be configured as repository secrets in `Taegost/shared-workflows`. `secrets: inherit` only applies when called via `workflow_call`.

### U2. Create the caller workflow template

**Goal:** Create a thin caller workflow that each consumer repo will copy into `.github/workflows/build-and-push.yml`.

**Files:** `examples/caller-workflow.yml`

**Approach:** The exact content specified in the user's request:

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
      schedule_trigger: ${{ github.event_name == 'schedule' }}
    secrets: inherit
```

**Test expectation:** none — this is a template file.

**Verification:** The file matches the specified content exactly.

### U3. Write the README

**Goal:** Replace the placeholder README with documentation covering the workflow's purpose, required secrets, and consumer setup instructions.

**Files:** `README.md`

**Approach:** The README should contain:
- A one-paragraph description of what the workflow does (multi-arch Docker build, push to Docker Hub, Cosign signing, README sync)
- A "Required Secrets" section listing the three secrets with descriptions (matching the source workflow's comments), including a note that `workflow_dispatch` requires these secrets to be configured as repository secrets in the shared repo (not just via `secrets: inherit`)
- A "Usage" section with the caller workflow content and instructions to copy it into a consumer repo
- A note that `v1.0.0` is the initial release tag shown in the example, but consumers should use whatever version tag fits their needs (e.g., `@main` for latest, or a specific release tag)
- A "Supported triggers" section clarifying that the shared workflow itself triggers on `workflow_call` and `workflow_dispatch`, while the caller workflow (shown in the Usage section) handles the four consumer-facing triggers (push tags, pull_request, schedule, workflow_dispatch)

**Test expectation:** none — documentation file.

**Verification:** The README covers all required secrets, includes the caller workflow snippet, and correctly references `v1.0.0`.

### U4. Confirm LICENSE and push tag

**Goal:** Confirm the existing MIT LICENSE is correct. After merging the workflow to `main`, push a `v1.0.0` tag so the caller workflow's `@v1.0.0` reference resolves.

**Files:** `LICENSE` (no changes expected)

**Approach:** Read the existing LICENSE, confirm it is MIT with copyright "Mike Wheway". **U4 depends on U1–U3 being committed and pushed to `main` first.** After the repo content is on `main`, create and push the `v1.0.0` tag — it must point to the commit that contains `docker-build-push.yml`.

**Test expectation:** none — metadata file and git tag.

**Verification:** `LICENSE` contains MIT text; `v1.0.0` tag exists on the commit that contains the shared workflow.

---

## Scope Boundaries

**Deferred to follow-up work:**
- Replacing the duplicated workflows in the four consumer repos (`DevOps-Toolbox`, `aws-ddns-docker`, `hermes-sandbox`, `claude-code-env`) with the caller workflow. This plan creates the shared repo content only.
- Adding the workflow to `hermes-sandbox` (noted as "not yet added" in the user's request).

## Sources

- Source workflow: `https://raw.githubusercontent.com/Taegost/claude-code-env/main/.github/workflows/build-and-push.yml` (fetched and reviewed)
