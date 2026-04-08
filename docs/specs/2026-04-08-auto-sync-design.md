# Auto-Sync: Automated Doc Freshness & Regeneration

**Date:** 2026-04-08
**Status:** Draft

## Problem

The ddmr-developer-context repo contains ~4,000 lines of architectural documentation across 28 files. This documentation drifts from reality as services evolve. Currently, updates depend on developers noticing stale info and fixing it manually — which doesn't happen reliably.

## Solution

A weekly GitHub Action that:
1. Detects which services have changed significantly since their doc was last reviewed
2. Regenerates mechanical sections (event topology, team ownership) from source-of-truth configs
3. For services with significant changes, dispatches Claude to read the diffs and intelligently update the docs
4. Opens a single PR with all updates for human review

Most weeks: cheap API calls, no AI cost, exits silently. When drift is detected: targeted Claude invocations on only the affected services.

## Architecture

```
Weekly cron
    │
    ├── Phase 1: Mechanical sync (shell script, GitHub API only)
    │   ├── Read catalog-info.yaml from each service repo → update team ownership in CLAUDE.md
    │   ├── Read topic.json files from event-bus-configuration-topics → update event topology in event-layer.md
    │   └── Check for new repos with system: blueprints not yet in services/
    │
    ├── Phase 2: Change detection (shell script, GitHub API only)
    │   ├── For each service repo, query: git commits to src/ since doc's Last reviewed date
    │   ├── Score by: commit count, files changed, whether key paths were touched
    │   └── Flag services above threshold
    │
    └── Phase 3: Intelligent update (Claude, only for flagged services)
        ├── For each flagged service: fetch the git diff since last review
        ├── Read the current service doc
        ├── Prompt Claude: "Here's the diff, here's the doc, what's stale? Update the doc."
        └── Write updated doc
    │
    └── Open PR (if any changes)
```

## Phase 1: Mechanical Sync

### Event Topology

Source of truth: `jamf/event-bus-configuration-topics/envs/prod/pdd/`

The script reads all `topic.json` files via `gh api`, extracts topic name, owner, and subscriber names, and regenerates the topic ownership table in `docs/event-layer.md`. The auto-generated section is fenced:

```markdown
<!-- AUTO:TOPICS:START -->
| Topic | Namespace | Producer | Consumers |
|-------|-----------|----------|-----------|
| ... | ... | ... | ... |
<!-- AUTO:TOPICS:END -->
```

The script only replaces content between these markers. Everything outside is human-maintained.

### Team Ownership

Source of truth: `catalog-info.yaml` in each service repo.

The script reads `spec.owner` and `metadata.annotations.jsm/team` from each known service repo. If ownership has changed, it updates the team grouping in `CLAUDE.md` and the `**Owner:**` line in the corresponding `services/*.md` file.

The service list in `CLAUDE.md` is fenced with markers. The `**Owner:**` line in service docs is updated by simple text replacement.

### New Service Detection

The script maintains a list of known repos in `scripts/repos.txt`. It queries the GitHub API for all repos in the `jamf` org with topic or description matching "blueprints" or known team names, and compares against the known list. New repos are logged as warnings in the PR body.

## Phase 2: Change Detection

For each service repo in `scripts/repos.txt`, the script:

1. Reads the `Last reviewed` date from the corresponding `services/<name>.md`
2. Queries the GitHub API: `GET /repos/jamf/<name>/commits?since=<last-reviewed>&path=src/`
3. Scores the change volume:
   - Number of commits touching `src/`
   - Number of unique files changed
   - Whether key files were touched (bonus weight):
     - Route/controller definitions (`**/rest/**`, `**/controller/**`, `**/handler/**`)
     - DynamoDB/database model code (`**/dynamo/**`, `**/repository/**`, `**/entity/**`)
     - Pulsar/event config (`**/config/*Pulsar*`, `**/config/*Messaging*`, `**/config/*Event*`)
     - Helm values (`helm/**/values.yaml`)
     - Build dependencies (`build.gradle.kts`, `pom.xml`, `package.json`)

4. Flags the service if: commits > 10, OR key files touched > 3, OR any route/controller definition changed

Thresholds are configurable in `scripts/sync-config.yaml`.

## Phase 3: Intelligent Update

Only runs for flagged services. Uses `anthropics/claude-code-action` or a direct Claude API call.

For each flagged service:

1. Fetch the commit log and diff summary since `Last reviewed` date via GitHub API
2. Read the current `services/<name>.md`
3. Prompt Claude:

