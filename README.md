[![Apache 2.0](https://img.shields.io/badge/license-Apache2.0-orange.svg)](./LICENSE)

# proto

Single source of truth for Solti's Protobuf / gRPC contracts.

The schema lives here; each consumer generates its own bindings (Rust via `prost`/`tonic`, Go via `buf`).

## Layout

The repo root is the `buf` module root. File paths match package names (`buf` `STANDARD` lint).

```
solti/
  task/v1/      # task management API
    types.proto
    api.proto
  discover/v1/  # agent discovery / heartbeat
    discovery.proto
  raft/v1/      # internal control-plane replication
    raft.proto
```

## Packages

| Package             | Service           | Purpose                                                         |
|---------------------|-------------------|-----------------------------------------------------------------|
| `solti.task.v1`     | `TaskService`     | Task management on the agent. The control-plane is the client.  |
| `solti.discover.v1` | `DiscoverService` | Agent registers and sends heartbeats to the control-plane.      |
| `solti.raft.v1`     | —                 | Internal Raft FSM state replicated between control-plane nodes. |

## Development

Tasks run `buf` inside a Docker image, so local runs match CI. Requires [Taskfile](https://taskfile.dev/) and Docker.

```shell
task ci/lint      # buf lint
task ci/format    # clang-format check (aligned style)
task ci/build     # buf build (compile the schema)
task ci/breaking  # buf breaking against main
task format/fix   # clang-format -i (auto-format)
```

## Versioning

Schema is versioned in the package path (`v1`).

## License

[Apache License, Version 2.0](LICENSE)

## Contributing

Found a bug? Have an idea? [Open an issue](https://github.com/soltiHQ/proto/issues) or send a PR.

<div>
  <a href="https://github.com/soltiHQ/sdk"><img alt="Dashboards" src="https://img.shields.io/badge/SDK-f46800?style=for-the-badge&logo=grafana&logoColor=white"></a>
  <a href="https://github.com/soltiHQ/taskvisor"><img alt="Taskvisor" src="https://img.shields.io/badge/Taskvisor-2c3e50?style=for-the-badge&logo=rust&logoColor=white"></a>
</div>
