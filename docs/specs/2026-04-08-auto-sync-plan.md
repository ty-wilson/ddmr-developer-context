# Auto-Sync Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Automated weekly GitHub Action that regenerates mechanical doc sections from source repos, detects significant service changes, and dispatches Claude to update stale service docs — opening a PR for human review.

**Architecture:** Three-phase pipeline: (1) shell scripts read GitHub API to sync event topology and team ownership, (2) shell script scores commit volume per service since last review, (3) Claude API call updates docs for flagged services. All orchestrated by a single GitHub Action on a weekly cron.

**Tech Stack:** Bash, `gh` CLI, `jq`, `yq`, GitHub Actions, Anthropic API (curl)

---

## File Structure

```
ddmr-developer-context/
├── scripts/
│   ├── sync.sh                  ← Main entry point, orchestrates phases 1-3
│   ├── sync-config.yaml         ← Repo list, thresholds, limits
│   ├── mechanical-sync.sh       ← Phase 1: topic table + ownership updates
│   ├── change-detection.sh      ← Phase 2: git log scoring, outputs flagged list
│   ├── intelligent-update.sh    ← Phase 3: Claude API calls for flagged services
│   └── lib/
│       ├── gh-helpers.sh        ← Shared functions: API reads, file fetching
│       └── marker-replace.sh    ← Replace content between AUTO markers
├── .github/
│   └── workflows/
│       └── auto-sync.yml        ← Weekly cron + manual dispatch
├── CLAUDE.md                    ← Gets AUTO markers around team-grouped service list
└── docs/
    └── event-layer.md           ← Gets AUTO markers around topic ownership table
```

---

### Task 1: Create sync-config.yaml with the full repo list

**Files:**
- Create: `scripts/sync-config.yaml`

- [ ] **Step 1: Write sync-config.yaml**

Write `scripts/sync-config.yaml`:

```yaml
# Repos tracked by auto-sync. Each entry maps to a services/<name>.md doc.
repos:
  # DDmR Team
  - name: scoping-engine
    org: jamf
    key_paths:
      - "src/**/rest/**"
      - "src/**/dynamo/**"
      - "src/**/config/*Pulsar*"
      - "src/**/config/*Messaging*"
  - name: declaration-service
    org: jamf
    key_paths:
      - "src/**/rest/**"
      - "src/**/handler/**"
      - "src/**/config/**"
  - name: declaration-storage-service
    org: jamf
    key_paths:
      - "src/**/handler/**"
      - "src/**/dynamo/**"
      - "src/**/config/*Pulsar*"
  - name: blueprint-component-custom-declarations
    org: jamf
    key_paths:
      - "src/**/controller/**"
      - "src/**/service/**"
  - name: ddmr-jwt-sidecar
    org: jamf
    key_paths:
      - "src/**/filter/**"
      - "src/**/config/**"
  - name: ddmr-authorizer-tenant
    org: jamf
    key_paths:
      - "src/**/handler/**"
      - "src/**/repository/**"

  # Ocean Team
  - name: blueprint-management-service
    org: jamf
    key_paths:
      - "src/**/controller/**"
      - "src/**/entity/**"
      - "src/**/pulsar/**"
  - name: blueprint-components-registry-service
    org: jamf
    key_paths:
      - "src/**/controller/**"
      - "src/**/entity/**"
  - name: blueprint-component-declarations-service
    org: jamf
    key_paths:
      - "src/**/controller/**"
      - "src/**/service/**"
  - name: blueprint-component-sw-update-service
    org: jamf
    key_paths:
      - "src/**/controller/**"
      - "src/**/service/**"

  # Goldminers Team
  - name: configuration-profile-service
    org: jamf
    key_paths:
      - "src/**/controller/**"
      - "src/**/service/**"
      - "src/**/entity/**"
  - name: configuration-profile-plist-migrator
    org: jamf
    key_paths:
      - "src/**/handler/**"
      - "src/**/service/**"
  - name: mdm-schema-ingest-inbound-adapter
    org: jamf
    doc_name: mdm-schema-ingest
    key_paths:
      - "ingest/src/**"
      - "transformation/src/**"

  # Other Teams
  - name: tenants-odin
    org: jamf
    key_paths:
      - "src/**/controller/**"
      - "src/**/service/**"
  - name: device-declaration-reporting-service
    org: jamf
    key_paths:
      - "src/**/controller/**"
      - "src/**/service/**"
  - name: micro-frontend-hub
    org: jamf
    key_paths:
      - "apps/**/src/**"
      - "libs/**/src/**"
  - name: json-schema-form-generator
    org: jamf
    key_paths:
      - "src/**"

# Event topology source
event_topics:
  repo: jamf/event-bus-configuration-topics
  path: envs/prod/pdd

# Change detection thresholds
thresholds:
  min_commits: 10
  min_key_files: 3
  any_route_change: true

# Cost caps
limits:
  max_ai_updates_per_run: 5

# Claude config
claude:
  model: claude-sonnet-4-6
  max_diff_lines: 500
```

