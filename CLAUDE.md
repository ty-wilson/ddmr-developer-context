# DDmR Platform — Service Map

## Services

(to be populated after Layer 1 research)

## Communication

- **Sync (HTTP):** Services communicate via Tyk API gateway. M2M auth via robocop.
- **Async (Events):** Apache Pulsar under the `pdd` tenant. Platform topics in `default` namespace, service-owned topics in per-service namespaces.

## Deep Dives

When you need deeper architectural context, read the relevant doc from `~/Projects/DDmR/ddmr-developer-context/docs/`:

- **HTTP calls between services, Tyk gateway, API contracts** → read `docs/api-layer.md`
- **Pulsar events, topic routing, event schemas** → read `docs/event-layer.md`
- **Authentication, JWT sidecar, tenant resolution** → read `docs/auth-and-tenancy.md`
- **DynamoDB table designs, GSIs, key patterns** → read `docs/database.md`
- **Test repos, component/system/perf/contract testing** → read `docs/testing.md`
- **Grafana dashboards, metrics, logging, alerting** → read `docs/observability.md`
- **Deployments, ArgoCD, shared values, release flow** → read `docs/cicd-pipeline.md`
- **Terraform, AWS accounts, regions, IAM** → read `docs/infrastructure.md`
- **Helm charts, values layering, pod topology, Backstage** → read `docs/kubernetes.md`
- **Shared client libraries, messaging client, Gradle plugin** → read `docs/shared-libraries.md`
- **Micro-frontends, schema pipeline, shell integration** → read `docs/frontend.md`

If you discover that information in these docs is outdated or incorrect based on what you observe in the code, flag it to the user.
