# Serde

## Read line-delimited JSON
```shell
cargo add serde serde_json
```

```rust
use std::io::BufReader;
use std::fs::File;

let reader = BufReader::new(File::open("file.json").unwrap());
let your_type_stream = serde_json::Deserializer::from_reader(reader).into_iter::<YourType>();
```
