---
title: Chapter 1. Introduction
slug: /
---

Rust aims to improve on traditional systems-level languages such as C/C++. It offers the same low-level control and high performance, but adds strong safety guarantees. It introduces an ownership and borrowing model that allows the compiler to check and enforce memory safety and data-race freedom at compile time, without the runtime cost of garbage collection. Rust also includes various high-level features common in modern languages, such as type inference, closures, iterators, async/await, and a package and build system. Rust follows the principle of zero-cost abstractions, meaning that higher-level features compile to lower-level code that runs as fast as equivalent code written by hand. With its combination of performance, safety, and expressiveness, Rust is well suited to both systems-level programming and application-level development.

### Quick Start

```bash
# Installed in ~/.rustup and ~/.cargo
curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh
# Add this to ~/.bashrc
. "$HOME/.cargo/env"

rustc --version
rustup update
rustup doc
rustup self uninstall

rustup component list
# LSP, unnecessary for Zed
rustup component add rust-analyzer

rustc main.rs

# Build system and package (crate) manager, also generate .git and .gitignore if not in any repo
cargo new hello_world
cd hello_world
# Or in the CWD
mkdir src
mv main.rs src
cargo init --name hello_world

cargo build
cargo build --release
cargo run
cargo run --release
cargo check
```

Crates are source-code packages managed by Cargo, typically compiled locally and statically linked rather than using shared libraries. Rust prioritises optimisation and does not guarantee a stable native ABI, although it can compile to shared libraries with a C-compatible ABI or use external libraries via its Foreign Function Interface (FFI). Note that the linker can discard unused code sections to prevent static linking from unnecessarily increasing binary size. Furthermore, Rust supports Link-Time Optimisation (LTO) across crate boundaries to improve performance and potentially reduce binary size through additional optimisations. Additionally, crates can be private and distributed via Git repositories or private registries.

Crates conventionally follow semantic versioning (_major_._minor_._patch_), where Cargo determines compatibility based on the leftmost non-zero version component. Before 1.0, the minor version normally acts as the compatibility boundary when the major version zero implies the lack of a stable API.

Dependency versions are declared in `Cargo.toml` as requirements and locked in `Cargo.lock` for building, and the latter will be updated if the former changes. The lockfile can also be updated by `cargo update`, which is also based on `Cargo.toml` but can update dependencies to the latest compatible versions without changing `Cargo.toml`. Locking dependency versions supports reproducible builds, but note that `Cargo.lock` is only used when the crate is built as the root (not as a dependency). This is a trade-off because a crate should not force downstream applications to use its exact dependency versions. Consequently, reproducibility does not mean that applications reproduce all the same dependency versions that their dependencies used during development and testing. Following semantic versioning conventions is therefore crucial for Cargo to manage dependency updates and compatibility correctly.

### Cargo

```bash
# Sources are typically cached in ~/.cargo/registry/.

# Executable, compiled globally and installed to ~/.cargo/bin/
cargo install ripgrep --version ^14
cargo install --git https://github.com/bradhezh/my-crate-repo
cargo install --list
cargo uninstall ripgrep

# Libs for the project, compiled with individual projects into their target/

# Update Cargo.toml and Cargo.lock, as well as fetch and cache sources
# --dev: For testing, benchmarking, and examples, like test frameworks and assertion libs, ignored
#   by consumers
# --build: For building, usually required by build.rs, like C compilers and pkgconfig, also used by
#   consumers to build this as their dependencies
# --git: From a git repo
# --path: From a local path
cargo add rand@0.10

# Update Cargo.toml and Cargo.lock, but remain cached
cargo remove rand
# Update Cargo.lock only
cargo update

# Documentation including all dependencies
cargo doc --open
```

```toml title="Cargo.toml"
[package]
name = "hello_world"
version = "0.1.0"
edition = "2024"
# Block `cargo publish` from uploading to Crates.io
#publish = false

[dependencies]
# Default (^) semantics: the latest compatible version (>= 0.10.3 and < 0.11.0 here)
rand = "0.10.3"
# The exact version
#rand = "=0.10.3"
# With the exact major and minor but the latest patch version
#cfg-if = "~1.0.5"
#my_crate_name = { git = "https://github.com/bradhezh/my-crate-repo", branch = "master" }
...
```

Apart from its unique memory management model, Rust features distinct approaches to concepts common to almost all programming languages. For example, variables are immutable by default, which is particularly beneficial for concurrency. The language minimises undefined behaviour, requiring programmers to be explicit about their intent.

Data types have fixed widths (e.g. `i32` rather than `int`), and numbers have definite overflow behaviour, while allowing programmers to explicitly specify how to handle integer overflows. Rust also makes a clear distinction between expressions and statements, preventing C-style chained assignments (e.g. `x = y = 5`). Expressions implicitly return an empty tuple `()` by default, which is referred to as the unit value and type. In fact, blocks, along with control flow constructs such as `if`/`else`, `match`, `loop`, `while`, and `for`, are all expressions. Rust also supports variable shadowing, allowing for more expressive code without unnecessary variable renaming.

Rust enforces strict static typing and disallows implicit type coercion. Array access is checked at runtime, with a predictable panic for out-of-bounds memory access, instead of the undefined behaviour typical of C. Moreover, Rust macros operate directly on the Abstract Syntax Tree (AST) rather than simple textual replacement, making metaprogramming significantly safer and more robust.

### Guessing Game

```rust title="src/main.rs"
use std::cmp::Ordering;
use std::io;

use rand::prelude::*;

fn main() {
  let secret = rand::rng().random_range(1..=100);
  loop {
    println!("Please input your guess.");
    let mut guess = String::new();
    io::stdin()
      .read_line(&mut guess)
      .expect("Failed to read line");
    let guess: u32 = match guess.trim().parse() {
      // Note that `=>` is a separator between a pattern and an arm. A closure is like `|x| x + 1`.
      Err(_) => continue,
      Ok(n) => n,
    };
    println!("Your guess: {guess}");

    match guess.cmp(&secret) {
      Ordering::Equal => {
        println!("You win!");
        break;
      }
      Ordering::Less => println!("Too small!"),
      Ordering::Greater => println!("Too big!"),
    }
  }
}
```