- [ ] **Step 2: Commit**

```bash
git add scripts/sync-config.yaml
git commit -m "auto-sync: add sync-config.yaml with full repo list and thresholds"
```

---

### Task 2: Create shared helper library

**Files:**
- Create: `scripts/lib/gh-helpers.sh`
- Create: `scripts/lib/marker-replace.sh`

- [ ] **Step 1: Write gh-helpers.sh**

Write `scripts/lib/gh-helpers.sh`:

```bash
#!/usr/bin/env bash
# Shared helper functions for GitHub API access via gh CLI.
# Requires: gh CLI authenticated, GH_TOKEN set, jq installed.

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
REPO_ROOT="$(cd "$SCRIPT_DIR/../.." && pwd)"
LOG_FILE="${REPO_ROOT}/scripts/sync.log"

log() {
  echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] $*" | tee -a "$LOG_FILE"
}

warn() {
  echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] WARNING: $*" | tee -a "$LOG_FILE"
}

# Fetch a single file from a repo via GitHub API. Returns raw content.
# Usage: gh_fetch_file "jamf/scoping-engine" "catalog-info.yaml"
gh_fetch_file() {
  local repo="$1"
  local path="$2"
  gh api "repos/${repo}/contents/${path}" --jq '.content' 2>/dev/null | base64 -d 2>/dev/null || echo ""
}

# List files in a directory recursively via GitHub API.
# Usage: gh_list_tree "jamf/event-bus-configuration-topics" "envs/prod/pdd"
gh_list_tree() {
  local repo="$1"
  local path="$2"
  local branch
  branch=$(gh api "repos/${repo}" --jq '.default_branch' 2>/dev/null)
  gh api "repos/${repo}/git/trees/${branch}?recursive=1" --jq ".tree[] | select(.path | startswith(\"${path}\")) | .path" 2>/dev/null
}

# Get commits since a date for a given path in a repo.
# Usage: gh_commits_since "jamf/scoping-engine" "2026-04-07" "src/"
gh_commits_since() {
  local repo="$1"
  local since="$2"
  local path="${3:-}"
  local url="repos/${repo}/commits?since=${since}T00:00:00Z&per_page=100"
  if [ -n "$path" ]; then
    url="${url}&path=${path}"
  fi
  gh api "$url" 2>/dev/null
}

# Get commit count since a date for a path.
# Usage: gh_commit_count "jamf/scoping-engine" "2026-04-07" "src/"
gh_commit_count() {
  gh_commits_since "$1" "$2" "$3" | jq 'length'
}

# Get list of files changed in commits since a date.
# Usage: gh_changed_files "jamf/scoping-engine" "2026-04-07"
gh_changed_files() {
  local repo="$1"
  local since="$2"
  local shas
  shas=$(gh_commits_since "$repo" "$since" | jq -r '.[].sha')
  for sha in $shas; do
    gh api "repos/${repo}/commits/${sha}" --jq '.files[].filename' 2>/dev/null
  done | sort -u
}

# Read sync-config.yaml. Requires yq.
# Usage: config_get '.thresholds.min_commits'
config_get() {
  yq "$1" "${REPO_ROOT}/scripts/sync-config.yaml"
}

# Get repo list from config.
# Usage: config_repos | while read -r name org doc_name; do ... done
config_repos() {
  yq -r '.repos[] | [.name, .org, (.doc_name // .name)] | @tsv' "${REPO_ROOT}/scripts/sync-config.yaml"
}

# Get key_paths for a repo.
# Usage: config_key_paths "scoping-engine"
config_key_paths() {
  local name="$1"
  yq -r ".repos[] | select(.name == \"${name}\") | .key_paths[]" "${REPO_ROOT}/scripts/sync-config.yaml"
}

# Initialize the log file
init_log() {
  echo "# Auto-Sync Run: $(date -u +%Y-%m-%dT%H:%M:%SZ)" > "$LOG_FILE"
  echo "" >> "$LOG_FILE"
}
```

