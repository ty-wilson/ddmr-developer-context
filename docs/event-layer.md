# Event Layer

Last reviewed: 2026-07-29. Re-verified against code on that date: the Scoping Engine listener settings and producer message keys, the fully-qualified topic assembly rule, the `sub.json` layout in `event-bus-resources-configuration`, and the alert set emitted by its Helm chart (there are now six alerts, not four; two DLQ alerts were added since the previous review). NOT re-verified: the namespace-ownership and per-topic Producer / Known Consumers / Schema / Compat tables, which date from 2026-05-19 and were derived from `main` at that time. The Operational Notes are incident-derived and were edited only for emphasis.

## Overview

DDmR services communicate asynchronously over Apache Pulsar using the `pdd` tenant. All topic, namespace, tenant, and subscription definitions are version-controlled in the `event-bus-resources-configuration` repo. The Pulsar broker is shared across the platform, so the `pdd/default/` namespace is considered platform-owned and cannot be changed without broader consultation.

**Trap: `event-bus-configuration-topics` is archived.** It was replaced by `event-bus-resources-configuration` in April 2026. The archived repo is still clonable and still greps, so it is easy to read a stale topic definition and conclude a topic or subscription does not exist. Confirm which repo you are in before drawing conclusions.

**Trap: topic definition owner is not the producer.** The `owner` field in `topic.json` identifies who owns the *topic definition*, not who publishes to it. Jamf Pro Server (jamf-messaging) produces `device-group-changed` and `device-management-channel-changed` even though DDmR owns those topic definition entries.

---

## Tenant and Namespace Ownership

| Namespace              | Owner         | Notes                                                      |
|------------------------|---------------|------------------------------------------------------------|
| `pdd/default`          | Platform Core | Shared platform topics; changes require cross-team sign-off |
| `pdd/scoping-engine`   | DDmR          | Owned by Scoping Engine; internal self-communication topics |
| `pdd/blueprints`       | Ocean (DDmR)  | Blueprint deployment workflow topics                       |
| `pdd/apple-ddm`        | Wolfpack      | DDM status reporting from Jamf Pro                         |
| `pdd/mms`              | PowerPC       | Apple media / VPP topics                                   |
| `pdd/compliance-benchmark` | Mars / Red | Compliance rule evaluation topics                         |

---

## Topic Ownership Table

These tables are the expensive-to-rediscover core of this document: rebuilding the consumer columns means grepping roughly 60 repos. Producers and consumers change slowly and are worth trusting for orientation. `Schema` and `Compat` are declared in each `topic.json` and are the cheapest thing here to re-read, so confirm those in the repo before relying on them for a compatibility decision. `(none registered)` means no subscription was declared in the repo at review time, not that nothing consumes the topic.

### pdd/default

| Topic (short name)                        | Producer                        | Known Consumers                                                       | Schema | Compat            |
|-------------------------------------------|---------------------------------|-----------------------------------------------------------------------|--------|-------------------|
| `device-group-changed`                    | Jamf Pro Server (jamf-messaging) | Scoping Engine, compliance-benchmark-report-service                  | Off    | FULL              |
| `device-management-channel-changed`       | Jamf Pro Server (jamf-messaging) | Scoping Engine, blueprint-report-aggregation-service                 | Off    | FULL              |
| `device-scope-membership-changed`         | Scoping Engine                  | blueprint-report-aggregation-service                                  | Off    | FULL              |
| `declaration-assignment-changed`          | Declaration Storage Service     | jamf-school-apns-service, mux (pro-messaging)                         | Off    | FULL              |
| `blueprint-deployment-changed`            | blueprint-deployment-service    | blueprint-management-service                                          | On     | FORWARD_TRANSITIVE |
| `device-devicename-changed`               | Jamf Pro Server (jamf-messaging) | compliance-benchmark-report-service                                  | On     | FORWARD           |
| `device-extensionattribute-state`         | Jamf Pro Server (jamf-messaging) | compliance-benchmark-report-service                                  | On     | FORWARD           |
| `device-identity-certificate-issued`      | Jamf Pro Server (jamf-messaging) | device-identity-mapping-service                                      | On     | FORWARD           |
| `device-management-state`                 | Jamf Pro Server (jamf-messaging) | compliance-benchmark-report-service, device-identity-mapping-service | On     | FORWARD           |
| `device-operatingsystem-changed`          | Jamf Pro Server (jamf-messaging) | compliance-benchmark-report-service                                  | On     | FORWARD           |
| `enrollment-ca-changed`                   | Jamf Pro Server (jamf-messaging) | device-identity-mapping-service                                      | On     | FORWARD           |
| `scim-group-state`                        | scim-directory-service          | mux (pro-messaging)                                                   | Off    | FULL              |
| `scim-user-membership-changed`            | scim-directory-service          | mux (pro-messaging)                                                   | Off    | FULL              |
| `scim-user-state`                         | scim-directory-service          | mux (pro-messaging)                                                   | Off    | FULL              |
| `scim-user-username-changed`              | scim-directory-service          | mux (pro-messaging)                                                   | Off    | FULL              |

