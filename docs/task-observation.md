---
title: Task observation
description: Consume Task snapshots, collection watches, run history, conditions, and live output without mixing their cursors or guarantees.
---

# Task observation

Task state, attempt history, and captured output use separate APIs. Each answers a different question and carries its own identity, cursor, retention, and loss semantics.

## Choose the observation source

| Need | RPC | Identity or cursor | Completeness |
|---|---|---|---|
| Current Task resources | `ListTasks` | `continue`, collection `resource_version` | Paginated collection snapshot |
| Changes to Task resources | `WatchTasks` | watch `resource_version` | Stream of matching collection changes |
| Attempt history | `ListTaskRuns` | Task name, `task_uid`, run `continue` | Paginated retained history |
| Current live output | `StreamTaskLogs` | Task name and exact `task_uid` | Lossy live tail |

Do not move a token or resource version from one API into another unless the target field explicitly accepts it.

## Page Task snapshots

`ListTasks` applies slot, phase, and label filters together. Multiple phases are alternatives within the phase filter.

A non-empty `continue` token resumes the previous collection snapshot. Repeat the original filters unchanged on every page. The response returns:

- one page of `Task` resources;
- the snapshot's opaque `resource_version`;
- an opaque `continue` token for the next page;
- an optional `remaining_item_count`.

For `ListTasks` and `ListTaskRuns`, `limit = 0` selects the default page size of 100. The maximum accepted value is 1000.

Do not parse, compare numerically, or construct continuation tokens or resource versions.

## Watch collection changes

`WatchTasks` applies the same filter categories as `ListTasks`. Its event types are:

| Event | Meaning |
|---|---|
| `ADDED` | A Task entered the watched collection. |
| `MODIFIED` | A watched Task changed. |
| `DELETED` | A Task left the watched collection. |

Each event carries a complete Task snapshot for that change.

An absent `resource_version` or the value `"0"` requests current matching objects followed by later changes. A non-zero value is opaque. A terminal `OutOfRange` means the requested position is no longer retained; start a fresh observation cycle instead of replaying the expired value.

Pagination continuation and watch position are different mechanisms.

## Read desired and observed generations

The stored resource separates desired state from controller observation:

```text
Task.metadata.generation ── desired state revision ──► Task.status.observed_generation
```

When both values match, status describes the current desired generation. When they differ, status still describes an older observed generation.

`TaskStatus.attempt` is the current or latest attempt within `observed_generation`. It is not a global attempt counter across every generation.

## Handle conditions as typed status

Each `TaskCondition` carries:

- `type`, a stable condition identity;
- `status`, with `TRUE`, `FALSE`, or `UNKNOWN`;
- `observed_generation`, which binds the condition to desired state;
- `last_transition_time` in Unix milliseconds;
- `reason`, a stable machine-readable category;
- `message`, a human-readable diagnostic.

Branch on condition type, status, observed generation, and documented reason values. Do not parse `message` for application behavior.

## Bind run history to one incarnation

`ListTaskRuns` returns attempts oldest first. Each `TaskRunInfo` contains desired-state `generation`, attempt number, phase, timestamps, optional error and exit code, and the workload GVK captured for that run.

The page-level `task_uid` identifies the Task incarnation whose history is being read. It stays fixed across continuation pages. Deleting and recreating a Task under the same name produces a different UID.

Run-history `continue` and `resource_version` values belong to the run snapshot. They are not Task collection cursors.

## Treat live output as lossy

`StreamTaskLogs` requires both the Task name and exact Task UID. The UID prevents a stream from silently following a different incarnation created under the same name.

The stream emits four event shapes:

| Event | Contract |
|---|---|
| `RunStarted` | Opens one generation and attempt and records its start time. |
| `OutputChunk` | Carries one retained stdout or stderr line without its delimiter. |
| `RunFinished` | Closes one generation and attempt and may include an exit code. |
| `Lagged` | Reports the number of events and retained line bytes missed by this subscriber. |

`OutputChunk.seq` is a per-stream sequence. `line` contains exact retained bytes. `truncated = true` means bytes were omitted from the end.

A robust consumer tracks output by Task UID, generation, attempt, stream kind, and sequence:

```text
RunStarted(generation, attempt)
├──► OutputChunk(stdout, seq ...)
├──► OutputChunk(stderr, seq ...)
├──► Lagged(skipped, skipped_bytes) ──► record an observation gap
└──► RunFinished(generation, attempt)
```

The stream is a live tail, not an output archive. `Lagged` is a gap inside a live stream. A terminal gRPC status ends the stream itself.

## Interpret Task phases

`TaskPhase` records logical lifecycle state:

| Phase | Meaning |
|---|---|
| `UNSPECIFIED` | Zero-value sentinel. It is not a lifecycle state. |
| `PENDING` | Desired generation is stored and awaits runtime observation. |
| `RUNNING` | An attempt has started. |
| `SUCCEEDED` | A successful outcome was recorded. |
| `FAILED` | An attempt, fatal, runtime, or non-cancel-like admission failure was recorded. It does not mean retries are exhausted. |
| `TIMEOUT` | An attempt exceeded its configured deadline. Retry policy can still start another attempt. |
| `CANCELED` | Logical cancellation or a cancel-like admission result was recorded. Physical exit is not implied. |
| `EXHAUSTED` | An eligible failure stopped because restart policy allowed no further attempt or the retry budget was reached. |

`TaskRunInfo.phase` describes one attempt. `TaskStatus.phase` is the latest recorded logical state of the Task. A later retry or reconciliation can start another attempt after a terminal-looking phase.

Use recorded resources and run history for logical outcomes. Do not infer physical process state from a phase alone.

## Use the exact schema

- [`ListTasks`, `WatchTasks`, `ListTaskRuns`, and `StreamTaskLogs`](../solti/task/v1/api.proto)
- [`TaskStatus`, conditions, phases, and run history`](../solti/task/v1/types.proto)
