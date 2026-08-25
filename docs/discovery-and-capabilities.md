---
title: Discovery and capabilities
description: Register an agent endpoint, send heartbeat snapshots, and advertise the runners available for Task routing.
---

# Discovery and capabilities

`solti.discover.v1.DiscoverService.Sync` is the agent-to-control-plane registration and heartbeat call. It publishes how to reach the agent and which workloads its registered runners accept.

## Follow the call direction

```text
agent ── SyncRequest ──► control plane
         └── endpoint + heartbeat state + AgentCapabilities

control plane ── SyncResponse ──► agent
                 └── accepted/rejected + reason + suggested retry delay
```

The agent is the `Sync` client. The control plane is the `DiscoverService` server. This is the reverse of `TaskService`, which is served by the agent.

## Send one heartbeat snapshot

Each `SyncRequest` carries:

- stable agent identity and display name;
- endpoint address and `GRPC` or `HTTP` endpoint type;
- uptime, operating-system name, architecture, and platform;
- send time in Unix seconds;
- agent-supplied metadata;
- the API version spoken by the endpoint;
- the agent's heartbeat interval in seconds;
- current runner capabilities.

`ENDPOINT_TYPE_UNSPECIFIED` is invalid and is rejected by the server. `api_version = 1` identifies v1 of the agent endpoint API. It is not the Contracts repository release.

This Protobuf text-format example shows one heartbeat shape:

```protobuf
id: "agent-west-1"
name: "West build agent"
endpoint: "agent-west-1.internal:7443"
uptime_seconds: 86400
os: "linux"
arch: "amd64"
platform: "unix"
ts: 1787616000
metadata { key: "environment" value: "production" }
endpoint_type: ENDPOINT_TYPE_GRPC
api_version: 1
heartbeat_interval_s: 15
capabilities {
  runners {
    name: "container-runner"
    labels { key: "runtime" value: "oci" }
    workloads {
      api_version: "example.workloads/v1"
      kind: "ContainerJob"
    }
  }
}
```

The endpoint, timestamps, labels, and workload GVK are illustrative. Use values from the running agent and its registered runners.

## Advertise routing capabilities

`AgentCapabilities.runners` contains registered runner instances. Each `RunnerCapability` declares:

- a unique runner name;
- labels matched by `TaskSpec.runner_selector`;
- accepted workload API-version and kind pairs.

A capability snapshot describes what the agent advertises at heartbeat time. It does not itself select a runner for a Task. Task routing also uses the workload GVK and optional structured runner selector.

Read [Workloads and routing](workloads-and-routing.md) for selector and slot behavior.

## Handle the response

`SyncResponse.success` reports whether the control plane accepted the heartbeat. When it is false, `reason` carries a readable rejection cause. `retry_after_s` is a suggested delay before the next `Sync` call.

The schema does not define precedence between `retry_after_s` and the agent's own heartbeat scheduler. That policy belongs to the implementations using the contract.

Do not parse `reason` as a stable category. The response has no typed rejection enum in v1.

## Keep deployment policy outside the payload

The request advertises endpoint coordinates. It does not define:

- endpoint authentication or authorization;
- TLS identity or trust roots;
- network reachability;
- how the control plane marks missed heartbeats stale;
- runner tie-breaking when several capabilities match.

Those policies must be agreed by the agent and control-plane deployments.

## Use the exact schema

- [`DiscoverService`, heartbeat request, and response](../solti/discover/v1/discovery.proto)
- [`AgentCapabilities` and runner types](../solti/agent/v1/types.proto)
