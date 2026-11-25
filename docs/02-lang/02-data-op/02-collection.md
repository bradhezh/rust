---
title: 2.2.2. Collections and Iteration
---

Collections, including vectors (`Vec<T>`), strings (`String`), and hash maps (`HashMap<K, V>`), are types holding heap values. In memory, elements are organised in contiguous buffers with flexible lengths, rather than scattered as linked nodes, to maximises CPU cache locality and avoids pointer-chasing overheads. Under the hood, `String` is a wrapper around a vector of UTF-8 bytes (`Vec<u8>`).

### Vectors

```rust title="Vectors"
fn vector() {
  let v: Vec<i32> = Vec::new();
  //[]
  println!("{v:?}");
  let v = vec![0, 1, 2];
  //[0, 1, 2]
  println!("{v:?}");
  let mut v = Vec::new();
  v.push(0);
  v.push(0);
  //[0, 0]
  println!("{v:?}");

  // Get Option<&T>, None for out-of-bounds, and `unwrap` panics for None.
  let r0 = v.get(0).unwrap();
  //0
  println!("{r0}");
  // Get Option<&mut T>, None for out-of-bounds, and `unwrap` panics for None.
  let r1 = v.get_mut(1).unwrap();
  *r1 = 1;
  /* Violate simultaneous references rules although r0 and r1 point to different elems
  println!("{r0}");
  */
  //[0, 1]
  println!("{v:?}");

  // Auto-deref returned references, panic for out-of-bounds
  v[0] = v[1];
  //1, 1
  println!("{}, {}", v[0], v[1]);
}
```

### Strings

```rust title="Strings"
fn string() {
  let mut s = "a".to_string();
  // `char` is Unicode with 4 bytes.
  s.push('😻');
  s.push_str(", hello");
  // fn add(self, s: &str) -> String
  s = s + ", world";
  // Like println! (auto-ref) but return String
  s = format!("{s}!");
  //a😻, hello, world!
  println!("{s}");

  /* Error: the type str cannot be indexed by {integer}
  let a = s[0];
  */
  /* Slice on bytes, panic on part of any char's bytes, Error: end byte index 2 is not a char
   * boundary; it is inside '😻' (bytes 1..5 of string)
  let bytes = &s[1..2];
  */
  let bytes = &s[7..];
  //hello, world!
  println!("{bytes}");

  //heooo
  println!("{}", "hello".replace('l', "o"));
  s.replace_range(0..1, "Hello, ");
  //Hello, 😻, hello, world!
  println!("{s}");
  assert!(s.contains("ell"));
  assert!(s.contains('😻'));
  assert!(s.contains(&['a', 'e']));
  assert!(s.contains(|c: char| c.is_alphabetic()));
  assert!(!"😻💝".contains(|c: char| c.is_alphabetic()));
}
```

### Hash Maps

```rust title="Hash Maps"
use std::collections::HashMap;

fn map() {
  let mut m = HashMap::new();
  // Overwrite
  m.insert(String::from("key1"), 1);
  m.insert(String::from("key2"), 2);
  //{"key1": 1, "key2": 2}
  println!("{m:?}");
  // First get the Entry enum for the key, then insert the value only when that key doesn't exist,
  // always return `&mut T` to the final payload
  //1
  println!("{}", m.entry(String::from("key1")).or_insert(10));
  //10
  println!("{}", m.entry(String::from("key3")).or_insert(10));
  // Get Option<&V> (Option<&mut V> for get_mut)
  //Some(10)
  println!("{:?}", m.get("key3"));
  //0
  println!("{}", m.get("key4").copied().unwrap_or(0));
}
```

In modern languages, iteration acts as a pattern that data can support via iterators, which are used by `for...in`, adapters, and consumers to traverse yielded items. In Rust, iterators are specified by the `Iterator` trait, which types can implement to support iteration.

### `Iterator`

```rust title="Iterator"
struct Range {
  from: i32,
  to: i32,
}

impl Iterator for Range {
  type Item = i32;

  fn next(&mut self) -> Option<Self::Item> {
    if self.from > self.to {
      None
    } else {
      let current = Some(self.from);
      self.from += 1;
      current
    }
  }
}

fn iterate() {
  let rng = Range { from: 0, to: 5 };
  //0 //1 //2 //3 //4 //5
  /* Equivalent to
  // into_iter here returns `self`, to be explained later.
  let mut iter = rng.into_iter();
  while let Some(i) = iter.next() {
  */
  for i in rng {
    println!("{i}");
  }
}
```

