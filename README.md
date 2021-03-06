## SideFuzz: Fuzzing for side-channel vulnerabilities

[![docs](https://docs.rs/sidefuzz/badge.svg)](https://docs.rs/sidefuzz)
[![crates.io](https://meritbadge.herokuapp.com/sidefuzz)](https://crates.io/crates/sidefuzz)
[![docs](https://docs.rs/sidefuzz/badge.svg)](https://docs.rs/sidefuzz)
[![patreon](https://img.shields.io/badge/patreon-donate-green.svg)](https://patreon.com/phayes)
[![flattr](https://img.shields.io/badge/flattr-donate-green.svg)](https://flattr.com/@phayes)
<a href="https://github.com/taibangle/awesome-china"><img src="https://raw.githubusercontent.com/phayes/awesome-china/master/badges/awesome-china.png" width="98"></a>
<a href="https://github.com/taibangle/awesome-china"><img src="https://raw.githubusercontent.com/phayes/awesome-china/master/badges/很棒-中国.png" width="66"></a>

SideFuzz is an adaptive fuzzer that uses a genetic-algorithm optimizer in combination with t-statistics to find side-channel (timing) vulnerabilities in cryptography compiled to [wasm](https://webassembly.org).

Fuzzing Targets can be found here: https://github.com/phayes/sidefuzz-targets

### How it works

SideFuzz works by counting instructions executed in the [wasmi](https://github.com/paritytech/wasmi) wasm interpreter. It works in two phases:

**Phase 1.** Uses a genetic-algorithm optimizer that tries to maximize the difference in instructions executed between two different inputs. It will continue optimizing until subsequent generations of input-pairs no longer produce any meaningful differences in the number of instructions executed. This means that it will optimize until it finds finds a local optimum in the fitness of input pairs.

**Phase 2.** Once a local optimum is found, the leading input-pairs are sampled until either:

- A large t-statistic (p = 0.001) is found, indicating that there is a statistically significant difference in running-time between the two inputs. This is indicative of a timing side-channel vulnerability; or

- The t-statistic stays low, even after significant sampling. In this case the candidate input pairs are rejected and SideFuzz returns to phase 1, resuming the genetic-algorithm optimizer to find another local optimum.

### What it gets you

Fuzzing with SideFuzz shows that your Rust code can be constant-time, but doesn't show that it _is_ constant-time on all architectures. This is because LLVM backends [can and will](http://www.reparaz.net/oscar/misc/cmov.html) ruin constant-time Rust / LLVM-IR when compiling to machine-code. SideFuzz should be considered a "good first step" to be followed up with [dudect-bencher](https://crates.io/crates/dudect-bencher) and [ctgrind](https://github.com/RustCrypto/utils/tree/master/ctgrind). It should also be noted that proper compiler support for constant-time code-generation is an unsolved problem in the Rust ecosystem. There have been some ideas around using [cranelift](https://github.com/CraneStation/cranelift) for constant-time code generation, but things are still in the brainstorming phase.

## Installation

```
rustup target add wasm32-unknown-unknown
git clone git@github.com:phayes/sidefuzz.git
cd sidefuzz && cargo install --path .
```

(Cannot currently do `cargo install sidefuzz` because of [this issue](https://github.com/phayes/sidefuzz/issues/12))

## Creating a Rust fuzz target

Creating a target in rust is very easy.

```rust
// lib.rs
#[no_mangle]
pub extern "C" fn fuzz() {
  let input = sidefuzz::fetch_input(32); // 32 bytes of of fuzzing input as a &[u8]
  sidefuzz::black_box(my_hopefully_constant_fn(input));
}
```

```toml
# Cargo.toml
[lib]
crate-type = ["cdylib"]

[dependencies]
sidefuzz = "0.1.1"
```

Compile and fuzz the target like so:

```bash
cargo build --release --target wasm32-unknown-unknown                # Always build in release mode
sidefuzz fuzz ./target/wasm32-unknown-unknown/release/my_target.wasm # Fuzzing!
```

Results can be checked like so:

```bash
sidefuzz check my_target.wasm 01250bf9 ff81f7b3
```

When fixing variable-time code, sidefuzz can also help with `sidefuzz count` to quickly count the number of instructions executed by the target.

```bash
sidefuzz count my_target.wasm 01250bf9
```

## Creating a fuzz target in other languages

SideFuzz works with Go, C, C++ and other langauges that compile to wasm.

The wasm module should provide four exports:

1. Memory exported to "memory"

2. A function named "fuzz". This function will be repeatedly called during the fuzzing process.

3. A function named "input_pointer" that returns an i32 pointer to a location in linear memory where we can can write an array of input bytes. The "fuzz" function should read this array of bytes as input for it's fuzzing.

4. A function named "input_len" that returns an i32 with the desired length of input in bytes.

## FAQ

#### 1. Why wasm?

Web Assembly allows us to precisely track the number of instructions executed, the type of instructions executed, and the amount of memory used. This is much more precise than other methods such as tracking wall-time or counting CPU cycles.

#### 2. Why do I always need to build in release mode?

Many constant-time functions include calls to variable-time `debug_assert!()` functions that get removed during a release build. Rust's and LLVM optimizer may also mangle supposedly constant-time code in the name of optimization, introducing subtle timing vulnerabilities. Running in release mode let's us surface these issues.

#### 3. I need an RNG (Random Number Generator). What do?

You should make use of a PRNG with a static seed. While this is a bad idea for production code, it's great for fuzzing. See the [rsa_encrypt_pkcs1v15_message target](https://github.com/phayes/sidefuzz-targets/blob/master/rsa_encrypt_pkcs1v15_message/src/lib.rs) for an example on how to do this.

#### 4. What's up with `black_box` ?

`sidefuzz::black_box` is used to avoid dead-code elimination. Because we are interested in exercising the fuzzed code instead of getting results from it, the exported `fuzz` function doesn't return anything. The Rust optimizer sees all functions that don't return as dead-code and will try to eliminate them as part of it's optimizations. `black_box` is a function that is opaque to the optimizer, allowing us to exercise functions that don't return without them being optimized away. It should be used whenever calling a function that doesn't return anything or where we are ignoring the output returned.

#### 5. The fuzzer gave me invalid inputs, what now?

You should panic (causing a wasm trap). This will signal to the fuzzer that the inputs are invalid.

#### 6. I need to do some variable-time set-up. How do I do that?

You should use [`lazy_static`](https://crates.io/crates/lazy_static) to do any set-up work (like generating keys etc). The target is always run once to prime lazy statics before the real fuzzing starts.

## Related Tools

1. `dudect-bencher`. An implementation of the DudeCT constant-time function tester. In comparison to SideFuzz, this tool more closely adheres to the original dudect design. https://crates.io/crates/dudect-bencher

2. `ctgrind`. Tool for checking that functions are constant time using Valgrind. https://github.com/RustCrypto/utils/tree/master/ctgrind

## Further Reading

1. "DifFuzz: Differential Fuzzing for Side-Channel Analysis", Nilizadeh, Noller, Păsăreanu.
   https://arxiv.org/abs/1811.07005

2. "Dude, is my code constant time?", Reparaz et al. https://eprint.iacr.org/2016/1123.pdf

3. "Rust, dudect and constant-time crypto in debug mode", brycx.
   https://brycx.github.io/2019/04/21/rust-dudect-constant-time-crypto.html

## Contributors

1.  Patrick Hayes ([linkedin](https://www.linkedin.com/in/patrickdhayes/)) ([github](https://github.com/phayes)) - Available for hire.
