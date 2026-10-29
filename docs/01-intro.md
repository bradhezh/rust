---
title: Chapter 1. Introduction
slug: /
---

Rust aims to improve on traditional systems-level languages like C/C++. It offers the same low-level control and high performance, but adds strong safety guarantees. It introduces an ownership and borrowing model allowing the compiler to check and enforce memory safety and data-race freedom at compile time, without the runtime cost of garbage collection. Rust also includes various high-level features common in modern languages, such as type inference, pattern matching, modules, generics, closures, iterators, concurrency primitives, and async/await, as well as a built-in package and build system. Rust follows the principle of zero-cost abstractions, meaning that higher-level features that compile to lower-level code can run as fast as the code written manually. With its combination of performance, safety, and expressiveness, Rust is well suited to both systems-level programming and application-level development.

### Quick Start

```bash
# installed into ~/.rustup and ~/.cargo
$ curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | bash

# ~/.bashrc
...
. "$HOME/.cargo/env"
...

# ~/.cargo/env
...
export PATH="$HOME/.cargo/bin:$PATH"
...

$ rustc --version
$ rustup update
$ rustup self uninstall
$ rustup doc

$ rustup component list
$ rustup component add rust-analyzer
$ nvim
:CocInstall coc-rust-analyzer
" ~/.config/nvim/init.vim
...
" calling rustfmt (installed by default)
nmap <leader>f <Plug>(coc-format)
...

$ rustc main.rs

# build system and package (crate) manager
# also generating .git and .gitignore (/target) if not in any repository
$ cargo new helloworld
# or in cwd
$ cargo init
$ cargo build
$ cargo build --release
# build (only if necessary) and run
$ cargo run
$ cargo run --release
# compile check only
$ cargo check
```
