---
title: Workloads and routing
description: Select a versioned workload, match a compatible runner, and coordinate slot conflicts independently.
---

# Workloads and routing

Every public Task workload has an API version, a kind, and one JSON body. An eligible runner must match the workload identity and the optional label selector.

Slot admission is a separate decision. It defines how the agent handles a busy execution lane after routing.

## Send one body form

`TaskWorkload` has one public form: `ExtensionTask`. Its `RawExtension.raw` is bytes containing one UTF-8 JSON object, the workload body. That is the case for every workload type, built-in or application-defined.

The body is validated against the JSON Schema that the declaring runner advertises for the workload's `api_version` and `kind`. The schema decides what a valid body is, and the runner interprets it. The schema requires the identity fields but does not publish a universal list of GVK strings. Obtain accepted API-version and kind pairs from `AgentCapabilities` or from the serving implementation's versioned documentation.

Embedded workloads intentionally have no public wire representation. The former typed fields (field numbers 3, 4 and 5) are reserved and are not accepted.

## Describe a type's body with a JSON Schema

An agent declares each workload type it accepts as an API-version and kind pair plus the fingerprint of a JSON Schema. The schema describes the body of a workload of that type. The control plane obtains the schema through [discovery](discovery-and-capabilities.md) and validates a workload body against it before placement.

See [Send one body form](#send-one-body-form) for how the body travels.

## Match a runner capability

An agent advertises registered runner instances through `AgentCapabilities`:

```protobuf
runners {
  name: "extension-runner"
  labels { key: "runtime" value: "extension" }
  labels { key: "region" value: "west" }
  workloads {
    api_version: "example.workloads/v1"
    kind: "Report"
  }
}
```

The values are illustrative. A Task workload is eligible only when a runner advertises the same API-version and kind pair.

`TaskSpec.runner_selector` can narrow that set by runner labels:

```protobuf
runner_selector {
  match_labels { key: "runtime" value: "extension" }
  match_expressions {
    key: "region"
    operator: SELECTOR_OPERATOR_IN
    values: "west"
    values: "central"
  }
}
```

`match_labels` and every expression are ANDed. `IN` and `NOT_IN` require values. `EXISTS` and `DOES_NOT_EXIST` require an empty values list.

The schema does not define tie-breaking when more than one runner matches. That behavior belongs to the serving implementation.

## Keep selector formats distinct

`TaskSpec.runner_selector` is a structured `LabelSelector` message. `ListTasksRequest.label_selector` and `WatchTasksRequest.label_selector` are strings using label-selector syntax.

Do not serialize the structured runner selector into the collection-filter field. They select different labeled objects and have different wire types.

## Coordinate a slot

`TaskSpec.slot` is a non-empty execution lane with a maximum length of 64 characters. Tasks that share a slot are subject to one admission policy:

| Policy | Busy-slot behavior |
|---|---|
| `ADMISSION_POLICY_DROP_IF_RUNNING` | Ignore the new Task and report success. |
| `ADMISSION_POLICY_REPLACE` | Cancel the running Task and start the new Task. |
| `ADMISSION_POLICY_QUEUE` | Wait until the slot is free. |
| `ADMISSION_POLICY_UNSPECIFIED` | Zero-value sentinel; it does not select a policy. |

Runner routing and slot admission solve different problems:

```text
workload GVK + runner_selector ──► compatible runner

slot + admission policy ─────────► busy-slot behavior
```

Do not use runner labels as a concurrency key. Do not use a slot to claim a particular runner implementation.

## Use the exact schema

- [`TaskWorkload`, selectors, `TaskSpec`, and policies](../solti/task/v1/types.proto)
- [`AgentCapabilities` and runner workload types](../solti/agent/v1/types.proto)
