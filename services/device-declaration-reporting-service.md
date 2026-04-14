# Device Declaration Reporting Service

Last reviewed: 2026-04-07

> **Point-in-time snapshot.** Verify critical claims against the actual code before acting on them.

**Owner:** Jabberwocky team

## Summary

Device Declaration Reporting Service (DRS) is a Spring Boot / Java 21 service (standard MVC, not WebFlux) owned by the jabberwocky team. Its role is device-oriented reporting of Declaration Management declarations — presenting per-device declaration status in a way that is useful to callers who need to know what declarations are currently applied to a given device and whether those declarations are in a desired state. The service is deployed to production. The only implemented endpoint is a placeholder `/test` route; major integration points (M2M auth, Pulsar consumption of status reports, S3/MinIO storage, Declaration Storage Service calls) are stubbed out or commented in config. Treat this service's API surface as unstable.

---

## API Endpoints

The service runs on port 8080 (application) and 8079 (management/actuator).

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/test` | Placeholder endpoint. Returns the string `"Test successful"`. Not intended for production use. |
| `GET` | `/actuator/health/liveness` | Kubernetes liveness probe. Port 8079. |
| `GET` | `/actuator/health/readiness` | Kubernetes readiness probe. Port 8079. |
| `GET` | `/actuator/prometheus` | Prometheus metrics scrape endpoint. Port 8079. |

No tenant-scoped or device-scoped reporting endpoints are implemented.

---

## Data Model

No persistent data store is owned by this service. The README notes S3 (via AWS SDK, with MinIO as a local stand-in) as the intended storage backend, but there is no S3 client code, no MinIO configuration in `application-local.yaml`, and no MinIO service in `docker-compose.yaml`.

**Upstream data source:** Apple MDM status reports published to the `persistent://pdd/apple-ddm/statusreport` Pulsar topic. The Pulsar subscription (`device-declaration-reporting-service`) is defined in commented-out config and not wired up.

---

## Dependencies

| Dependency | How it is used |
|---|---|
| Declaration Storage Service (DSS) | Listed in README as a direct dependency; integration not implemented in code |
| MDM Server (Jamf Pro) | Listed in README as a direct dependency; integration not implemented in code |
| Apache Pulsar (`pdd/apple-ddm/statusreport`) | Consumer of Apple DDM status reports; commented out in `application.yaml` |
| AWS S3 / MinIO | Storage backend described in README; no client code or local configuration |
| Jamf Maple (`com.jamfsoftware.mdm:maple`) | MDM integration library; pulled in but no usage present |
| M2M auth (`jamf.platform.m2m`) | Configured in `application-m2m.yaml`; disabled (`authentication-enabled: false`) and commented out in all environments |

---

## Gotchas

**This service is a skeleton.** The only live endpoint is the `/test` placeholder. Do not build integrations against it expecting stable behavior.

**M2M auth is disabled everywhere.** The `m2m` Spring profile is active in production per `values-prod.yaml`, but `authentication-enabled` is set to `false` and the actual secret wiring is commented out. Requests are not authenticated.

**Pulsar is excluded in local dev.** `application-local.yaml` explicitly excludes `PulsarAutoConfiguration`, so the service starts without a Pulsar connection locally.

**S3/MinIO is not wired up locally.** The README describes a local dev workflow using MinIO, but neither `application-local.yaml` nor `docker-compose.yaml` configure it (docker-compose only starts Pulsar). There is no S3 client code in the service.

**Tests are commented out in CI.** The `test` job in the GitHub Actions workflow is disabled.

**Owner:** jabberwocky team — `#help-jabberwocky` on Slack.