- [ ] **Step 2: Write marker-replace.sh**

Write `scripts/lib/marker-replace.sh`:

```bash
#!/usr/bin/env bash
# Replace content between AUTO markers in a file.
#
# Marker format:
#   <!-- AUTO:<SECTION>:START -->
#   ...content replaced...
#   <!-- AUTO:<SECTION>:END -->
#
# Usage: marker_replace "docs/event-layer.md" "TOPICS" "new content here"

set -euo pipefail

marker_replace() {
  local file="$1"
  local section="$2"
  local new_content="$3"
  local start_marker="<!-- AUTO:${section}:START -->"
  local end_marker="<!-- AUTO:${section}:END -->"

  if ! grep -q "$start_marker" "$file"; then
    echo "ERROR: Start marker '${start_marker}' not found in ${file}" >&2
    return 1
  fi
  if ! grep -q "$end_marker" "$file"; then
    echo "ERROR: End marker '${end_marker}' not found in ${file}" >&2
    return 1
  fi

  # Build replacement: start marker + new content + end marker
  local tmpfile
  tmpfile=$(mktemp)

  awk -v start="$start_marker" -v end="$end_marker" -v content="$new_content" '
    $0 ~ start { print; printf "%s\n", content; skip=1; next }
    $0 ~ end   { skip=0 }
    !skip       { print }
  ' "$file" > "$tmpfile"

  mv "$tmpfile" "$file"
}
```

- [ ] **Step 3: Make scripts executable and commit**

```bash
chmod +x scripts/lib/gh-helpers.sh scripts/lib/marker-replace.sh
git add scripts/lib/
git commit -m "auto-sync: add shared helper libraries (gh-helpers, marker-replace)"
```

---

### Task 3: Implement Phase 1 — Mechanical Sync

**Files:**
- Create: `scripts/mechanical-sync.sh`
- Modify: `docs/event-layer.md` (add AUTO markers around topic table)
- Modify: `CLAUDE.md` (add AUTO markers around team-grouped service list)

- [ ] **Step 1: Add AUTO markers to event-layer.md**

Read `docs/event-layer.md`, find the topic ownership tables (the `### pdd/default`, `### pdd/scoping-engine`, etc. sections with their tables), and wrap them with:

```markdown
<!-- AUTO:TOPICS:START -->
(existing topic tables stay here)
<!-- AUTO:TOPICS:END -->
```

Place the START marker just before `### pdd/default` and the END marker after the last topic table section.

- [ ] **Step 2: Add AUTO markers to CLAUDE.md**

Read `CLAUDE.md`, wrap the team-grouped service lists (from `### DDmR Team` through the end of `### External Services`) with:

```markdown
<!-- AUTO:OWNERSHIP:START -->
(existing team-grouped list stays here)
<!-- AUTO:OWNERSHIP:END -->
```

- [ ] **Step 3: Write mechanical-sync.sh**

Write `scripts/mechanical-sync.sh`:

