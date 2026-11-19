---
title: 2.1.2. Borrow Checking and Lifetimes
---

The compiler checks the validity of referenced data via the borrow checker, which ensures that any reference lifetime is entirely contained within the lifetime of the referenced data. Data lifetimes extend until the data goes out of scope (deallocated) or becomes invalid (moved), while reference lifetimes extend to the last use of the reference. Note that lifetimes here do not refer to data survival duration in memory at runtime, but rather to the scope and usage extent of identifiers based on static analysis. While the exact data to be referenced at runtime cannot always be determined at compile time, the borrow checker operates conservatively, rejecting any runtime risk of dangling references.

### Borrow Checker

```rust title="Borrow Checker"
fn borrow_checker() {
  let mut opt = String::new();
  io::stdin()
    .read_line(&mut opt)
    .expect("Failed to read line");
  let opt = match opt.trim().to_lowercase().as_str() {
    "y" | "yes" => true,
    _ => false,
  };

  let s = String::from("hello");
  // Reference to `s` or "world" depending on runtime, while the borrow checker conservatively
  // compares `r` with `s`.
  let r = if opt { &s } else { "world" };
  /* Error: cannot move out of `s` because it is borrowed
  let s1 = s;
  */
  println!("{r}");
}
```

The borrow checker operates within functions, meaning that it does not look into function calls to infer the data referenced by returned references. Instead, it applies reference lifetime relationships between inputs and outputs of function calls from function signatures. If an output reference is specified to have the same lifetime as an input reference, the borrow checker treats that output as that input, or its reborrow if it is reborrowed, for lifetime checking. This applies to data validity as well as simultaneous references and reborrows rules. For example, it ensures that the lifetime of that output is contained within the lifetime of the data referenced by that input. If no lifetimes are specified, the borrow checker attempts to infer the relationship. If there is only one input reference, or multiple ones but one of them is `&self` or `&mut self` (as in methods), it infers that output references have the same lifetime as that one.

Reference lifetimes are specified as reference type specifiers. While there are concrete lifetimes such as `'static`, those specifiers are more often as generic parameters. Note that both input and output reference lifetimes can be flexible across different function calls, but this flexibility is constrained at compile time. Moreover, the relationship can still be fixed as generic logic. Based on this, the compiler instantiates generic lifetime parameters to concrete reference lifetimes for each function call.

### Lifetimes

```rust title="Lifetimes"
// For lifetime checking, the output should be treated as the input, or an immutable/mutable
// reborrow from the input accordingly if that input is a mutable reference and therefore reborrowed
// by `s` as an immutable/mutable one, no matter whether that output itself is immutable or mutable.
/* Auto-inferred as `fn new_ref<'a>(s: &'a mut str) -> &'a str`
fn new_ref(s: &mut str) -> &str {
*/
/* Auto-inferred as `fn new_ref<'a>(s: &'a str) -> &'a str` */
fn new_ref(s: &str) -> &str {
  s
  /* Note that there's no type specifier for reborrowing, so there's no way to specify return values
   * with reborrowing relationships to params.
  &*s
  */
}

/* Error: missing lifetime specifier; this function’s return type contains a borrowed value, but the
 * signature does not say whether it is borrowed from `x` or `y`
fn longest(x: &str, y: &str) -> &str {
*/
// The output should be treated as both inputs for lifetime checking, so the output lifetime should
// be contained within the smaller lifetime of the data referenced by those inputs.
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
  if x.len() > y.len() { x } else { y }
}

fn lifetime() {
  let mut s1 = String::from("hello");
  let r1 = &mut s1;
  /* r1 is a mutable reference and therefore reborrowed by the param as an immutable/mutable one, so
   * for lifetime checking, r2 is treated as an immutable/mutable reborrow from r1 accordingly, even
   * if r2 itself is immutable. */
  let r2 = new_ref(r1);
  /* The original blocked
  r1.push_str(", world");
  */
  /* If r1 is reborrowed by the param as a mutable one, this will also be blocked. */
  //hello
  println!("{r1}");
  /* Without Reborrow, this violates simultaneous references rules.
  let r2 = &s1;
  */
  //hello
  println!("{r2}");
  // The original resumed
  //hello
  println!("{r1}");

  let result;
  /* If s2 is within this block scope, the function call will fail with Error: s2 does not live long
   * enough.
  {
  */
  let s2 = String::from("a");
  // `result` references "hello" at runtime, while the borrow checker compares it with both s1 and
  // s2 based on the lifetime relationship from the function signature.
  result = longest(&s1, &s2);
  /*
  }
  */
  //hello
  println!("{result}");
}
```

In generic programming, types act as a typical kind of generic parameter. As type specifiers, lifetimes are included in instantiation when type parameters are instantiated to reference types. As mentioned earlier, the compiler implicitly reborrows mutable references when assigning them to typed destinations. When the destination is a generic type parameter, the compiler uses implicit reborrowing only if it is explicitly specified as `&mut`. When the destination is a bare type parameter, the compiler uses copy or move semantics strictly based on the instantiated type.

### Generic References

```rust title="Generic References"
fn mut_ref<T>(r: &mut T) -> &mut T {
  r
}

fn _foo<T>(x: T) -> T {
  x
}

fn generic_ref() {
  let mut s = String::from("hello");
  let r1 = &mut s;
  /* Implicit Reborrow */
  let r2 = mut_ref(r1);
  /* Move for mutable references
  let r2 = _foo(r1);
  */
  /* Lifetimes instantiated within reference types
  let _s = s;
  */
  /* Reborrow: The original blocked
  println!("{r1}");
  */
  r2.push_str(", world");
  //hello, world
  println!("{r2}");
  // Reborrow: The original resumed: hello, world
  /* Move: Error */
  println!("{r1}");
}
```
