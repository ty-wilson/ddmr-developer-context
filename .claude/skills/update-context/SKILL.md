---
name: update-context
description: Update the shared DDmR developer context docs when you discover new or incorrect architectural information during work in any DDmR repo.
---

# Update DDmR Developer Context

## When to Use

- You discovered information in the code that contradicts the context docs
- You learned about a new service, topic, endpoint, or pattern that isn't documented
- You traced a cross-service flow and now understand it better than the docs describe
- The developer asks you to update the context docs

## Find the Repo

The `ddmr-developer-context` repo is a sibling of the repo you're currently in:

```bash
CONTEXT_REPO="$(git rev-parse --show-toplevel)/../ddmr-developer-context"
ls "$CONTEXT_REPO/docs/" 2>/dev/null || echo "not found"
```

If not found, ask the developer where it is.

## Match Discovery to Doc

| Topic | File |
|-------|------|
| API routes, Tyk, HTTP contracts | `docs/api-layer.md` |
| Pulsar topics, events, producers/consumers | `docs/event-layer.md` |
| JWT sidecar, M2M auth, tenancy | `docs/auth-and-tenancy.md` |
| DynamoDB tables, keys, GSIs | `docs/database.md` |
| Test repos, testing patterns | `docs/testing.md` |
| Grafana, metrics, logging | `docs/observability.md` |
| ArgoCD, releases, shared values | `docs/cicd-pipeline.md` |
| Terraform, AWS accounts, IAM | `docs/infrastructure.md` |
| Helm charts, K8s, Backstage | `docs/kubernetes.md` |
| Client libraries, Gradle plugin | `docs/shared-libraries.md` |
| MFE, schema pipeline, shell | `docs/frontend.md` |
| Service inventory changes | `CLAUDE.md` (Layer 0) |

## Make the Update

1. Read the target file to understand its current structure
2. Edit the relevant section — correct wrong info, add new info, or remove stale info
3. Update the `Last reviewed` date at the top to today's date
4. Keep within limits: Layer 0 < 100 lines, Layer 1 docs 100-200 lines

## After Updating

Tell the developer what changed and suggest they review and commit:

```bash
cd <path-to-ddmr-developer-context>
git diff
git add <file>
git commit -m "update <doc>: <what changed>"
```

Do not commit automatically — let the developer review first.

## Rules

- Only document what you've verified in code — no speculation
- Don't rewrite entire docs when only a section needs updating
- Don't add implementation details that belong in the service's own CLAUDE.md
- If an update would push a doc over its line limit, trim less important content
