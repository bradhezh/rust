---
title: 2.4. Modules and Privacy
---

Within a package, there can be one binary crate rooted at `src/main.rs`, or multiple binary crates rooted at each file in `src/bin/`, but at most only one library crate rooted at `src/lib.rs`. Crates are usually module trees from those roots.

A module is declared via `mod <name>` in its parent, with inline code or code in like `src/<parent_name>/<mod_name>.rs` or `src/<parent_name>/<mod_name>/mod.rs`, but if the parent is the root, there is no `<parent_name>` part. If a binary crate in `src/bin/` has modules, they should be separated in a sub-directory, and rooted at `src/bin/<sub_dir>/main.rs`. All items in a module, including functions, types, and submodules, are private by default, meaning that they are accessible only within that module and its descendants. When marked `pub`, items in module _M_ become accessible in any module where _M_ and all its ancestors are accessible. Note that the fields of public structs are still private by default, whereas the variants of public enums are also public.

Assume that _P_ is the parent of _M_, _C_ is a public child of _M_, and _R_ is a descendant of _P_. First, _M_ being accessible in _P_ causes public items (_C_) in _M_ to be accessible in _P_, which then causes public items in _C_ to be accessible in _P_, and so on. Ultimately, public items in a public module are accessible throughout a public ancestor chain from that module. This reasoning can also start from _R_ because _P_ and _M_ are naturally accessible in _R_ even if _M_ is private. Consequently, _P_ acts as the accessibility boundary of all public items from private _M_, no matter whether they are directly in _M_ or there is a public module chain from them up to _M_, which can be referred to as a private ceiling. In other words, public items under a private ceiling are only accessible throughout the parent tree of that ceiling. Additionally, since roots are always accessible, public items with a public module chain up to the root are accessible in other crates.

Items in modules are identified via paths like `<crate_name>::<parent_name>::<mod_name>::<item_name>`, where `crate` refers to the current crate. Paths can also be relative to the current module, where `super` acts like `..`. Additionally, `use` creates shortcuts to paths, bringing those items into the current module scope while still being subject to the privacy rules mentioned earlier. In practice, it is idiomatic to bring types into scope but bring their parents for functions. Moreover, `pub use` re-exports the shortcut as a public item from the current module, like `<mod_name>::<shortcut_name>`, which can short-circuit the privacy chain without violations. For example, if module _M_ uses `pub use <private_mod>::<pub_item>`, its siblings can use `use <M>::<pub_item>` to access the item.

### Privacy Rules

```rust title="src/lib.rs"
pub mod m;
```

```rust title="src/m.rs"
mod pri_c;
pub mod pub_c;
```

```rust title="src/m/pri_c.rs"
pub fn bar() {
  println!("Public item under a private ceiling");
}
```

```rust title="src/m/pub_c.rs"
use super::pri_c;

pub fn foo() {
  println!("Public items under a private ceiling are accessible in its parent tree.");
  pri_c::bar();
}
```

```rust title="src/bin/mod_demo.rs"
// Package name for the lib crate
use rust_demo::m::pub_c;

fn main() {
  println!("Public items with a public chain up to its root are accessible in other crates.");
  pub_c::foo();
}
```

```bash
cargo run --bin mod_demo
```

### `use` Syntax

```rust title="use"
use rand::prelude::*;
use std::{cmp::Ordering, io};
use std::io::{self, Write};
```

Paths can be specified in `pub(in <path>)` as an extra restriction in addition to the privacy rules. Basically, it prevents the item from being accessible outside the path boundary, which cannot be broken even via re-exporting. Similarly, `pub(crate)` restricts the boundary to the current crate, `pub(super)` is equivalent to `pub(in super)`, and `pub(self)` is equivalent to `pub(in self)`, effectively meaning not using `pub` at all.

Note that crates are source code compiled locally and statically linked. Without a stable native ABI, Rust supports source-code level updatability based on modules and privacy. Typically, private items are updatable implementations with public items as interfaces, including crate APIs with a public module chain up to the root, and internal APIs under private ceilings.
