---
title: Contract model
description: See the public request directions, internal Raft data flow, package imports, and ownership boundaries.
---

# Contract model

Contracts defines two public request paths and one internal persistence path. The direction matters because the agent serves Task management but calls the control plane for discovery.

## Follow the public traffic

```text
remote client or control plane ── TaskService ──► agent
                                                   │
                                                   └── DiscoverService.Sync ──► control plane
                                                          └── request includes AgentCapabilities
```

`TaskService` is exposed by the agent. Its messages carry desired Task state, stored identity, observed status, attempt history, and live output events.

`DiscoverService` is exposed by the control plane. An agent calls `Sync` to publish its endpoint, heartbeat state, and current runner capabilities.

The public schema does not require both sides to use the same implementation language. A Rust client and a Go server interoperate when both generate compatible bindings from the same schema snapshot.

## Follow the internal Raft path

```text
control-plane mutation ──► Command ──► Raft log ──► replica FSMs

full FSM state ──────────► Snapshot ──────────────► compaction or restore
```

`solti.raft.v1` declares messages, not an RPC service. `Command` carries one atomic group of state mutations. `Snapshot` carries the full declared FSM state together with a format guard.

This package is internal to the control plane. It is a persistence and replication boundary, not a public agent contract.

## Understand package composition

```text
solti.task.v1

solti.discover.v1 ──► solti.agent.v1

solti.raft.v1 ──┬──► solti.agent.v1
                └──► google.protobuf.Struct
```

Generate every imported package from the same repository release. A discovery consumer needs `solti.discover.v1` and `solti.agent.v1`. An internal Raft consumer also needs `solti.agent.v1` plus its Protobuf well-known-type dependency.

## Keep version markers separate

Several values contain `v1` or `1`, but they do not select the same thing:

- a repository release selects one exact schema snapshot;
- a Protobuf package suffix such as `solti.task.v1` identifies a wire API generation;
- `TaskManifest.api_version` identifies the Task resource API;
- `SyncRequest.api_version` identifies the API served by an agent endpoint;
- `SnapshotHeader.version` identifies the internal Raft snapshot format.

Read [Versioning](versioning.md) before upgrading any consumer.

## Know who owns behavior outside the wire

Contracts owns message shapes, service signatures, field identities, and documented wire semantics. Generated-code layout belongs to the selected language tools.

The serving application and deployment own storage, endpoint authentication, authorization, TLS policy, network reachability, runner implementation, and operational limits unless a schema field states otherwise.
See [Production boundaries](production-boundaries.md) for the complete integration checklist.