In this example, the data itself serves as the iterator with its state changed during iteration. A more mature pattern uses iterator factories to separate data from its iterators, which serve as its cursors with independent "current position" states for multiple iterations. In Rust, iterator factories are specified in `IntoIterator` as the `into_iter` method, which takes `self`. There are typically three kinds of factories implemented on the same type. First, it is implemented for the type itself (`D<T>`), for iterators yielding items (`T`), moving the data if it follows move semantics. Second, it is implemented for immutable references to the type (`&D<T>`), for iterators yielding immutable references to items (`&T`), without moving the data. Last, it is implemented for `&mut D<T>`, for iterators yielding `&mut T`. In fact, `for` relies on iterator factories even for types directly implementing `Iterator` (as in that example), because the standard library provides a blanket `IntoIterator` implementation for all iterators by returning `self`.

The `Iterator` trait also provides consumers such as `sum`, `collect`, and `for_each`, which consume the cursor to the end, as well as adapters such as `map`, `filter`, and `enumerate`, which adapt the iterator into a new one. They all take `self` and typically move the original iterator. Alongside `into_iter` from `IntoIterator` for the type itself, types typically provide `iter` and `iter_mut` as well. These two methods serve as those factories for immutable or mutable reference iterators (`(&data).into_iter()` and `(&mut data).into_iter()`), used directly on the type rather than its references.

### Iterator Factories

In the previous example, because `Range` yields integers on the fly without stored items, mutable reference iterators are inapplicable, while immutable reference iterators can simply yield `i32` since `i32` follows `Copy`. To demonstrate the complete pattern, `Pair<T>` with two stored items is used.

```rust title="Pair<T> as Data with Stored Items"
#[derive(Debug)]
struct Pair<T> {
  first: T,
  second: T,
}
```

```rust title="Item Iterators"
// Cursor
struct PairIntoIter<T> {
  // Items can't be moved behind a reference to the data in `next(&mut self)` — note that you can
  // never take ownership via references and therefore can never move referenced data, so Option<T>
  // with `take` is used to swap items instead of `Some(self.pair.<item>)` moving them.
  pair: Pair<Option<T>>,
  current: usize,
}

// Over T
impl<T> Iterator for PairIntoIter<T> {
  type Item = T;

  fn next(&mut self) -> Option<Self::Item> {
    let result = match self.current {
      0 => self.pair.first.take(),
      1 => self.pair.second.take(),
      _ => None,
    };
    if self.current < 2 {
      self.current += 1;
    }
    result
  }
}

// For Pair<T>
impl<T> IntoIterator for Pair<T> {
  type Item = T;
  type IntoIter = PairIntoIter<T>;

  fn into_iter(self) -> Self::IntoIter {
    PairIntoIter {
      pair: Pair {
        first: Some(self.first),
        second: Some(self.second),
      },
      current: 0,
    }
  }
}

fn iterate() {
  let pair = Pair {
    first: String::from("hello"),
    second: String::from("world"),
  };
  //hello //world
  for s in pair {
    println!("{s}");
  }
  /* Moved, Error
  println!("{pair:?}");
  */
}
```

