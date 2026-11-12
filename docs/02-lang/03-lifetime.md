---
title: 2.3. Borrow Checking and Lifetimes
---

The compiler checks the validity of referenced data via the borrow checker, which ensures that any reference lifetime is entirely contained within the lifetime of the referenced data. Data lifetimes extend until the data goes out of scope (deallocated) or becomes invalid (moved), while reference lifetimes extend to the last use of the reference. While the exact data to be referenced at runtime cannot always be determined at compile time, the borrow checker operates conservatively, rejecting any runtime risk of dangling references.

### Borrow Checker

```rust title="Borrow Checker"
fn borrow_checker() {
  // 1. The elem type is inferred as integers, which can default to i32.
  let mut opt = [0; 1];
  io::stdin()
    // 2. Then determined as u8 for the demand. Note that type inference might not be entirely
    // determined at declarations.
    .read_exact(&mut opt)
    .expect("Failed to read");
  let opt = match opt[0] {
    b'y' | b'Y' => true,
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

The borrow checker operates within functions, meaning that it does not look into function calls to deduce the data referenced by returned references (outputs). Instead, it uses the lifetime relationship between parameter references (inputs) and outputs from function signatures. For example, if an output is specified to have the same lifetime as an input, the borrow checker treats the output as the input for lifetime checking. In other words, it ensures that the output lifetime is contained within the lifetime of the data referenced by the input. Lifetimes are specified as another kind of generic parameters. If no lifetimes specified, the borrow checker will deduce the relationship. If there is only one input lifetime, it will be specified as the output lifetime. If there are multiple input lifetimes and one of them is `&self` or `&mut self` (methods), the lifetime of `self` will be specified as the output lifetime.

### Lifetimes

```rust title="Lifetimes"
/* Error: missing lifetime specifier; this function’s return type contains a borrowed value, but the
 * signature does not say whether it is borrowed from `x` or `y`
fn longest(x: &str, y: &str) -> &str {
*/
// The output is treated as both `x` and `y` for lifetime checking, meaning the output lifetime
// should be contained within the smaller lifetime of the data referenced by `x` and the data
// referenced by `y`.
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
  if x.len() > y.len() { x } else { y }
}

/* Auto-deduced as `fn new_ref<'a>(input: &'a String) -> &'a str`, meaning the output is treated as
 * `input` for lifetime checking
fn new_ref(input: &String) -> &str {
*/
fn new_ref(input: &String) -> &'static str {
  println!("{input}");
  "hello"
}

fn lifetime() {
  let s1 = String::from("longest");
  let result;
  /* If s2 is within this block scope, the function call will fail with Error: s2 does not live long
   * enough.
  {
  */
  let s2 = String::from("a");
  // `result` references "longest" at runtime, while the borrow checker compares it with both s1 and
  // s2 based on the lifetime relationship from the function signature.
  result = longest(&s1, &s2);
  /*
  }
  */
  //longest
  println!("{result}");

  //longest
  let r = new_ref(&s1);
  // With auto-deduction, `r` would be treated as a reference to s1, and this would fail with Error:
  // cannot move out of s1 because it is borrowed.
  let _s1 = s1;
  //hello
  println!("{r}");
}
```

Furthermore, structs (including enums with struct variants) with reference fields must be generic with lifetime parameters for the borrow checker to enforce lifetime checking on the struct, without looking into associated functions to deduce the data referenced by its fields.

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

// `_` as the elided lifetime, here is the only one input lifetime.
fn new_ref1(input: &String) -> Ref<'_> {
  Ref { _s: input }
}

fn struct_ref_lifetime() {
  let s = String::from("hello");
  let r = new_ref1(&s);
  /* Error: cannot move out of `s` because it is borrowed
  let s1 = s;
  */
  //Ref { _s: "hello" }
  println!("{r:?}");
}
```

Similarly to types, reference lifetimes are essentially static contracts determined at compile time, which typically do not exist and therefore cannot be manipulated at runtime in traditional statically typed languages. Generic programming provides a way to make these rigid contracts programmable, based on the compiler supporting parameterised types and lifetimes used in contract logic to generate flexible contracts. While traditional code is executed by the machine and reusable across different runtime data, generic code is executed by the compiler and reusable across different static contracts (particularly types). Unlike dynamic types and decorators relying on runtime metadata, generic programming operates purely at compile time for zero-cost abstractions — the priority of Rust's design philosophy.

As mentioned earlier, the compiler implicitly reborrows mutable references for assignment with typed destinations. Note that when destinations are generic type parameters, the compiler uses implicit reborrowing only if the parameter is explicitly specified as `&mut`. When the destination is a bare type parameter, the compiler uses copy or move semantics strictly based on the instantiated type.

### Generic Mutable References

```rust title="Generic Mutable References"
#[derive(Debug)]
enum MutRef<'a, T> {
  Ref(&'a mut T),
}

fn generic_mut_ref() {
  let mut s = String::from("hello");
  let r1 = &mut s;
  /* Implicit reborrow */
  let mut r2 = MutRef::Ref(r1);
  /* Some(T): Move for mutable references
  let mut r2 = Some(r1);
  */
  /* Reborrow: The original blocked
  println!("{r1}");
  */
  match r2 {
    /* MutRef: Note that Ref(&'a mut T) serves as the pattern source, not the dest slot for `r`
     * here, so `ref` is used to avoid Move for mutable references. */
    MutRef::Ref(ref mut r) => r.push_str(", world"),
    /* Some(T)
    Some(ref mut r) => r.push_str(", world"),
    _ => (),
    */
  }
  //MutRef: Ref("hello, world")
  //Some(T): Some("hello, world")
  println!("{r2:?}");
  // Reborrow: The original resumed: hello, world
  /* Move: Error: borrow of moved value: r1 */
  println!("{r1}");
}
```
