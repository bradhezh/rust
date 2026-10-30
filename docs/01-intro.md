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
cargo init

cargo build
cargo build --release
cargo run
cargo run --release
cargo check
```

Crates are source-code packages managed by Cargo, typically compiled locally and statically linked rather than using shared libraries. Rust prioritises optimisation and does not guarantee a stable native ABI, although it can compile to shared libraries with a C-compatible ABI or use external libraries via its Foreign Function Interface (FFI). Note that the linker can discard unused code sections to prevent static linking from unnecessarily increasing binary size. Furthermore, Rust supports Link-Time Optimisation (LTO) across crate boundaries to improve performance and potentially reduce binary size through additional optimisations. Crates conventionally follow semantic versioning (_major_._minor_._patch_), where Cargo determines compatibility based on the leftmost non-zero version component. Before 1.0, the minor version normally acts as the compatibility boundary when the major version is zero, implying the lack of a stable API. Additionally, crates can be private and distributed via Git repositories or private registries.

### Cargo

```bash
# Sources are typically cached in ~/.cargo/registry/.
# Executable, compiled globally and installed to ~/.cargo/bin/
cargo install ripgrep --version ^14
cargo install --git https://github.com/bradhezh/my-crate-repo
cargo install --list
cargo uninstall ripgrep
# Libs for the project, compiled with individual projects into their target/
cargo add rand@0.10
# Update with version semantics — keeping the leftmost non-zero version No. for compatibility.
# Versions will be locked in Cargo.lock unless explicitly updated via this cmd or Cargo.toml.
cargo update
```

```toml title="Cargo.toml"
[package]
# Block `cargo publish` from uploading to crates.io
#publish = false
...
[dependencies]
# Default (^) version semantics
rand = "0.10.3"
# Only patches can update, even for post-1.0 versions.
#cfg-if = "~1.0.5"
# Nothing can update.
#rand = "=0.10.3"
my_crate_name = { git = "https://github.com/bradhezh/my-crate-repo", branch = "master" }
...
```
