# Event Layer

Last reviewed: 2026-04-07

## Overview

DDmR services communicate asynchronously over Apache Pulsar using the `pdd` tenant. All topic definitions are version-controlled in the `event-bus-configuration-topics` repo. The Pulsar broker is shared across the platform, so the `pdd/default/` namespace is considered platform-owned and cannot be changed without broader consultation.

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

| Topic (short name)                        | Full path                                                  | Producer                        | Known Consumers                                                       | Schema validation    | Compatibility         |
|-------------------------------------------|------------------------------------------------------------|---------------------------------|-----------------------------------------------------------------------|----------------------|-----------------------|
| `device-group-changed`                    | `pdd/default/device-group-changed`                         | Platform Core                   | Scoping Engine, compliance-benchmark-report-service                   | Off                  | FULL                  |
| `device-management-channel-changed`       | `pdd/default/device-management-channel-changed`            | Platform Core                   | Scoping Engine, blueprint-report-aggregation-service                  | Off                  | FULL                  |
| `device-scope-membership-changed`         | `pdd/default/device-scope-membership-changed`              | Scoping Engine                  | blueprint-report-aggregation-service                                  | Off                  | FULL                  |
| `declaration-assignment-changed`          | `pdd/default/declaration-assignment-changed`               | Declaration Storage Service     | jamf-school-apns-service, mux (pro-messaging)                         | Off                  | FULL                  |
| `blueprint-deployment-changed`            | `pdd/default/blueprint-deployment-changed`                 | blueprint-deployment-service    | blueprint-management-service                                          | On                   | FORWARD_TRANSITIVE    |
| `device-sync`                             | `pdd/scoping-engine/device-sync`                           | Scoping Engine                  | Scoping Engine (self)                                                 | Off                  | FORWARD_TRANSITIVE    |
| `api-request`                             | `pdd/scoping-engine/api-request`                           | Scoping Engine                  | Scoping Engine (self)                                                 | Off                  | FORWARD_TRANSITIVE    |
| `blueprint-component-translation-changed` | `pdd/blueprints/blueprint-component-translation-changed`   | blueprint-management-service    | blueprint-management-service                                          | On                   | FORWARD_TRANSITIVE    |
| `blueprint-deployment-task`               | `pdd/blueprints/blueprint-deployment-task`                 | blueprint-management-service    | blueprint-deployment-service                                          | On                   | FORWARD_TRANSITIVE    |
| `statusreport`                            | `pdd/apple-ddm/statusreport`                               | Jamf Pro (ddm-statusreporting)  | blueprint-report-aggregation-service                                  | Off                  | FULL                  |
| `verified-rules`                          | `pdd/compliance-benchmark/verified-rules`                  | compliance-benchmark-engine     | compliance-benchmark-report-service                                   | On                   | BACKWARD              |

---

## Event Flow (Scoping Engine Focus)

```
Platform Core
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

Both Scoping Engine and Declaration Storage Service follow the same `@ConfigurationProperties(prefix = "messaging.topics")` pattern:

```kotlin
// Scoping Engine defaults
tenant = "pdd"
platformNamespace = "default"
scopingEngineNamespace = "scoping-engine"
deviceSyncTopicName = "device-sync"
apiRequestTopicName = "api-request"
scopeChangeTopicName = "device-scope-membership-changed"
channelChangeTopicName = "device-management-channel-changed"
groupChangeTopicName = "device-group-changed"
subscriptionBase = "scoping-engine"
topicNameSuffix = ""          // optional per-environment suffix
```

`EventTopicConfig` assembles `TopicInfo` objects from these properties. The fully-qualified topic name is always:

```
persistent://<tenant>/<namespace>/<baseShortName>[-<suffix>]
```

Subscription names follow: `<subscriptionBase>-<shortName>[-<consumerGroupSuffix>]`

The `topicNameSuffix` and `messaging.consumer-group-suffix` properties allow sandboxes and staging environments to use isolated topics and subscriptions without code changes (e.g., `device-sync-sbox`, `scoping-engine-device-sync-sbox`).

---

## event-bus-configuration-topics: Where Topic Definitions Live

Repository: `event-bus-configuration-topics`

Topic definitions live under:
```
envs/
  prod/
    pdd/
      <namespace>/
        <topic-name>/
          topic.json     # topic settings, properties, subscriptions per region
          readme.md      # description and owner contact
```

Each `topic.json` declares:
- `name` — fully-qualified topic path (`pdd/<namespace>/<topic>`)
- `settings.schemaEnforceValidation` — whether the broker rejects schema-incompatible messages
- `settings.schemaCompatibilityStrategy` — e.g., `FULL`, `FORWARD_TRANSITIVE`, `BACKWARD`
- `properties` — owner, domain, system, component metadata
- `subscriptions` — per-region subscription registrations (each with owner metadata and optional alert configuration)

### Adding a New Topic

1. Create a directory under `envs/prod/pdd/<namespace>/<topic-name>/`.
2. Add `topic.json` with name, settings, properties, and subscriptions for each region (`ap-northeast-1`, `eu-central-1`, `us-east-2`).
3. Add `readme.md` describing purpose and owner Slack channel.
4. Open a PR — the CI pipeline applies the config to Pulsar via automation.
5. If the topic belongs to `pdd/default/`, coordinate with all teams that consume the namespace before merging.

---

## Key Rules

- **`pdd/default/` is platform-owned.** Topics in this namespace (`device-group-changed`, `device-management-channel-changed`, `device-scope-membership-changed`, `declaration-assignment-changed`, etc.) are consumed by multiple teams. Schema or topic changes require broader consultation before merging.
- **`pdd/scoping-engine/` is DDmR-owned.** `device-sync` and `api-request` are internal implementation details of the Scoping Engine and can be changed by the DDmR team without external sign-off.
- **All SE listeners use `Key_Shared`.** Per-device ordering is guaranteed as long as the message key is `tenantId + deviceId`. Do not change the key strategy without understanding the ordering implications.
- **Listeners start disabled.** Never rely on Spring's normal `autoStartup` path for SE listeners — `PulsarWatchdog` owns their lifecycle.
