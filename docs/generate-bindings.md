---
title: Generate bindings
description: Generate coherent Rust or Go bindings from one exact Contracts release without copying schema sources into a consumer.
---

# Generate bindings

Generate all required packages from one exact repository release. Keep generator configuration in the consuming repository, where language plugins, runtime versions, output paths, and client/server choices belong.

Do not copy `.proto` files into a consumer repository. Do not commit generated bindings. Fetch or check out the canonical schema during development and CI, then generate into build output or another ignored directory.

## Prepare one schema checkout

Clone Contracts outside the consumer repository and select an exact release tag:

```sh
export SOLTI_PROTO_DIR=/path/to/solti-proto
export SOLTI_PROTO_REF=YOUR_RELEASE_TAG

git clone https://github.com/soltiHQ/proto.git "$SOLTI_PROTO_DIR"
git -C "$SOLTI_PROTO_DIR" fetch --tags
git -C "$SOLTI_PROTO_DIR" checkout --detach "$SOLTI_PROTO_REF"
buf build "$SOLTI_PROTO_DIR"
```

`/path/to/solti-proto` and `YOUR_RELEASE_TAG` are placeholders. Replace them with an external checkout path and an existing release tag. CI should resolve the same immutable reference on every run.

Choose entry schemas by responsibility:

| Consumer | Entry schemas |
|---|---|
| Task client or server | `solti/task/v1/types.proto`, `solti/task/v1/api.proto` |
| Discovery client or server | `solti/agent/v1/types.proto`, `solti/discover/v1/discovery.proto` |
| Internal Raft persistence | `solti/agent/v1/types.proto`, `solti/raft/v1/raft.proto` |

Generating all three public packages together is valid. Generate `solti.raft.v1` only for a control-plane internal consumer.

## Generate Rust with prost and tonic

The Rust example uses [`prost`](https://crates.io/crates/prost), [`tonic`](https://crates.io/crates/tonic), [`tonic-prost`](https://crates.io/crates/tonic-prost), [`tonic-prost-build`](https://crates.io/crates/tonic-prost-build), and [`protoc-bin-vendored`](https://crates.io/crates/protoc-bin-vendored):

```sh
cargo add prost tonic tonic-prost
cargo add --build protoc-bin-vendored tonic-prost-build
```

Pin the selected crate versions through the consumer's dependency policy and `Cargo.lock`.

With `SOLTI_PROTO_DIR` set as shown above, this `build.rs` generates the public client packages into Cargo's `OUT_DIR`:

```rust
use std::{env, path::PathBuf};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("cargo:rerun-if-env-changed=SOLTI_PROTO_DIR");

    let root = PathBuf::from(env::var("SOLTI_PROTO_DIR")?);
    let protos = [
        root.join("solti/task/v1/types.proto"),
        root.join("solti/task/v1/api.proto"),
        root.join("solti/agent/v1/types.proto"),
        root.join("solti/discover/v1/discovery.proto"),
    ];

    for proto in &protos {
        println!("cargo:rerun-if-changed={}", proto.display());
    }

    let mut prost = tonic_prost_build::Config::new();
    prost.protoc_executable(protoc_bin_vendored::protoc_bin_path()?);

    tonic_prost_build::configure()
        .build_client(true)
        .build_server(false)
        .compile_with_config(prost, &protos, &[root])?;

    Ok(())
}
```

Use `.build_server(true)` in a crate that implements either service. A Task client and a discovery client need only generated client code.

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

The package names in `include_proto!` must match the declarations in the schema. Generated Rust module layout around those packages is consumer-owned.

## Generate Go with Buf

Keep a generation template in the Go consumer. Replace the module prefix with the real import path used by that consumer:

```yaml
version: v2
clean: true
managed:
  enabled: true
  override:
    - file_option: go_package_prefix
      value: example.com/your/module/gen
plugins:
  - remote: buf.build/protocolbuffers/go:v1.36.11
    out: gen
    opt:
      - paths=source_relative
  - remote: buf.build/grpc/go:v1.5.1
    out: gen
    opt:
      - paths=source_relative
```

The plugin versions above are exact generator versions, not Contracts versions. Review and update them independently.
See Buf's [`buf.gen.yaml` v2 reference](https://buf.build/docs/configuration/v2/buf-gen-yaml/) for the remote-plugin and managed-mode fields.

Generate the public packages from the tagged checkout:

```sh
buf generate "$SOLTI_PROTO_DIR" \
  --template buf.gen.solti.yaml \
  --path "$SOLTI_PROTO_DIR/solti/task/v1" \
  --path "$SOLTI_PROTO_DIR/solti/agent/v1" \
  --path "$SOLTI_PROTO_DIR/solti/discover/v1"
```

Add `gen/` to the consumer's ignore rules and run generation before compilation in local and CI workflows.
Run the consumer's normal Go dependency command, such as `go mod tidy`, after generation when the generated imports change.

For the internal control-plane package, use a separate target and include both its own directory and the imported agent package:

```sh
buf generate "$SOLTI_PROTO_DIR" \
  --template buf.gen.solti.yaml \
  --path "$SOLTI_PROTO_DIR/solti/agent/v1" \
  --path "$SOLTI_PROTO_DIR/solti/raft/v1"
```

Do not include the internal package in a public agent SDK.

## Make generation reproducible

Pin each independent input:

- the exact Contracts release;
- the generator and runtime versions;
- language-specific generation options;
- the generated package set.

Then regenerate from an empty output directory in CI and compile the consumer. A successful schema build proves that the `.proto` graph is valid. It does not prove that stale generated code in another directory matches the selected release.

The Rust calls used above are documented by the [`tonic-prost-build` crate](https://docs.rs/tonic-prost-build/latest/tonic_prost_build/) and its [`Builder`](https://docs.rs/tonic-prost-build/latest/tonic_prost_build/struct.Builder.html). The vendored compiler path comes from [`protoc_bin_path`](https://docs.rs/protoc-bin-vendored/latest/protoc_bin_vendored/fn.protoc_bin_path.html).
