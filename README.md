# DDmR Developer Context

Shared architectural knowledge for the DDmR platform, designed for both human developers and AI coding assistants (Claude Code).

## How It Works

Claude Code automatically reads `CLAUDE.md` files from the current working directory **and every parent directory** up to the filesystem root. Since all DDmR repos are cloned under a shared parent directory (e.g., `~/Projects/DDmR/`), placing a `CLAUDE.md` at that level means it's loaded into every Claude session in any child repo — `scoping-engine`, `declaration-service`, `micro-frontend-hub`, etc.

This repo contains that shared `CLAUDE.md` plus deeper reference docs. A symlink connects the two:

```
~/Projects/DDmR/
├── CLAUDE.md → ddmr-developer-context/CLAUDE.md   ← symlink, auto-loaded every session
├── ddmr-developer-context/
│   ├── CLAUDE.md          ← Layer 0: service map (~90 lines, always in context)
│   ├── README.md
│   └── docs/              ← Layer 1: deep dives (~100-200 lines each, loaded on demand)
│       ├── api-layer.md
│       ├── event-layer.md
│       └── ...
├── scoping-engine/        ← your DDmR repos
├── declaration-service/
└── ...
```

**Layer 0** (the symlink target) is concise enough to always be in context without significant cost. It contains a service map and pointers telling Claude when to read the deeper docs.

**Layer 1** docs are only read when Claude determines from the task at hand that it needs more context — e.g., tracing an event flow, debugging a deployment, or understanding a DynamoDB schema.

## Setup

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed
- DDmR repos cloned under a shared parent directory (the path doesn't need to be `~/Projects/DDmR/` — any shared parent works, just adjust the commands below)

### Option A: Use the setup skill (recommended)

1. **Clone this repo** next to your other DDmR repos:
   ```bash
   git clone git@github.com:jamf/ddmr-developer-context.git
   ```

2. **Open Claude Code** inside the cloned repo and run `/setup`. The skill will find your DDmR repos, determine the correct parent directory, and create the symlink for you.

### Option B: Manual setup

1. **Clone this repo** next to your other DDmR repos:
   ```bash
   cd ~/Projects/DDmR
   git clone git@github.com:jamf/ddmr-developer-context.git
   ```

2. **Create the symlink** at the common parent directory of all your DDmR repos:
   ```bash
   ln -s ~/Projects/DDmR/ddmr-developer-context/CLAUDE.md ~/Projects/DDmR/CLAUDE.md
   ```

   If your repos are somewhere else (e.g., `~/code/jamf/`), adjust both paths:
   ```bash
   ln -s ~/code/jamf/ddmr-developer-context/CLAUDE.md ~/code/jamf/CLAUDE.md
   ```

### Verify

Open Claude Code in any DDmR repo and ask "what services are in the DDmR platform?" Claude should answer from the service map without you pointing it at any file.

### Removing the symlink

```bash
rm ~/Projects/DDmR/CLAUDE.md   # removes the symlink only, not the file it points to
```

## Structure

- **CLAUDE.md** — Layer 0: concise service map, always loaded
- **docs/*.md** — Layer 1: domain deep dives, loaded on demand by Claude when needed

### Layer 1 Docs

| Doc | Domain |
|-----|--------|
| [api-layer.md](docs/api-layer.md) | Tyk gateway, service-to-service HTTP, API contracts |
| [event-layer.md](docs/event-layer.md) | Pulsar topics, event schemas, producer/consumer map |
| [auth-and-tenancy.md](docs/auth-and-tenancy.md) | JWT sidecar, robocop M2M, tenant resolution |
| [database.md](docs/database.md) | DynamoDB table designs, GSIs, key patterns |
| [testing.md](docs/testing.md) | Component/system/perf/contract test repos |
| [observability.md](docs/observability.md) | Grafana dashboards, metrics, logging |
| [cicd-pipeline.md](docs/cicd-pipeline.md) | ArgoCD, components repo, shared values, releases |
| [infrastructure.md](docs/infrastructure.md) | Terraform, AWS accounts, regions, IAM |
| [kubernetes.md](docs/kubernetes.md) | Helm charts, values layering, pod topology |
| [shared-libraries.md](docs/shared-libraries.md) | Messaging client, storage clients, Gradle plugin |
| [frontend.md](docs/frontend.md) | MFE architecture, schema pipeline, shell integration |

## Keeping Docs Updated

When you discover new or incorrect architectural information while working in any DDmR repo, Claude can update these docs directly. The Layer 0 CLAUDE.md instructs Claude to read the update instructions at `.claude/skills/update-context/SKILL.md` when it detects stale information or when you ask it to update the docs.

The workflow:
1. You're working in `scoping-engine` and discover the event-layer doc is missing a new consumer
2. Tell Claude "update the context docs" — it reads the update skill, finds this repo, and edits the relevant doc
3. You review the change, commit, and PR

## Contributing

- When you discover outdated or incorrect information while working, fix it here via PR.
- Each Layer 1 doc has a `Last reviewed` date at the top — update it when you verify content is still accurate.
- Keep Layer 0 under 100 lines. If a service needs more than one line, the detail belongs in a Layer 1 doc.
- Layer 1 docs should stay in the 100-200 line range. If one grows beyond that, consider splitting it.
