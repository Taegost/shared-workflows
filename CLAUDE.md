# shared-workflows

Reusable GitHub Actions workflows shared across Taegost's projects.

## Architecture

The repo follows a **reusable workflow + caller template** pattern:

- **Shared workflow** (`.github/workflows/docker-build-push.yml`) — the reusable `workflow_call` workflow that contains the actual pipeline logic
- **Caller template** (`examples/caller-workflow.yml`) — a thin workflow that consumer repos copy into their `.github/workflows/` directory

Consumer repos reference the shared workflow at a pinned tag (e.g., `@v1.0.0`).

## Key Technical Decisions

**`github.ref` replaces `github.event_name` in all conditionals.** In a reusable workflow called via `workflow_call`, `github.event_name` is always `"workflow_call"` — it does not reflect the original trigger event. All conditionals must use `github.ref` instead, which IS correctly inherited from the caller.

**Schedule trigger context passed via input parameter.** `github.ref` alone cannot distinguish schedule triggers from default-branch pushes (both resolve to `refs/heads/main`). The caller passes `event_name: ${{ github.event_name }}` as a string input so the reusable workflow can check `inputs.event_name == 'schedule'` for `latest`-only tagging. See `docs/solutions/design-patterns/reusable-workflow-event-context-propagation.md`.

**Secrets passed via `secrets: inherit`.** The caller forwards all repository/org secrets. No `inputs` block is needed for secrets — the shared workflow reads them directly.

## Documented Solutions

`docs/solutions/` — documented solutions to past problems (bugs, best practices, workflow patterns), organized by category with YAML frontmatter (`module`, `tags`, `problem_type`). Relevant when implementing or debugging in documented areas.

## Conventions

- Action versions in workflows must match the source workflow exactly — no upgrades without explicit decision
- All existing comments from source workflows are preserved during migration
- The `v1.0.0` tag must point to the commit that contains the shared workflow file
- `workflow_dispatch` requires secrets configured as repository secrets in the shared repo (not just via `secrets: inherit`)
