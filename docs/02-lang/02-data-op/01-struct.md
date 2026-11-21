---
title: 2.2.1. Structs and Enums
---

Structs and enums are basic user-defined types in Rust, and both can have associated functions (methods). Note that they are nominal rather than structural. Unlike traditional enums only with constant values, Rust enums can also have different structs as variants, which can be matched and destructured via `match` and patterns. In Rust, enums serve as an algebraic type with flexible structures, providing a way to manipulate types in addition to generics.

Structs (including enums as variant structs) are also regarded as owners if they contain fields holding heap values, but other fields can still remain valid even if those heap-holding fields are partially moved. Additionally, types with reference fields must be generic over lifetime parameters, for the borrow checker to enforce lifetime checking without looking into associated functions to infer the data referenced by fields. Note that move semantics is used by default. To opt into copy semantic, both the type itself and all its fields must implement `Copy`, while mutable references and heap-holding types can never implement `Copy`. Alternatively, types can implement `Drop`, which disables partial moving.

### Structs

```rust title="Structs"
#[derive(Debug)]
struct Label {
  _text: String,
  width: u32,
  height: u32,
}

// There can be multiple impl blocks for one type.
impl Label {
  // Associated functions. `Self` is an alias of the type.

  fn new(text: &str, width: u32, height: u32) -> Self {
    Self {
      _text: String::from(text),
      width,
      height,
    }
  }

  // Methods
  // &self: `self: &Self`
  // &mut self: `self: &mut Self`
  // self: `self: Self`
  fn size(&self) -> u32 {
    // Auto-deref for field access
    self.width * self.height
  }
}

#[derive(Debug, Clone, Copy)]
// Tuple struct
struct Pos(i32, i32);

// Unit-like struct without any field but can implement traits.
struct AlwaysEqual;

fn structs() {
  let _u = AlwaysEqual;

  // Structs implementing Copy
  let pos1 = Pos(1, 1);
  let Pos(_x, _y) = pos1;
  // Copy
  let mut pos2 = pos1;
  pos2.0 = 2;
  pos2.1 = 2;
  //Pos(1, 1), Pos(2, 2)
  println!("{pos1:?}, {pos2:?}");

  // Default Move
  let mut label1 = Label::new("Label 1", 100, 10);
  let label2 = Label::new("Label 2", 200, 20);
  //Label { _text: "Label 1", width: 100, height: 10 }
  println!("{label1:?}");
  /* Move: label2 (or label2._text) invalidated, "Label 1" dropped */
  label1 = label2;
  /* Or (Partial Move)
  label1._text = label2._text;
  // Or (Partial Move with Copy for others)
  label1 = Label { ..label2 };
  */
  // label1 is the owner of "Label 2".
  //Label { _text: "Label 2", ...}
  println!("{label1:?}");
  /* label2 (or label2._text) is invalid. Error: borrow of (partially) moved value: label2
  println!("{label2:?}");
  println!("{}", label2._text);
  // Partial Move: Other fields remain valid.
  println!("{}, {}", label2.width, label2.height);
  */

  // Auto-ref/deref for method calls
  //4000, 4000
  println!("{}, {}", label1.size(), Label::size(&label1));
} // label1 drops "Label 2" right before it goes out of scope.
```

### Struct Reference Lifetimes

```rust title="Struct Reference Lifetimes"
/* Error: missing lifetime specifier
struct Ref {
  s: &str,
}
*/
#[derive(Debug)]
struct Ref<'a> {
  _s: &'a str,
}

// '_ as the elided lifetime, inferred to the same lifetime as the only input.
/*
fn new_ref(s: &mut str) -> Ref<'_> {
*/
fn new_ref(s: &str) -> Ref<'_> {
  Ref { _s: s }
}

fn struct_ref_lifetime() {
  let mut s = String::from("hello");
  let r1 = &mut s;
  /* For lifetime checking, r2 is treated as an immutable/mutable reborrow from r1 accordingly. */
  let r2 = new_ref(r1);
  /* The original blocked
  r1.push_str(", world");
  */
  /* If r1 is reborrowed by the param as a mutable one, this will also be blocked. */
  //hello
  println!("{r1}");
  /* Without Reborrow, this violates simultaneous references rules.
  let r2 = &s;
  */
  //Ref { _s: "hello" }
  println!("{r2:?}");
  // The original resumed
  //hello
  println!("{r1}");
}
```

