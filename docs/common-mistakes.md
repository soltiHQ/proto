---
title: Common mistakes
description: Avoid version, identity, routing, observation, and internal-state mistakes when integrating Contracts.
---

# Common mistakes

Most integration failures come from mixing boundaries that look similar on the wire but have different ownership or lifetime rules.

## Using `v1` as an exact dependency version

`solti.task.v1` identifies a wire API generation. It does not select one exact schema snapshot.

Pin a repository release and generate every imported package from that release. Read [Versioning](versioning.md).

## Mixing packages from different releases

Discovery imports agent capability types. Internal Raft state imports the same package. Generating imports from another checkout can create a binding set that never existed as one schema snapshot.

Use one root checkout and one generation invocation per package set.

## Copying schema or generated code into a consumer

Copied `.proto` files create a second source of truth. Committed generated bindings can silently drift from the selected release.

Keep generator configuration in the consumer. Fetch the canonical Contracts release and regenerate into ignored build output during local and CI workflows.

## Applying stale desired state without preconditions

An unconditional `ApplyTask` can overwrite a newer stored revision. Read the current Task and carry its UID and resource version when stale writes are unsafe.

On `Aborted`, decode `WriteConflictDetails`. Do not parse the status message.

## Treating a Task name as permanent identity

A name is a stable address. A UID identifies one incarnation under that name.

Keep the UID with run history and log subscriptions. A deleted and recreated Task is not the same incarnation.

## Treating cancellation as physical process exit

`CANCELED` is a logical phase. Force-aborted task code can remain physically active.

Use a separate implementation-owned barrier when physical exit or external lease release is required.

## Treating live logs as an archive

`StreamTaskLogs` can emit `Lagged`. It has no replay cursor in v1.

Record gaps and use deployment-owned durable logging when complete output is required.

## Reusing the wrong cursor

Task list continuation, Task collection resource version, run-history continuation, and watch position have API-specific meaning.

Do not parse them. Do not pass a cursor to a different API because both fields are strings.

## Confusing runner routing with slot admission

Workload GVK and `runner_selector` choose an eligible runner. `slot` and `admission` coordinate overlap.

A slot is not a runner selector. Runner labels are not a concurrency key.

## Sending a structured selector as a collection filter

`TaskSpec.runner_selector` is a `LabelSelector` message. List and watch requests accept `label_selector` as a string.

Use the wire type declared by the target field.

## Exposing internal Raft messages

`solti.raft.v1` is an internal persistence contract and contains secret-bearing state. It is not a public service schema.

Generate it only for the control-plane persistence boundary and protect logs, snapshots, backups, and diagnostics.

## Parsing human-readable diagnostics

Status messages, `TaskCondition.message`, discovery rejection `reason`, and execution `error` strings are readable diagnostics.

Branch on gRPC status codes, typed conflict reasons, condition type/status/reason, enums, and explicit fields instead.
