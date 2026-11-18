---
title: 2.1.1. Ownership and Borrowing
---

Smart pointers themselves are variables on the stack or static area, but the actual data resides on the heap, which can be understood as the real value of the variable, merely referenced via a pointer. This perspective shifts heap management away from pointer operations controlled by programmers toward value semantics managed by the compiler. Beneath the surface, Rust variables holding heap values operate similarly to unique smart pointers. As the sole owner, a variable deallocates its heap value via its `drop` method, which the compiler calls right before the variable goes out of scope. Meanwhile, for assignment (including passing parameters), the compiler invalidates the original owner and drops any pre-existing heap value losing its owner, to ensure strict move semantics. Additionally, raw heap data can only be managed in Unsafe Rust, while in Safe Rust, applications rely on standard library types to hold heap values, such as `String` and `Vec`, which already implement the `Drop` trait (with `drop` in it). Ultimately, this simplified model is entirely enforced by the compiler and standard library within Safe Rust.

The compiler uses move semantics by default for assignment, which transfers ownership to ensure that any heap value can have only one owner at any time and heap values without any owner are dropped right away. This is suitable for variables holding heap values, but neither static nor stack variables need to transfer ownership for assignment. In fact, they implement the `Copy` trait to opt into copy semantics, including all scalars, as well as arrays and tuples if all their constituent types implement `Copy`. Obviously, no type can implement both `Copy` and `Drop`. Note that global variables might need synchronisation mechanisms for concurrency safety. For example, mutable globals require locks or atomics to prevent data races.

Each assignment causing an ownership transfer would result in verbose code. Rust provides references for borrowing without taking ownership. References act as pointers or fat pointers with specialised structures for referenced data. They can point to stack or static variables such as `&i32` and `&[i32; 5]`, or variables holding heap values such as `&String`. Furthermore, They can point to values such as strings or arrays, possibly via slicing, regardless of whether those values reside on the heap, the stack, or the static area. References to stack array values can be of type `&[T; length]`, or `&[T]` for array slices because slices are fat pointers with lengths that cannot be determined at compile time. String slices and variables with static string values are both `&str` references, which are also fat pointers. Rust provides slices to avoid floating indexes lacking data validity guarantees. The compiler ensures data validity while there are still references to it to be used. For example, the compiler prevents moving from variables with references to be used.

References should follow copy semantics since they do not take ownership, and immutable references do implement `Copy`. However, Rust disallows any mutable reference to be used simultaneously with any other reference to the same data to prevent data races. Consequently, mutable references still follow move semantics to invalidate the original to ensure that any data can have only one mutable reference to be used. This causes verbose code again. Similarly to borrowing other than assignment, Rust provides reborrowing from references other than reference assignment. Via `&*original` or `&mut *original`, it borrows the referenced data again but temporarily restricts the original until the reborrow is no longer used. Specifically, mutable reborrows completely block the original, while immutable reborrows block only mutation, aligning with the rule for simultaneous references.

Moreover, the compiler implicitly uses reborrowing when assigning mutable references to typed destinations, such as parameters or via type annotations, particularly convenient for calling functions with mutable references. Additionally, the compiler can implicitly apply referencing or dereferencing in certain contexts to simplify code, such as auto-ref/deref for method calls, and deref coercion when passing parameters, but note that references are not aliases.

Similarly to the ownership model, the compiler ensures borrowing and reborrowing safety. Based on this, Rust provides a clear, predictable memory management model that guarantees memory and concurrency safety with minimal runtime cost.

### Ownership

```rust title="Ownership"
fn ownership() {
  // Variables holding heap values
  let mut s1 = String::from("hello");
  let s2 = String::from("world");
  //hello
  println!("{s1}");
  // Move: s2 is invalidated as the original owner of "world"; "hello" losing its owner is dropped.
  s1 = s2;
  // s1 is the owner of "world".
  //world
  println!("{s1}");
  /* s2 is invalid. Error: borrow of moved value: s2
  println!("{s2}");
  */

  // Stack variables
  let a = (0, 'a');
  // Copy
  let mut b = a;
  b.1 = 'b';
  //(0, 'a'), (0, 'b')
  println!("{a:?}, {b:?}");
} // s1 drops "world" right before it goes out of scope.
```

