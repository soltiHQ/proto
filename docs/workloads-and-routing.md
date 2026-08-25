---
title: Workloads and routing
description: Select a versioned workload, match a compatible runner, and coordinate slot conflicts independently.
---

# Workloads and routing

Every public Task workload has an API version, a kind, and exactly one typed specification. An eligible runner must match the workload identity and the optional label selector.

Slot admission is a separate decision. It defines how the agent handles a busy execution lane after routing.

## Choose a workload representation

`TaskWorkload` supports four public forms:

| Form | Desired configuration |
|---|---|
| `SubprocessTask` | A direct command or interpreter-backed script, environment, working directory, and non-zero-exit behavior. |
| `WasmTask` | A WebAssembly module path, arguments, and environment. |
| `ContainerTask` | An OCI image, optional entrypoint override, arguments, and environment. |
| `ExtensionTask` | One UTF-8 JSON value interpreted by an application-provided runner. |

Embedded workloads intentionally have no public wire representation.

The selected `oneof spec` must agree with `api_version` and `kind`. The schema requires the identity fields but does not publish a universal list of built-in GVK strings. Obtain accepted API-version and kind pairs from `AgentCapabilities` or from the serving implementation's versioned documentation.

## Configure each built-in shape

For a subprocess:

- `CommandMode.command` is an executable name or path resolved directly, with `args` passed to it;
- `ScriptMode.interpreter` selects the executable that runs the script;
- `ScriptMode.body` carries base64-encoded script content;
- `cwd` is optional and uses the agent default when absent;
- `fail_on_non_zero` decides whether a non-zero process exit is a failure.

For a container, an absent `command` uses the image default. A present `ContainerCommand` is an explicit entrypoint override, even when its `items` list is empty.

For an extension, `RawExtension.raw` is bytes containing one UTF-8 JSON value. Validation and interpretation of that JSON belong to the selected runner.

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