```
You are updating a service summary doc for <service-name>.

## Current doc
<contents of services/<name>.md>

## Changes since last review (<date>)
<git log --oneline --stat output>

## Key diffs
<abbreviated diff of the most-changed files>

## Instructions
- Update the doc to reflect any changes visible in the diffs
- Do NOT remove information that isn't contradicted by the diffs — it may still be correct
- Update the Last reviewed date to today
- Keep the same structure and style
- Keep under 200 lines
- If a change is ambiguous, leave the original and add a note: "(verify: may have changed per <commit>)"
```

4. Write the updated doc
5. Add to the PR

### Cost Control

- Phase 1 and 2: ~50 GitHub API calls total. Free.
- Phase 3: Only runs for flagged services. Typical week: 0-2 services flagged. Worst case (major refactor week): 5-6 services. Each invocation is one Claude call with ~2K tokens of doc + ~5K tokens of diff context. Very cheap.
- The `sync-config.yaml` has a `max-ai-updates-per-run` cap (default: 5) to prevent runaway costs.

## File Structure

```
ddmr-developer-context/
├── scripts/
│   ├── sync.sh                  ← Main entry point (phases 1-3)
│   ├── sync-config.yaml         ← Thresholds, repo list, caps
│   ├── mechanical-sync.sh       ← Phase 1: event topology, ownership
│   ├── change-detection.sh      ← Phase 2: git log scoring
│   └── intelligent-update.sh    ← Phase 3: Claude invocation
├── .github/
│   └── workflows/
│       └── auto-sync.yml        ← Weekly cron + manual dispatch
└── docs/
    └── event-layer.md           ← Has AUTO: markers for topic table
```

## GitHub Action

```yaml
name: Auto-Sync Docs
on:
  schedule:
    - cron: '0 9 * * 1'    # Weekly Monday 9am UTC
  workflow_dispatch:         # Manual trigger

permissions:
  contents: write
  pull-requests: write

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run mechanical sync
        run: ./scripts/mechanical-sync.sh
        env:
          GH_TOKEN: ${{ secrets.ORG_READ_TOKEN }}
      - name: Run change detection
        run: ./scripts/change-detection.sh
        env:
          GH_TOKEN: ${{ secrets.ORG_READ_TOKEN }}
      - name: Run intelligent updates (if needed)
        if: steps.change-detection.outputs.flagged_count > 0
        run: ./scripts/intelligent-update.sh
        env:
          GH_TOKEN: ${{ secrets.ORG_READ_TOKEN }}
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      - name: Create PR (if changes)
        run: |
          if [ -n "$(git status --porcelain)" ]; then
            git checkout -b auto-sync/$(date +%Y-%m-%d)
            git add -A
            git commit -m "auto-sync: update docs from source repos"
            git push -u origin HEAD
            gh pr create \
              --title "auto-sync: weekly doc update $(date +%Y-%m-%d)" \
              --body "$(cat scripts/sync.log)" \
              --reviewer "${{ github.actor }}"
          fi
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Auto-Generated Section Markers

Human-authored content and auto-generated content coexist in the same files. Auto-generated sections are fenced:

```markdown
Some human-written context above.

<!-- AUTO:TOPICS:START -->
This content is regenerated by scripts/mechanical-sync.sh.
Do not edit manually — changes will be overwritten.

| Topic | ... |
<!-- AUTO:TOPICS:END -->

More human-written context below.
```

The script replaces everything between START and END markers, preserving everything outside. Multiple marker pairs can exist in the same file.

## sync-config.yaml

```yaml
repos:
  - name: scoping-engine
    org: jamf
    key_paths: ["src/**/rest/**", "src/**/dynamo/**", "src/**/config/*Pulsar*"]
  - name: declaration-storage-service
    org: jamf
    key_paths: ["src/**/handler/**", "src/**/dynamo/**"]
  # ... one entry per service

event_topics_repo: jamf/event-bus-configuration-topics
event_topics_path: envs/prod/pdd

thresholds:
  min_commits: 10
  min_key_files: 3
  any_route_change: true

limits:
  max_ai_updates_per_run: 5

scripts:
  claude_model: claude-sonnet-4-6
  max_diff_tokens: 5000
```

## What This Does NOT Do

- Does not auto-merge PRs. A human always reviews.
- Does not update Layer 1 cross-cutting docs (api-layer, database, testing, etc.) — those require understanding across multiple repos simultaneously.
- Does not create new service docs from scratch — only updates existing ones. New service detection logs a warning.
- Does not modify source repos — this is read-only from the perspective of service repos.

## Secrets Required

- `ORG_READ_TOKEN`: GitHub PAT or App token with `repo:read` scope across the `jamf` org
- `ANTHROPIC_API_KEY`: For Phase 3 Claude invocations
- `GITHUB_TOKEN`: Default token for creating PRs in this repo
