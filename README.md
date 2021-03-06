# grpc-actix

A fast, actor-based [gRPC] implementation in Rust.

`grpc-actix` generates code for sending and receiving RPCs via gRPC. It also provides a high-level
server for building applications that respond to RPCs.

`grpc-actix` builds upon [`prost`] for generating Rust types from [Protocol Buffers] sources,
[`actix`] for the actor framework, and [`hyper`] for HTTP/2 client and server support.

## Usage

Add this to your `Cargo.toml`:

```toml
[dependencies.grpc-actix]
git = "https://github.com/tokenio/grpc-actix"
branch = "develop"
```

and this to your crate root:

```rust
extern crate grpc_actix;
```

Code generated by `grpc-actix` also depends on several other libraries. Crates that include
generated code will need to have the following in their `Cargo.toml`:

```toml
[dependencies]
actix = "0.7"
bytes = "0.4"
prost = "0.4"
prost-derive = "0.4"

# Needed only if client code generation is enabled.
http = "0.1"

# Needed only if server code generation is enabled.
futures = "0.1"
hyper = { git = "https://github.com/tokenio/hyper" }
```

and the following in their root:

```rust
extern crate actix;
extern crate bytes;
extern crate prost;
#[macro_use]
extern crate prost_derive;

// Needed only if client code generation is enabled.
extern crate http;

// Needed only if server code generation is enabled.
extern crate futures;
extern crate hyper;
```

## Code Generation

`grpc-actix` by itself only provides the support code for Protocol Buffers and gRPC types, including
shared client and server code. The `grpc-actix-build` crate contains support for generating Rust
code from `.proto` files in conjunction with the `prost-build` crate.

The easiest way to generate code is to use a custom build script. To do so, add this to your
`Cargo.toml`:

```toml
[build-dependencies]
prost-build = "0.4"

[build-dependencies.grpc-actix-build]
git = "https://github.com/tokenio/grpc-actix"
branch = "develop"
```

and this to your `build.rs` script:

```rust
extern crate grpc_actix_build;
extern crate prost_build;
```

## Client Usage

Client calls are done by sending `actix` messages to a client actor. Client actors are created by
`grpc-actix-build` for each service.

The following illustrates basic setup of the `actix` runtime, spawning a client actor, and issuing
an RPC request:

```rust
extern crate actix;
extern crate bytes;
extern crate http;

// "example" contains the generated protobuf/gRPC code for a service named
// "Service".
mod example;

use actix::prelude::*;
use http::uri;

use bytes::Bytes;

// Client actors are named after their service name with the suffix "Client"
// added. "Request" is the input message type used by our example RPC.
use example::{service, ServiceClient, Request};

// Start actix.
let sys = System::new("example");

// Start the client actor in the current thread.
let address = Bytes::from("192.168.1.100");
let client = ServiceClient::new(uri::Scheme::HTTP, uri::Authority::from_shared(address).unwrap());
let client_addr = client.start();

// Create a future for issuing the RPC request. "example_call" is the name of
// the RPC in the .proto file; `grpc-actix-build` will export an actor message
// type called "ExampleCall" in a submodule named after the service.
let request = Request {};
let result = client_addr.send(service::ExampleCall::from(request));

// Spawn the future.
Arbiter::spawn(
    result
        .map(|res| {
            match res {
                Ok(_result) => {
                    println!("Received result message.");
                }
                Err(e) => {
                    eprintln!("Received error status: {}", e);
                }
            }
        })
        .map_err(|e| {
            eprintln!("Failed to send actor message.");
        })
);

sys.run();
```

## Server Usage

`grpc-actix` provides a multithreaded server that can dispatch incoming RPCs to user-defined actors.

```rust
extern crate actix;
extern crate grpc_actix;
extern crate num_cpus;

// "example" contains the generated protobuf/gRPC code for a service named
// "Service".
mod example;

use actix::prelude::*;

use grpc_actix::{Status, UnaryResponse};

// "Request" is the input message type used by our example RPC, and "Response"
// is the output message type.
use example::{service, ServiceDispatch, Request, Response};

// Actor type for handling "Service" messages.
struct ServiceServer;

impl Actor for ServiceServer {
    type Context = Context<Self>;
}

impl Handler<service::ExampleCall> for ServiceServer {
    type Result = Result<UnaryResponse<Response>, Status>;

    fn handle(&mut self, _msg: service::ExampleCall, _ctx: &mut Self::Context) -> Self::Result {
        let response = Response {};
        Ok(response.into())
    }
}

let mut sys = System::new("example");
let server = Server::spawn(|| ServiceServer, num_cpus::get())
    .wait()
    .expect("failed to start gRPC server");
sys.block_on(server.bind(([0, 0, 0, 0], 50051).into(), |mut service| {
    ServiceDispatch::add_to_service(&mut service);
})).expect("server error");
sys.run();
```

[`actix`]: https://actix.rs/
[gRPC]: https://grpc.io/
[`hyper`]: https://hyper.rs/
[`prost`]: https://github.com/danburkert/prost
[Protocol Buffers]: https://developers.google.com/protocol-buffers/
