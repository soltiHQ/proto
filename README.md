[![Apache 2.0](https://img.shields.io/badge/license-Apache2.0-orange.svg)](./LICENSE)

# proto

Single source of truth for Solti's Protobuf / gRPC contracts.

The schema lives here; each consumer generates its own bindings (Rust via `prost`/`tonic`, Go via `buf`).

[Documentation](https://solti.io/docs/contracts/latest/) · [Generate bindings](docs/generate-bindings.md) · [Versioning](docs/versioning.md)

## Layout

The repo root is the `buf` module root. File paths match package names (`buf` `STANDARD` lint).

```
solti/
  task/v1/      # task management API
    types.proto
    api.proto
  agent/v1/     # shared agent capability types
    types.proto
  discover/v1/  # agent discovery / heartbeat
    discovery.proto
  raft/v1/      # internal control-plane replication
    raft.proto
```

## Packages

| Package             | Service           | Purpose                                                         |
|---------------------|-------------------|-----------------------------------------------------------------|
| `solti.task.v1`     | `TaskService`     | Task management on the agent. The control-plane is the client.  |
| `solti.agent.v1`    | —                 | Runner and workload capabilities advertised during discovery.   |
| `solti.discover.v1` | `DiscoverService` | Agent registers and sends heartbeats to the control-plane.      |
| `solti.raft.v1`     | —                 | Internal Raft FSM state replicated between control-plane nodes. |

The Task, Agent, and Discover packages are public contracts. `solti.raft.v1` is internal control-plane state.

## Development

Commands delegate to the shared `soltiHQ/actions@v1` Proto Taskfile, which runs
the local tools directly. Requires [Taskfile](https://taskfile.dev/),
[Buf](https://buf.build/docs/cli/installation/), and `clang-format`. On macOS,
the formatter bundled with Xcode Command Line Tools is detected automatically.
CI pins Buf 1.50.0 and clang-format 18.

```shell
task fmt          # clang-format -i (auto-format)
task ci/fmt       # clang-format check (aligned style)
task ci/lint      # buf lint
task ci/build     # buf build (compile the schema)
task ci/breaking  # buf breaking against main
```

## Versioning

Repository releases and Protobuf package paths are separate version markers. Pin generated bindings to an exact repository
tag such as `v1.0.0`. The `v1` package suffix identifies the wire API generation and does not select an exact schema snapshot.
Generate every imported package from the same tag.

See [Generate bindings](docs/generate-bindings.md) for a complete checkout and generation flow, and [Versioning](docs/versioning.md) for compatibility rules.

## Contributing

Found a bug? Have an idea? [Open an issue](https://github.com/soltiHQ/proto/issues) or send a PR.

<div>
  <a href="https://github.com/soltiHQ/sdk"><img alt="SDK" src="https://img.shields.io/badge/SDK-2c3e50?style=for-the-badge&logo=rust&logoColor=white"></a>
  <a href="https://github.com/soltiHQ/taskvisor"><img alt="Taskvisor" src="https://img.shields.io/badge/Taskvisor-2c3e50?style=for-the-badge&logo=rust&logoColor=white"></a>
  <a href="https://github.com/soltiHQ/podium"><img alt="Podium" src="https://img.shields.io/badge/Podium-2c3e50?style=for-the-badge&logo=go&logoColor=white"></a>
</div>
