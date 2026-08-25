---
title: Quick start
description: Pin one Contracts release, validate it with Buf, and make a read-only TaskService call.
---

# Quick start

This flow checks out one immutable schema snapshot, compiles it with Buf, and optionally calls `ListTasks` without generating language bindings.

## Get one release

You need Git and [Buf](https://buf.build/docs/cli/installation/). Clone the repository, inspect the available release tags, and enter the exact tag your application will consume:

```sh
git clone https://github.com/soltiHQ/proto.git solti-proto
cd solti-proto
git fetch --tags
git tag --list 'v*'
printf 'Contracts release tag: '
read -r SOLTI_PROTO_REF
git checkout --detach "$SOLTI_PROTO_REF"
```

Do not enter `main` or another moving branch. Record the selected tag in the consuming application's dependency or build configuration.

## Validate the schema

Run the checks that do not need another repository revision:

```sh
buf lint
buf build
```

`buf lint` checks the repository's `STANDARD` lint policy. `buf build` resolves imports and compiles every package in this snapshot.

The repository's full CI also checks formatting and compares schema compatibility with `main`.

## Inspect a Task endpoint

If you have an agent endpoint and [`grpcurl`](https://github.com/fullstorydev/grpcurl), this read-only call lists up to ten Tasks without relying on server reflection:

```sh
grpcurl \
  -plaintext \
  -import-path . \
  -proto solti/task/v1/api.proto \
  -d '{"limit": 10}' \
  AGENT_HOST:PORT \
  solti.task.v1.TaskService/ListTasks
```

`AGENT_HOST:PORT` is a placeholder. The `-plaintext` flag is valid only for a plaintext endpoint. Authentication, TLS, and endpoint addressing are deployment-owned; use the flags required by that deployment.

The response is a `ListTasksResponse`. Treat its `resource_version` and `continue` values as opaque. Read [Task observation](task-observation.md) before implementing pagination or watches.

## Generate application bindings

The repository contains schemas rather than checked-in Rust or Go bindings. Continue with [Generate bindings](generate-bindings.md) and generate only the package set needed by the application:

| Integration | Entry schema |
|---|---|
| Task client or server | `solti/task/v1/api.proto` |
| Discovery client or server | `solti/discover/v1/discovery.proto` |
| Internal Raft persistence | `solti/raft/v1/raft.proto` |

Imports must come from the same checkout. Keep the checkout reference and generator versions independently pinned.

## Continue with one workflow

- To create or apply desired Task state, read [Task resources](task-resources.md).
- To advertise an agent, read [Discovery and capabilities](discovery-and-capabilities.md).
- To persist control-plane FSM state, read [Internal Raft state](raft-state.md).
