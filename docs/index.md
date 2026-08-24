---
title: Solti API Contracts
description: Understand the public Protobuf boundary for task management, agent discovery, and runner capabilities.
---

# Solti API Contracts

These Protobuf definitions are the implementation-neutral wire boundary between a Solti agent and remote clients or control planes.
Each consumer generates its own language bindings from the same tagged schema snapshot.

## Choose a public package

| Package | Contract |
|---|---|
| `solti.task.v1` | Manage desired Task resources and observe their execution on an agent. |
| `solti.agent.v1` | Describe an agent's registered runners and accepted workload types. |
| `solti.discover.v1` | Register an agent endpoint and send heartbeat state to a control plane. |

Start with the [contract model](contract-model.md), then open the guide for the traffic you implement:

- [Task resources](task-resources.md) for create, apply, cancel, delete, and write safety.
- [Task observation](task-observation.md) for lists, watches, run history, and live output.
- [Workloads and routing](workloads-and-routing.md) for workload types and runner selection.
- [Task API errors](task-errors.md) for gRPC status handling and structured write conflicts.
- [Discovery and capabilities](discovery-and-capabilities.md) for agent heartbeats.
- [Generate bindings](generate-bindings.md) for reproducible Rust and Go generation from one tag.

## Use the tagged schema as the exact reference

This guide explains cross-message semantics and operational boundaries. It does not duplicate every field, enum value, or field number.
For the exact contract, read the `.proto` files from the same repository tag as this guide.

`solti.raft.v1` is internal control-plane replication state. It is intentionally excluded from this public guide.
