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

To remove a crate.
```shell
cargo rm <crate>
```

## Debugging
### Configure log level and more
To set the default log level to info and set the log level to debug for a specific crate.
```shell
RUST_LOG=info,<specific_crate>=debug cargo run
```

### Trace all the socket connections
```shell
cargo build && strace -e 'connect' ./target/debug/<your_app>
```

To only keep strace outputs.
```shell
cargo build && strace -e 'connect' ./target/debug/<your_app> > /dev/null
```

To trace all the connections form the children (threads)
```shell
cargo build --quiet --release && strace -f -e 'connect' ./target/release/<your_app>
```

### Debug a segfault with GDB
```shell
cargo build && gdb --quiet --args ./target/debug/<your_app>
```

### Set a breakpoint on a specific syscall
In the following example we set a breakpoint on the system call "connect" (i.e. socket connect).
```shell
cargo build --quiet && gdb --quiet --args ./target/debug/<your_app>
(gdb) catch syscall connect
(gdb) r
Starting your app
...
<stop on the cathpoint>
(gdb) c
... continue
...
(gdb) info threads
... display info on threads
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

### To report structured logs
```shell
cargo add tracing tracing-subscriber
```
```rust
use color_eyre::Report;
use tracing::info;
use tracing_subscriber::EnvFilter;

fn main() -> Result<(), Report> {
    init()?;

    info!("Hello");

    let param1 = "First parameter";
    let param2 = "Second parameter";

    // % --> Display
    // ? --> Debug
    info!(%param1, param2 = ?param2, "An additional message");
    
    Ok(())
}

fn init() -> Result<(), Report> {
    if std::env::var("RUST_LIB_BACKTRACE").is_err() {
        std::env::set_var("RUST_LIB_BACKTRACE", "1")
    }
    color_eyre::install()?;

    if std::env::var("RUST_LOG").is_err() {
        std::env::set_var("RUST_LOG", "info")
    }
    tracing_subscriber::fmt::fmt()
        .with_env_filter(EnvFilter::from_default_env())
        .init();

    Ok(())
}
```

### To enable backtrace
```shell
RUST_BACKTRACE=1 cargo run
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