### Enums

```rust title="Enums"
#[derive(Debug)]
enum Message<'a> {
  // Unit struct
  Quit,
  // Struct
  Pos { x: i32, y: i32 },
  // Tuple structs
  Text(String),
  MutRef(&'a mut String),
}

impl<'a> Message<'a> {
  fn foo(&mut self) {
    match self {
      Message::Quit => println!("Quit"),
      Message::Pos { x, y } => println!("Pos: {x}, {y}"),
      /* Note that "matching on a reference type with a non-reference pattern implicitly borrows the
       * contents", so don't use `ref` here as Explicit Borrow. */
      Message::Text(s) => println!("Text: {s}"),
      /* Or use a reference pattern without Implicit Borrow, and therefore `ref` should be used here
       * for Explicit Borrow to avoid Move for String.
      &mut Message::Text(ref s) => println!("Text: {s}"),
      */
      /* Note that MutRef(&'a mut String) serves as the pattern source, not the dest slot for `s`
       * here, which still "implicitly borrows the contents", resulting in `&mut &mut String`, which
       * still works here due to auto-deref.
      Message::MutRef(s) => {
      */
      /* Or reference pattern with Explicit Borrow to avoid Move for mutable references, `ref mut`
       * here still results in `&mut &mut String`. */
      &mut Message::MutRef(ref mut s) => {
        s.push_str(", world");
        println!("MutRef: {s}");
      }
    }
  }
}

fn enums() {
  let mut quit = Message::Quit;
  let mut pos = Message::Pos { x: 1, y: 2 };
  let mut text = Message::Text(String::from("hello"));
  let mut s = String::from("hello");
  let mut mut_ref = Message::MutRef(&mut s);
  //Quit
  quit.foo();
  //Pos: 1, 2
  pos.foo();
  //Text: hello
  text.foo();
  //Text("hello")
  println!("{text:?}");
  //MutRef: hello, world
  mut_ref.foo();
  //MutRef("hello, world")
  println!("{mut_ref:?}");
  //hello, world
  println!("{s}");
}
```

With enums as variant structs, `match` provides type-based branching other than purely value-based `if`/`else`, while `match` supports value-based branching as well. Additionally, Rust provides `if let` and `let...else` as syntax sugar for `match` to pick only one pattern of values. For example, Rust does not support the null value directly, while the standard library provides the `Option<T>` enum for values that could be nothing or something. Nullish values can be conveniently filtered via `let...else`. Note that to opt into copy semantic, generic types like `Option<T>` typically implement `Copy` under a trait bound `T: Copy`, where `T` acts as the inner field type. Consequently, `Option<i32>` and `Option<&String>` follow copy semantics whereas `Option<String>` and `Option<&mut String>` fall back to move semantics.

### Options

```rust title="Options"
/*
enum Option<T> {
  None,
  Some(T),
}
*/

fn op_on_nullable<T: std::fmt::Debug>(x: Option<T>, i: i32) {
  /* Equivalent to
  let x = match x {
    Some(v) => v,
    None => return,
  };
  */
  let Some(x) = x else {
    return;
  };
  /* Or
  if let Some(x) = x {
    ...
  }
  */

  // Value-based
  match i {
    0 => println!("{x:?} and ZERO"),
    10 => println!("{x:?} and ten"),
    /* Equivalent to
    _ => println!("{x:?} and {i}"),
    */
    other => println!("{x:?} and {other}"),
    /* Do nothing
    _ => (),
    */
  }
}

fn options() {
  let null: Option<i32> = None;
  let five = Some(5);
  let a = Some('a');
  let s = String::from("hello");
  /* Borrow */
  let hello = Some(&s);
  /* Move
  let hello = Some(s);
  */
  // All Copy
  op_on_nullable(null, 0);
  //5 and ZERO
  op_on_nullable(five, 0);
  //'a' and ten
  op_on_nullable(a, 10);
  // Borrow: Copy for Option<&String>, Move for Option<&mut String>
  // Move: Move for Option<String>
  //"hello" and 1
  op_on_nullable(hello, 1);
  //None, Some(5), Some('a')
  println!("{null:?}, {five:?}, {a:?}");
  // Copy: Some("hello"), hello
  /* Move: Error */
  println!("{hello:?}, {s}");
}
```

