# Rust snippets and tips

## Links to other pages
* [Memory management and optimization](memory.md)
* [Linux distributions](linux-distributions.md)
* [Protobuf and gRPC](protobuf-grpc.md)

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

### List the dependency graph of a crate
First install ‘cargo-tree’ and use one of the following commands depending on the use case.  
```
cargo install cargo-tree
cargo tree
cargo tree -p <sub-crate>
cargo tree —features serde_json 
```

### Dependency resolution (summary of this [page](https://doc.rust-lang.org/cargo/reference/resolver.html))
Rule 1: versions are considered compatible if their left-most *non-zero* major/minor/patch component is the same. 
Rule 2: for versions with leading zeros (could be on multiple levels), the rule 1 is applied once the zeros are removed.

Some examples to illustrate these rules:
* 1.0.1 and 1.3.4 are considered compatible. 
* 0.1.0 and 0.1.2 are considered compatible.
* 0.1.0 and 0.2.0 are not compatible.
* 0.0.1 and 0.0.2 are not compatible.

When multiple packages specify a dependency for a common package, the resolver attempts to ensure that they use the same version of that common package, as long as they are within a SemVer compatibility range. It also attempts to use the greatest version currently available within that compatibility range. 

If multiple packages have a common dependency with semver-incompatible versions, then Cargo will allow this, but will build two separate copies of the dependency. 

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
