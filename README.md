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

## Consuming

Vendor this repo as a git submodule and generate bindings locally; pin to a commit or tag. 
Code generation and `go_package` / output paths are the consumer's concern (`buf` managed mode on the Go side, `build.rs` on the Rust side).

## Development

Tasks run `buf` inside a Docker image, so local runs match CI. Requires [Taskfile](https://taskfile.dev/) and Docker.

```shell
task ci/lint      # buf lint
task ci/format    # buf format --diff --exit-code (check)
task ci/build     # buf build (compile the schema)
task ci/breaking  # buf breaking against main
task format/fix   # buf format -w (auto-format)
```

## CI

| Trigger             | Checks                          |
|---------------------|---------------------------------|
| push (non-`main`)   | format, lint, build             |
| pull request        | format, lint, build, breaking   |

`breaking` compares against `main`, so backward-incompatible edits fail the PR.

## Versioning

Schema is versioned in the package path (`v1`). 
