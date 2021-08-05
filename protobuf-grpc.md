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

## Tonix
Tonic is a crate implementing the gRPC protocol on top of HTTP/2 and Prost. 
```shell
cargo add tonic
```

### Build gRPC spec with Tonic
TBD
