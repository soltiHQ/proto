---
title: Discovery and capabilities
description: Register an agent endpoint, send heartbeat state, and advertise the runners available for task routing.
---

# Discovery and capabilities

`DiscoverService.Sync` is the agent-to-control-plane registration and heartbeat call. It publishes how to reach the agent and what the agent can run.

## Send one heartbeat snapshot

Each `SyncRequest` carries:

- stable agent identity and display name;
- endpoint address and whether it serves gRPC or HTTP;
- uptime, operating-system, architecture, platform, and send time;
- agent metadata and advertised heartbeat interval;
- the agent API version spoken at the endpoint;
- registered runner capabilities.

`api_version` reports the endpoint API version, where `1` means v1. It is distinct from the repository release tag.

## Advertise routing capabilities

`AgentCapabilities` lists registered runner instances. Each entry gives the runner name, routing labels, and accepted workload API-version and kind pairs.

A control plane can use this snapshot to understand which public workloads the agent advertises. The capability snapshot does not select a runner; `TaskSpec.runner_selector` requests label-based selection when desired work is applied.

## Handle the response

`SyncResponse.success` says whether the control plane accepted the heartbeat. When it is false, `reason` carries the rejection cause.
`retry_after_s` can request a delay before the next call.

The request advertises endpoint coordinates but does not define endpoint authentication, authorization, or reachability policy.
