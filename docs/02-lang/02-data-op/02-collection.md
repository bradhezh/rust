---
title: 2.2.2. Collections
---

Collections, including vectors (`Vec<T>`), strings (`String`), and hash maps (`HashMap<K, V>`), are all types holding heap values.

Vectors are more of dynamic arrays rather than linked lists in memory layouts.

### Vectors

```rust title="Vectors"

```

String: Vector of UTF-8 encoded bytes (`Vec<u8>`)

```rust title="Strings"
let data = "initial contents";
// From the Display trait
let s = data.to_string();
// The method also works on a literal directly:
let s = "initial contents".to_string();
let s = String::from("initial contents");
s.push('l'); // 4 bytes Unicode as scalar values of `char`
s.push_str("bar");

let s1 = String::from("Hello, ");
let s2 = String::from("world!");
// fn add(self, s: &str) -> String
let s3 = s1 + &s2; // note s1 has been moved here and can no longer be used
let s1 = String::from("tic");
let s2 = String::from("tac");
let s3 = String::from("toe");
let s = s1 + "-" + &s2 + "-" + &s3;
// Auto-refs params like println! but returns String
let s = format!("{s1}-{s2}-{s3}");

/* Error: the type `str` cannot be indexed by `{integer}`
let h = s[0];
*/
// Bytes, panic on part of a char's bytes, e.g. Error: end byte index 1 is not a char boundary;
// it is inside <some-char> (bytes 0..2 of string)
let s = &hello[0..1];
// `char`s
for c in "Зд".chars() {
    println!("{c}");
}
// Bytes
for b in "Зд".bytes() {
    println!("{b}");
}
// Grapheme clusters needs 3rd-party crates, not provided by std.
contains, replace
```

```rust title="Hash Maps"
use std::collections::HashMap;
let mut scores = HashMap::new();
// Overwrite
scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);
// The return value of the entry method is an enum called Entry that represents a value that might
// or might not exist. The or_insert method on Entry is defined to return a mutable reference to the value for the corresponding Entry key if that key exists, and if not, it inserts the parameter as the new value for this key and returns a mutable reference to the new value.
scores.entry(String::from("Yellow")).or_insert(50);
let team_name = String::from("Blue");
// get method returns an Option<&V>; copied to get an Option<i32> rather than an Option<&i32>;
// unwrap_or to set score to zero if scores doesn’t have an entry for the key.
let score = scores.get(&team_name).copied().unwrap_or(0);
for (key, value) in &scores {
  println!("{key}: {value}");
}

let text = "hello world wonderful world";

    let mut map = HashMap::new();

// split_whitespace method returns an iterator over subslices
    for word in text.split_whitespace() {
        let count = map.entry(word).or_insert(0);
        *count += 1;
    }
```
