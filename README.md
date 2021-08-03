# Rust snippets and tips

## Links to other pages
* [Memory management and optimization](memory.md)
* [Linux distributions](linux-distributions.md)

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

### Detect unused crates
```shell
cargo install cargo-udeps --locked
cargo +nightly udeps
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

To keep strace outputs.
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

To stop a program right before it exits
```shell
(gdb) catch syscall exit exit_group
(gdb) r
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
    if std::env::var("RUST_BACKTRACE").is_err() {
        std::env::set_var("RUST_BACKTRACE", "1")
    }
    color_eyre::install()?;

    Ok(())
}
```

* If you want panics and errors to both have backtraces, set RUST_BACKTRACE=1;
* If you want only errors to have backtraces, set RUST_LIB_BACKTRACE=1;
* If you want only panics to have backtraces, set RUST_BACKTRACE=1 and RUST_LIB_BACKTRACE=0.

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

## Compiler explorer
### Generate assembler from a Rust program
```
https://rust.godbolt.org
```

## Iterator
### Chain multiple iterators
```rust
let a = 0..3;
let b = 3..6;
let c = 6..9;
let all = a.chain(b).chain(c);
```

## Lifetime
### Lifetime on generic type parameter 
```rust
pub fn values<'a>(&'a self) -> Box<Iterator<Item = &i32> + 'a> {
    // ...
}
```
