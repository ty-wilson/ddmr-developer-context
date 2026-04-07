# DDmR Developer Context

Shared architectural knowledge for the DDmR platform. Designed to be automatically loaded by Claude Code when working in any DDmR repo.

## Setup

1. Clone this repo under your DDmR project directory:
   ```bash
   git clone <repo-url> ~/Projects/DDmR/ddmr-developer-context
   ```

2. Symlink the Layer 0 doc to the DDmR parent directory:
   ```bash
   ln -s ~/Projects/DDmR/ddmr-developer-context/CLAUDE.md ~/Projects/DDmR/CLAUDE.md
   ```

Claude Code reads `CLAUDE.md` from every parent directory. The symlink ensures the service map is loaded in every DDmR repo session automatically.

## Structure

- **CLAUDE.md** — Layer 0: concise service map, always loaded (~60 lines)
- **docs/*.md** — Layer 1: domain deep dives, loaded on demand by Claude when needed

## Contributing

When you discover outdated information while working in a DDmR repo, fix it here via PR. Each doc has a `Last reviewed` date at the top — update it when you verify content is still accurate.

## Layer 1 Docs

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
