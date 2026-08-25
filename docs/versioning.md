---
title: Versioning
description: Pin an exact Contracts release and keep repository, package, resource, endpoint, and snapshot versions distinct.
---

# Versioning

Contracts has several independent version markers. A consumer must know which boundary each marker identifies.

## Keep the version domains separate

| Marker | Example in this schema | What it identifies |
|---|---|---|
| Repository release | A tag such as `v0.1.1` | One exact snapshot of every `.proto` file and this guide. |
| Documentation compatibility line | `0.1` | The major/minor release line served by this documentation. |
| Protobuf package suffix | `solti.task.v1` | A wire API generation and generated namespace. |
| Task resource API | `TaskManifest.api_version = "solti.io/v1"` | The public Task resource shape. |
| Agent endpoint API | `SyncRequest.api_version = 1` | The API version spoken at the advertised endpoint. |
| Raft snapshot format | `SnapshotHeader.version = 1` | The internal snapshot serialization format. |

Changing one marker does not automatically change the others.

## Pin an exact release

Generate all required packages from one exact repository tag. Do not use `main` as a production dependency and do not combine imports from separate checkouts.

The documentation build reads the exact semantic version from the repository's root `version` file. CI verifies that `docs/site.yml` declares the matching major/minor compatibility line.

Follow [Generate bindings](generate-bindings.md) for consumer-owned generator configuration and an external tagged schema checkout.

## Understand compatibility checks

The repository's [`buf.yaml`](../buf.yaml) uses Buf `FILE` breaking-change policy. CI compares schema changes with `main` in addition to compiling and linting the schema.

A passing breaking-change check establishes compatibility under that configured Buf policy. It does not test runtime behavior, generated client migration, storage migration, or interoperability with every deployed consumer.

Internal `solti.raft.v1` state remains part of the tagged source snapshot even though it is not a public agent contract. Snapshot restore must also validate `SnapshotHeader.magic` and `SnapshotHeader.version`.

## Upgrade a consumer deliberately

1. Select the target Contracts release tag.
2. Review the schema and documentation changes between the current and target tags.
3. Regenerate the complete required package set from the target tag.
4. Compile the consumer from an empty generated-output directory.
5. Test client/server interoperability for the RPCs the consumer uses.
6. For an internal Raft consumer, test command decoding and snapshot restore with the supported stored formats.
7. Update the pinned Contracts reference only after those checks pass.

Do not edit copied schema or generated output to bridge an incompatibility. Change generator configuration or upgrade the consumer against the canonical contract.

## Use tagged source for field details

This guide explains semantics and integration boundaries. Read the tagged `.proto` files for exact RPC signatures, field types, numbers, enum values, reserved identities, and comments.

Keeping field-level identities in the schema avoids maintaining a second wire definition in prose.