### pdd/scoping-engine

| Topic (short name) | Producer       | Known Consumers       | Schema | Compat            |
|--------------------|----------------|-----------------------|--------|-------------------|
| `device-sync`      | Scoping Engine | Scoping Engine (self) | Off    | FORWARD_TRANSITIVE |
| `api-request`      | Scoping Engine | Scoping Engine (self) | Off    | FORWARD_TRANSITIVE |

### pdd/blueprints

| Topic (short name)                        | Producer                     | Known Consumers              | Schema | Compat            |
|-------------------------------------------|------------------------------|------------------------------|--------|-------------------|
| `blueprint-deployment-task`               | blueprint-management-service | blueprint-deployment-service | On     | FORWARD_TRANSITIVE |
| `blueprint-component-translation-changed` | blueprint-management-service, blueprint-component-sw-update-service | blueprint-management-service | On     | FORWARD_TRANSITIVE |

### pdd/apple-ddm

| Topic (short name) | Producer                       | Known Consumers                       | Schema | Compat |
|--------------------|--------------------------------|---------------------------------------|--------|--------|
| `statusreport`     | Jamf Pro (ddm-statusreporting) | blueprint-report-aggregation-service  | Off    | FULL   |

### pdd/mms

| Topic (short name)          | Producer    | Known Consumers | Schema | Compat |
|-----------------------------|-------------|-----------------|--------|--------|
| `apple-media-app-changed`   | mms-pigeon  | (none registered) | On   | (none) |
| `apple-media-asset-assignment` | mms-pigeon | (none registered) | On  | (none) |
| `apple-media-asset-changed` | mms-pigeon  | (none registered) | On   | (none) |
| `apple-media-media-changed` | mms-pigeon  | (none registered) | On   | (none) |
| `apple-media-token-state`   | mms-pigeon  | (none registered) | On   | (none) |
| `apple-media-user-state`    | mms-pigeon  | (none registered) | On   | (none) |

### pdd/compliance-benchmark

| Topic (short name) | Producer                     | Known Consumers                        | Schema | Compat   |
|--------------------|------------------------------|----------------------------------------|--------|----------|
| `verified-rules`   | compliance-benchmark-engine  | compliance-benchmark-report-service    | On     | BACKWARD |

---

## Event Flow (Scoping Engine Focus)

```
Jamf Pro Server (jamf-messaging)
  │
  ├─── device-group-changed (pdd/default) ──────────────────────► Scoping Engine
  │                                                                    │
  └─── device-management-channel-changed (pdd/default) ───────────────┤
                                                                       │
Scoping Engine ◄──── device-sync (pdd/scoping-engine) ◄──────────────┤ (self-loop: deferred processing)
Scoping Engine ◄──── api-request (pdd/scoping-engine) ◄──────────────┘ (self-loop: async API fan-out)
                                                                       │
Scoping Engine ──── device-scope-membership-changed (pdd/default) ────► blueprint-report-aggregation-service

Declaration Storage Service
  └─── declaration-assignment-changed (pdd/default) ────────────► jamf-school-apns-service
                                                                  ► mux (pro-messaging)

blueprint-management-service ─── blueprint-deployment-task (pdd/blueprints) ──► blueprint-deployment-service
blueprint-deployment-service ─── blueprint-deployment-changed (pdd/default)  ──► blueprint-management-service
blueprint-management-service ─── blueprint-component-translation-changed ──────► blueprint-management-service (self)
blueprint-component-sw-update-service ─── blueprint-component-translation-changed ──► blueprint-management-service

Jamf Pro (ddm-statusreporting)
  └─── statusreport (pdd/apple-ddm) ────────────────────────────► blueprint-report-aggregation-service
```

---

## Scoping Engine: Consumer Details

Every `@PulsarListener` in Scoping Engine is declared with the same four settings, and that uniformity is the durable, non-obvious part:

- `subscriptionType = Key_Shared`. Messages with the same key always route to the same consumer instance, preserving per-device ordering.
- `autoStartup = "false"`. Spring does not start the containers at context refresh; `PulsarWatchdog` owns the lifecycle instead.
- `schemaType = SchemaType.JSON`.
- `concurrency` bound to the `messaging.consumer.pulsarListenerConcurrency` property.

Enumerate the current listener set rather than trusting a list here. The declaring class has been renamed at least once, so grep for the annotation, not the filename:

```bash
grep -rn '@PulsarListener' scoping-engine/src/main/kotlin/
```

### Listener Startup: PulsarWatchdog

