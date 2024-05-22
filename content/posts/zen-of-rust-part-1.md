+++
title = 'Zen of Rust'
date = 2024-05-21
draft = false
+++
# Rust for Pythonistas

---
### The Zen of... Rust?
Does rust follow the zen of python? Does it do it well? In this series we'll take a look at some of rust's most salient features and why you, as a pythonista, may come to really vibe with some of rust's Zen. 🦀 (come on, doesn't that crab look chill af?)

---
### The Zen of Python

```Long time Pythoneer Tim Peters succinctly channels the BDFL’s guiding principles for Python’s design into 20 aphorisms, only 19 of which have been written down.```[^1].

If you open up a python shell and type `import this` and hit enter, you will be greeted with Tim Peters' Zen of python.

```
The Zen of python, by Tim Peters

Beautiful is better than ugly.
Explicit is better than implicit.
Simple is better than complex.
Complex is better than complicated.
Flat is better than nested.
Sparse is better than dense.
Readability counts.
Special cases aren't special enough to break the rules.
Although practicality beats purity.
Errors should never pass silently.
Unless explicitly silenced.
In the face of ambiguity, refuse the temptation to guess.
There should be one-- and preferably only one --obvious way to do it.
Although that way may not be obvious at first unless you're Dutch.
Now is better than never.
Although never is often better than *right* now.
If the implementation is hard to explain, it's a bad idea.
If the implementation is easy to explain, it may be a good idea.
Namespaces are one honking great idea -- let's do more of those!
```

