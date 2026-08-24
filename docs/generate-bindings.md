---
title: Generate bindings
description: Pin one schema tag and generate coherent Rust or Go bindings from that exact snapshot.
---

# Generate bindings

Generate bindings from an exact repository tag, never from a moving branch. Keep generator configuration in the consuming repository, where language plugins, package paths, runtime versions, and output ownership belong.

## Check out one schema snapshot

```shell
git clone https://github.com/soltiHQ/proto.git proto
git -C proto checkout --detach v1.0.0
buf build proto
```

Replace `v1.0.0` with the exact release consumed by the application. Generate every imported public package from this checkout:

| Entry | Generated packages |
|---|---|
| `solti/task/v1/api.proto` | `solti.task.v1` |
| `solti/discover/v1/discovery.proto` | `solti.discover.v1` and imported `solti.agent.v1` types |

`solti.raft.v1` is internal control-plane state and is not part of the public agent binding set.

## Generate Rust with prost and tonic

Add the runtime and build dependencies selected by the consuming repository:

```shell
cargo add prost tonic tonic-prost
cargo add --build protoc-bin-vendored tonic-prost-build
```

These crates are generator tooling, not part of the Solti wire contract. Keep
their selected versions and `Cargo.lock` in the consuming repository.

Generate the public packages from `build.rs`:

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut prost = tonic_prost_build::Config::new();
    prost.protoc_executable(protoc_bin_vendored::protoc_bin_path()?);

    tonic_prost_build::configure()
        .build_client(true)
        .build_server(false)
        .compile_with_config(
            prost,
            &[
                "proto/solti/task/v1/types.proto",
                "proto/solti/task/v1/api.proto",
                "proto/solti/agent/v1/types.proto",
                "proto/solti/discover/v1/discovery.proto",
            ],
            &["proto"],
        )?;
    Ok(())
}
```

Expose the generated packages with their Protobuf nesting intact:

```rust
pub mod solti {
    pub mod agent {
        pub mod v1 {
            tonic::include_proto!("solti.agent.v1");
        }
    }
    pub mod discover {
        pub mod v1 {
            tonic::include_proto!("solti.discover.v1");
        }
    }
    pub mod task {
        pub mod v1 {
            tonic::include_proto!("solti.task.v1");
        }
    }
}
```

Commit `Cargo.lock` for applications. If generated files are committed, add a test that regenerates them and rejects drift.

## Generate Go with Buf

The schema intentionally has no universal `buf.gen.yaml`. A Go consumer can keep this template in its own repository and replace the module prefix:

```yaml
version: v2
managed:
  enabled: true
  override:
    - file_option: go_package_prefix
      value: example.com/your/module/gen
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen
    opt:
      - paths=source_relative
  - remote: buf.build/grpc/go
    out: gen
    opt:
      - paths=source_relative
```

Run it against the tagged checkout and exclude the internal Raft package:

```shell
buf generate proto \
  --template buf.gen.go.yaml \
  --path solti/task/v1 \
  --path solti/agent/v1 \
  --path solti/discover/v1
```

Keep the template and generator versions under consumer review. Regenerate only after deliberately changing the pinned schema tag.
