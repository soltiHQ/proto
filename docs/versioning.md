---
title: Versioning
description: Pin one repository release, distinguish it from package versions, and validate changes before tagging.
---

# Versioning

The repository release and the Protobuf package path are separate version markers. Consumers need both to identify the contract they generated.

## Pin the repository snapshot

`VERSION` contains the exact repository release version. A release tag adds the `v` prefix, for example `VERSION` `1.0.0` corresponds to tag `v1.0.0`.

Pin consumers to the exact repository tag and generate all required packages from that snapshot. This keeps imported messages and service definitions aligned.
Follow [Generate bindings](generate-bindings.md) for the tagged checkout and consumer-owned generator configuration.

## Read the package path

Public packages currently use the `v1` namespace:

- `solti.task.v1`
- `solti.agent.v1`
- `solti.discover.v1`

The package suffix identifies the wire API generation. It is not a replacement for the exact repository tag.

This guide belongs to documentation compatibility line `1.0`, derived from repository version `1.0.0`.

## Validate before a tag

The repository checks formatting, Buf lint, schema build, and breaking changes against `main`. Documentation validation checks that this guide declares the same exact version and compatibility line.

The public guide excludes internal `solti.raft.v1` state. That package remains part of the tagged source snapshot for its control-plane consumers.

## Use tagged source for field-level details

The 1.0 documentation does not declare an external generated schema reference. Read the tagged `.proto` files for exact RPC signatures, field types, field numbers, enum values, and comments.

Keeping semantic guidance here and field-level definitions in the schema avoids maintaining two copies of the same contract.
