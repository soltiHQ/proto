---
title: Task resources
description: Manage desired Task state safely and distinguish cancellation from deletion.
---

# Task resources

`TaskService` manages named desired-state resources on an agent. A `TaskManifest` is write input; a returned `Task` adds server-owned identity, version, generation, timestamps, and observed status.

## Choose an operation

| RPC | Purpose |
|---|---|
| `CreateTask` | Create one named Task from a manifest. |
| `ApplyTask` | Create or update desired state declaratively. |
| `GetTask` | Read one Task by name. |
| `ListTasks` | Read a filtered, paginated collection snapshot. |
| `WatchTasks` | Stream collection changes. |
| `ListTaskRuns` | Read attempt history for one Task incarnation. |
| `CancelTask` | Stop current reconciliation or request a terminal logical outcome while retaining the Task and its history. |
| `DeleteTask` | Request a terminal logical outcome and purge retained state. |
| `StreamTaskLogs` | Live-tail captured output and attempt boundaries. |

The observation RPCs have additional contracts in [Task observation](task-observation.md).

## Separate desired and observed state

`TaskManifest` contains user-owned metadata and `TaskSpec`. The specification selects a slot, workload, timeout, restart behavior, backoff, admission behavior, retry budget, and optional runner selector.

`Task` contains the stored specification plus `ObjectMeta` and `TaskStatus`.
`metadata.generation` identifies a desired-state generation. `status.observed_generation` identifies the latest generation processed by the controller.

## Protect writes from stale clients

`ApplyTask`, `CancelTask`, and `DeleteTask` accept optional `WritePreconditions`:

- `uid` binds the write to one resource incarnation, even if the same name is deleted and recreated.
- `resource_version` binds the write to one stored revision.

With no preconditions, `ApplyTask` is an unconditional upsert and cancel or delete has no identity or version check.
A failed write condition is represented by an `Aborted` gRPC status carrying `WriteConflictDetails` and one or more typed causes.
See [Task API errors](task-errors.md) for the complete status-code and conflict-detail contract.

## Distinguish cancel from delete

`CancelTask` retains desired Task state and run history. It stops current reconciliation or requests a terminal logical outcome for the current runtime, but it does not suppress later reconciliation.

`DeleteTask` purges retained resource state and history after requesting a terminal logical outcome.

In both cases, a force-aborted outcome does not prove that task code has physically stopped.