```rust title="Immutable Reference Iterators"
// Cursor
struct PairIter<'a, T> {
  pair: &'a Pair<T>,
  current: usize,
}

// Over &T
impl<'a, T> Iterator for PairIter<'a, T> {
  type Item = &'a T;

  // If the output has the same lifetime as the input, the output would be treated as a mutable
  // reborrow from the input for lifetime checking, because `&mut self` implicitly reborrows the
  // input. This would prevent `next` from being called on a mutable iterator reference again while
  // the previous output is still to be used, due to simultaneous mutable references and reborrows
  // rules. To avoid this, the trait specifies different lifetimes for the input and output:
  // fn next(&'1 mut self) -> Option<&'a T>
  fn next(&mut self) -> Option<Self::Item> {
    let result = match self.current {
      // &self.pair.first borrows `first` behind the `self` reference and therefore has the same
      // lifetime as `self` ('1), but with Copy it can be copied into another lifetime ('a).
      0 => Some(&self.pair.first),
      1 => Some(&self.pair.second),
      _ => None,
    };
    if self.current < 2 {
      self.current += 1;
    }
    result
  }
}

impl<T> Pair<T> {
  fn iter(&self) -> PairIter<'_, T> {
    PairIter {
      pair: self,
      current: 0,
    }
  }
}

// For &Pair<T>
impl<'a, T> IntoIterator for &'a Pair<T> {
  type Item = &'a T;
  type IntoIter = PairIter<'a, T>;

  fn into_iter(self) -> Self::IntoIter {
    self.iter()
  }
}

fn iterate() {
  let pair = Pair {
    first: String::from("hello"),
    second: String::from("world"),
  };
  //hello //world
  for s in &pair {
    println!("{s}");
  }
  //Pair { first: "hello", second: "world" }
  println!("{pair:?}");
}
```

```rust title="Mutable Reference Iterators"
// Cursor
struct PairIterMut<'a, T> {
  // Without Copy, `&mut self.pair.<item>` with lifetime '1 can't be copied into another one with
  // lifetime 'a in `next`, so Option<&'a mut T> with 'a used instead.
  pair: Pair<Option<&'a mut T>>,
  current: usize,
}

// Over `&mut T`
impl<'a, T> Iterator for PairIterMut<'a, T> {
  type Item = &'a mut T;

  fn next(&mut self) -> Option<Self::Item> {
    let result = match self.current {
      0 => self.pair.first.take(),
      1 => self.pair.second.take(),
      _ => None,
    };
    if self.current < 2 {
      self.current += 1;
    }
    result
  }
}

impl<T> Pair<T> {
  fn iter_mut(&mut self) -> PairIterMut<'_, T> {
    PairIterMut {
      pair: Pair {
        first: Some(&mut self.first),
        second: Some(&mut self.second),
      },
      current: 0,
    }
  }
}

// For `&mut Pair<T>`
impl<'a, T> IntoIterator for &'a mut Pair<T> {
  type Item = &'a mut T;
  type IntoIter = PairIterMut<'a, T>;

  fn into_iter(self) -> Self::IntoIter {
    self.iter_mut()
  }
}

fn iterate() {
  let mut pair = Pair {
    first: String::from("hello"),
    second: String::from("world"),
  };
  for s in &mut pair {
    s.push_str("!");
  }
  //Pair { first: "hello!", second: "world!" }
  println!("{pair:?}");
}
```

### Collection Iteration

```rust title="Collection Iteration"
fn collection_iter() {
  let mut v1 = vec![String::from("hello"), String::from("world")];
  let v2: Vec<_> = v1.iter().map(|x| format!("{x}!")).collect();
  //["hello", "world"], ["hello!", "world!"]
  println!("{v1:?}, {v2:?}");
  let v2: Vec<_> = v1.iter_mut().map(|x| x.push_str("!")).collect();
  //["hello!", "world!"], [(), ()]
  println!("{v1:?}, {v2:?}");
  let v2: Vec<_> = v1.into_iter().map(|x| format!("{x}!")).collect();
  //["hello!!", "world!!"]
  println!("{v2:?}");
  // Unlike the closure in `map` takes `Self::Item`, the closure in `filter` takes `&Self::Item`.
  let v2: Vec<_> = v2.into_iter().filter(|x| *x != "hello!!").collect();
  //["world!!"]
  println!("{v2:?}");
  //3
  println!("{}", vec![0, 1, 2].iter().sum::<i32>());

  // `char`s
  //😻 //💝
  for c in "😻💝".chars() {
    println!("{c}");
  }
  // Bytes
  //240 //159 //152 //187 //240 //159 //146 //157
  for b in "😻💝".bytes() {
    println!("{b}");
  }
  // Grapheme clusters need 3rd-party crates, not provided by std.

  let mut m = HashMap::new();
  for word in "hello world wonderful world".split_whitespace() {
    let count = m.entry(word).or_insert(0);
    *count += 1;
  }
  //hello: 1 //world: 2 //wonderful: 1
  for (k, v) in m {
    println!("{k}: {v}");
  }
}
```
