---
title: Internal Raft state
description: Encode atomic control-plane FSM commands and guarded full-state snapshots without treating them as a public API.
---

# Internal Raft state

`solti.raft.v1` is the internal serialization contract for control-plane Raft state. It declares log-entry and snapshot messages. It does not declare an RPC service and is not part of the public agent API.

Only control-plane code that applies Raft commands or persists and restores FSM snapshots should generate this package.

## Separate incremental commands from full snapshots

```text
state changes ──► Command { repeated Op } ─────────► one atomic Raft log entry

all FSM objects ──► Snapshot { header + collections } ──► compaction or restore
```

Every `Op` in one `Command` applies in a single all-or-nothing transaction on each replica's FSM. Preserve operation order when encoding and applying the repeated field.

`Snapshot` carries the complete set of object collections declared by the schema. `SnapshotHeader` guards the snapshot format before restoration.

## Choose one operation variant

`Op` is a `oneof`. Exactly one mutation variant is selected for each entry in `Command.ops`.

| State area | Operation families |
|---|---|
| Agents | upsert or delete an agent; upsert or delete its stored credential |
| Users and access | upsert or delete users, roles, credentials, and verifiers |
| Sessions | create or delete sessions, delete by user, rotate refresh hash, or revoke |
| Desired delivery | upsert or delete Specs; upsert or delete Rollouts; delete Rollouts by Spec |
| Events | notify a transient event, append an event record, or delete matching issue events |

Upsert variants carry full message snapshots. Delete variants carry the target identity or a typed parameter message.

## Understand the persisted models

| Message | Stored purpose |
|---|---|
| `AgentMsg` | Agent endpoint, heartbeat state, metadata, labels, and runner capabilities. |
| `AgentCredentialMsg` | Per-agent bearer secret learned during discovery. |
| `UserMsg` | External subject, profile, direct permissions, and assigned roles. |
| `RoleMsg` | Named permission set. |
| `CredentialMsg` | Authentication mechanism and mechanism-specific secrets. |
| `VerifierMsg` | Verification material associated with a credential. |
| `SessionMsg` | Session ownership, refresh hash, expiration, revocation, and timestamps. |
| `SpecMsg` | Desired task specification, target selection, runner labels, and revision counters. |
| `RolloutMsg` | One Spec's synchronization state on one target agent. |
| `ExecutionRecordMsg` | Retained terminal run observation and its capture provenance. |

These messages mirror internal control-plane models. They are not replacements for the public `TaskManifest`, `Task`, or discovery messages.

## Handle special mutation payloads

`SessionRotateRefreshMsg` carries a new refresh-token hash, expiration time and optional producer-captured `updated_at_ns`. It never carries the raw refresh token. The control-plane implementation preserves the stored modification time when replaying a legacy command without that field. A present zero represents the Unix epoch. `SessionMsg` has additive timestamp presence flags for modification, revocation, expiration and creation. They preserve valid epoch instants in snapshots without changing existing scalar fields or their nonzero legacy meaning. A missing zero retains the historical unset interpretation.

`SessionRevokeMsg` identifies the session and its revocation time. The implementation uses that explicit operation time for modification metadata; replicas must not derive persisted metadata from their local clocks.

`EventRecordMsg` carries an activity-event kind and payload. `EventDeleteIssuesMsg` removes issue-ring entries matching both kind and target ID. `event_notify` carries a transient UI refresh event name.

The schema defines the serialized payloads. Ring capacity, event delivery, and subscriber behavior belong to the control-plane implementation.

## Restore only a compatible snapshot

`SnapshotHeader` currently declares:

- `magic = "SOLTI-SNAP"`;
- writer `version = 3`; the current implementation also reads version 2.

Validate both fields before applying snapshot contents. This format version is independent of the repository release and the `solti.raft.v1` package suffix.

The declared snapshot collections contain agents, agent credentials, users, roles, credentials, verifiers, sessions, Specs, Rollouts, and retained execution observations. The exact field numbers and types in [`raft.proto`](../solti/raft/v1/raft.proto) are the persistence contract.

## Protect secret-bearing state

The internal messages include raw agent bearer tokens, credential secrets, verifier material, and refresh-token hashes. In particular, `AgentCredentialMsg.token` must never be exposed or logged.

Treat Raft logs, snapshots, diagnostics, backups, and generated debug output as potentially secret-bearing. Encryption, access control, redaction, and key lifecycle are deployment and implementation responsibilities.

## Generate one coherent internal package set

`solti.raft.v1` imports `solti.agent.v1.AgentCapabilities` and `google.protobuf.Struct`. Generate those dependencies from the same Contracts checkout and the selected Protobuf runtime.

Do not mix Raft messages from one repository release with public agent types from another. Follow [Generate bindings](generate-bindings.md) and [Versioning](versioning.md).
