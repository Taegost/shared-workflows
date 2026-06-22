# Shared GitHub Actions — Deduplication Plan

---

## Part 1: Source of Truth — Which Workflow Is Most Current?

After pulling and comparing all four repos:

| Repo | Has tag validation? | Has semver verify step? | Cosign OIDC flag | Notes |
|---|---|---|---|---|
| `aws-ddns-docker` | ❌ | ❌ | ❌ | Oldest iteration |
| `DevOps-Toolbox` | ✅ | ✅ | ✅ | Header comment falsely says "push to main triggers" — YAML doesn't match |
| `claude-code-env` | ✅ | ✅ | ✅ | Cleanest; no misleading comments |
| `hermes-sandbox` | — | — | — | Repo is a stub; no workflow at all yet |

**→ Use `claude-code-env` as the source of truth.** It is functionally identical to
`DevOps-Toolbox` for all the important logic, but doesn't carry the incorrect
header comment that claims pushes to `main` trigger a build (they don't — the
YAML only has `push: tags:`, which fires only on tag pushes, not branch pushes).

The specific "missing v" guard in `claude-code-env` is this step:

```yaml
- name: Validate tag format
  if: startsWith(github.ref, 'refs/tags/')
  run: |
    TAG="${GITHUB_REF#refs/tags/}"
    if [[ ! "$TAG" =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
      echo "::error::Tag '$TAG' is not valid semver (expected vMAJOR.MINOR.PATCH)"
      exit 1
    fi
```

> **Note:** The condition is being adapted here from `github.event_name == 'push'`
> to `startsWith(github.ref, 'refs/tags/')` — this is intentional and explained in
> Part 2 below.

---

## Part 2: How to Create a Reusable GitHub Action

### Reusable Workflow vs. Composite Action — Which Do You Want?

GitHub gives you two ways to share automation logic:

| | Reusable Workflow | Composite Action |
|---|---|---|
| Unit of sharing | An entire workflow (jobs + steps) | A set of steps (no jobs) |
| Called from | A `job` level `uses:` | A `step` level `uses:` |
| Trigger logic | Lives in the CALLER's workflow | Lives in the CALLER's workflow |
| `github.event_name` in shared code | Always `workflow_call` ⚠️ | Inherits caller's event name ✅ |
| Consumer boilerplate | Minimal — just triggers + one `uses:` line | More — full workflow skeleton in each repo |

**For your use case: Reusable Workflow.** You want consumers to have as little
boilerplate as possible. The caller just specifies triggers and points at the
shared workflow. That said, there is one important gotcha detailed below.

---

### The `github.event_name` Gotcha (Important)

When a reusable workflow is called via `workflow_call`, `github.event_name` inside
the CALLED workflow is always `"workflow_call"` — never `"push"`, `"pull_request"`,
`"schedule"`, etc.

Your existing workflow uses `github.event_name` in several conditions:

```yaml
# These break in a called workflow:
if: github.event_name != 'pull_request'
if: github.event_name == 'push' && startsWith(github.ref, 'refs/tags/')
```

**The fix:** `github.ref` IS correctly inherited from the caller. You can
distinguish all the cases you need using just the ref:

| Scenario | `github.ref` pattern |
|---|---|
| Tag push (`v1.2.3`) | `refs/tags/v1.2.3` |
| Pull request | `refs/pull/1/merge` |
| Branch push / scheduled / dispatch | `refs/heads/main` |

Replace the conditions like this:

```yaml
# BEFORE (breaks in reusable workflow):
if: github.event_name != 'pull_request'
if: github.event_name == 'push' && startsWith(github.ref, 'refs/tags/')
push: ${{ github.event_name != 'pull_request' }}

# AFTER (works in reusable workflow):
if: ${{ !startsWith(github.ref, 'refs/pull/') }}
if: ${{ startsWith(github.ref, 'refs/tags/') }}
push: ${{ !startsWith(github.ref, 'refs/pull/') }}
```

---

### Repository Structure

Create a new public repository — suggested name: `Taegost/shared-workflows`.

```
shared-workflows/
├── .github/
│   └── workflows/
│       └── docker-build-push.yml   ← the reusable workflow
├── README.md
└── LICENSE
```

The file MUST live under `.github/workflows/`. This is a GitHub requirement —
reusable workflows cannot be placed elsewhere in the repo.

---

### The Reusable Workflow (`docker-build-push.yml`)

The key change is the `on:` block. It MUST include `workflow_call:` to be callable
by other repos. You can keep `workflow_dispatch:` alongside it so you can manually
run it from the shared repo itself for testing.

```yaml
on:
  workflow_call:
    secrets:
      DOCKERHUB_USERNAME:
        required: true
      DOCKERHUB_TOKEN:
        required: true
      DOCKERHUB_IMAGENAME:
        required: true
  workflow_dispatch:    # Optional: lets you test from the shared repo directly
```

> Explicitly declaring secrets under `workflow_call:` serves as documentation even
> if callers use `secrets: inherit`. It makes the contract clear.

All conditions that used `github.event_name` get the `ref`-based replacements
described above.

---

### The Caller Workflow (goes in each consumer repo)

Each repo replaces its existing `build-and-push.yml` with this thin caller:

```yaml
# .github/workflows/build-and-push.yml
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
    secrets: inherit
```

That's the entire file. All the actual logic lives in the shared repo.

> **`secrets: inherit` behavior:** This passes all secrets available in the calling
> workflow's context (org-level and repo-level) to the called workflow. It works
> across public repos owned by the same personal account. If you ever hit issues
> with it, the explicit fallback is:
> ```yaml
> secrets:
>   DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
>   DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
>   DOCKERHUB_IMAGENAME: ${{ secrets.DOCKERHUB_IMAGENAME }}
> ```

