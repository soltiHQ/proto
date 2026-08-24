---
title: Task API errors
description: Branch on gRPC status codes and typed write-conflict causes without parsing diagnostic messages.
---

# Task API errors

Use the gRPC status code as the top-level machine-readable category. Status messages and cause messages are diagnostics for operators; clients must not parse them to choose behavior.

## Handle status codes

| gRPC code | Meaning |
|---|---|
| `InvalidArgument` | The request, selector, continuation token, phase, or page limit is invalid. |
| `Unauthenticated` | The configured authentication boundary did not accept a credential. |
| `PermissionDenied` | The authenticated identity is not allowed to perform the operation. |
| `AlreadyExists` | `CreateTask` targeted an existing Task name. |
| `Aborted` | A write precondition did not match the current Task. Decode `WriteConflictDetails`. |
| `NotFound` | The requested Task or public resource does not exist or is not visible. |
| `Unimplemented` | The serving application does not implement the requested operation. |
| `ResourceExhausted` | A request or response item is too large, or configured service capacity is exhausted. |
| `OutOfRange` | The requested collection, run, or watch position is no longer retained. |
| `Unavailable` | The service is shutting down or cannot currently accept work. |
| `Internal` | An unexpected server failure occurred; the client receives no internal diagnostic. |

Authentication and authorization policy are deployment-owned. `Unauthenticated` and `PermissionDenied` apply when the serving application enables those boundaries.

## Decode write conflicts

`ApplyTask`, `CancelTask`, and `DeleteTask` can carry `WritePreconditions`. A mismatch returns `Aborted`, and the raw gRPC status-details bytes encode one `WriteConflictDetails` message.

Each `WriteConflictCause` has a typed reason:

| Reason | Related field | Meaning |
|---|---|---|
| `WRITE_CONFLICT_REASON_UID_MISMATCH` | `preconditions.uid` | The requested UID differs from the stored Task incarnation. |
| `WRITE_CONFLICT_REASON_RESOURCE_VERSION_MISMATCH` | `preconditions.resourceVersion` | The requested resource version differs from the stored revision. |
| `WRITE_CONFLICT_REASON_PRECONDITION_FAILED` | optional | A failed precondition has no more specific v1 category. |

The agent does not intentionally emit `WRITE_CONFLICT_REASON_UNSPECIFIED`. Branch on the enum reason, use `field` to locate the failed request value, and present `message` only as a readable diagnostic.

## Treat stream failures as terminal

A watch can begin successfully and later terminate with a gRPC status. In particular, `OutOfRange` means the retained watch position expired; start a new list-and-watch cycle instead of replaying the same resource version.

Live output is lossy by contract. A `Lagged` event is an observation gap inside a healthy stream, while a terminal gRPC status ends the stream itself.
