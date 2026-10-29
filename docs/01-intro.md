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
