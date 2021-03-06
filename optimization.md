# Optimizations

## Flamegraph

```shell
cargo install flamegraph

flamegraph <target binary> <call parameters>
```

## Profiling with perf + Firefox

```shell
perf record -g --call-graph dwarf ./my-binary
perf script -F +pid > ./my-file-output
```

Then visit profiler.firefox.com in an FF browser and load the file generated by the second command to see stack traces, timing diagrams, etc.

## Assembler code generated

Visit the [Godbolt compiler](https://godbolt.org) explorer site with the -O argument to turn on optimizations.

## Profiling compilation phases

`cargo build --release -Ztimings`

For a more detailed breakdown of where time is spent in the compiler.

`cargo rustc -- -Ztime-passes`

## Auto-vectorization techniques

TBD
https://www.nickwilcox.com/blog/autovec/ 
http://cliffle.com/p/dangerust/6/
https://github.com/novacrazy/numeric-array
https://llvm.org/docs/Vectorizers.html (LLVM doc)

## Unsafe cast

`std::mem::transmute` is a very versatile function in Rust. It converts between any two types as long as they’re the same size, without running any code or changing any data -- it simply reinterprets the bits of one type as the other.

## Compare performance of multiple command lines

See hyperfine

## Common optimizations 

* Enable LTO
* Use Jemalloc
* Try compile with RUSTFLAGS=’-C target-cpu=native’
* Use MaybeUnint
* Use different maps or hashers (aHash or FNV)
* Use tokio-uring or Glommio
* Multiple threads with individual tokio runtime