Kateryna Koidan explains how to "translate these beautifully written principles into actionable insights" in her blog post, [What is the Zen of python?](https://learnpython.com/blog/zen-of-python/).

## Beautiful is better than ugly.
“What you do, the way you think, makes you beautiful.” ― Scott Westerfeld, Uglies

Beautiful code is in the eye of the next person who has to maintain it a year after you've left.
Ugly code wakes up an SRE at 1:46am on a Wednesday.

I've written a lot of production python code. I've maintained far, far more. Readable code that is easy to follow is easy to debug and easier to maintain. This is important for python applications, as many bugs are only discovered at runtime, and often when under load.

But even the cleanest code can be defeated by unexpected data formats, missing fields, etc. Unless you take very deliberate measures to catch and handle exceptions, it can be near impossible to track down the source of intermittent crashes, spurious bugs, and incomplete execution paths.

While python lends itself to getting from idea to working code quickly, it does so with some deep trade offs that I wish we were more upfront about as a community. I don't intend to give much attention to the pain points of python in production, as I'd really rather show how painless rust can be. And while not always the most beautiful at first glance, I think you will come to appreciate the thought that went into making coding in rust truly delightful.

## Explicit is better than implicit

Rust is an explicitly typed language, like C/C++, Java, etc. The syntax is very similar to python type hints. However, rust strictly enforces types and will not compile if there are any mis-matched types in your program.

```rust
// this is a string literal.
let var: str = "the quick brown fox";

// this is how you create an empty List[int] in rust, except they're called Vectors.
let nums: i32 = Vec::new();

// functions take typed parameters
// this checks if 2 is in a vector and returns some index, or returns none
fn has_two(nums: Vec<i32>) -> Option<i32> {
    for i, num in enumerate(nums) {
        if num == 2 {
            return Some(i)
        }
    }
    None
}
```

The `Option` type in rust is pretty special. It's an `enum` type with two variants, `Some(T)`, and `None`. You see, in rust there is no null. The None enum variant shouldn't be confused with python's None, which is essentially just another way to say null. In rust it is fundamentally different, it's a variant of a concrete type.

Why does this matter? Well, it enables some extremely powerful patterns for flow control and error handling.

Consider the case of looking for a key in a dictionary. Rust calls dictionaries `Hashmaps`.

```rust

let s = Hashmap::from([("key", "val"), ("foo", "bar")]);

// foo is a valid key in the map so value is going to get an Option::Some type returned to it
let value = s.get("foo");

match value {
    Some(a) => println!("{}", a),
    None => println!("foo not found")
}

// baz isn't a valid key in this map, so value will get an Option::None
let value = s.get("baz");

match value {
    Some(a) => println!("{}", a),
    None => println!("baz not found")
}
```
output:
```
bar
baz not found
```

What happens if we leave out one of these cases, or `match arms` as we call them in rust?
```rust
let value = s.get("baz");

match value {
    Some(a) => println!("{}", a),
}
```
```
error[E0004]: non-exhaustive patterns: `None` not covered
   --> src/main.rs:9:11
    |
9   |     match value {
    |           ^^^^^ pattern `None` not covered
    |
note: `Option<&&str>` defined here
   --> /Users/piper/.rustup/toolchains/stable-aarch64-apple-darwin/lib/rustlib/src/rust/library/core/src/option.rs:570:1
    |
570 | pub enum Option<T> {
    | ^^^^^^^^^^^^^^^^^^
...
574 |     None,
    |     ---- not covered
    = note: the matched value is of type `Option<&&str>`
help: ensure that all possible cases are being handled by adding a match arm with a wildcard pattern or an explicit pattern as shown
    |
10  ~         Some(a) => println!("{}", a),
11  ~         None => todo!(),
    |

For more information about this error, try `rustc --explain E0004`.
error: could not compile `rust-examples` (bin "rust-examples") due to 1 previous error
```

Uh, oh! We got a compilation error! It's telling us that we haven't covered `None`.

```
help: ensure that all possible cases are being handled by adding a match arm with a wildcard pattern or an explicit pattern as shown
    |
10  ~         Some(a) => println!("{}", a),
11  ~         None => todo!(),
```

The compiler tells us how to fix our code. We have to cover the None pattern, but we can use the todo! macro to let the compiler know we intend to implement this later[^2].

Rust has a similar enum type called `Result`. Where `Option` is used when an action may return no value, `Result` is used when an action can produce an error. It also has two variants, Ok(T), and Err(E).

```rust
use std::fs::File;
use std::io::Read;

fn main() {
    // open a file
    let file = File::open("some.txt");
    match file {
        Ok(file) => {
            // read the file...
        },
        Err(err) => {
            println!("failed to open file: {err}")
        }
    }
}
```

Because `match` requires all variants of an enum to be handled, you're naturally forced to handle errors as well as the optimistic cases.

This also gives rise to a delightful (or what I think is a delightful) pattern for message handling that prevents you from having those "oops I forgot to handle that one weird message type!!" moments.

```rust

enum Message {
    Connect(String),
    Disconnect(String),
    Publish(String, Post),
    Subscribe(String),
}

struct Post {
    body: String,
    // .. additional fields omitted
}

fn handle_message(msg: Message) -> Result<(), std::io::Error> {
    match msg {
        Message::Connect(broker) => {
            connect(broker);
            Ok(())
        }
        Message::Disconnect(broker) => {
            disconnect(broker);
            Ok(())
        }
        Message::Publish(topic, post) => {
            publish(topic, post);
            Ok(())
        }
        Message::Subscribe(topic) => {
            subscribe(topic);
            Ok(())
        }
    }
}
fn connect() {
    todo!()
}
fn disconnect() {
    todo!()
}
fn publish(topic: String, post: Post) {
    todo!()
}
fn subscribe(topic: String) {
    todo!()
}

```

As you can see, we handle every message variant, and if we didn't we'd get a similar compiler error as before.
We've also established a simple api contract, as any message must be of one of the variants and every variant must be handled.
Of course this is an incredibly over-simplified example, but by simply accepting or returning a single concrete type, we've prevented a large number of potential runtime bugs from malformed messages or unhandled message types. That's peace of mind. That is Zen.


## Simple is better than complex.

If you've enjoyed what you're read so far, please come back for part 2.


---
footnotes
[^1]: From the abstract of [(PEP 20)](https://peps.python.org/pep-0020/).
[^2]: Note, todo! is just shorthand for panic! with a specific message: “not yet implemented”
