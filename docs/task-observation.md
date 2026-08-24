---
title: Task observation
description: Consume collection snapshots, watches, run history, and live output without treating them as the same stream.
---

# Task observation

Task state, attempt history, and captured output use separate APIs. Their cursors and versions are opaque values with API-specific meaning.

## Page Task snapshots

`ListTasks` applies slot, phase, and label filters together. Multiple phases are alternatives within the phase filter.

A non-empty `continue` token resumes the previous collection snapshot. Repeat the original filters unchanged when requesting the next page.
The response returns the snapshot's `resource_version`, the next token, and an optional remaining-item count.

For both `ListTasks` and `ListTaskRuns`, `limit = 0` selects the default page size of 100. The maximum accepted value is 1000; a larger value returns `InvalidArgument`.

Do not parse or construct continuation tokens or resource versions.

## Watch collection changes

`WatchTasks` applies the same filter categories and emits `ADDED`, `MODIFIED`, and `DELETED` events.
An absent resource version or the value `"0"` requests current matching objects followed by later changes.

Each event contains a complete Task snapshot for that change. The watch stream is distinct from paginated list continuation.

## Bind run history to an incarnation

`ListTaskRuns` returns attempts oldest first. Each run is identified within the Task by desired-state `generation` and `attempt`.

The page-level `task_uid` identifies the Task incarnation whose history is being read. It stays fixed across continuation pages.
Deleting and recreating a Task under the same name produces a different UID, which lets a client reject history from the wrong incarnation.

Run-history continuation and `resource_version` belong to the run snapshot, not the Task collection snapshot.

## Treat live output as a lossy observation path

`StreamTaskLogs` requires the exact Task UID obtained from a Task resource. A UID mismatch returns `NotFound` instead of subscribing to a different incarnation under the same name.

The request UID identifies every event returned by the stream. A successfully opened stream stays pinned to that Task UID and the generation visible at subscription time. Deleting and recreating the same Task name never retargets the existing stream.

`StreamTaskLogs` emits four event kinds:

- `RunStarted` opens one generation and attempt.
- `OutputChunk` carries exact retained line bytes, the source stream, a per-stream sequence, and the time the agent read the line.
- `RunFinished` closes the current attempt and may carry an exit code.
- `Lagged` reports events and retained bytes skipped because the subscriber fell behind.

The stream is a live tail, not a complete attempt archive. A consumer must treat `Lagged` as an explicit gap. Identify an attempt by Task UID, generation, and attempt number.

## Interpret Task phases

`TaskPhase` is a logical state recorded by the agent:

| Phase | Meaning |
|---|---|
| `UNSPECIFIED` | Zero-value sentinel. The agent rejects it in phase filters and does not return it as a lifecycle state. |
| `PENDING` | The desired generation is stored and awaits runtime observation. |
| `RUNNING` | An attempt has started. |
| `SUCCEEDED` | A successful attempt outcome was recorded. |
| `FAILED` | An attempt failed, or execution ended through a fatal or runtime failure or a non-cancel-like admission rejection. It does not mean retry policy was exhausted. |
| `TIMEOUT` | The agent observed that an attempt exceeded its configured deadline. Retry policy may still start another attempt. |
| `CANCELED` | Logical cancellation, force-abort, or a cancel-like admission rejection was recorded. Physical exit is not implied. |
| `EXHAUSTED` | A retry-eligible, non-fatal failure stopped because restart policy allowed no further attempt or the retry limit was reached. |

`TaskRunInfo.phase` describes one attempt. `TaskStatus.phase` is the latest recorded logical state of the Task. A later retry or reconciliation can start another attempt after a terminal attempt phase.

Use the Task resource and run history for recorded outcomes; do not infer physical process state from a logical phase alone.
