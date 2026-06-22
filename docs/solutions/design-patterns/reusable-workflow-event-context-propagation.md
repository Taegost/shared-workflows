---
title: "Passing Event Context into Reusable GitHub Actions Workflows"
date: "2026-06-21"
last_updated: "2026-06-21"
category: docs/solutions/design-patterns
module: shared-workflows
problem_type: design_pattern
component: development_workflow
severity: medium
applies_when:
  - "Reusable workflows called via workflow_call need to distinguish trigger events"
  - "Schedule triggers must apply different logic than push or tag triggers"
  - "github.event_name is unavailable inside the reusable workflow"
tags:
  - github-actions
  - reusable-workflows
  - workflow-call
  - event-context
  - schedule-trigger
  - tagging-strategy
---

# Passing Event Context into Reusable GitHub Actions Workflows

## Context

When a reusable workflow is invoked via `workflow_call`, `github.event_name` is still present inside the reusable workflow but is always set to the string `"workflow_call"` -- it does NOT reflect the original trigger event that started the caller. This means a reusable workflow cannot determine whether it was invoked by a schedule trigger, a tag push, a pull request, or any other event by checking `github.event_name`.

In a concrete case, a reusable Docker build-and-push workflow needed to apply different tagging strategies depending on the trigger: semver tags get versioned tags, while schedule triggers should only update the `latest` tag. The plan's transformation rules replaced `github.event_name`-based conditionals with `github.ref`-based equivalents, but this approach fails for schedule triggers because `github.ref` on a schedule event defaults to the default branch ref, making it indistinguishable from a regular push-to-main.

The intended behavior was: schedule triggers run the full pipeline (like a semver tag release) but only update the `latest` tag. The `github.ref`-based approach could not distinguish these cases.

## Guidance

**Pattern: Pass event context as an explicit input parameter.**

The calling workflow has access to the real `github.event_name`. By passing this value (or a derived boolean) as an input to the reusable workflow, the reusable workflow can branch on the actual trigger event.

**Reusable workflow definition -- declare the input:**

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
```

**Caller -- pass the real event context:**

```yaml
jobs:
  build:
    uses: Taegost/shared-workflows/.github/workflows/docker-build-push.yml@v1.0.0
    with:
      schedule_trigger: ${{ github.event_name == 'schedule' }}
    secrets: inherit
```

**Reusable workflow -- branch on the input:**

```yaml
- name: Determine Docker tags
  id: tags
  run: |
    if [[ "${{ inputs.schedule_trigger }}" == "true" ]]; then
      echo "tags=latest" >> "$GITHUB_OUTPUT"
    else
      # Apply semver versioned tagging
      echo "tags=${{ steps.meta.outputs.version }}" >> "$GITHUB_OUTPUT"
    fi
