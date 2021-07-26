# Rust snippets and tips

## Memory
### Mesure memory consumption programmatically
Works on Linux.
```shell
cargo add jemallocator
cargo add jemalloc-ctl
```

In your main.rs file add the following global allocator declaration.
```rust
#[cfg(not(target_env = "msvc"))]
#[global_allocator]
static ALLOC: jemallocator::Jemalloc = jemallocator::Jemalloc;
```

In the rust file where you want to measure the memory consumption.
```rust
use jemalloc_ctl::{stats, epoch};

// ...
let epoch = epoch::mib().unwrap();
let allocated_mib = stats::allocated::mib().unwrap();

let allocated_before = allocated_mib.read().unwrap();
let _buf = vec![0i64; 16];
epoch.advance().unwrap();
let allocated_after = allocated_mib.read().unwrap();
println!("{} bytes allocated", allocated_after - allocated_before);
```
