---
title: "PR #1 review: grep logic flaw and cosign per-tag signing"
date: 2026-06-22
category: logic-errors
module: shared-workflows
problem_type: logic_error
component: tooling
root_cause: logic_error
resolution_type: code_fix
severity: high
symptoms:
  - "Workflow passes semver tag verification when no version tags are generated"
  - "Cosign signing makes N separate OIDC token exchanges for N tags on the same digest"
tags:
  - github-actions
  - docker
  - cosign
  - code-review
  - semver
  - grep
  - sigstore
  - reusable-workflows
---

# PR #1 review: grep logic flaw and cosign per-tag signing

## Problem

PR #1 of the shared Docker build-and-push workflow produced 8 code review findings. After investigation, 2 were confirmed as real bugs requiring fixes:

1. **Grep check logic flaw** — The "Verify semver tags were generated" step used `grep -qv "sha-"` which matches any non-SHA content (including `latest`), defeating the safety gate.
2. **Cosign per-tag signing** — Cosign signed each tag separately via `xargs`, making N redundant OIDC token exchanges and Rekor entries instead of signing the digest once.

## Symptoms

- The workflow completes successfully on tag pushes even when metadata-action fails to produce semver version tags — no error is surfaced until a consumer tries to pull by version.
- Cosign signing step makes 5 separate OIDC token exchanges and Rekor entries for a standard v1.2.3 release (tags: `1.2.3`, `1.2`, `1`, `latest`, `sha-abc123`), multiplying latency and transient failure risk.

## What Didn't Work

**Rejected: Secrets context unavailable in workflow-level `env:` blocks**

One finding claimed `${{ secrets.* }}` doesn't work in workflow-level `env:` blocks. This is incorrect for reusable workflows — secrets declared in `on.workflow_call.secrets` are available throughout the entire workflow, including `env:` blocks. No fix needed.

**Rejected: Tag pushes always update `latest`**

One finding claimed any semver tag push unconditionally updates `latest`. The `enable` condition on the `latest` tag entry was already correct: `startsWith(github.ref, 'refs/tags/')` evaluates to true on tag pushes, generating `latest` — which is the intended behavior per the documented tagging strategy.

## Solution

### Bug 1: Grep check — replace inversion with positive semver pattern match

Before:
```yaml
TAGS="${{ steps.meta.outputs.tags }}"
if ! echo "$TAGS" | grep -qv "sha-"; then
  echo "::error::No semver tags generated — verify your tag follows vMAJOR.MINOR.PATCH"
  exit 1
fi
```

After:
```yaml
TAGS="${{ steps.meta.outputs.tags }}"
# Check for at least one full MAJOR.MINOR.PATCH version tag (e.g. :1.2.3 or :v1.2.3).
# The metadata-action produces both v-prefixed and non-prefixed semver tags.
# This catches cases where metadata-action produced no version tags (e.g. invalid tag context).
# A simple "grep -qv sha-" would falsely pass on non-SHA tags like "latest".
if ! echo "$TAGS" | grep -qP ':v?\d+\.\d+\.\d+'; then
  echo "::error::No semver tags generated — verify your tag follows vMAJOR.MINOR.PATCH"
  exit 1
fi
```

### Bug 2: Cosign — sign digest once instead of per-tag

Before:
```yaml
env:
  TAGS: ${{ steps.meta.outputs.tags }}
  DIGEST: ${{ steps.build-and-push.outputs.digest }}
run: echo "${TAGS}" | xargs -I {} cosign sign --yes --oidc-provider=github-actions {}@${DIGEST}
```

After:
```yaml
env:
  DIGEST: ${{ steps.build-and-push.outputs.digest }}
# Sign the digest once (not per-tag). All tags point to the same digest,
# so a single signature covers every tag. When a tag like "latest" later
# drifts to a new digest, the old signature correctly does not apply.
# This avoids redundant OIDC token exchanges and Rekor transparency log entries.
run: cosign sign --yes --oidc-provider=github-actions ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${DIGEST}
```

## Why This Works

**Bug 1:** The Perl regex `:v?\d+\.\d+\.\d+` matches exactly the pattern semver tags produce — with or without the `v` prefix. The metadata-action generates both `v`-prefixed and non-prefixed tags (e.g. `v1.2.3` and `1.2.3`). From the tag output:
```
docker.io/user/image:v1.2.3     ← matches (v-prefixed semver)
docker.io/user/image:1.2.3      ← matches (three dot-separated numeric groups)
docker.io/user/image:1.2        ← no match (only two groups)
docker.io/user/image:1          ← no match (single digit)
docker.io/user/image:latest     ← no match (no digits)
docker.io/user/image:sha-abc123 ← no match (starts with "sha-")
```

The old `grep -qv "sha-"` had a double negation: it checked "are there lines NOT containing sha-", which is true whenever `latest` is present. The new approach checks directly for what matters.

**Bug 2:** Cosign signatures are keyed by digest, not tag. All tags point to the same image digest at build time. A single `cosign sign image@digest` covers every tag. When `latest` later drifts to a new digest, the old signature correctly does not apply to the new content.

## Prevention

1. **Avoid double-negative grep patterns.** When verifying the *presence* of something specific, use a positive match (`grep -q 'pattern'`) rather than negating an absence check (`! grep -qv 'other-pattern'`). Double negation is hard to reason about.

2. **Sign cosign images by digest, not by tag.** `cosign sign <image>@<digest>` is both more efficient and semantically correct. Document this pattern in workflow comments.

3. **Code review findings require reproduction.** Both bugs were confirmed through careful analysis. Two other findings from the same review were plausible but incorrect on investigation. Reproduce or prove each finding before accepting it.

4. **Test grep validation with negative cases.** The original check would have been caught if tested with a tag push producing only `latest` + `sha-*` (no semver). Add test cases for edge scenarios.

## Related

- `docs/solutions/design-patterns/reusable-workflow-event-context-propagation.md` — related context on `github.ref` vs `github.event_name` in reusable workflows
- `docs/2026-06-21-shared-github-actions-dedup-plan.md` — documents the source workflow's semver validation step (the first validation, not the grep verification fixed here)