In Rust, there are common patterns for wrappers (of type `W<T>`, e.g. `Option<T>`) with inner payload values (of type `T`), including projection methods, materialisation methods, swapping methods, and direct extractors. Projection or materialisation methods return a new wrapper. Swapping methods return either a new wrapper (e.g. `Option<T>`) or the payload by value (e.g. `T` for `Cell<T>` and `RefCell<T>`). Direct extractors return the payload by value or by reference.

Typically, `as_ref`/`as_mut` projects `W<T>` to `W<&T>`/`W<&mut T>`, while `as_deref`/`as_deref_mut` projects `W<T>` to `W<&T::Target>`/`W<&mut T::Target>` under a trait bound `T: Deref`/`T: DerefMut` where `Target` is specified. For example, `String` specifies both `Deref::Target` and `DerefMut::Target` as `str`, so `Option<String>::as_deref()` returns `Option<&str>` and `Option<String>::as_deref_mut()` returns `Option<&mut str>`. Similarly, since `&mut T` implements `Deref<Target = T>`, `as_deref` can also downgrade `W<&mut T>` to `W<&T>`. For example, `Option<&mut i32>::as_deref()` returns `Option<&i32>`. However, `&T` does not implement `DerefMut`, preventing `W<&T>` from being upgraded to `W<&mut T>`. Projection methods take `&self`/`&mut self` without moving the wrapper.

The `copied` (under `T: Copy`) or `cloned` (under `T: Clone`) method copies or clones the referenced payload from `W<&T>`/`W<&mut T>` into `W<T>`. Note that materialisation methods take `self`, moving it if the wrapper follows move semantics. For example, `Option<&mut i32>::copied()` moves the wrapper, whereas `Option<&mut i32>::as_deref().copied()` avoids moving it.

The `replace` method swaps the payload with a new value, while `take` swaps it with a default/empty state — either `W: Default` (e.g. `Option::None`) or `T: Default` (for `Cell<T>` and `RefCell<T>`). Swapping methods take `&mut self`.

The `unwrap` method returns `T`, panicking if the wrapper is in an empty or unsuccessful state. It takes `self`, moving it if the wrapper follows move semantics. In contrast, `get_or_insert` takes `&mut self` and returns `&mut T`, inserting a payload only when the wrapper is in its empty state such as `Option::None`.

### Wrapper Patterns

```rust title="Wrapper Patterns"
fn wrapper() {
  //Type: Option<String>
  let mut w = Some(String::from("hello"));
  //Type: Option<&mut String>
  let p = w.as_mut();
  //Type: &mut String
  let v = p.unwrap();
  v.push_str(", world!");
  //Type: Option<&String>
  let p = w.as_ref();
  //Some("hello, world!")
  println!("{p:?}");
  //Type: Option<&mut str>
  let p = w.as_deref_mut();
  //Type: &mut str
  let v = p.unwrap();
  v[..1].make_ascii_uppercase();
  //Type: Option<&str>
  let p = w.as_deref();
  //Some("Hello, world!")
  println!("{p:?}");

  //Type: Option<i32>
  let mut w = Some(5);
  //Type: Option<&mut i32>
  let p1 = w.as_mut();
  /* Move for Option<&mut i32>
  let m = p1.copied();
  */
  /* Downgrade */
  //Type: Option<&i32>
  let p2 = p1.as_deref();
  //Type: Option<i32>
  let m = p2.copied();
  //Type: &mut i32
  let v = p1.unwrap();
  // Don't use p1 and p2 any more since p1 is moved (unwrapped) and p2 borrows from p1.
  *v += 1;
  //Some(5), Some(6)
  println!("{m:?}, {w:?}");

  let mut w = None;
  //None, Some(1)
  println!("{:?}, {w:?}", w.replace(1));
  //Some(1), None
  println!("{:?}, {w:?}", w.take());
  //1
  println!("{:?}", w.get_or_insert(1));
  //Some(1)
  println!("{w:?}");
}
```
