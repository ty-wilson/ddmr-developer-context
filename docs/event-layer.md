# Event Layer

Last reviewed: 2026-04-22

## Overview

DDmR services communicate asynchronously over Apache Pulsar using the `pdd` tenant. All topic, namespace, tenant, and subscription definitions are version-controlled in the `event-bus-resources-configuration` repo (which replaced the now-archived `event-bus-configuration-topics` in April 2026). The Pulsar broker is shared across the platform, so the `pdd/default/` namespace is considered platform-owned and cannot be changed without broader consultation.

> **Important:** The `owner` field in `topic.json` identifies who owns the *topic definition*, not who produces messages to it. Jamf Pro Server (jamf-messaging) produces `device-group-changed` and `device-management-channel-changed` even though DDmR owns the topic definition entries.

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
| `blueprint-component-translation-changed` | blueprint-management-service | blueprint-management-service | On     | FORWARD_TRANSITIVE |

### pdd/apple-ddm

| Topic (short name) | Producer                       | Known Consumers                       | Schema | Compat |
|--------------------|--------------------------------|---------------------------------------|--------|--------|
| `statusreport`     | Jamf Pro (ddm-statusreporting) | blueprint-report-aggregation-service  | Off    | FULL   |

### pdd/mms

| Topic (short name)          | Producer    | Known Consumers | Schema | Compat |
|-----------------------------|-------------|-----------------|--------|--------|
| `apple-media-app-changed`   | mms-pigeon  | (none registered) | On   | —      |
| `apple-media-asset-assignment` | mms-pigeon | (none registered) | On  | —      |
| `apple-media-asset-changed` | mms-pigeon  | (none registered) | On   | —      |
| `apple-media-media-changed` | mms-pigeon  | (none registered) | On   | —      |
| `apple-media-token-state`   | mms-pigeon  | (none registered) | On   | —      |
| `apple-media-user-state`    | mms-pigeon  | (none registered) | On   | —      |

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

Jamf Pro (ddm-statusreporting)
  └─── statusreport (pdd/apple-ddm) ────────────────────────────► blueprint-report-aggregation-service
```

---

## Scoping Engine: Consumer Details

All four listeners in `PulsarRouter` use identical settings:

| Listener method             | Topic bean         | Event type                  | Handler                        |
|-----------------------------|--------------------|-----------------------------|--------------------------------|
| `handleDeviceGroupChanged`  | `groupChangeTopic` | `DeviceGroupChangedEvent`   | `DeviceGroupChangedHandler`    |
| `handleDeviceSync`          | `deviceSyncTopic`  | `DeviceSyncEvent`           | `DeviceSyncHandler`            |
| `handleScopingEngineApiRequest` | `apiRequestTopic` | `ApiRequestEvent`        | `ApiRequestEventHandler`       |
| `handleDeviceChannelChanged`| `channelChangeTopic` | `DeviceChannelChangedEvent` | `DeviceChannelChangedHandler` |

All listeners are declared with:
- `subscriptionType = Key_Shared` — messages with the same key are always routed to the same consumer instance, preserving per-device ordering
- `autoStartup = "false"` — Spring does not start the containers at context refresh; `PulsarWatchdog` takes over
- `schemaType = SchemaType.JSON`
- Concurrency driven by `messaging.consumer.pulsarListenerConcurrency`

### Listener Startup: PulsarWatchdog

`PulsarWatchdog` is an `ApplicationListener<ContextRefreshedEvent>`. On startup it iterates over every registered listener container and:

1. Sets `startupFailurePolicy = StartupFailurePolicy.RETRY`
2. Installs a custom `RetryTemplate`: infinite retries, exponential backoff starting at 5 s, multiplier 1.5, capped at 5 min (with jitter)
3. Calls `container.start()`

This means a Pulsar connection outage at boot time will not crash the pod — it will keep retrying in the background until the broker is reachable.

---

## Scoping Engine: Producer Details

`PulsarService` sends three event types, all asynchronously via `sendAsync()`:

| Method                              | Topic                                    | Message key                    | Notes                                   |
|-------------------------------------|------------------------------------------|--------------------------------|-----------------------------------------|
| `sendDeviceSyncEvent`               | `pdd/scoping-engine/device-sync`         | `tenantId + deviceId`          | Optional delivery delay supported       |
| `sendApiRequestEvent`               | `pdd/scoping-engine/api-request`         | `tenantId`                     |                                         |
| `sendDeviceScopeMembershipChangedEvent` | `pdd/default/device-scope-membership-changed` | `tenantId + deviceId`  |                                         |

---

## Topic Configuration: MessagingTopicProperties

Scoping Engine uses `@ConfigurationProperties(prefix = "messaging.topics")`. Key defaults: `tenant=pdd`, `platformNamespace=default`, `scopingEngineNamespace=scoping-engine`. `EventTopicConfig` assembles fully-qualified names as `persistent://pdd/<namespace>/<baseName>[-<suffix>]`. The `topicNameSuffix` and `messaging.consumer-group-suffix` properties isolate sandbox/staging environments without code changes.

---

## event-bus-resources-configuration: Where Topic Definitions Live

Topic and subscription definitions live under `configuration/<catalog>/<env>/topics/<tenant>/<namespace>/<topic>/` (catalogs are `platform` and `jpro`; envs are `dev`, `stage`, `sbox`, `prod`). Each topic directory contains `topic.json` (declaring `name`, `settings`, `properties`) and a `subscriptions/<sub-name>/sub.json` file per subscription — subscriptions are no longer nested in the topic file. Resources are applied by the Pulsar Operator via ArgoCD.

To add a topic or subscription: create the directory/files, open a PR. The `run-and-commit.yaml` workflow regenerates `values/service-type-specific/<catalog>/values-<env>.yaml` on Linux/py3.12 and auto-commits back to the branch; `pr-checks.yml` then validates no drift. Note: local regeneration on macOS can produce filesystem-ordering diffs that CI won't accept — let CI regenerate.

### Subscription alerting

Each `sub.json` may include an `alertConfiguration` block. The helm chart (`helm/event-bus-resources-configuration/templates/prometheus-rules.yaml`) emits a `PrometheusRule` per topic with up to four alerts gated on `alertConfiguration.enabled`: `TopicStorageSizeTooHigh` (topic-level), `HighRedeliveryRate`, `SubscriptionUnackedMessages`, `HighSubscriptionBacklog`. To disable an individual alert while keeping others on, use `alertOverrides.<AlertName>.disabled: true` — the override key is `disabled`, not `enabled: false`. Alerts are labeled with the topic's `properties.{team,component,system,domain}`, so `pdd/default/` topics with DDmR ownership route alerts to DDmR even when fired by another team's subscription.

---

## Key Rules

- **`pdd/default/` is platform-owned.** Topics in this namespace are consumed by multiple teams. Schema or topic changes require broader consultation before merging.
- **`pdd/scoping-engine/` is DDmR-owned.** `device-sync` and `api-request` are internal implementation details of the Scoping Engine and can be changed by the DDmR team without external sign-off.
- **All SE listeners use `Key_Shared`.** Per-device ordering is guaranteed as long as the message key is `tenantId + deviceId`. Do not change the key strategy without understanding the ordering implications.
- **Listeners start disabled.** Never rely on Spring's normal `autoStartup` path for SE listeners — `PulsarWatchdog` owns their lifecycle.
- **Topic definition owner ≠ producer.** The `properties.owner` in `topic.json` reflects who registered the topic definition, not who sends messages to it. Always verify producers by reading actual service code.
