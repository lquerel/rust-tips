# Memory management and optimization

## Measure the size of a scalar value, struct, or array
This approach doesn't follow references.

```rust
struct Container<T> {
    items: [T; 3],
}

fn main() {
    use std::mem::size_of_val;

    let cu8 = Container {
        items: [1u8, 2u8, 3u8],
    };
		println!("size of cu8 = {} bytes", size_of_val(&cu8));  // returns 3

    let cu32 = Container {
        items: [1u32, 2u32, 3u32],
    };
    println!("size of cu32 = {} bytes", size_of_val(&cu32));  // returns 12
}
```

## Measure memory consumption programmatically
This will compile and run on Linux.
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

## Use better memory allocators
These days one of the best memory allocator is [mimalloc](https://github.com/microsoft/mimalloc) (Microsoft) followed by glibc 2.31 
and [jemalloc](https://github.com/jemalloc/jemalloc) (Facebook) according to this [test](https://www.linkedin.com/pulse/testing-alternative-c-memory-allocators-pt-2-musl-mystery-gomes/) and this [one](https://www.linkedin.com/pulse/linux-testing-alternative-c-memory-allocators-emerson-gomes/). 

> ToDo: Find more recent comparison tests.

## Change the default Musl memory allocator
The default Musl memory allocator is super slow according to this [test](https://www.linkedin.com/pulse/testing-alternative-c-memory-allocators-pt-2-musl-mystery-gomes/).
Better to use [mimalloc](https://github.com/microsoft/mimalloc) (Microsoft) or [jemalloc](https://github.com/jemalloc/jemalloc) (Facebook).

See ripgrep for an example.
