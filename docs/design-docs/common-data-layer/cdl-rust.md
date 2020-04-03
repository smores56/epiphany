# Epiphany Common Data Layer (Rust implementation) design document

Affected version: N/A


## 1. Goals

Implement the Common Data Layer (An intermediate data gateway connecting 
resource requests to their appropriate backend in the appropriate format) 
in the [Rust][rust] language.


## 2. Input Consumption

The suggested allowed input channels to the CDL are [Kafka][kafka], [gRPC][gRPC], and a
[REST API][rest]. Their interaction with the Rust language are as follows:

### 2.1 Kafka

[Kafka][kafka], the distributed streaming platform, has bindings already to the Rust
language, found [here][kafka rust]. The crate (the name for Rust libraries, see appendix)
is a client for currently-deployed Kakfa instances, providing functionality for the Producer
API, the Consumer API, and the Client API for Kafka. Once Kafka has been deployed, the
following code (taken from the crate documentation) could consume a message stream:

```rust
use kafka::consumer::{Consumer, FetchOffset, GroupOffsetStorage};

// type-level mutability, everything is default immutable in Rust
let mut consumer =
   Consumer::from_hosts(vec!("localhost:9092".to_owned()))
      .with_topic_partitions("my-topic".to_owned(), &[0, 1])
      .with_fallback_offset(FetchOffset::Earliest)
      .with_group("my-group".to_owned())
      .with_offset_storage(GroupOffsetStorage::Kafka)
      .create()
      .unwrap(); // `unwrap`s should be removed in production code

loop {
  for ms in consumer.poll().unwrap().iter() {
    for m in ms.messages() {
      println!("{:?}", m); // debug print the message
    }
    consumer.consume_messageset(ms);
  }
  consumer.commit_consumed().unwrap();
}
```

### 2.1 gRPC

[gRPC][gRPC], the remote procedure call framework, traditionally integrates into an
environment by defining a [schema file][protobuf schema file] of the format for
communication, and then generating code in the target language at compile time
(or runtime for some dynamic languages, like Python). For Rust, there are multiple
implementations of the gRPC protocol, but the [tonic][tonic] library is likely the best
due to its:

- popularity
- async implementation
- great documentation
- clean, type-safe implementation (with macros)

The [helloworld tutorial][tonic helloworld] comprehensively covers setup of the rust codebase
and the relevant `.proto` file. Here is a sample of the `helloworld` gRPC example:

```rust
use tonic::{transport::Server, Request, Response, Status};

use hello_world::greeter_server::{Greeter, GreeterServer};
use hello_world::{HelloReply, HelloRequest};

pub mod hello_world {
    // this macro makes the helloworld gRPC available in Rust
    tonic::include_proto!("helloworld");
}

#[derive(Debug, Default)]
pub struct MyGreeter {}

#[tonic::async_trait]
impl Greeter for MyGreeter {
    // this can be run asyncronously (in line with RPC philosophy)
    async fn say_hello(
        &self,
        request: Request<HelloRequest>,
    ) -> Result<Response<HelloReply>, Status> {
        println!("Got a request: {:?}", request);

        // simple, automatically generated types for use with gRPC
        let reply = hello_world::HelloReply {
            message: format!("Hello {}!", request.into_inner().name).into(),
        };

        Ok(Response::new(reply))
    }
}
```

### 2.3 REST API

The most well-defined of the potential entry points to the CDL, a [REST API][rest] can be
built with any one of many options online. The most popular options are:

1) [Actix Web][actix] (blazingly fast)
2) [Rocket][rocket] (macro heavy, non-async, but very concise)
3) [Warp][warp] (clean, fast, slight issue with error-handling)

Please visit each individual framework above for README examples, and if you would like
to see even more frameworks, please visit the [Are We Web Yet?][are we web yet] page for
longer list of popular web frameworks. The most popular framework is definitely
[Actix Web][actix], which is commonly among the best in
[speed benchmark comparisons][actix benchmark] among popular frameworks in all languages.

