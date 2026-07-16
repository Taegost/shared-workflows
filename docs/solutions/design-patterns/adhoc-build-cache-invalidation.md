---
title: "Preventing Stale Docker Layer Cache From Masking Base Image Patches on Ad-Hoc Runs"
date: "2026-07-16"
last_updated: "2026-07-16"
category: docs/solutions/design-patterns
module: shared-workflows
problem_type: design_pattern
component: docker_build_cache
severity: medium
applies_when:
  - "A scheduled or manually-dispatched rebuild exists specifically to pick up upstream base image changes (e.g. security patches)"
  - "The build uses a layer cache (e.g. cache-from: type=gha) that can satisfy the base image layer from a previous run instead of pulling it fresh"
  - "workflow_dispatch and workflow_call share the same input name so a single flag works from either trigger path"
tags:
  - github-actions
  - docker
  - buildx
  - cache
  - reusable-workflows
  - workflow-dispatch
  - schedule-trigger
---

# Preventing Stale Docker Layer Cache From Masking Base Image Patches on Ad-Hoc Runs

## Context

The shared Docker build-and-push workflow has a weekly `schedule` trigger whose entire purpose is to rebuild the image and pick up upstream base-image security patches, even when nothing in the consumer repo has changed. The build step used `cache-from: type=gha` unconditionally, so a scheduled run could hit the GHA layer cache and reuse a previously-cached base-image layer instead of pulling a fresh one — silently defeating the point of the schedule. A manual `workflow_dispatch` run, typically triggered specifically to force a fresh rebuild ("don't wait for the weekly schedule"), hit the exact same problem.

## Guidance

**Pattern: default cache reads off for ad-hoc rebuild triggers, keep cache writes on, and expose one opt-in boolean input shared by both `workflow_call` and `workflow_dispatch`.**

- `cache-from` (cache **read**) is what causes a stale layer to be reused — this is the side that needs to be conditional.
- `cache-to` (cache **write**) should stay unconditional even on ad-hoc runs, so the freshly-pulled layers still populate the shared cache for the next tag-push or PR build to benefit from.
- Declare the same input name/type in both `on.workflow_call.inputs` and `on.workflow_dispatch.inputs` on the reusable workflow. GitHub Actions binds whichever trigger actually fired to the same `inputs.*` context, so a single expression in the job works regardless of whether the workflow was called by a consumer or dispatched directly.

**Reusable workflow — declare the input on both trigger blocks:**

```yaml
on:
  workflow_call:
    inputs:
      event_name:
        description: 'Caller github.event_name — required so this workflow can detect schedule/manual triggers'
        required: false
        default: ''
        type: string
      enable_from_cache:
        description: 'Whether schedule or manual (workflow_dispatch) runs may read from the Docker layer cache. Defaults to false.'
        required: false
        default: false
        type: boolean
    secrets:
      # ...
  workflow_dispatch:
    inputs:
      enable_from_cache:
        description: 'Whether this manual run may read from the Docker layer cache. Defaults to false.'
        required: false
        default: false
        type: boolean
```

**Reusable workflow — conditional cache-from, unconditional cache-to:**

```yaml
- name: Build and push Docker image
  uses: docker/build-push-action@v7.0.0
  with:
    cache-from: ${{ (!(inputs.event_name == 'schedule' || inputs.event_name == 'workflow_dispatch' || github.event_name == 'workflow_dispatch') || inputs.enable_from_cache) && 'type=gha' || '' }}
    cache-to: type=gha,mode=max
```

An empty string for `cache-from` results in `docker/build-push-action` omitting `--cache-from` entirely, producing a full rebuild.

The condition covers three distinct trigger paths with one expression:
1. `inputs.event_name == 'schedule'` — a consumer's caller workflow forwarded a schedule-triggered run via `workflow_call` (see [reusable-workflow-event-context-propagation.md](reusable-workflow-event-context-propagation.md) for why `event_name` must be passed explicitly rather than read from `github.event_name`).
2. `inputs.event_name == 'workflow_dispatch'` — a consumer manually ran their own caller workflow, which forwarded the trigger via `workflow_call`.
3. `github.event_name == 'workflow_dispatch'` — this reusable workflow was dispatched directly (no caller involved), the shared-repo manual-testing path.

**Caller template — forward the input, defaulting safely on non-dispatch triggers:**

```yaml
on:
  workflow_dispatch:
    inputs:
      enable_from_cache:
        description: 'Read from the Docker layer cache on this manual run (default: false)'
        required: false
        default: false
        type: boolean
jobs:
  build:
    uses: Taegost/shared-workflows/.github/workflows/docker-build-push.yml@vX
    with:
      event_name: ${{ github.event_name }}
      enable_from_cache: ${{ inputs.enable_from_cache || false }}
    secrets: inherit
```

`inputs.enable_from_cache` is only populated when the caller itself was `workflow_dispatch`-triggered; on push/PR/schedule triggers it evaluates to an empty value, so `|| false` makes the fallback explicit instead of passing an undefined value into a boolean input.

## Why This Matters

A cache is supposed to be an optimization, not a correctness hazard. When the entire purpose of a trigger (schedule, or an ad-hoc manual run) is to force freshness, an unconditional cache-from silently undermines that purpose — the workflow reports success, an image is pushed, but it may not actually contain the upstream patch the rebuild was meant to pick up. This class of bug is hard to notice because nothing fails; the build just quietly reuses old layers.

Making this an opt-in, off-by-default input for ad-hoc triggers is a breaking change for any consumer that was relying on scheduled/manual builds being cache-accelerated — but the prior behavior was arguably never correct for those triggers in the first place.

## When to Apply

- A reusable workflow has a trigger whose purpose is to force a rebuild independent of source changes (schedule, manual dispatch, "refresh" triggers).
- The build uses a layer cache that could satisfy an upstream/base dependency from a stale previous run.
- You need one input usable from both `workflow_call` and `workflow_dispatch` entry points — declare it identically in both places rather than trying to share a single `inputs` block across triggers (GitHub Actions doesn't allow that; the same name/type in each is the supported pattern).

## Related

- [Passing Event Context into Reusable GitHub Actions Workflows](reusable-workflow-event-context-propagation.md) — the `event_name` input pattern this fix depends on to distinguish schedule/manual triggers inside a `workflow_call`.
- [Docker build-and-push reusable workflow plan](../../plans/2026-06-21-001-feat-docker-build-push-workflow-plan.md) — original workflow implementation.