---

### Versioning and Pinning Strategy

The `@ref` after the workflow path pins to a specific version. Options:

| Ref format | Example | Behavior |
|---|---|---|
| Tag | `@v1.0.0` | Pinned to exact release — **recommended for production** |
| Branch | `@main` | Always latest — drifts without warning |
| SHA | `@a1b2c3d` | Pinned to exact commit — most secure, hardest to maintain |

**Recommended approach:**
- Tag releases of `shared-workflows` with semver (e.g. `v1.0.0`)
- Consumer repos pin to a tag (`@v1.0.0`)
- When you update the shared workflow, cut a new tag and update the pin in each consumer

This is exactly the "change the pin" workflow you described. One update in the shared
repo, then a one-line change in each consumer to point at the new tag.

> **Best practice deviation note:** Pinning to a tag rather than a commit SHA is
> slightly less secure (tags are mutable — they can be force-pushed). For a personal
> homelab with public repos this is fine. If you ever move to an org with security
> requirements, pin to SHAs and use Dependabot to auto-PR updates.

---

### Permissions

The called workflow needs `id-token: write` for Cosign OIDC signing. Permissions
declared in the CALLED workflow take precedence over the caller for that workflow's
jobs. The existing `permissions:` block in your source workflow is correct and should
be kept as-is in the shared workflow:

```yaml
permissions:
  contents: read
  packages: write
  id-token: write
```

---

### Visibility Requirements

- The `shared-workflows` repo MUST be **public** for other repos to call it (for a
  personal account without org-level settings to allow private repo reuse).
- All your consumer repos are already public, so this is a non-issue.

---

### Full Checklist for Setup

1. Create `Taegost/shared-workflows` as a public repo
2. Add `.github/workflows/docker-build-push.yml` with:
   - `on: workflow_call:` (+ optional `workflow_dispatch:` for manual testing)
   - All `github.event_name`-based conditions replaced with `github.ref`-based ones
   - The `push:` flag on `build-push-action` updated accordingly
3. Tag the first release: `v1.0.0`
4. In each consumer repo, replace the existing workflow with the thin caller shown above
5. Verify each consumer's Actions tab shows the job running from the shared workflow
6. Add `hermes-sandbox`'s workflow once its Dockerfile is in place

---

## Part 3: Claude Code Prompt

Paste this prompt verbatim into a Claude Code session opened at the root of the new
`shared-workflows` repo (after creating it on GitHub and cloning locally):

---

```
You are helping me create a new GitHub repository called `shared-workflows` 
(github.com/Taegost/shared-workflows). This repo will contain a single reusable 
GitHub Actions workflow that is currently duplicated across four of my repos:

  - Taegost/DevOps-Toolbox
  - Taegost/aws-ddns-docker
  - Taegost/hermes-sandbox (workflow not yet added)
  - Taegost/claude-code-env

The source of truth is the workflow at:
  https://raw.githubusercontent.com/Taegost/claude-code-env/main/.github/workflows/build-and-push.yml

Read that file first before doing anything else.

## What I need you to do

### Step 1 — Plan (do not write code yet)

Read the source workflow from the URL above, then produce a written plan that covers:

1. The full content of `.github/workflows/docker-build-push.yml` in the shared repo,
   with all `github.event_name`-based conditions replaced with `github.ref`-based 
   equivalents (see transformation rules below).
2. The full content of a thin caller workflow that each consumer repo will use.
3. The README content for the shared repo.
4. A LICENSE file (MIT, same as the source repos).

Transformation rules for `github.event_name` → `github.ref`:
  - `github.event_name != 'pull_request'`  
    → `!startsWith(github.ref, 'refs/pull/')`
  - `github.event_name == 'push' && startsWith(github.ref, 'refs/tags/')`  
    → `startsWith(github.ref, 'refs/tags/')`
  - `push: ${{ github.event_name != 'pull_request' }}` (in build-push-action)  
    → `push: ${{ !startsWith(github.ref, 'refs/pull/') }}`

These changes are required because `github.event_name` in a reusable workflow 
called via `workflow_call` is always the string `"workflow_call"`, not the 
original trigger event name. `github.ref` IS correctly inherited from the caller.

The `on:` block of the shared workflow should be:
```yaml
on:
  workflow_call:
    secrets:
      DOCKERHUB_USERNAME:
        required: true
      DOCKERHUB_TOKEN:
        required: true
      DOCKERHUB_IMAGENAME:
        required: true
  workflow_dispatch:
```

The caller workflow (used in each consumer repo) should be:
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
    secrets: inherit
```

### Step 2 — Implement (after I approve the plan)

Only after I approve the plan:
1. Create `.github/workflows/docker-build-push.yml` in this repo
2. Create `README.md`
3. Create `LICENSE` (MIT, copyright Taegost)

Do NOT create or modify anything in the consumer repos — those changes are manual 
and I will handle them after this repo is tagged and released.

## Constraints

- Do not guess at action versions. Use the exact versions already present in the 
  source workflow (e.g. `actions/checkout@v6`, `docker/build-push-action@v7.0.0`, etc.)
- Preserve all existing comments from the source workflow; update only what is 
  necessary to make the reusable workflow work correctly
- The shared workflow's header comment block should be updated to reflect that this 
  is now a CALLED workflow, not a standalone workflow
- Do not add inputs beyond what is described — `secrets: inherit` handles all 
  credential passing
- Do not add a `push: branches: main` trigger (the existing source workflow does not 
  actually support this despite a comment in one version claiming it does)
