---
name: update-context
description: Update the shared DDmR developer context docs when you discover new or incorrect architectural information during work in any DDmR repo.
---

# Update DDmR Developer Context

## What these docs are for

They are a **map, not the territory**. Their job is to answer "which of the ~60 repos do I care about, who owns it, how do these pieces relate, and where should I look next." They are explicitly **not** a source of truth for current values or current state.

The organizing principle for every edit:

> **Document what is expensive to rediscover. Point at what is cheap to verify.**

A fact you can confirm with one command (a version number, a current config value, a GSI list, a subscriber count) is *cheap*. Never assert its value; document the **mechanism** and include the command to check it. A fact that takes an hour to reconstruct (why two teams solved something differently, which services participate in a flow, what an event means to its consumers, where an ownership boundary sits, a trap that has burned people before) is *expensive*. That belongs in prose, because rediscovering it is the cost these docs exist to remove.

Write so a reader is nudged to verify rather than to trust. Overconfident phrasing in these docs causes real downstream errors.

**Provenance is durable; state is not.** "Removed in DDMR-1035 (2026-04-02)" survives indefinitely and explains history. "Is not currently deployed" is wrong the moment someone deploys it. Prefer the former.

## When to Use

- You discovered information in the code that contradicts the context docs
- You learned about a new service, topic, endpoint, or pattern that isn't documented
- You traced a cross-service flow and now understand it better than the docs describe
- The developer asks you to update the context docs

## Find the Repo

The context repo lives at `~/Projects/DDmR/ddmr-developer-context`. Verify with:

```bash
ls ~/Projects/DDmR/ddmr-developer-context/docs/ 2>/dev/null || echo "not found"
```

Do NOT use `git rev-parse --show-toplevel` to find it. That returns the worktree path when in a worktree, breaking the relative path. Always use the absolute path above.

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
| Service internals, purpose, ownership | `services/<service-name>.md` |
| Service inventory changes | `CLAUDE.md` (Layer 0) |
| Point-in-time design analysis | `docs/specs/YYYY-MM-DD-<topic>.md` (see below) |

## Make the Update

1. Read the target file to understand its current structure
2. Edit the relevant section. Correct wrong info, add new info, remove stale info
3. Update the `Last reviewed` date at the top to today's date
4. Where you assert something checkable, add the command to check it
5. Respect the line budgets (below)

### Line budgets

- **Layer 0 (`CLAUDE.md`): hard limit under 100 lines.** This is loaded into *every* session in every sibling repo, so it is the only place the budget genuinely matters. Be ruthless. It should be little more than the service map, ownership, and the pointer table.
- **Layer 1 (`docs/*.md`): aim 100-300, but do not drop verified, expensive-to-rediscover content solely to fit.** These are read on demand, so length costs nothing until someone opens the file. If a doc is bursting, prefer splitting by topic or moving service-specific detail to Layer 2 over deleting knowledge.
- **Layer 2 (`services/*.md`): under 200 lines.**

## What NOT to Document

- **Specific values that will drift.** No version numbers ("schema version 27"), no current counts ("all four listeners", "sends three event types"), no dependency versions, no tuning constants (backoff multipliers, timeouts), no "as of <date> the value is X", no IDs that are lookups rather than stable identifiers. Document the *mechanism* that produces the value plus how to read it. Stable identifiers (AWS account numbers, regions, table names, topic names, hostnames) are fine.
- **Current-state snapshots of in-progress work.** Branch names, open PRs, "this is currently a stub", "not yet deployed", "team X is working on Y". These invert fast. If the *architecture* matters, describe the shape and note it is unmerged, or put the analysis in `docs/specs/` where a dated artifact is expected.
- **Implementation details** that belong in the service's own CLAUDE.md, or in Layer 2 if genuinely needed. Internal class names, method-to-handler tables, and package structure are usually single-repo detail with no cross-repo navigational value, and they drift on every refactor.
- **Speculation.** Only document what you verified. If you must record an inference, label it as one.

### Consumer and caller lists: keep them

An explicit exception to the "no snapshots" rule. DDmR services are stable, and knowing who consumes a topic or calls an endpoint is high-value for change planning. **Correct these lists rather than deleting them.** They are expensive to rediscover (you would have to grep 60 repos), which is exactly why they belong here. "(none registered)" is useful information, not a gap.

### Hard-won traps: keep them

Non-obvious behavior that has burned someone (a metric that means something other than its name, a flag whose override key is `disabled` rather than `enabled: false`, a cursor reset that does not clear a delay queue, a local regeneration that produces diffs CI rejects) is the highest-value content in these docs. It is durable, and it is nearly impossible to rediscover without repeating the incident. Never prune it for length.

## Reduce overconfidence

- **Prefer mechanism over value.** "The MFEs request the version returned by `GET /v1/version`, which can lag the SSM latest-ingested value" survives; "the version is 27" does not.
- **Add a verification recipe** for anything a reader might build on. `docs/dss-401-tenant-mismatch-playbook.md` is the model: task-shaped, with a "Where to find the data" section of literal commands. Copy that shape. A doc that asserts checkable facts but gives no way to check them is the pattern most likely to mislead.
- **Separate verified from inferred.** If a claim was reasoned rather than observed, say so inline. Unmarked claims are read as verified.
- **Be especially careful with negative and absolute claims** ("no service queries this", "X does not support Y", "only Z does this"). These are usually derived from grepping merged `main` and get falsified by unmerged work or an unsearched path. State what you searched.
- **Check for in-flight branches before concluding another team has not built something.** `origin/main` does not show unmerged work, and concluding "service X does not do Y" from main alone has produced real errors:
  ```bash
  git -C <repo> fetch origin --quiet
  git -C <repo> for-each-ref --sort=-committerdate \
    --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -20
  ```
- **Distinguish "last edited" from "last re-verified against code."** A date on a file that was only reformatted tells a reader nothing about freshness.

## Specs vs reference docs

`docs/specs/YYYY-MM-DD-<topic>.md` holds **dated, point-in-time analysis**: design proposals, investigations, comparisons, decision records. Specs *may* contain current-state detail, branch names, and effort estimates, because the date is in the filename and a reader knows to treat them as a snapshot.

Reference docs (Layers 0-2) must stay durable. When an investigation produces both, put the durable architecture in the reference doc and the analysis in a spec, and cross-link.

## After Updating

Tell the developer what changed and suggest they review and commit:

```bash
cd ~/Projects/DDmR/ddmr-developer-context
git diff
git add <file>
git commit -m "update <doc>: <what changed>"
```

Do not commit automatically. Let the developer review first.

## Rules

- Don't rewrite entire docs when only a section needs updating
- Read the context repo's own CLAUDE.md for conventions if unsure about structure
- If you are correcting something you previously got wrong, record the correction so it is not re-derived
