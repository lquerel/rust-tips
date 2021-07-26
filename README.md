# Rust snippets and tips

## Cargo
### Add crates with the command line
```shell
cargo install cargo-edit
cargo add <crate_name>
```
To specify a specific crate version and some features
```shell
cargo add tokio@1.9.0 --features full
```

## Error handling
### To nicely report errors in an "application"
```shell
cargo add color-eyre
```
In you main.rs file.
```rust
use color_eyre::Report;

fn main() -> Result<(), Report> {
    init()?;

    // Some code

    Ok(())
}

fn init() -> Result<(), Report> {
    if std::env::var("RUST_LIB_BACKTRACE").is_err() {
        std::env::set_var("RUST_LIB_BACKTRACE", "1")
    }
    color_eyre::install()?;

    Ok(())
}
```

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
