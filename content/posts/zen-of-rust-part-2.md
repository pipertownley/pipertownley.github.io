+++
title = 'Zen of Rust (Part 2)'
date = 2024-07-15
draft = false
+++
# Rust for Pythonistas

---
### Part 2

Hi! Thanks for coming back for the second part of Rust for Pythonistas, where we've been exploring how Rust upholds the Zen of Python.

In Part 1 we touched on the first tenant of The Zen of Python, `Beautiful is better than ugly`, before taking a bit of a deeper dive into `Explicit is better than implicit`. In this part, we won't waste anymore time with introductions or recaps because...

## Simple is better than complex

From the earliest days of computer programming, managing complexity has been a significant challenge. Simplified models of complex systems allow us to utilize these systems without constantly dealing with intricate details.

However, when we need to modify behavior, enhance efficiency, or troubleshoot issues, having a deep understanding of the underlying systems is invaluable.

One of the most challenging aspects of programming is dealing with memory allocation and deallocation, which can lead to issues like memory leaks, dangling pointers, and buffer overflows.

A major attraction of Python is that you can write very complex programs without ever thinking about memory management. This is great for quickly delivering new features. However, this convenience comes with significant tradeoffs in runtime performance, which can lead to increased infrastructure costs. (And decreased site reliability, but that's a rant for another time.)

Rust, on the other hand, requires the developer to consider memory usage; however, its model is designed to be straightforward and safe. The first layer of safety is that variables are immutable unless you explicitly declare them as mutable. This fundamental principle reduces unintended side effects and enhances code reliability.

Building on this, rust introduces the concept of ownership, which ensures that each piece of data has a single owner at any point in time. An "owner" in rust is the variable that holds the data. The data is automatically deallocated when the owner goes out of scope, eliminating the need for a garbage collector. This makes Rust both efficient and predictable in terms of performance.

For example:
```rust
fn main() {
    let s = String::from("hello"); // s is the owner of the string
    println!("{}", s);
} // s goes out of scope here, and the memory is automatically freed
```

In addition to ownership, Rust introduces the concepts of borrowing and lifetimes to further manage memory safely and efficiently.

Borrowing allows you to reference data without taking ownership of it.

There are two types of borrowing:

Immutable borrowing: you can read the data, but not modify it.
```rust
fn main() {
    let s = String::from("hello");
    let s_ref = &s; // s_ref borrows s immutably
    println!("{}", s_ref); // s_ref can read the data
    // s_ref cannot modify the data
}
```
Mutable borrowing: you can both read and modify the data.
```rust
fn main() {
    let mut s = String::from("hello");
    let s_ref = &mut s; // s_ref borrows s mutably
    s_ref.push_str(", world"); // s_ref can modify the data
    println!("{}", s_ref);
}
```
You can borrow immutable references as many times as you like, but mutable references are like the Highlander; there can be only one. These are key rules in Rust's borrowing system:

1. You can have multiple immutable references to a piece of data at the same time.
2. You can have exactly one mutable reference to a piece of data at a time.
3. You cannot have both a mutable and an immutable references to the same data at the same time.

These rules ensures that when you have a mutable reference to data, you have exclusive access to that data. This also ensures that when you read data, its not going to change out from underneath you.

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s; // no problem
    let r2 = &s; // no problem
    println!("{} and {}", r1, r2);
    // r1 and r2 are no longer used after this point

    let r3 = &mut s; // OK
    println!("{}", r3);
}
```
This code compiles because the immutable references r1 and r2 are moved into println's scope and are deallocated when println returns.

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s; // no problem
    let r2 = &s; // no problem
    let r3 = &mut s; // BIG PROBLEM

    println!("{}, {}, and {}", r1, r2, r3);
}
```

This would result in a helpful compiler error:

```rust
error[E0502]: cannot borrow `s` as mutable because it is also borrowed as immutable
 --> src/main.rs:6:14
  |
4 |     let r1 = &s; // no problem
  |              -- immutable borrow occurs here
5 |     let r2 = &s; // no problem
6 |     let r3 = &mut s; // BIG PROBLEM
  |              ^^^^^^ mutable borrow occurs here
7 |
8 |     println!("{}, {}, and {}", r1, r2, r3);
  |                                -- immutable borrow later used here

```

Another critical aspect of Rust's memory management is the concept of lifetimes. Lifetimes are a way for Rust to track how long references to data are valid.

Rust's concept of lifetimes might seem daunting at first, but it's a fundamental part of how Rust ensures memory safety. Lifetimes track how long references to data are valid, preventing dangling references that can lead to undefined behavior.

```rust
fn main() {
    let r;
    {
        let x = 5;
        r = &x;
        println!("r: {}", r);  // This works because x is still in scope
    }
    // println!("r: {}", r);  // This would cause an error because x is out of scope
}
```

In this example, because `r` is defined in the main scope, it's lifetime is the same as main. But `x` is defined inside an inner scope of main. When the inner scope finishes, `x` will be automatically dropped.
This is

Let's uncomment that last line and see what error we get.

```rust
error[E0597]: `x` does not live long enough
 --> src/main.rs:5:13
  |
4 |         let x = 5;
  |             - binding `x` declared here
5 |         r = &x;
  |             ^^ borrowed value does not live long enough
6 |         println!("r: {}", r);  // This works because x is still in scope
7 |     }
  |     - `x` dropped here while still borrowed
8 |     println!("r: {}", r);  // This would cause an error because x is out of scope
  |                       - borrow later used here

For more information about this error, try `rustc --explain E0597`.
```

Like before, `x` is dropped when the inner scope finishes, but `r` still holds a reference to `x`! In C/C++ we'd have a null reference at runtime, but Rust catches this for us when we try to compile.

Rust is usually able to infer lifetimes for you, but sometimes Rust won't be able to make assurances about lifetimes. This is often the case when a function takes a reference and also returns a reference. We can use lifetime annotations as a way to tell the Rust compiler that the returned reference will be valid as long as the input reference is valid.

In this example, the `first_word` function takes a string slice with an explicit lifetime `'a` and returns a string slice with the same lifetime `'a`.

```rust
fn main() {
    let string1 = String::from("hello");
    let result = first_word(&string1);
    println!("The first word is '{}'", result);
}

fn first_word<'a>(s: &'a str) -> &'a str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
```

## Complex is better than complicated.
Stay tuned for part 3!