`PulsarWatchdog` is an `ApplicationListener<ContextRefreshedEvent>`. On startup it iterates over every registered listener container, sets `startupFailurePolicy = StartupFailurePolicy.RETRY`, installs a custom `RetryTemplate` with an exponential backoff policy, and calls `container.start()`. Read the backoff constants from `PulsarWatchdog.kt`; they have been tuned.

**Trap: a pod can be Ready and consuming nothing.** The retry policy is unbounded, so a broker outage at boot does not crash the pod or fail its probes. The container keeps retrying in the background indefinitely while the pod reports healthy. When "no events are being processed" but the deployment looks fine, check consumer connection counts on the subscription before you look for a logic bug.

---

## Scoping Engine: Producer Details

`PulsarService` publishes asynchronously via `sendAsync()`. Two facts matter beyond the method list (which `grep -rn 'fun send' scoping-engine/src/main/kotlin/` regenerates):

- **`device-sync` supports a delivery delay.** It is the only path that passes a `Duration` through to Pulsar's `deliverAfter`, which is what makes the self-feedback retry loop and the delayed-message metric behavior below possible.
- **Device-scoped messages are keyed `tenantId + deviceId`.** The `Key_Shared` per-device ordering guarantee depends entirely on that key. Changing the key strategy silently breaks ordering without any error.

---

## Topic Configuration: MessagingTopicProperties

Scoping Engine uses `@ConfigurationProperties(prefix = "messaging.topics")`. `EventTopicConfig` assembles fully-qualified names as `persistent://<tenant>/<namespace>/<baseName>[-<suffix>]`. Read the tenant and namespace defaults from `MessagingTopicProperties.kt` rather than quoting them.

The `topicNameSuffix` and `messaging.consumer-group-suffix` properties isolate sandbox/staging/perf environments without code changes: the same code produces distinct topic and subscription names per environment purely from config. That mechanism is also the cause of the zombie-subscription failure mode described in Operational Notes.

---

## event-bus-resources-configuration: Where Topic Definitions Live

Topic and subscription definitions live under `configuration/<catalog>/<env>/topics/<tenant>/<namespace>/<topic>/` (catalogs are `platform` and `jpro`; envs are `dev`, `stage`, `sbox`, `prod`). Each topic directory contains `topic.json` (declaring `name`, `settings`, `properties`). Resources are applied by the Pulsar Operator via ArgoCD.

**Since the April 2026 migration to this repo, each subscription is its own `subscriptions/<name>/sub.json` file.** Older examples and any snippet you find from `event-bus-configuration-topics` will show subscriptions nested inside the topic file. Copying the nested form into this repo produces a definition the generator ignores.

To add a topic or subscription: create the directory/files, open a PR. The `run-and-commit.yaml` workflow regenerates `values/service-type-specific/<catalog>/values-<env>.yaml` on a Linux runner and auto-commits back to the branch; `pr-checks.yml` then validates no drift.

**Trap: do not regenerate the values files on macOS.** `scripts/generate_values.py` picks up the local filesystem's directory ordering, so a macOS run produces a reordered diff that CI will reject even when the content is correct. Push the config change and let CI regenerate.

**Trap: one values file serves both clusters.** The same `values-<env>.yaml` is consumed by commercial and HC clusters for an env, and each cluster's Pulsar Operator provisions the identical topic/subscription set on its own broker. There is no per-cluster split, so a subscription added for a deployment that exists on only one side is created as an orphan on the other, where it accumulates backlog with no consumer.

### Subscription alerting

Each `sub.json` may include an `alertConfiguration` block. The Helm chart (`prometheus-rules.yaml`) emits a `PrometheusRule` per topic, with each alert gated on `alertConfiguration.enabled`. The alert names, which are what you actually search for in Grafana and Alertmanager, are `TopicStorageSizeTooHigh` (topic-level), `HighRedeliveryRate`, `SubscriptionUnackedMessages`, and `HighSubscriptionBacklog`, plus two that are emitted **only for DLQ topics** (gated on an `isDLQ` condition in the template): `DLQBacklogNotEmpty` and `DLQMessageRate`. So a non-DLQ topic gets four, a DLQ topic gets six. Regenerate the set with `grep -n 'alert:' helm/event-bus-resources-configuration/templates/prometheus-rules.yaml`, and see that repo's own `docs/ALERTING.md` for thresholds and override examples, which it documents in more detail than this file should duplicate.

Alerts are labeled with the topic's `properties.{team,component,system,domain}`, so `pdd/default/` topics with DDmR ownership route alerts to DDmR even when fired by another team's subscription.

**Trap: the per-alert override key is `disabled`, not `enabled`.** To silence one alert while keeping the others, use `alertOverrides.<AlertName>.disabled: true`. Writing `alertOverrides.<AlertName>.enabled: false` parses fine, changes nothing, and leaves the alert firing.

