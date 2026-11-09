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