_Note: the issue with error handling in Warp is that, unlike most Rust code, it throws_
_error handling up to the root of the application, which means that your application_
_will compile if it doesn't handle a type of error, but your users will simply get a_
_blank 500 response instead of an appropriate error. Otherwise, the framework is fast_
_and very nice to use, but it feels orthogonal to how Rust usually handles errors._

### 2.4 Putting it all together

Most modern web frameworks make the assumption that they are the only thing receiving input
from the bound port, so they consume all input from the network. The same holds true for
Rust, where virtually all frameworks, included the ones listed above, assume that they will
be the only listening process (a reasonable assumption in most use cases).

Since we need to take in input from Kafka, gRPC, and REST, we have two options:

Method                                     | Pros
-------------------------------------------|------
Spawn multiple threads, one for each       | Deployment is easier, single deployment
Use multiple binaries (see Appendix A.1.3) | Distributes resources more naturally, can deploy multiple instances


## 3 Communication with the Schema Registry

The Schema Registry maintains a mutable list of possible formats that data can be provided
to the CDL in. The registry will be an in-memory store using (for now)
[Apache Ignite][apache ignite], but potentially a different tool like [Redis][redis] or
[Memcached][memcached]. The meta-format of the legal formats for data is yet to be determined,
but will generally take the form of a key tagging the format, an input format (from the request), and an output format (to send to storage).

### 3.1 Store Interaction

As mentioned, the current plan for an in-memory store is to use Apache Ignite, but
to no fault of its own, the current open-source bindings to the Apache Ignite (Thin) Client
are incomplete, and thus definitely not robust. The incomplete client can be found
[here][ignite rust], and could be forked and completed, but would require trusting a partial
implementation. Alternatively, a ground-up minimum work-required implementation could be
undertaken, but would require a not insignificant amount of work.

If we continue with this implementation, the best course of action is likely to bind via
C-bindings to the C++ provided thin client of Apache Ignite. There are multiple provided
clients from Apache, but the C++ client has "no runtime" (unlike the Java or Go clients),
and bindings in C can be auto-generated with one of many Rust libraries.


### 3.2 Alternative Stores

However, for both Redis and Memcached, there exist well-defined bindings:
[redis for rust][redis rust] and [memcache for rust][memcache rust].


## A Appendix

### A.1 Rust Tips

1) For finding crates (the Rust version of a library) for any given purpose, the first place
to check is [crates.io][crates.io], the front-end of the centralized package repository
of the Rust language.

2) `build.rs` is a construct for running code during the compilation of a Rust program
or library. More information can be found [here][build.rs] on the subject.

3) Rust supports a project containing multiple binaries, you can read more on that
[here][multiple binaries]. This allows common code to be used between the binaries.

[rust]: https://www.rust-lang.org/
[gRPC]: https://grpc.io/
[kafka]: https://kafka.apache.org/
[rest]: https://en.wikipedia.org/wiki/Representational_state_transfer
[kafka rust]: https://docs.rs/crate/kafka/0.8.0
[protobuf schema file]: https://grpc.io/docs/guides/#working-with-protocol-buffers
[tonic]: https://github.com/hyperium/tonic
[tonic helloworld]: https://github.com/hyperium/tonic/blob/master/examples/helloworld-tutorial.md
[actix]: https://github.com/actix/actix-web
[warp]: https://github.com/seanmonstar/warp
[rocket]: https://github.com/SergioBenitez/Rocket
[crates.io]: https://crates.io/
[build.rs]: https://doc.rust-lang.org/cargo/reference/build-scripts.html
[multiple binaries]: https://thefullsnack.com/en/multiple-binaries-rust.html
[are we web yet]: http://www.arewewebyet.org/topics/frameworks/
[actix benchmark]: https://www.techempower.com/benchmarks/#section=data-r18&hw=ph&test=query
[apache ignite]: https://ignite.apache.org/
[redis]: https://redis.io/
[memcached]: https://memcached.org/
[redis rust]: https://docs.rs/crate/redis/0.15.1
[memcache rust]: https://docs.rs/crate/memcache/0.14.0
[ignite rust]: https://github.com/isapego/ignite-rust
[ignite cpp]: https://apacheignite-cpp.readme.io/docs/thin-client