```

The boolean input is explicit, self-documenting, and decouples the reusable workflow from any knowledge of the caller's trigger mechanism.

**Full transformation table for `github.event_name` → `github.ref` replacements:**

When migrating a standalone workflow to a reusable `workflow_call` workflow, apply these transformations to every `github.event_name` conditional:

| Original condition | Reusable workflow replacement |
|---|---|
| `github.event_name == 'push' && startsWith(github.ref, 'refs/tags/')` | `startsWith(github.ref, 'refs/tags/')` |
| `github.event_name != 'pull_request'` | `${{ !startsWith(github.ref, 'refs/pull/') }}` |
| `push: ${{ github.event_name != 'pull_request' }}` | `push: ${{ !startsWith(github.ref, 'refs/pull/') }}` |
| `enable={{is_default_branch}}` (metadata-action tagging) | `enable=${{ github.ref == format('refs/heads/{0}', github.event.repository.default_branch) \|\| startsWith(github.ref, 'refs/tags/') \|\| inputs.schedule_trigger }}` |

The `github.ref` on a pull request is `refs/pull/N/merge`, so `startsWith(github.ref, 'refs/pull/')` is the correct inverse of `github.event_name != 'pull_request'`. The `{{is_default_branch}}` template variable from `docker/metadata-action` may not resolve correctly in a reusable workflow context, so use an explicit `format()` expression instead.

## Why This Matters

The core technical constraint is that `github.event_name` inside a `workflow_call` is always `"workflow_call"` -- GitHub does not propagate the original event name into the reusable workflow's context. Any logic that branches on the trigger event will fail silently or produce incorrect behavior if it relies on `github.event_name` being the original trigger.

Using `github.ref` as a substitute is a common workaround that works for tag pushes and pull requests, but it breaks for schedule triggers because the ref defaults to the default branch. This creates a class of bugs that are difficult to diagnose: the workflow runs successfully but applies the wrong tagging strategy, and there is no obvious error because all steps complete without failure.

The pattern of passing event context as an input parameter eliminates this ambiguity entirely. It makes the caller responsible for providing the information it has, and the reusable workflow responsible for acting on it. This is the correct separation of concerns: the reusable workflow should not need to know how it was invoked, only what behavior to apply.

Failure to follow this pattern leads to:
- Schedule-triggered workflows that tag images the same way as push-to-main, producing incorrect `latest` tags
- Silent misconfiguration where the workflow appears to succeed but produces wrong output
- Fragile conditionals that break when new trigger types are added

## When to Apply

- The reusable workflow needs to differentiate behavior based on the trigger event (schedule, push, pull_request, tag, etc.)
- `github.event_name` is referenced inside a reusable workflow called via `workflow_call`
- The workflow uses `github.ref` as a proxy for trigger type and schedule triggers are involved
- Multiple caller workflows trigger the same reusable workflow with different event types
- The reusable workflow must remain generic and not depend on the caller's specific trigger configuration

## Examples

**Before -- broken: ref-based conditional fails for schedule triggers**

Reusable workflow:
```yaml
- name: Determine tags
  id: tags
  run: |
    if [[ "${{ github.ref }}" == refs/tags/v* ]]; then
      # Semver tag push - apply versioned tags
      echo "tags=${GITHUB_REF#refs/tags/}" >> "$GITHUB_OUTPUT"
    else
      # Default branch push or schedule - only update latest
      echo "tags=latest" >> "$GITHUB_OUTPUT"
    fi
```

On a schedule trigger, `github.ref` is `refs/heads/main`, so the else branch runs and only `latest` is tagged -- this accidentally works but is semantically wrong. If the schedule trigger is supposed to run the full pipeline (including versioned tagging logic), this approach cannot express that intent.

**After -- correct: explicit input parameter**

Reusable workflow declares `schedule_trigger` input and branches on it:
```yaml
- name: Determine tags
  id: tags
  run: |
    if [[ "${{ inputs.schedule_trigger }}" == "true" ]]; then
      # Schedule trigger: run full pipeline, only update latest tag
      echo "tags=latest" >> "$GITHUB_OUTPUT"
    elif [[ "${{ github.ref }}" == refs/tags/v* ]]; then
      # Semver tag push: apply versioned tags
      echo "tags=${GITHUB_REF#refs/tags/}" >> "$GITHUB_OUTPUT"
    else
      # Default branch push: apply latest tag
      echo "tags=latest" >> "$GITHUB_OUTPUT"
    fi
```

Caller:
```yaml
jobs:
  build:
    uses: Taegost/shared-workflows/.github/workflows/docker-build-push.yml@v1.0.0
    with:
      schedule_trigger: ${{ github.event_name == 'schedule' }}
    secrets: inherit
```

This correctly identifies schedule triggers regardless of which branch the schedule runs against, and allows the reusable workflow to apply the intended tagging strategy for each trigger type.

## Related

- [Docker build-and-push reusable workflow plan](../../plans/2026-06-21-001-feat-docker-build-push-workflow-plan.md) -- the implementation plan that uses this pattern
- [GitHub Actions deduplication research](../../../2026-06-21-shared-github-actions-dedup-plan.md) -- background on the `github.event_name` gotcha and `github.ref`-based transformations
- GitHub Actions documentation on `workflow_call` events and context propagation
