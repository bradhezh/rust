---
title: 2.1.3. Smart Pointers
---

Although variables holding heap values operate similarly to unique smart pointers, and references can point to data without taking ownership, Rust still provides smart pointers to explicitly point to heap data with ownership. For instance, types cannot contain themselves as `struct Node { next: Node }`, while using references like `struct Node<'a> { next: Option<&'a Node<'a>> }` introduces unnecessary complexity to manage ownership, lifetimes, and mutation. Smart pointers with ownership remain the appropriate way to handle this use case — `struct Node { next: Option<Box<Node>> }`.

`Box<T>` acts as unique smart pointers. Apart from recursive types, it is commonly used to point to heap trait objects as `Box<dyn Trait>`, where `dyn Trait` represents all concrete types implementing `Trait`, and all objects of those types are regarded as objects of that trait. In Rust, traits specify virtual functions, so `dyn Trait` is not a generic type, while concrete pointers can be coerced to trait pointers, which are fat pointers with `data_ptr` and `vtable_ptr`. If `T` implements `Trait`, `&T` or `Box<T>` can be coerced to `&dyn Trait` or `Box<dyn Trait>` respectively, referred to as unsized coercion. The compiler sets `data_ptr` to the concrete pointer value and `vtable_ptr` to the static address of the virtual function table for `T` implementing `Trait`. Note that there is one virtual function table for each type implementing each trait, all determined at compile time. Both `&dyn Trait` and `Box<dyn Trait>` are used to dynamically bind virtual functions.

As mentioned earlier, the compiler applies deref coercion. Types implement the `Deref`/`DerefMut` trait to support deref coercion. If `T` implements `Deref<Target = D>`/`DerefMut<Target = D>`, `&T`/`&mut T` can be coerced to `&D`/`&mut D`. For example, `&String`/`&mut String` can be coerced to `&str`/`&mut str` because `String` implements `Deref<Target = str>`/`DerefMut<Target = str>`. Similarly, `Box<T>` implements `Deref<Target = T>`/`DerefMut<Target = T>`, so `&Box<i32>`/`&mut Box<i32>` can be coerced to `&i32`/`&mut i32`.

However, the compiler prioritises unsized coercion for the `&dyn Trait`/`&mut dyn Trait` destination. Consequently, `&Box<dyn Trait>`/`&mut Box<dyn Trait>` cannot be coerced to `&dyn Trait`/`&mut dyn Trait` via deref coercion, while treated as `&T` for unsized coercion requiring `T` (`Box<dyn Trait>`) to implement `Trait`. While deref coercion fails in this case, the `AsRef`/`AsMut` trait with the `as_ref`/`as_mut` method can be used for types to act as references. For example, `Box<T>` and `String` implement `AsRef`/`AsMut`, where `as_ref`/`as_mut` returns `&str`/`&mut str` for `String` and `&T`/`&mut T` (`&dyn Trait`/`&mut dyn Trait`) for `Box<T>` (`Box<dyn Trait>`). Additionally, since `Box<T>` is a pointer, `&*`/`&mut *` can be used directly on `Box<dyn Trait>` for the same effect.

On the other hand, `Box<dyn Trait>` with ownership cannot be created from `&dyn Trait` without ownership, unless the trait specifies a dedicated copy or clone method for types to implement. It can be created from concrete objects via `Box<T>::new(x: T) -> Self`, which moves (or copies, if `T` implements `Copy`) the object to the heap if it is on the stack, unless `T` is zero-sized.

### Trait Pointers

```rust title="Trait Pointers"
trait Speak {
  fn speak(&self);
}

struct Dog;

impl Speak for Dog {
  fn speak(&self) {
    println!("Woof!");
  }
}

struct Cat;

impl Speak for Cat {
  fn speak(&self) {
    println!("Meow!");
  }
}

fn converse(o: &dyn Speak) {
  o.speak();
}

fn trait_ptr() {
  let dog = Dog;
  let cat = Box::new(Cat);

  // &Dog -> &dyn Speak
  //Woof!
  converse(&dog);
  /* Error: the trait bound `Box<Cat>: Speak` is not satisfied
  converse(&cat);
  // Box<Cat> -> Box<dyn Speak + 'static> (here `+ 'static` means even if concrete objects contain
  // references, those references must outlive 'static)
  let speak: Box<dyn Speak> = cat;
  // Error: the trait bound `Box<dyn Speak>: Speak` is not satisfied
  converse(&speak);
  */
  // Box<Cat>::as_ref() -> &Cat -> &dyn Speak
  //Meow!
  converse(cat.as_ref());
  // &*Box<Cat> -> &Cat -> &dyn Speak
  //Meow!
  converse(&*cat);
}
```
