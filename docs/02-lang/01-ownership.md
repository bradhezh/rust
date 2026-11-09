---
title: 2.1. Ownership and Borrowing
---

The management of memory — specifically the heap — has long been a frequent source of program bugs and vulnerabilities. Beyond code itself, programs use three primary kinds of memory for data — static areas, the stack, and the heap. Typically, compilers allocate global or static variables and certain constants in a static area throughout the program lifecycle. Function parameters and local variables are allocated and deallocated at runtime with function calls on the stack, while compilers can still generate push and pop instructions for them. The key point is that both static and stack data have fixed sizes and patterns for allocation and deallocation, as well as fixed addresses or offsets for accessing them. All fixed factors can be determined at compile time, for compilers to know when and how much memory to allocate or deallocate, and how to access the allocated data.

However, programs also deal with a different kind of data. Not only its size, but even whether it needs to be allocated at all, cannot be determined at compile time — for instance, a string from user input. Programs must therefore handle whatever occurs exactly at runtime, allocating the exact space and deallocating it when it is no longer needed. The heap serves as dynamic memory for this purpose. Note that "dynamic" here means not only runtime allocation and deallocation, but also those allocation factors being flexible at runtime. Compilers can only allocate fixed-size pointers to reference dynamic heap data, while programmers have to write logic for heap allocation, deallocation, and access. Consequently, traditional compilers have no way to prevent programmers from introducing related bugs, including memory leaks, dangling pointers, and double-free errors.

A typical solution is Garbage Collection (GC), which intermittently runs at runtime to trace all reachable heap data and deallocate any unreachable one. With GC, programmers can allocate and reference heap data without manual heap management, but GC introduces heavy runtime overheads. Alternatively, C++ provides smart pointers for automatic heap management, typically relying on reference counting and implemented via copy constructors, copy assignment operator overrides, and destructors. Typically, the first two override copy semantics to automatically increase the reference count. Meanwhile, destructors are automatically called right before smart pointers are destroyed, to decrease the reference count and deallocate the managed heap data when its reference count reaches 0. However, C++ smart pointers involve a complex management model with shared pointers, weak pointers, unique pointers, copy and move semantics, as well as references and rvalue references. This complex model is managed by programmers, without mandatory compiler enforcement.

Rust avoids GC to prioritise performance. It draws inspiration from C++ smart pointers but simplifies the model with strict compiler enforcement.

Smart pointers themselves are variables on the stack or static area, but the actual data resides on the heap, which can be understood as the real value of the variable, merely referenced via a pointer. This perspective shifts heap management away from pointer operations controlled by programmers toward value semantics managed by the compiler. Beneath the surface, Rust variables holding heap values operate similarly to unique smart pointers. As the sole owner, a variable deallocates its heap value via its `drop` method, which the compiler calls right before the variable goes out of scope. Meanwhile, for assignment (including passing parameters), the compiler invalidates the original owner and drops any pre-existing heap value losing its owner, to ensure strict move semantics. Additionally, raw heap data can only be managed in Unsafe Rust, while in Safe Rust, applications rely on types like `String`, `Box`, and `Vec` to hold heap values. These standard library types all implement the `Drop` trait (with `drop` in it). Ultimately, this simplified model is entirely enforced by the compiler and standard library within Safe Rust.

The compiler uses move semantics by default for assignment, which transfers ownership to ensure that any heap value can have only one owner at any time and heap values without any owner are dropped right away. This is suitable for variables holding heap values, but neither static nor stack variables need to transfer ownership for assignment. In fact, they implement the `Copy` trait to opt into copy semantics, including all scalars, as well as arrays and tuples if all their constituent types implement `Copy`. Obviously, no type can implement both `Copy` and `Drop`. Note that global variables might need synchronisation mechanisms for concurrency safety. For example, mutable globals require locks or atomics to prevent data races.

Each assignment causing an ownership transfer would result in verbose code. Rust provides references for borrowing without taking ownership. References act as pointers or fat pointers with specialised structures for referenced data. They can point to stack or static variables such as `&i32` and `&[i32; 5]`, or variables holding heap values such as `&String`. Furthermore, They can point to values such as strings or arrays, possibly via slicing, regardless of whether those values reside on the heap, the stack, or the static area. References to stack array values can be of type `&[T; length]`, or `&[T]` for array slices because slices are fat pointers with lengths that cannot be determined at compile time. String slices and variables with static string values are both `&str` references, which are also fat pointers. Rust provides slices to avoid floating indexes lacking data validity guarantees. The compiler ensure data validity while there are still references to it to be used. For example, the compiler prevents moving from a variable with references to be used.