### References

```rust title="References"
fn borrow() {
  let s = String::from("hello");
  // Borrow: Without taking ownership
  refs(&s);
  //hello
  println!("{s}");
}

static G_VAR: i32 = 5;

// The arg can also be &String due to deref coercion.
fn refs(s: &str) {
  // Point to a static variable (&i32)
  let r = &G_VAR;
  //5
  println!("{r}");

  let mut s = String::from(s);
  // Point to a heap string value (&mut str)
  let r1 = &mut s[..1];
  // Auto-ref/deref for method calls
  r1.make_ascii_uppercase();
  // Point to a variable holding a heap value (&String)
  let r1 = &s;
  /* Still with reference r1 to be used, Error: cannot move out of `s` because it is borrowed
  let s1 = s;
  */
  // Point to a static string value (&str)
  /* Variables being mutable means their values can be changed, while references being mutable means
   * they should point to mutable data and can change its value. Static values aren't mutable.
   * Error: expected `&mut str`, found `&'static str`
  let r2: &mut str = "this world ";
  */
  let r2 = "this world ";
  // Point to a static string value (&str)
  let r2 = &r2[5..10];
  //Hello, world!
  println!("{r1}, {r2}!");

  let mut a = [1, 1];
  // Point to a stack array value (slice) (&mut [i32])
  let r1 = &mut a[..1];
  r1[0] = 0;
  // Point to a stack variable (&[i32; 2])
  let r1 = &a;
  // Point to a stack array value (&mut [i32; 4])
  let r2 = &mut [2, 2, 2, 2];
  r2[2] = 3;
  // Point to a stack array value (slice) (&[i32])
  let r2 = &r2[1..3];
  //[0, 1], [2, 3]
  println!("{r1:?}, {r2:?}");
}
```

### Reborrowing

```rust title="Reborrowing"
fn reborrow() {
  let mut i = 5;

  // Immutable references
  let r1 = &i;
  // Copy
  let r2 = r1;
  /* Error: cannot assign to `i` because it is borrowed
  i = 6;
  */
  //5, 5
  println!("{r1}, {r2}");
  // Simultaneous immutable references with immutable reborrows
  //5, 5, 5
  println!("{r1}, {r2}, {}", &*r2);
  /* Error: cannot borrow *r2 as mutable, as it is behind a `&` reference
  println!("{}", &mut *r2);
  */

  // Mutable references
  let r1 = &mut i;
  /* If r2 is to be used here, that mutable borrow r1 will fail with Error: cannot borrow `i` as
   * mutable because it is also borrowed as immutable.
  println!("{r2}");
  */
  // Move: r1 is invalidated as the original.
  let r2 = r1;
  *r2 = 6;
  //6
  println!("{r2}");
  // Immutable Reborrow: Mutation is blocked.
  /* Equivalent to implicit reborrowing via the type annotation
  let r3: &i32 = r2;
  */
  let r3 = &*r2;
  /* Error: cannot assign to *r2 because it is borrowed
   *r2 = 7;
   */
  //6, 6
  println!("{r2}, {r3}");
  // Mutable Reborrow: r2 is blocked as the original.
  /* Equivalent to implicit reborrowing via the type annotation
  let r3: &mut i32 = r2;
  */
  let r3 = &mut *r2;
  /* Error: cannot borrow r2 as immutable because it is also borrowed as mutable
  println!("{r2}");
  */
  /* Error: cannot assign to *r2 because it is borrowed
   *r2 = 7;
   */
  *r3 = 7;
  //7
  println!("{r3}");
  // Reborrow r3 is no longer used, and original r2 is resumed.
  *r2 = 8;
  //8
  println!("{r2}");
  /* r1 is invalid. Error: borrow of moved value: r1
  println!("{r1}");
  */

  let mut s = String::new();
  // Implicit reborrowing for passing params
  io::stdin().read_line(&mut s).expect("Failed to read line");
  // Resumed
  println!("{s}");
}
```
