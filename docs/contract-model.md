---
title: Contract model
description: See which side serves each API and how the public packages compose without sharing an implementation.
---

# Contract model

The public schema defines two independent request paths. Task management targets an agent, while discovery originates at an agent.

## Follow the traffic

```text
remote client or control plane ── TaskService ──► agent

agent ── DiscoverService.Sync ──► control plane
          └── AgentCapabilities
```

`solti.task.v1.TaskService` is exposed by the agent. Its messages describe desired Task state, stored resource identity, observed status, history, and output events.

`solti.discover.v1.DiscoverService` receives agent registration and heartbeat state. The request embeds `solti.agent.v1.AgentCapabilities`, which lists runner instances and the workload types each runner accepts.

Neither direction requires the client and server to share an implementation language.

## Keep one coherent schema snapshot

Generate all public packages from one repository tag. Imports between `solti.discover.v1` and `solti.agent.v1` are resolved inside that snapshot.

The repository tag and the package suffix have different roles:

- A tag such as `v1.0.0` selects one complete schema snapshot.
- The `v1` package path names the wire API generation.

Do not substitute one marker for the other when pinning generated bindings.

## Know what stays outside the schema

The repository owns Protobuf messages, enums, services, and field identities.
Consumers own generated bindings and application behavior around the wire contract.

Storage, endpoint authentication, authorization, TLS policy, and network topology are not defined by these messages.
Internal `solti.raft.v1` state is not a public agent contract.