```bash
#!/usr/bin/env bash
# Phase 1: Mechanical sync — regenerate event topology and team ownership
# from source-of-truth configs via GitHub API.

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "${SCRIPT_DIR}/lib/gh-helpers.sh"
source "${SCRIPT_DIR}/lib/marker-replace.sh"

sync_event_topology() {
  log "Phase 1a: Syncing event topology from event-bus-configuration-topics..."

  local topics_repo
  topics_repo=$(config_get '.event_topics.repo')
  local topics_path
  topics_path=$(config_get '.event_topics.path')

  # Find all topic.json files
  local topic_files
  topic_files=$(gh_list_tree "$topics_repo" "$topics_path" | grep 'topic.json$')

  local table=""
  local current_namespace=""

  while IFS= read -r topic_file; do
    local content
    content=$(gh_fetch_file "$topics_repo" "$topic_file")
    [ -z "$content" ] && continue

    local topic_name namespace short_name owner schema compat
    topic_name=$(echo "$content" | jq -r '.name')
    # Extract namespace: pdd/<namespace>/<topic> -> <namespace>
    namespace=$(echo "$topic_name" | cut -d'/' -f2)
    short_name=$(echo "$topic_name" | cut -d'/' -f3)
    owner=$(echo "$content" | jq -r '.properties.owner // "unknown"')
    schema=$(echo "$content" | jq -r '.settings.schemaEnforceValidation // false')
    compat=$(echo "$content" | jq -r '.settings.schemaCompatibilityStrategy // "NONE"')

    # Collect consumers across all regions
    local consumers
    consumers=$(echo "$content" | jq -r '[.subscriptions[]?[]?.name] | unique | join(", ")' 2>/dev/null)
    [ -z "$consumers" ] && consumers="(none)"

    # Group by namespace
    if [ "$namespace" != "$current_namespace" ]; then
      [ -n "$current_namespace" ] && table="${table}\n"
      table="${table}\n### pdd/${namespace}\n\n"
      table="${table}| Topic | Owner | Consumers | Schema | Compat |\n"
      table="${table}|-------|-------|-----------|--------|--------|\n"
      current_namespace="$namespace"
    fi

    table="${table}| ${short_name} | ${owner} | ${consumers} | ${schema} | ${compat} |\n"

  done <<< "$topic_files"

  local event_layer_file="${REPO_ROOT}/docs/event-layer.md"
  marker_replace "$event_layer_file" "TOPICS" "$(echo -e "$table")"
  log "Event topology updated with $(echo "$topic_files" | wc -l | tr -d ' ') topics."
}

sync_team_ownership() {
  log "Phase 1b: Syncing team ownership from catalog-info.yaml files..."

  local ownership_changes=0

  config_repos | while IFS=$'\t' read -r name org doc_name; do
    local catalog
    catalog=$(gh_fetch_file "${org}/${name}" "catalog-info.yaml")
    [ -z "$catalog" ] && continue

    local owner jsm_team
    owner=$(echo "$catalog" | yq '.spec.owner' 2>/dev/null)
    jsm_team=$(echo "$catalog" | yq '.metadata.annotations."jsm/team"' 2>/dev/null)
    [ "$owner" = "null" ] && owner=""
    [ "$jsm_team" = "null" ] && jsm_team=""

    local display_team="${jsm_team:-$owner}"
    [ -z "$display_team" ] && continue

    # Update the Owner line in the service doc
    local service_doc="${REPO_ROOT}/services/${doc_name}.md"
    if [ -f "$service_doc" ]; then
      local current_owner
      current_owner=$(grep '^\*\*Owner:\*\*' "$service_doc" | sed 's/\*\*Owner:\*\* //')
      local expected_owner="${display_team} team"
      if [ "$current_owner" != "$expected_owner" ] && [ -n "$current_owner" ]; then
        sed -i '' "s/^\*\*Owner:\*\*.*/\*\*Owner:\*\* ${expected_owner}/" "$service_doc"
        log "  Updated ownership for ${name}: ${current_owner} → ${expected_owner}"
        ownership_changes=$((ownership_changes + 1))
      fi
    fi
  done

  log "Ownership check complete. ${ownership_changes} changes."
}

detect_new_services() {
  log "Phase 1c: Checking for new Blueprints-system repos..."

  local known_repos
  known_repos=$(config_get '.repos[].name' | sort)

  # Search for repos with system: blueprints in the jamf org
  # This is a heuristic — we check repos we know about, not all jamf repos
  # New service detection relies on the repo list being maintained in sync-config.yaml
  warn "New service detection is passive — add new repos to scripts/sync-config.yaml manually."
}

# Main
init_log
sync_event_topology
sync_team_ownership
detect_new_services
log "Phase 1 complete."
```

- [ ] **Step 4: Make executable and commit**

```bash
chmod +x scripts/mechanical-sync.sh
git add scripts/mechanical-sync.sh docs/event-layer.md CLAUDE.md
git commit -m "auto-sync: implement Phase 1 mechanical sync (topics + ownership)"
```

---

### Task 4: Implement Phase 2 — Change Detection

**Files:**
- Create: `scripts/change-detection.sh`

- [ ] **Step 1: Write change-detection.sh**

Write `scripts/change-detection.sh`:

```bash
#!/usr/bin/env bash
# Phase 2: Change detection — score each service's commit volume since
# its doc was last reviewed. Output a list of flagged services.

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "${SCRIPT_DIR}/lib/gh-helpers.sh"

FLAGGED_FILE="${REPO_ROOT}/scripts/.flagged-services"
> "$FLAGGED_FILE"

FLAGGED_COUNT=0

log "Phase 2: Detecting changes since last review..."

config_repos | while IFS=$'\t' read -r name org doc_name; do
  local service_doc="${REPO_ROOT}/services/${doc_name}.md"

  # Get the Last reviewed date from the service doc
  if [ ! -f "$service_doc" ]; then
    warn "No service doc for ${name} at ${service_doc}"
    continue
  fi

  local last_reviewed
  last_reviewed=$(grep '^Last reviewed:' "$service_doc" | head -1 | sed 's/Last reviewed: //' | cut -d' ' -f1)
  if [ -z "$last_reviewed" ]; then
    warn "No 'Last reviewed' date in ${service_doc}"
    continue
  fi

  log "  Checking ${name} (last reviewed: ${last_reviewed})..."

  # Count commits to src/ since last reviewed
  local commit_count
  commit_count=$(gh_commit_count "${org}/${name}" "$last_reviewed" "src/")

  if [ "$commit_count" -eq 0 ]; then
    log "    No changes since last review."
    continue
  fi

  log "    ${commit_count} commits to src/ since ${last_reviewed}"

  # Check thresholds
  local min_commits
  min_commits=$(config_get '.thresholds.min_commits')
  local flagged=false

  if [ "$commit_count" -ge "$min_commits" ]; then
    log "    FLAGGED: commit count ${commit_count} >= threshold ${min_commits}"
    flagged=true
  fi

  # Check key paths if not already flagged
  if [ "$flagged" = false ]; then
    local changed_files
    changed_files=$(gh_changed_files "${org}/${name}" "$last_reviewed")
    local key_file_hits=0

    while IFS= read -r key_path; do
      # Convert glob to grep pattern (rough approximation)
      local pattern
      pattern=$(echo "$key_path" | sed 's/\*\*/.*/' | sed 's/\*/.*/g')
      local matches
      matches=$(echo "$changed_files" | grep -c "$pattern" 2>/dev/null || true)
      key_file_hits=$((key_file_hits + matches))
    done < <(config_key_paths "$name")

    local min_key_files
    min_key_files=$(config_get '.thresholds.min_key_files')
    if [ "$key_file_hits" -ge "$min_key_files" ]; then
      log "    FLAGGED: ${key_file_hits} key files changed >= threshold ${min_key_files}"
      flagged=true
    fi
  fi

  if [ "$flagged" = true ]; then
    echo "${name}|${org}|${doc_name}|${last_reviewed}" >> "$FLAGGED_FILE"
    FLAGGED_COUNT=$((FLAGGED_COUNT + 1))
  fi
done

log "Phase 2 complete. ${FLAGGED_COUNT} services flagged for intelligent update."

# Output for GitHub Actions
if [ -n "${GITHUB_OUTPUT:-}" ]; then
  echo "flagged_count=${FLAGGED_COUNT}" >> "$GITHUB_OUTPUT"
fi
```

- [ ] **Step 2: Make executable and commit**

```bash
chmod +x scripts/change-detection.sh
git add scripts/change-detection.sh
git commit -m "auto-sync: implement Phase 2 change detection (commit scoring)"
```

---

### Task 5: Implement Phase 3 — Intelligent Update

**Files:**
- Create: `scripts/intelligent-update.sh`

- [ ] **Step 1: Write intelligent-update.sh**

Write `scripts/intelligent-update.sh`:

