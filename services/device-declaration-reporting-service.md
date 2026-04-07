# Device Declaration Reporting Service

Last reviewed: 2026-04-07

## Summary

Device Declaration Reporting Service (DRS) is a Spring Boot 3.5 / Java 21 service owned by the jabberwocky team. Its intended role is device-oriented reporting of Declaration Management declarations — presenting per-device declaration status in a way that is useful to callers who need to know what declarations are currently applied to a given device and whether those declarations are in a desired state. The service is deployed to production but is **early in development**: as of this writing, the only implemented endpoint is a placeholder `/test` route, and major integration points (M2M auth, Pulsar consumption of status reports, S3/MinIO storage, Declaration Storage Service calls) are all stubbed out or commented in config. Treat this service's API surface as unstable — no stable endpoints exist for external callers yet.

---

## API Endpoints

The service runs on port 8080 (application) and 8079 (management/actuator).

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/test` | Placeholder endpoint. Returns the string `"Test successful"`. Not intended for production use. |
| `GET` | `/actuator/health/liveness` | Kubernetes liveness probe. Port 8079. |
| `GET` | `/actuator/health/readiness` | Kubernetes readiness probe. Port 8079. |
| `GET` | `/actuator/prometheus` | Prometheus metrics scrape endpoint. Port 8079. |

No tenant-scoped or device-scoped reporting endpoints are implemented yet.

---

## Data Model

No persistent data store is owned by this service yet. The README notes S3 (via AWS SDK, with MinIO as a local stand-in) as the intended storage backend for declaration status report data — the local dev profile configures MinIO credentials and notes the bucket is created automatically on startup. No schema or key structure is defined in code at this time.

**Planned upstream data source:** Apple MDM status reports published to the `persistent://pdd/apple-ddm/statusreport` Pulsar topic. The Pulsar subscription (`device-declaration-reporting-service`) is defined in commented-out config but not yet wired up.

---

## Dependencies

| Dependency | How it is used |
|---|---|
| Declaration Storage Service (DSS) | Listed in README as a direct dependency; integration not yet implemented in code |
| MDM Server (Jamf Pro) | Listed in README as a direct dependency; integration not yet implemented in code |
| Apache Pulsar (`pdd/apple-ddm/statusreport`) | Planned consumer of Apple DDM status reports; commented out in `application.yaml` |
| AWS S3 / MinIO | Planned storage backend; configured in `application-local.yaml` for local dev, not yet active in production |
| Jamf Maple (`com.jamfsoftware.mdm:maple:4.3.1`) | MDM integration library; pulled in but no usage present yet |
| M2M auth (`jamf.platform.m2m`) | Configured in `application-m2m.yaml`; disabled (`authentication-enabled: false`) and commented out in all environments |

---

## Gotchas

**This service is a skeleton.** The only live endpoint is the `/test` placeholder. Do not build integrations against it expecting stable behavior — the API, data model, and event contracts are all TBD.

**M2M auth is disabled everywhere.** The `m2m` Spring profile is active in production per `values-prod.yaml`, but `authentication-enabled` is set to `false` and the actual secret wiring is commented out. Requests are not authenticated.

**Pulsar is excluded in local dev.** `application-local.yaml` explicitly excludes `PulsarAutoConfiguration`, so the service starts without a Pulsar connection locally.

**S3 bucket is auto-created locally.** When running with the `local` profile, the service expects a running MinIO instance (`docker compose up -d`) and creates the target bucket on startup. There is no equivalent in deployed environments yet.

**Tests are commented out in CI.** The `test` job in the GitHub Actions workflow is disabled. The test infrastructure exists (JUnit, Spring Boot Test) but coverage is minimal.

**Owner:** jabberwocky team — `#help-jabberwocky` on Slack.
