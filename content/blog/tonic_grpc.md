+++
title = "gRPC with Rust and Tonic"
date = "2024-02-17"
[taxonomies]
tags = ["rust", "grpc", "gnmi"]
+++
# Building a gRPC client with Rust
Rust is a language that I am very fond of. I like its expressiveness and the awesome typesystem, giving me a feeling of confidence in my code that is umatched by other languages by now. I also came across a gRPC-Service for network devices called [gNMI](https://www.openconfig.net/docs/gnmi/gnmi-specification/) when doing network related stuff in university. I thought it would be pretty cool to go and see how one could implement a client for it in Rust.

## Rusts gRPC Ecosystem
Rusts support for gRPC is currently in a very good state. There is an awesome crate called [tonic](https://github.com/hyperium/tonic) which provides a pretty solid implementation of gRPC over HTTP/2. Tonic also provides means to generate types from a `.proto` file via a buildscript and macros. While I was reading that, I thought that this could be little wonky, especially since the protobuf compiler does not support emitting rust out of the box. But I was quickly proven wrong by how easily I got a working example up and running. All I had to do was write a simple buildscript that looks in a folder called `proto`, tell it which files I need compiled and then include them via the `include_proto!` macro:

```rust
fn main() {
    let proto_dir = "proto";
    println!("cargo:rerun-if-changed={}", proto_dir);

    tonic_build::configure()
        .build_server(false)
        .compile_well_known_types(true)
        .compile(
            &[
                "proto/gnmi/gnmi.proto",
                "proto/gnmi_ext/gnmi_ext.proto",
                "proto/target/target.proto",
                "proto/collector/collector.proto",
                "proto/google.proto",
            ],
            &[proto_dir],
        ).expect("Failed to compile protobuf files");
}
```

```rust
mod gen {
    pub mod gnmi {
        tonic::include_proto!("gnmi");
    }

    pub mod gnmi_ext {
        tonic::include_proto!("gnmi_ext");
    }

    pub mod target {
        tonic::include_proto!("target");
    }

    pub mod google {
        pub mod protobuf {
            tonic::include_proto!("google.protobuf");
        }
    }
}
```

## Calling a gRPC service

With the code above, I had all type and service definitions available in the `gen` module. I had a few hiccups early regarding built in types in protobuf files, but the error messages were really helpful and I got it figured out. I then wrote asmall wrapper for the generated Client type that allows a user to nicely configure settings such as tls and credentials for authentication. The final code looked like this:

```rust
let mut client = Client::builder("https://test.com")
        .credentials("test", "pass")
        .build()
        .await
        .unwrap();

    let cap = client.capabilities().await;
```

This retrieves the supported gNMI capabilities from a target network device. I think it looks really clean and the effort to get here was pretty minimal. I think I will continue working on it, you can find the repository with all the code on my [GitHub](https://github.com/Lachstec/ginmi)

## Conclusion
Rusts support for gRPC is, as far as I can tell, pretty great and everything worked fine up until now. Don't let the pretty low version numbers on gRPC related crates deter you from trying it out. Whether or not it is ready for production use I can not tell, but for small hobby project, I can only recommend giving Rust and Tonic a try!
