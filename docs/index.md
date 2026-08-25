---
title: Solti API Contracts
description: Use the versioned Protobuf boundary for Task management, agent discovery, runner capabilities, and internal Raft state.
---

# Solti API Contracts

This repository is the canonical, implementation-neutral schema for communication between Solti components.
It defines public agent APIs and the internal state format used by the control plane.
Consumers generate language bindings from one exact repository release.

## Choose the contract you need

| Package | Audience | Contract |
|---|---|---|
| `solti.task.v1` | Agent clients and agent servers | Create desired Task resources, observe execution, and request cancellation or deletion. |
| `solti.agent.v1` | Discovery clients and servers | Describe registered runners and the workload types they accept. |
| `solti.discover.v1` | Agents and control planes | Register an agent endpoint and send heartbeat state to a control plane. |
| `solti.raft.v1` | Control-plane internals only | Encode replicated FSM commands and full Raft snapshots. |

The first three packages are public contracts. `solti.raft.v1` is an internal persistence contract and must not be exposed as an agent API.

## Follow the shortest path

If this is your first time here:

1. Read the [contract model](contract-model.md) to see which side serves each API.
2. Follow the [quick start](quick-start.md) to pin and validate one schema snapshot.
3. Use [Generate bindings](generate-bindings.md) for Rust or Go generation.
4. Open the guide for the package or workflow you implement.

| You need to | Continue with |
|---|---|
| Write or reconcile Task desired state | [Task resources](task-resources.md) |
| Configure retries, timeouts, and restart behavior | [Execution policies](execution-policies.md) |
| Select a workload, runner, and slot policy | [Workloads and routing](workloads-and-routing.md) |
| List, watch, inspect runs, or stream output | [Task observation](task-observation.md) |
| Handle gRPC failures and write conflicts | [Task API errors](task-errors.md) |
| Register an agent and advertise capabilities | [Discovery and capabilities](discovery-and-capabilities.md) |
| Implement Raft persistence inside the control plane | [Internal Raft state](raft-state.md) |
| Prepare an integration for deployment | [Production boundaries](production-boundaries.md) |

## Use the schema as the exact reference

This guide explains relationships, workflows, and operational boundaries. The `.proto` files from the same repository release remain the exact source for RPC signatures, field types, field numbers, enum values, reserved identities, and message comments:

- [`solti/task/v1/api.proto`](../solti/task/v1/api.proto)
- [`solti/task/v1/types.proto`](../solti/task/v1/types.proto)
- [`solti/agent/v1/types.proto`](../solti/agent/v1/types.proto)
- [`solti/discover/v1/discovery.proto`](../solti/discover/v1/discovery.proto)
- [`solti/raft/v1/raft.proto`](../solti/raft/v1/raft.proto)

Do not combine generated packages from different repository releases. Do not use a moving branch as a production dependency.

## Related projects

- [Solti SDK](https://github.com/soltiHQ/sdk)
- [Podium](https://github.com/soltiHQ/podium)
- [Taskvisor](https://github.com/soltiHQ/taskvisor)

Check the pinned Contracts dependency in the exact project release you are integrating.
