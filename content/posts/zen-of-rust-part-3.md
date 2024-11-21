+++
title = 'Zen of Rust (Part 3)'
date = 2024-11-20
draft = true
+++
# Rust for Pythonistas

---
### Part 3

Welcome back! I took a pretty long break but I'm glad to be back and I'm glad you're back. Let's dive right back in where we left off last time.

## Complex is better than complicated.
Rust has features to help keep complex code from being complicated, one of the most notable is zero-cost abstractions. This means you don't really need to be too concerned about the performance impacts of how you structure your code. The Rust compiler will generally choose the right optimization paths. This allows you to focus on writing well structured, maintainable code instead.

The basic building blocks of Rust's zero-cost abstractions we'll cover here are Traits, Generics, and Closures.

Traits are very similar to the concept of _interfaces_ in other languages. They define methods, that when implemented for a specific type, add new behaviors to that type.


Let's look at an example trait from the Rust Book[^1]:
Here we're defining a simple trait called Summary.
```rust
pub trait Summary {
    fn summarize(&self) -> String;
}
```
Traits can have one or more method signatures that describe the behaviors of the types that implement them. For a trait to be considered implemented for a type, the compiler will enforce that the type must implement all methods of the trait, accepting the same input types, and returning the same output types as in their signatures defined by the trait.

Taking another example from the Rust Book:
Here we have two distinct types, NewsArticle and Tweet, with an implementation of the Summary trait for both.
```rust
pub struct NewsArticle {
    pub headline: String,
    pub location: String,
    pub author: String,
    pub content: String,
}

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}

pub struct Tweet {
    pub username: String,
    pub content: String,
    pub reply: bool,
    pub retweet: bool,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("{}: {}", self.username, self.content)
    }
}
```

We can now use the _summarize()_ method for either of these types to get a string summarizing the data contained in each type.

```rust
let article = NewsArticle {
    headline: String::from("Awesome Things Happening"),
    location: String::from("Colorado"),
    author: String::from("Me"),
    content: String::from("Seriously, just wait until you read about ... ")
}

let tweet = Tweet {
    username: String::from("pipertownley@github.com"),
    content: String::from("Rust makes running Python in production feel irresponsible. Let's fight."),
    reply: false,
    retweet: false
}

println!("{}", article.summarize());
println!("{}", tweet.summarize());
```
Output:
```
Awesome Things Happening, by Me (Colorado)
pipertownley@github.com: Rust makes running Python in production feel irresponsible. Let's fight.
```

Cool, I hear you say, but how is using a trait different than simply implementing the method for the type?
That's a great question, because you can certainly do this:

```rust
pub struct NewsArticle {
    pub headline: String,
    pub location: String,
    pub author: String,
    pub content: String,
}

impl NewsArticle {
    fn summarize(&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}
```
Notice here we didn't use `Summary for` and instead we're implementing the method on the type directly. This is totally fine when all we're dealing with are concrete types. However, the real power of traits comes into play when we want to make generic code that can handle multiple types.

This is where Trait Bounds enters the chat.

Continuing with the example from the Rust Book we used earlier, let's say we want to aggregate both news article and tweet summaries together into a single list.

As we saw in Part 1, a list in Rust is called a Vector and vectors need to be declared with the type they're going to hold, so the compiler knows how much memory each item will need on the stack.

We could simply create our vector to accept a String. But then any kind of string could be placed into the vector, not just summaries. If this vec comes from a library or is used in many different places in a large code base, then it is certainly possible to misuse it by putting in arbitrary strings.

This may not seem like a big deal and its really not in this specific example, but if some of those strings happen to contain sensitive information, it could become one.

Besides, we have a much better way to handle this problem.

```rust
pub struct Feed {
    pub summaries: Vec<String>
}

impl Feed {
    fn new() -> Self {
        Self {
            summaries: Vec::new()
        }
    }

    fn add_summary<T: Summary>(&mut self, item: T) {
        self.summaries.append(item.summarize())
    }
}

let mut feed = Feed::new()

feed.add_summary(tweet)
feed.add_summary(article)

for summary in feed.summaries {
    printf!("{}", summary")
}
```

Let's a take a look at the first part of the signature for add_summary, we have some new weird syntax `<T: Summary>(..., item: T)`.
These are our trait bounds. This is how we tell the compiler that this function takes any type `T` that implements the `Summary` trait.

Together, traits and their bounds allow for powerful generic programming patterns that keep your code clean and easier to maintain.

Another feature of Rust the uses traits as its foundation are `Closures`. Closures in Rust are similar to lambda functions in Python, or an arrow function in Javascript. It's essentially an anonymous function that can capture its environment.

Simple closures in Rust:
```rust
// A basic closure that adds two numbers
let add = |a, b| a + b;
println!("5 + 3 = {}", add(5, 3));

// Closure capturing environment variable
let multiplier = 2;
let multiply = |x| x * multiplier;

 println!("7 * 2 = {}", multiply(7));
```

Rust builds this feature on top of three generic traits that are part of the std lib. These traits are called `Fn`, `FnMut`, and `FnOnce`.

When the Rust compiler processes a closure, it analyzes it, the environment it captures, and its return type. It then implicitly creates a struct and implements any of those three traits that are needed for this particular closure. These traits add the respective methods `call`, `call_mut`, and `call_once` to the resulting struct. Rust takes our anonymous function, and turns it into a struct containing all the captured data, as well a call to its corresponding method. This is all turned in to static function calls, entirely automatically, giving you the same performance, compile time checks, and memory safety behind Rust's growing popularity.

Now that we've covered Traits, Generics, and Closures, I'd like to talk about `Iterators`, which are built on top of all three. Along with traditional loops like `loop` and `while`, Rust also supports Iterator Transforms. An iterator in Rust is a trait with a number of methods, but the two most important, and required features of implementing the Iterator trait is an associated type Item, that marks the type of item to be iterated over, and a `next` method, which advances the iterator to the next item in the collection.

Any collection that wants to allow iteration over a set of items can implement the Iterator trait. The Iterator trait uses trait bounds internally to provide a large collection of generic methods automatically, that allow for interacting with a collection in a _functional_ manner. Some of the most useful are `for_each`, `filter`, `map`, and `reduce` methods, which all accept a closure as an argument.

```rust
let numbers = vec![1, 2, 3, 4, 5];

// These two examples are functionally the identical.
numbers.iter().for_each(|&x| println!("{}", x))

// Really, there is no performance difference, it compiles to the same instructions.
for number in numbers {
    println!("{}", number);
}
```

Wrapping up, we explored how Traits, Generics built on Trait Bounds, and Closures are the building blocks of Rust's zero-cost abstractions.
These powerful features allow for building other powerful features, such as Iterators, and allow you to structure complex behavior without complication.

Just a note before we go, in order to keep the time between posts more reasonable, I'm going to start combining more Zen of Python tenants into each part. It wasn't really my intent to create a new post for each tenant, but some simply take more space to explore and others will not be enough to justify their own posts going forward. So stay tuned for Part 4.

## Flat is better than nested.
...and probably a few others as well.

---
footnotes
[^1]: [(The Rust Programming Language a.k.a The Rust Book)](https://doc.rust-lang.org/stable/book/).
