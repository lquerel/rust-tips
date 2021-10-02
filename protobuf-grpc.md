# Protobuf and gRPC in Rust

## Prost
Prost is a crate implementing protobuf encode/decoding.
```shell
cargo add prost
cargo add prost-types   // if you need to use common protobuf types such as Struct, Timestamp, ...
```

### Create a Prost timestamp
```rust
let now = chrono::offset::Utc::now();
let timestamp = prost_types::Timestamp {
  seconds: now.timestamp(),
  nanos: now.timestamp_subsec_nanos() as i32,
};
```

### Build protobuf spec with Prost
TBD

## Tonic
Tonic is a crate implementing the gRPC protocol on top of HTTP/2 and Prost. 
```shell
cargo add tonic
```

### Build gRPC spec with Tonic
TBD

### Multiple instances of a gRPC service listening to the same port

Based on SO_REUSEPORT.

```rust
let service = YourGrpcService::default();

let addr: std::net::SocketAddr = "0.0.0.0:<port>".parse().unwrap();
let sock = socket2::Socket::new(
  match addr {
    SocketAddr::V4(_) => socket2::Domain::IPV4,
    SocketAddr::V6(_) => socket2::Domain::IPV6,
  },
  socket2::Type::STREAM,
  None,
).unwrap();

sock.set_reuse_address(true).unwrap();
sock.set_reuse_port(true).unwrap();
sock.set_nonblocking(true).unwrap();
sock.bind(&addr.into()).unwrap();
sock.listen(8192).unwrap();

let incoming = tokio_stream::wrappers::TcpListenerStream::new(TcpListener::from_std(sock.into()).unwrap());

Server::builder()
  .add_service(YourGrpcService::new(service))
  .serve_with_incoming(incoming)
  .await
  .unwrap();
```