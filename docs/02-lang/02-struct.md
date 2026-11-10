---
title: 2.2. Structs and Enums
---

### Structs

```rust title="Structs"
// Structs follow Move by default unless explicitly implementing Copy. Note that Rust types are
// nominal rather than structural.
#[derive(Debug)]
struct Label {
  // The struct (specifically the field) is the owner.
  text: String,
  width: u32,
  height: u32,
}

// There can be multiple impl blocks for a struct.
impl Label {
  // Associated functions. `Self` is a type alias of the struct.

  fn new(text: &str, width: u32, height: u32) -> Self {
    Self {
      text: String::from(text),
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
  //pos1 = Pos(1, 1); pos2 = Pos(2, 2)
  dbg!(pos1, pos2);

  // Default Move
  let mut label1 = Label::new("Label 1", 100, 10);
  let label2 = Label::new("Label 2", dbg!(100 * 2), 20);
  // Fields
  // println! auto-refs params, whereas dbg! not.
  //&label1.text = "Label 1"; label1.width = 100; label1.height = 10
  dbg!(&label1.text, label1.width, label1.height);
  // Move: label2 (or label2.text) invalidated, "Label 1" dropped
  label1 = label2;
  /* Or
  label1.text = label2.text;
  // Or field `text` moved while others copied
  label1 = Label { ..label2 };
  */
  // label1 is the owner of "Label 2".
  //&label1 = Label { text: "Label 2", ...}
  dbg!(&label1);
  /* label2 (or label2.text) is invalid. Error: borrow of (partially) moved value: label2
  dbg!(&label2);
  dbg!(&label2.text);
  // If only partially moved, left fields are still valid.
  dbg!(&label2.width, &label2.height);
  */

  // Auto-ref/deref for method calls
  //label1.size() = 4000; Label::size(&label1) = 4000
  dbg!(label1.size(), Label::size(&label1));
} // label1 drops "Label 2" right before it goes out of scope.
```

Traditionally, the values of an enum can only be one of the named constants defined for it, while in Rust, they can also be values of named types, and enums can also have methods. For example, with different structs as variants, a Rust enum can serve as a sum type with different structures. Furthermore, these struct variants can be matched and destructured via `match` and patterns, providing type-based branching other than purely value-based `if`/`else`, while `match` actually supports value-based branching as well.

Additionally, Rust provides `if let` and `let...else` as syntax sugar for `match` to pick only one pattern of values. For example, Rust does not support the null value directly while the standard library provides the `Option<T>` enum for values that could be nothing or something, which can be conveniently dealt with via `let...else`. Also, enums follow move semantics by default, exactly like structs. Note that for generic types to opt into copy semantics, the generic type itself and its type arguments are all required to implement `Copy`. For example, since `Option<T>` implements `Copy`, `Option<i32>` follows copy semantics whereas `Option<String>` falls back to move semantics.

### Enums

```rust title="Enums"
#[derive(Debug)]
enum Message {
  // Unit struct
  Quit,
  // Struct
  Pos { x: i32, y: i32 },
  // Tuple struct
  Write(String),
}

impl Message {
  fn foo(&self) {
    match self {
      Message::Quit => println!("Quit"),
      Message::Pos { x, y } => println!("Pos: {x}, {y}"),
      /* Note that "matching on a reference type with a non-reference pattern implicitly borrows the
       * contents", so don't use `ref` here to explicitly borrow again.
      Message::Write(ref s) => println!("Write: {s}"),
      */
      Message::Write(s) => println!("Write: {s}"),
    }
  }
}

fn enums() {
  let q = Message::Quit;
  let m = Message::Pos { x: 1, y: 2 };
  let w = Message::Write(String::from("hello"));
  //Quit
  q.foo();
  //Pos: 1, 2
  m.foo();
  //Write: hello
  w.foo();
  //w = Write("hello")
  dbg!(w);
}
```

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
    3 => println!("{x:?} and three"),
    7 => println!("{x:?} and Seven"),
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
  let five = Some(5);
  let a = Some('a');
  let s = String::from("hello");
  // Reference used to avoid Move for String
  let hello = Some(&s);
  let null: Option<i32> = None;
  // All Copy
  //5 and three
  op_on_nullable(five, 3);
  //'a' and Seven
  op_on_nullable(a, 7);
  //"hello" and 10
  op_on_nullable(hello, 10);
  op_on_nullable(null, 0);
  //five = Some(5); a = Some('a'); &s = "hello"; hello = Some("hello"); null = None
  dbg!(five, a, &s, hello, null);
}
```