```bash
#!/usr/bin/env bash
# Phase 3: Intelligent update — for each flagged service, fetch the diff
# since last review and ask Claude to update the service doc.

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "${SCRIPT_DIR}/lib/gh-helpers.sh"

FLAGGED_FILE="${REPO_ROOT}/scripts/.flagged-services"

if [ ! -f "$FLAGGED_FILE" ] || [ ! -s "$FLAGGED_FILE" ]; then
  log "Phase 3: No flagged services. Skipping."
  exit 0
fi

MAX_UPDATES=$(config_get '.limits.max_ai_updates_per_run')
MAX_DIFF_LINES=$(config_get '.claude.max_diff_lines')
CLAUDE_MODEL=$(config_get '.claude.model')
UPDATE_COUNT=0
TODAY=$(date -u +%Y-%m-%d)

log "Phase 3: Updating flagged services (max ${MAX_UPDATES})..."

while IFS='|' read -r name org doc_name last_reviewed; do
  if [ "$UPDATE_COUNT" -ge "$MAX_UPDATES" ]; then
    warn "Hit max AI updates cap (${MAX_UPDATES}). Remaining flagged services skipped."
    break
  fi

  log "  Updating ${name}..."

  # Fetch commit log since last review
  local commit_log
  commit_log=$(gh api "repos/${org}/${name}/commits?since=${last_reviewed}T00:00:00Z&path=src/&per_page=30" \
    --jq '.[] | "\(.sha[0:7]) \(.commit.message | split("\n")[0])"' 2>/dev/null | head -30)

  # Fetch abbreviated diff of recent commits (just stats, not full diff — to stay within token limits)
  local diff_summary=""
  local recent_shas
  recent_shas=$(echo "$commit_log" | head -10 | awk '{print $1}')
  for sha in $recent_shas; do
    local stat
    stat=$(gh api "repos/${org}/${name}/commits/${sha}" --jq '
      "### " + .sha[0:7] + " " + (.commit.message | split("\n")[0]) + "\n" +
      (.files | map(.filename + " (+" + (.additions|tostring) + " -" + (.deletions|tostring) + ")") | join("\n"))
    ' 2>/dev/null)
    diff_summary="${diff_summary}\n${stat}\n"
  done

  # Truncate diff to max lines
  diff_summary=$(echo -e "$diff_summary" | head -"$MAX_DIFF_LINES")

  # Read current doc
  local service_doc="${REPO_ROOT}/services/${doc_name}.md"
  local current_doc
  current_doc=$(cat "$service_doc")

  # Build the prompt
  local prompt
  prompt=$(cat <<PROMPT
You are updating a service summary doc for ${name}.

## Current doc
${current_doc}

## Changes since last review (${last_reviewed})
${commit_log}

## Key file changes
${diff_summary}

## Instructions
- Update the doc to reflect any changes visible in the commits and file changes
- Do NOT remove information that isn't contradicted by the changes — it may still be correct
- Update the "Last reviewed" date to ${TODAY}
- Keep the same structure and style
- Keep under 200 lines
- If a change is ambiguous, leave the original and add a note: "(verify: may have changed per <commit-sha>)"
- Output ONLY the updated markdown file content, no explanation
PROMPT
)

  # Call Claude API
  local response
  response=$(curl -s https://api.anthropic.com/v1/messages \
    -H "content-type: application/json" \
    -H "x-api-key: ${ANTHROPIC_API_KEY}" \
    -H "anthropic-version: 2023-06-01" \
    -d "$(jq -n \
      --arg model "$CLAUDE_MODEL" \
      --arg prompt "$prompt" \
      '{
        model: $model,
        max_tokens: 8192,
        messages: [{role: "user", content: $prompt}]
      }')")

  # Extract the text content from the response
  local updated_doc
  updated_doc=$(echo "$response" | jq -r '.content[0].text' 2>/dev/null)

  if [ -z "$updated_doc" ] || [ "$updated_doc" = "null" ]; then
    local error
    error=$(echo "$response" | jq -r '.error.message // "unknown error"' 2>/dev/null)
    warn "Claude API call failed for ${name}: ${error}"
    continue
  fi

  # Write the updated doc
  echo "$updated_doc" > "$service_doc"
  log "  Updated ${service_doc}"
  UPDATE_COUNT=$((UPDATE_COUNT + 1))

done < "$FLAGGED_FILE"

log "Phase 3 complete. ${UPDATE_COUNT} services updated."
```

- [ ] **Step 2: Make executable and commit**

```bash
chmod +x scripts/intelligent-update.sh
git add scripts/intelligent-update.sh
git commit -m "auto-sync: implement Phase 3 intelligent update (Claude API)"
```

---

### Task 6: Create the orchestrator script

**Files:**
- Create: `scripts/sync.sh`

- [ ] **Step 1: Write sync.sh**

Write `scripts/sync.sh`:

```bash
#!/usr/bin/env bash
# Main entry point for auto-sync. Runs all three phases sequentially.

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

echo "=== Auto-Sync: Starting ==="

# Phase 1: Mechanical sync
"${SCRIPT_DIR}/mechanical-sync.sh"

# Phase 2: Change detection
"${SCRIPT_DIR}/change-detection.sh"

# Phase 3: Intelligent update (only if services were flagged)
"${SCRIPT_DIR}/intelligent-update.sh"

echo "=== Auto-Sync: Complete ==="
echo "Check scripts/sync.log for details."
```