---

## Key Rules

- **`pdd/default/` is platform-owned.** Topics in this namespace are consumed by multiple teams. Schema or topic changes require broader consultation before merging.
- **`pdd/scoping-engine/` is DDmR-owned.** `device-sync` and `api-request` are internal implementation details of the Scoping Engine and can be changed by the DDmR team without external sign-off.
- **All SE listeners use `Key_Shared`.** Per-device ordering is guaranteed as long as the message key is `tenantId + deviceId`. Do not change the key strategy without understanding the ordering implications.
- **Listeners start disabled.** Never rely on Spring's normal `autoStartup` path for SE listeners; `PulsarWatchdog` owns their lifecycle.
- **Topic definition owner ≠ producer.** The `properties.owner` in `topic.json` reflects who registered the topic definition, not who sends messages to it. Always verify producers by reading actual service code.

---

## Operational Notes

These are non-obvious Pulsar behaviors and broker-metric semantics that have tripped up incident response. This file is the canonical home for the backlog metric semantics below; other docs point here.

**Trap: `pulsar_rate_in` counts produces at publish time, including delayed messages.** A call to `sendDeviceSyncEvent(event, delay)` uses Pulsar's `deliverAfter` and the message is recorded against `pulsar_rate_in` immediately on publish, not when its delivery time arrives. Consequence: the topic produce rate can look much higher than the visible-backlog growth rate because most of those produces are queued for future delivery.

**Backlog metric distinctions.** When triaging a "backlog growing" alert, these measure subtly different things:

| Metric | Scope | Includes delayed? | What it counts |
|---|---|---|---|
| `pulsar_subscription_back_log_no_delayed` | per subscription | no | ready-to-deliver messages |
| `pulsar_subscription_back_log` | per subscription | yes | all unacked messages |
| `pulsar_subscription_delayed` | per subscription | only delayed | messages awaiting their delivery time |
| `pulsar_msg_backlog` | per topic | yes (all subs) | retained storage across every subscription |
| `pulsar_rate_in` | per topic | yes (publish time) | produce rate, delayed counted immediately |
| `pulsar_subscription_msg_rate_out` | per subscription | yes (delivery time) | consume rate, includes redeliveries |

`rate_in > rate_out` is normal whenever scheduled retries are in flight, even when the visible backlog is stable.

**Trap: `reset-cursor --position latest` does NOT clear the delay queue.** Subscription cursor reset moves the visible read cursor forward, but messages already in the broker's delayed-delivery queue continue to mature and are delivered to whichever consumer holds their key when their delivery time arrives. To force-drain a self-feedback retry storm you must either remove the upstream cause (e.g., the DDB state that drives the retry) or use `pulsar-admin topics skip-all-messages` on the affected subscription.

**Trap: the per-env subscription suffix creates multiple parallel subscriptions on one topic, and dead ones never stop retaining.** Scoping Engine subscribes as `scoping-engine-device-sync-<suffix>`, where the suffix comes from `messaging.consumer-group-suffix` (`main`, `sbox`, `perf`, and so on, per env). Each subscription is an independent cursor and every subscription receives every message. A subscription whose consumer is no longer deployed (or was never deployed) becomes a zombie that accumulates retained messages forever, so `pulsar_msg_backlog` at the topic level can grow large even when the live subscription is fully drained. Cull zombies with `pulsar-admin topics unsubscribe <topic> <subscription>` once you are certain nothing consumes them.

---

## Where to find the data (verify rather than trust)

```bash
# Which listeners exist, and with what settings (class has been renamed before)
grep -rn -A7 '@PulsarListener' scoping-engine/src/main/kotlin/

# Producer methods, message keys, and which paths pass a delivery delay
grep -rn 'fun send\|mc.key\|deliverAfter' scoping-engine/src/main/kotlin/com/jamf/platform/scoping/service/PulsarService.kt

# Every topic definition for an env, with its schema/compat settings
find event-bus-resources-configuration/configuration/platform/stage/topics/pdd -name topic.json

# Every declared subscription for a namespace, and its alert config
find event-bus-resources-configuration/configuration/platform/stage/topics/pdd/scoping-engine -name sub.json \
  -exec sh -c 'echo "--- $1"; cat "$1"' _ {} \;

# The current alert set emitted per topic
grep -n 'alert:' event-bus-resources-configuration/helm/event-bus-resources-configuration/templates/prometheus-rules.yaml

# Live subscription state: cursors, backlog, delayed count, connected consumers
pulsar-admin topics stats persistent://pdd/scoping-engine/device-sync
```

Before concluding a topic has no consumer, check the other team's unmerged work as well as the definitions repo:

```bash
git -C <repo> fetch origin --quiet
git -C <repo> for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -20
```
