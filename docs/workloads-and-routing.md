---
title: Workloads and routing
description: Select a versioned workload type and route it to a compatible registered runner.
---

# Workloads and routing

Every public Task workload carries an API version, a kind, and one typed specification. These values identify the workload schema and participate in runner routing.

## Choose a workload representation

`TaskWorkload` supports four public forms:

| Form | Desired configuration |
|---|---|
| `SubprocessTask` | A command or interpreter-backed script, arguments, environment, working directory, and non-zero-exit behavior. |
| `WasmTask` | A WebAssembly module path, arguments, and environment. |
| `ContainerTask` | An OCI image, optional entrypoint override, arguments, and environment. |
| `ExtensionTask` | One UTF-8 JSON object interpreted by an application-provided runner. |

Embedded workloads intentionally have no public wire representation.
The schema describes desired configuration; the selected runner implementation defines how that workload executes.

## Match a runner

An agent publishes `AgentCapabilities`. Each `RunnerCapability` contains:

- a unique runner name;
- labels used by Task runner selectors;
- workload API-version and kind pairs accepted by that runner.

`TaskSpec.runner_selector` narrows eligible runners by labels. The workload API version and kind must also match a workload type advertised by the runner.

Discovery tells a control plane what the agent currently exposes. It does not add a new workload type to the Task schema.

## Coordinate a slot

`TaskSpec.slot` selects an execution lane. `AdmissionPolicy` defines what happens when that slot is already busy: drop the new Task, replace the running Task, or queue the new Task.

Admission behavior and runner selection solve different problems. The selector chooses an implementation; the slot coordinates concurrent desired work.