References should follow copy semantics since they do not take ownership, and immutable references do implement `Copy`. However, Rust disallows any mutable reference to be used simultaneously with any other reference to the same data to prevent data races. Consequently, mutable references still follow move semantics to invalidate the original to ensure that any data can have only one mutable reference to be used. This causes verbose code again. Similarly to borrowing other than assignment, Rust provides reborrowing from references other than reference assignment. Via `&*original` or `&mut *original`, it borrows the referenced data again but temporarily restricts the original until the reborrow is no longer used. Specifically, mutable reborrows completely block the original, while immutable reborrows block only mutation, aligning with the rule for simultaneous references.

Moreover, the compiler implicitly uses reborrowing for mutable reference assignment with typed destinations, such as parameters or via type annotations, particularly convenient for functions with mutable reference parameters. Additionally, the compiler can implicitly apply references or dereferences in certain contexts to simplify code, such as auto-ref/deref for method calls, and deref coercion when passing parameters, but note that references are not aliases.

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
fn main() {
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
  let r1 = &G_VAR;
  //5
  println!("{r1}");

  let mut s = String::from(s);
  // Point to a heap string value (&mut str)
  let r2 = &mut s[..1];
  // Auto-ref/deref for method calls
  r2.make_ascii_uppercase();
  // Point to a variable holding a heap value (&String)
  let r3 = &s;
  /* Still with reference r3 to be used, Error: cannot move out of `s` because it is borrowed
  let s1 = s;
  */
  // Point to a static string value (&str)
  /* Variables being mutable means their values can be changed, while references being mutable means
   * they should point to mutable data and can change its value. Static values aren't mutable.
   * Error: expected `&mut str`, found `&'static str`
  let r4: &mut str = "this world ";
  */
  let r4 = "this world ";
  // Point to a static string value (&str)
  let r5 = &r4[5..10];
  //Hello, world!
  println!("{r3}, {r5}!");

  let mut a = [1, 1];
  // Point to a stack array value (slice) (&mut [i32])
  let r6 = &mut a[..1];
  r6[0] = 0;
  // Point to a stack variable (&[i32; 2])
  let r7 = &a;
  // Point to a stack array value (&mut [i32; 2])
  let r8 = &mut [2, 2];
  // Point to a stack array value (slice) (&mut [i32])
  let r9 = &mut r8[1..];
  r9[0] = 3;
  //[0, 1], [2, 3]
  println!("{r7:?}, {r8:?}");
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
  let r3 = &mut i;
  /* If r2 is to be used here, that mutable borrow r3 will fail with Error: cannot borrow `i` as
   * mutable because it is also borrowed as immutable.
  println!("{r2}");
  */
  // Move: r3 is invalidated as the original.
  let r4 = r3;
  /* r3 is invalid. Error: borrow of moved value: r3
  println!("{r3}");
  */
  *r4 = 6;
  //6
  println!("{r4}");

  // Immutable reborrow: Mutation is blocked.
  let r5 = &*r4;
  /* Error: cannot assign to *r4 because it is borrowed
   *r4 = 7;
   */
  //6, 6
  println!("{r4}, {r5}");
  // Mutable reborrow: r4 is blocked as the original.
  /* Equivalent to a implicit reborrow via the type annotation
  let r6: &mut i32 = r4;
  */
  let r6 = &mut *r4;
  /* Error: cannot borrow r4 as immutable because it is also borrowed as mutable
  println!("{r4}");
  */
  /* Error: cannot assign to *r4 because it is borrowed
   *r4 = 7;
   */
  *r6 = 7;
  //7
  println!("{r6}");
  // Reborrows r5 and r6 are no longer used, and original r4 is resumed.
  *r4 = 8;
  //8
  println!("{r4}");

  let mut guess = String::new();
  // Implicit reborrow for passing params
  io::stdin()
    .read_line(&mut guess)
    .expect("Failed to read line");
  // Resumed
  println!("{guess}");
}
```