- [ ] **Step 2: Make executable and commit**

```bash
chmod +x scripts/sync.sh
git add scripts/sync.sh
git commit -m "auto-sync: add orchestrator script (sync.sh)"
```

---

### Task 7: Create the GitHub Action workflow

**Files:**
- Create: `.github/workflows/auto-sync.yml`

- [ ] **Step 1: Write auto-sync.yml**

Write `.github/workflows/auto-sync.yml`:

```yaml
name: Auto-Sync Docs

on:
  schedule:
    - cron: '0 9 * * 1'  # Weekly Monday 9am UTC
  workflow_dispatch:       # Manual trigger

permissions:
  contents: write
  pull-requests: write

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install yq
        run: |
          sudo wget -qO /usr/local/bin/yq https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64
          sudo chmod +x /usr/local/bin/yq

      - name: Configure git
        run: |
          git config user.name "auto-sync[bot]"
          git config user.email "auto-sync[bot]@users.noreply.github.com"

      - name: Run mechanical sync
        id: mechanical
        run: ./scripts/mechanical-sync.sh
        env:
          GH_TOKEN: ${{ secrets.ORG_READ_TOKEN }}

      - name: Run change detection
        id: detection
        run: ./scripts/change-detection.sh
        env:
          GH_TOKEN: ${{ secrets.ORG_READ_TOKEN }}

      - name: Run intelligent updates
        if: steps.detection.outputs.flagged_count != '0'
        run: ./scripts/intelligent-update.sh
        env:
          GH_TOKEN: ${{ secrets.ORG_READ_TOKEN }}
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}

      - name: Create PR if changes detected
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          if [ -z "$(git status --porcelain -- docs/ services/ CLAUDE.md)" ]; then
            echo "No changes detected. Exiting."
            exit 0
          fi

          BRANCH="auto-sync/$(date +%Y-%m-%d)"
          git checkout -b "$BRANCH"
          git add docs/ services/ CLAUDE.md
          git commit -m "auto-sync: weekly doc update $(date +%Y-%m-%d)"
          git push -u origin "$BRANCH"

          gh pr create \
            --title "auto-sync: weekly doc update $(date +%Y-%m-%d)" \
            --body "$(cat scripts/sync.log)" \
            --label "auto-sync"
```

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/auto-sync.yml
git commit -m "auto-sync: add GitHub Action workflow (weekly cron + manual dispatch)"
```

---

### Task 8: Add .gitignore entries and clean up

**Files:**
- Modify: `.gitignore`

- [ ] **Step 1: Add sync artifacts to .gitignore**

Append to `.gitignore`:

```
# Auto-sync temporary files
scripts/sync.log
scripts/.flagged-services
```

- [ ] **Step 2: Commit**

```bash
git add .gitignore
git commit -m "auto-sync: gitignore temporary sync artifacts"
```

---

### Task 9: Test locally

- [ ] **Step 1: Test mechanical sync (dry run)**

```bash
cd ~/Projects/DDmR/ddmr-developer-context
GH_TOKEN=$(gh auth token) ./scripts/mechanical-sync.sh
```

Verify: `scripts/sync.log` has output, `docs/event-layer.md` topic table is updated between markers, ownership lines in `services/*.md` are unchanged (or correctly updated if any drifted).

- [ ] **Step 2: Test change detection**

```bash
GH_TOKEN=$(gh auth token) ./scripts/change-detection.sh
```

Verify: `scripts/sync.log` shows commit counts per service. `scripts/.flagged-services` lists any services over threshold (may be empty if docs are fresh from today).

- [ ] **Step 3: Test intelligent update (optional, costs money)**

Only if `.flagged-services` is non-empty:

```bash
GH_TOKEN=$(gh auth token) ANTHROPIC_API_KEY=<your-key> ./scripts/intelligent-update.sh
```

Verify: flagged service docs are updated, `Last reviewed` date changed to today.

- [ ] **Step 4: Test full pipeline**

```bash
GH_TOKEN=$(gh auth token) ./scripts/sync.sh
```

Verify: all three phases run in sequence, `sync.log` has the full run output.

- [ ] **Step 5: Commit any marker additions and push**

```bash
git add -A
git commit -m "auto-sync: finalize markers and test artifacts"
git push
```
