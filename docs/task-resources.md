---
title: Task resources
description: Create and apply desired Task state, use stored identity safely, and distinguish cancellation from deletion.
---

# Task resources

`solti.task.v1.TaskService` manages named desired-state resources on an agent. A client writes a `TaskManifest`. The agent returns a `Task` that adds server-owned identity, version, generation, timestamps, and observed status.

## Choose an operation

| RPC | Purpose |
|---|---|
| `CreateTask` | Create one named Task. An existing name returns `AlreadyExists`. |
| `ApplyTask` | Declaratively create or update one named Task. |
| `GetTask` | Read one Task by name. |
| `ListTasks` | Read a filtered, paginated collection snapshot. |
| `WatchTasks` | Stream changes in the filtered Task collection. |
| `ListTaskRuns` | Read attempt history for one Task incarnation. |
| `CancelTask` | Stop current reconciliation or request a terminal logical outcome while retaining desired state and history. |
| `DeleteTask` | Request a terminal logical outcome and purge retained state. |
| `StreamTaskLogs` | Live-tail captured output and attempt boundaries. |

The observation RPCs have separate cursor, retention, and loss contracts. Read [Task observation](task-observation.md) before using them.

## Separate write input from stored state

```text
TaskManifest
├── api_version
├── kind
├── user metadata
└── spec
      │
      └── stored by agent ──► Task
                                  ├── api_version and kind
                                  ├── ObjectMeta
                                  ├── spec
                                  └── status
```

`TaskManifest` contains only user-owned metadata and desired `TaskSpec`. It deliberately excludes server identity and status.

`Task.metadata.uid` identifies one resource incarnation. Deleting and recreating the same name produces a different incarnation. `resource_version` is an opaque stored revision. `generation` identifies a revision of desired state.

`Task.status.observed_generation` identifies the latest desired generation processed by the controller. Do not treat a write response as proof that execution has already reached that generation.

## Build one manifest

This Protobuf text-format example shows the complete shape of an extension workload:

```protobuf
manifest {
  api_version: "solti.io/v1"
  kind: "Task"
  metadata {
    name: "daily-report"
    labels { key: "team" value: "billing" }
    annotations { key: "owner" value: "reporting-service" }
  }
  spec {
    slot: "billing-report"
    workload {
      api_version: "example.workloads/v1"
      kind: "Report"
      extension {
        spec { raw: "{\"format\":\"pdf\"}" }
      }
    }
    timeout_ms: 30000
    restart: RESTART_POLICY_ON_FAILURE
    backoff {
      jitter: JITTER_POLICY_FULL
      first_ms: 1000
      max_ms: 30000
      factor: 2.0
    }
    admission: ADMISSION_POLICY_QUEUE
    max_retries: 3
    runner_selector {
      match_labels { key: "runtime" value: "extension" }
    }
  }
}
```

`example.workloads/v1`, `Report`, and the labels are illustrative values. A real workload API version and kind must match a pair advertised by the selected runner. The extension `raw` field contains one UTF-8 JSON value.

Read [Execution policies](execution-policies.md) for timeout and restart rules, and [Workloads and routing](workloads-and-routing.md) for workload and runner selection.

## Choose create or apply

Use `CreateTask` when duplicate creation must fail. Use `ApplyTask` when the caller owns desired state and needs create-or-update behavior.

An `ApplyTask` without preconditions is an unconditional upsert. It can overwrite a newer stored revision. A controller that reads and then writes should normally carry the identity and revision it observed.

## Protect writes from stale clients

`ApplyTask`, `CancelTask`, and `DeleteTask` accept optional `WritePreconditions`:

- `uid` binds the write to one resource incarnation;
- `resource_version` binds the write to one stored revision.

A safe read-modify-write flow is:

```text
GetTask(name)
├── keep task.metadata.uid
└── keep task.metadata.resource_version
          │
          └──► ApplyTask(manifest, preconditions)
                   ├── success ──► returned Task is the new stored snapshot
                   └── Aborted ──► decode conflict details and re-read
```

Preconditions are optional. Omitting them is an explicit choice to accept an unconditional write. See [Task API errors](task-errors.md) for the typed conflict contract.

## Distinguish cancel from delete

`CancelTask` retains the desired Task resource and its run history. It stops current reconciliation or requests a terminal logical outcome for the current runtime. Cancellation does not suppress later reconciliation.

`DeleteTask` requests a terminal logical outcome and purges retained resource state.

In both cases, a force-aborted logical outcome does not prove that task code has physically stopped. Do not use cancellation acknowledgement as a process-isolation or external-side-effect barrier.

## Use the exact schema

- [`TaskService` and request messages](../solti/task/v1/api.proto)
- [`TaskManifest`, `Task`, metadata, and status](../solti/task/v1/types.proto)
