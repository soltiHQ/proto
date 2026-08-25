---
title: Task API errors
description: Branch on gRPC status codes and typed write-conflict causes without parsing diagnostic messages.
---

# Task API errors

Use the gRPC status code as the top-level machine-readable category. Treat status messages and cause messages as operator diagnostics, not as a client decision API.

## Handle status categories

| gRPC code | Contract meaning |
|---|---|
| `InvalidArgument` | The request, selector, continuation token, phase, or page limit is invalid. |
| `Unauthenticated` | The configured authentication boundary did not accept a credential. |
| `PermissionDenied` | The authenticated identity is not allowed to perform the operation. |
| `AlreadyExists` | `CreateTask` targeted an existing Task name. |
| `Aborted` | A write precondition did not match the stored Task. Decode `WriteConflictDetails`. |
| `NotFound` | The requested Task or visible public resource does not exist. |
| `Unimplemented` | The serving application does not implement the requested operation. |
| `ResourceExhausted` | A request or response item is too large, or configured service capacity is exhausted. |
| `OutOfRange` | A requested collection, run, or watch position is no longer retained. |
| `Unavailable` | The service is shutting down or cannot currently accept work. |
| `Internal` | An unexpected server failure occurred without exposing an internal diagnostic. |

Authentication and authorization policy are deployment-owned. `Unauthenticated` and `PermissionDenied` apply when the serving application enables those boundaries.

## Decode write conflicts

`ApplyTask`, `CancelTask`, and `DeleteTask` can carry `WritePreconditions`. A mismatch returns `Aborted` with `WriteConflictDetails` in the status details.

The details identify the Task name and contain one or more `WriteConflictCause` values:

| Reason | Related request field | Meaning |
|---|---|---|
| `WRITE_CONFLICT_REASON_UID_MISMATCH` | `preconditions.uid` | Requested UID differs from the stored Task incarnation. |
| `WRITE_CONFLICT_REASON_RESOURCE_VERSION_MISMATCH` | `preconditions.resource_version` | Requested resource version differs from the stored revision. |
| `WRITE_CONFLICT_REASON_PRECONDITION_FAILED` | Optional | A failed precondition has no more specific v1 category. |
| `WRITE_CONFLICT_REASON_UNSPECIFIED` | None | Zero-value sentinel; the agent does not intentionally emit it. |

Use the gRPC status-details mechanism provided by the selected runtime to decode `WriteConflictDetails`. Branch on each typed `reason`. Use `field` to locate the failed input and show `message` only as readable context.

## Use a deliberate client decision tree

```text
RPC result
├── success ──► use the returned resource or acknowledgement
├── Aborted ──► decode conflict details
│                └──► re-read current state before deciding to reapply
├── InvalidArgument ──► fix the request; do not retry unchanged
├── Unauthenticated / PermissionDenied ──► fix deployment access
├── Unavailable / ResourceExhausted ──► apply caller-owned retry policy
└── other status ──► handle by typed code and operation context
```

The schema does not declare one universal retry policy. A caller must consider operation idempotency, deadlines, side effects, and whether the request carried write preconditions.

## Treat stream failures as terminal

A watch or log stream can open successfully and later end with a gRPC status.

For a watch, `OutOfRange` means the retained position expired. Start a fresh observation cycle instead of retrying the same resource version.

For live output, `Lagged` is a message inside a healthy stream and reports an observation gap. It is not a terminal gRPC status. A terminal status ends the stream.

## Use the exact schema

- [`WritePreconditions`, conflict details, and Task RPCs](../solti/task/v1/api.proto)
