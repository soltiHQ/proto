---
title: Production boundaries
description: Understand what the wire schema guarantees and what each integration or deployment must decide before production.
---

# Production boundaries

Contracts makes message and service boundaries explicit. It does not make every runtime, storage, security, or scheduling decision for applications that use those messages.

## Separate schema guarantees from deployment policy

| Defined by Contracts | Owned elsewhere |
|---|---|
| RPC names, directions, request and response shapes | Endpoint discovery mechanism and network topology |
| Field numbers, types, optional presence, enums, and `oneof` identities | Authentication, authorization, TLS, and credential transport |
| Task desired and observed resource shapes | Public Task storage durability and recovery policy |
| Runner capability and selector messages | Runner implementation and tie-breaking among matches |
| Opaque cursor and resource-version fields | Retention windows and storage capacity |
| Logical Task phases and log-gap events | Process isolation and external-side-effect fencing |
| Raft command and snapshot serialization | Raft transport, encryption, backup, and restore procedures |

Do not claim a deployment property from the presence of a schema field alone.

## Treat cancellation as logical state

`CancelTask` retains desired state and history. `DeleteTask` purges retained state. Both can request a terminal logical outcome while force-aborted task code remains physically active.

If correctness depends on a process having stopped or an external lease having been released, use an implementation-owned confirmation or fencing mechanism. The cancellation response is not that barrier.

## Treat live output as best-effort observation

`StreamTaskLogs` is a live tail. `Lagged` explicitly reports dropped events and retained bytes. The schema does not define an output archive or a replay cursor.

Use Task resources and `ListTaskRuns` for retained logical status. Use a deployment-owned logging system when complete or durable output is required.

## Preserve opaque identities

Do not parse or synthesize:

- Task `uid` values;
- Task or collection `resource_version` values;
- list continuation tokens;
- watch positions.

Store and return them through the API that issued them. Keep Task name and UID together whenever deletion and recreation can occur.

## Respect time units

The schema uses several explicit time units:

| Area | Unit |
|---|---|
| Discovery `ts` | Unix seconds |
| Discovery uptime and heartbeat interval | Seconds |
| Public Task timestamps, timeouts, intervals, and backoff | Milliseconds |
| Internal Raft model timestamps and heartbeat interval | Nanoseconds unless the field name says `ms` |

Do not infer units from the language's generated integer type. Use the field comment and suffix.

## Protect internal state

`solti.raft.v1` contains secret-bearing fields and complete internal model snapshots. It is not a public API surface.

Do not expose Raft payloads through agent endpoints, public logs, or unredacted diagnostics. The schema does not provide encryption or redaction.

## Check before deployment

- Pin one exact Contracts release and generate all imports from it.
- Pin generator and runtime versions independently.
- Decide which side owns each public service and endpoint lifecycle.
- Configure authentication, authorization, TLS, and network reachability.
- Use write preconditions where stale writes would be unsafe.
- Define retry and deadline policy for every unary and streaming RPC.
- Treat Task cancellation, log retention, and capability freshness according to their documented limits.
- Protect Raft logs and snapshots as internal secret-bearing state.
- Test schema generation, consumer compilation, and client/server interoperability together.

Read [Common mistakes](common-mistakes.md) for failure patterns found at integration boundaries.
