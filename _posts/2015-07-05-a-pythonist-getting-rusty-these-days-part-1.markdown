---
layout: post
title: "A Pythonist getting Rusty these days... (Part 1)"
date: 2015-07-05 20:39:06 +0530
comments: true
categories: [Python, Rust]
---

Though I've been playing with Python and JS for a while, getting into systems programming is something I've always wanted to do. The increased talks about Rust in the IRC (over the last few months), followed by the release of Rust 1.0 gave me a kickstart, which dragged me into it about two weeks back.

I got to see the beauty of Rust (thanks to the [wonderful book](http://doc.rust-lang.org/book/)) - its syntax, static type system, vast compile-time checking, etc., especially how it tackles the problem of memory safety by introducing a new concept called **ownership**, and it does all those without the use of a garbage collector. I liked it immediately!

Today, I'll try to explain some things I liked about the language. The upcoming posts are reserved for some serious topics like FFI and concurrency, and how I got around some of the worst situations I experienced (as a newbie who just got into the systems programming world).

<!-- more -->

## What I'd been doing?

*If you're just interested only in the code, please feel free to skip this part.*

It was quite hard for me at first, to understand how Rust worked. So, I decided to translate one of my Python works to Rust - a private diary which I'd written in Python[^1] about an year back.

As I got to know more and more about Rust, I began to look for ways to make my code more *efficient*. Honestly, I had to rewrite my entire diary to achieve that. I changed the data structure, the encryption scheme, the options it provided, etc., that one day it got quite big, that I had to move it to [a new repository](https://github.com/wafflespeanut/biographer).

At this point, I realized that translating the diary is probably a bad idea. Luckily, there was one thing I could offer. My diary supported *searching* through the encrypted stories. This is where scripting languages (like Python) suck, in the sense that they take too much time. So, I decided to integrate Rust into my diary, by writing a library (which can search at lightning speed!).

To achieve that, I had two things in mind - [FFI](http://en.wikipedia.org/wiki/Foreign_function_interface) and concurrency, which Rust handled quite *nicely*. That's where my life as a *Rustacean* began.

# Some things I liked...

*I liked a lot of things when I was making the library for my diary, which I'll elaborate in this post. We'll get into FFI and concurrency in future posts (because I've got a lot of stuff in those topics).*

The only thing that resembles C++ is how we call the methods or traits from modules (like `std::thread`). The **syntax** for all the other things are very different and they're the ones which impressed me at the first place. As an example, let's write a function for Caesar cipher.

In Python, we have this simple (yet amusing) comprehension,

``` python
def shift(text, amount):        # and, we have a 256-keyspace Caesar cipher
    return ''.join((chr((ord(char) + amount) % 256) for char in text))
    # assert shift('hello', 5) == 'mjqqt' and shift('mjqqt', 256-5) == 'hello'
```

In the case of Rust, I decided to work with bytes rather than strings (as it seemed inefficient)[^2]. Anyways, Rust doesn't support comprehension, but the equivalent version is somewhat nice.

``` rust
fn shift(text_bytes: &[u8], amount: u8) -> Vec<u8> {
    text_bytes.iter() // wrap around the boundary if the sum overflows
              .map(|byte| amount.wrapping_add(*byte))
              .collect()
} // we don't have to say `collect::<Vec<u8>>()` explicitly, thanks to Rust's type inference
```

I'm just iterating over the bytes, mapping a function over the values using a closure `|byte|` and finally collectng everything into a vector of integers. Notice the absence of `;` at the end of the expression. Since the function has a return type denoted by `->` (`Vec<u8>` in this case), Rust automatically *infers* this as the return value of the function.

Also, note that we've specified the types of each argument, which brings me to my next favorite thing - **static type system**. Rust expects me to specify the type of the argument (whether it's a `u8` or `&str` or `Vec<T>`), including the result (if any) of the function. This was a bit annoying, but it clearly tells that the coder has to know what types they're playing with. If you're not entirely sure of the type, then there's an elegant *workaround*.

Rust does **compile-time checking**, so that it reduces a program's runtime as much as possible. So, we can take advantage of this feature by deliberately introducing an error, so that it *shouts* the type for us when we try to  compile. Now, let's try adding some *nonsense* to our function,

``` rust
fn shift(text_bytes: &Vec<u8>, amount: u8) -> Vec<u8> {
    text_bytes.iter()   // wrap around the boundary if the sum overflows
              .map(|byte| amount.wrapping_add(*byte))
              // .collect()
              .blah()   // note the `blah`
}
```

Obviously, Rust doesn't compile! Instead, it shows a nice helpful error...[^3]


    Compiling demo v0.0.1 (file:///home/wafflespeanut/Desktop/demo)
    src/lib.rs:104:10: 104:16 error: no method named `blah` found for type
    `core::iter::Map<core::slice::Iter<_, u8>, [closure src/lib.rs:102:14: 102:47]>`
    in the current scope
    src/lib.rs:104         .blah()
                            ^~~~~~
    error: aborting due to previous error
    Could not compile `demo`.

    To learn more, run the command again with --verbose.


It tells us that the type `core::iter::Map<core::slice::Iter<_, u8>, [closure ...]>` doesn't implement our `blah` method (by which it's letting us know about the exact type), and it also points to the exact location where it has occurred (the same goes for warnings, which can come out of camel cases, deprecated APIs, unused variables, etc.). These errors were very helpful for me, especially when I'm not sure about how something should be done in Rust.

Apart from that, some things were much similar to (but more powerful than) Python, that sometimes I felt Rust's style as high-level. For example, we can unpack tuples and iterate over things just like we do in Python,

``` python
(x, y, z) = (1, 2, 3)   # sequence unpacking
for i in [3, 5, 7]:     # iterating over a list
    print i
```

In Rust, `let` does the same job, and `in` works more or less the same (in the sense, that it calls one of the [iterators](https://doc.rust-lang.org/std/iter/index.html#the-three-forms-of-iteration), just like the `iter()` in Python). Well, you can't search with the `in` operator like you do in Python, but *meh, how much can you expect from a systems language?*

``` rust
let (mut x, y, z) = (1, 2, 3);  // same unpacking
for i in &[3, 5, 7] {           // iterating directly over a slice
    println!("{}", i);
}
```

Now, there's a `mut` in `x`, and you can also notice the `&` operator in the slice, which brings us to the ultimate weapons of Rust - **mutability** and **ownership**. These are the things that allow Rust to guarantee memory safety[^4]. By default, variable bindings are *immutable* in Rust. In order to modify the binding in any way, you should specify it as `mut`, so that the binding becomes mutable, or else you can't change its value.

Regarding ownership, the concept is that the scope in which a variable is born is its owner. But, unlike other languages, we can transfer the ownership of a scope, so that even the scope (who was once the owner) can no longer access the variable once it's been *moved*. Here's an example,

``` rust
fn own(some: Vec<u8>) { }

fn main() {
    let (x, y) = (vec![1, 2, 3], vec![1, 2, 3]);
    let take = x;       // `take` is the owner of `x`
    own(y);             // `own` function is the owner of `y`
    println!("{:?}", (x, y));   // `println!` can't access both, and we get an error
}
```

I've used a heap-allocated vector here, because string, integers, etc. implement the [`Copy`](https://doc.rust-lang.org/std/marker/trait.Copy.html) trait by default, so that even if you try to move those types, only the ownership of the `Copy` is transferred and so your types are unaffected.

This is why I've always proceeded with caution when throwing *non-copyable* types (like vectors) here and there. I don't *move* the values unless it's absolutely necessary. Even then, I carry those inside a function, so that I can return some type back and compensate for the previous owner's *loss*. For example, here's something, which I call a *"string extractor"*.

``` rust
fn extract(bytes: Vec<u8>) -> String {  // requires a consumable vector, not a reference!
    String::from_utf8(bytes).unwrap()
}

let x = vec![97, 98, 99];
// let x = extract(&x);     // doesn't work!
let x = extract(x);         // `String` compensation for the owner who once owned a vector
```

The `from_utf8` method consumes the vector and returns a `String`. So, I can't do something like taking a pointer to a vector, because (obviously) a pointer is different from a vector, which (like I said) is checked by Rust in compile-time.

In Rust, things work quite differently when it comes to references (technically, it's called **"borrowing"**), which could be done either mutably or immutably. [Rust has two rules](http://doc.rust-lang.org/book/references-and-borrowing.html#the-rules) when it comes to borrowing,

> A variable binding can have only one of the following:

> - a single mutable reference
> - any number of immutable references

It's reasonable if you think about it. It's a great way to prevent data races. That said, we can achieve most of the things using immutable references like `&Foo` (for some type `Foo`). Now, what if you want read/write access to some value across thread boundaries? Rust has a lot of helpful and powerful APIs in its standard library. For example, for this particular problem, [`Arc<T>`](https://doc.rust-lang.org/std/sync/struct.Arc.html) and [`Mutex<T>`](https://doc.rust-lang.org/std/sync/struct.Mutex.html) can be used to execute the operations safely.

As far as I've seen, Rust offers various kinds of interesting options to easily get around most of the complex situations. For all the high-level coders out there, if you've always wanna get into systems programming, then I suggest "Rust" as a nice start!

<small>Thanks to Manish for reviewing this post and thanks to all the *Rustaceans* who helped me whenever I got into the [#rust](https://botbot.me/mozilla/rust/) channel at IRC. For more, see the [discussion about this post](https://www.reddit.com/r/rust/comments/3c7r1l/a_pythonist_getting_rusty_these_days/) on Reddit.</small>

[^1]: It *was* a very basic thing. It just wrote the stories into files, and my encryption scheme was nothing but a hexed shifting cipher. It was pathetic [until about a month back](https://github.com/wafflespeanut/scripts/blob/8850c831c10955b5c32d2710abfbfef916031792/Memorandum/Diary.py).

[^2]: Rust strings are valid UTF-8 by default, and so [`from_utf8`](https://doc.rust-lang.org/std/string/struct.String.html#method.from_utf8) method is likely to throw an error if the bytes are invalid. Also, Rust offers a [file-reader](https://doc.rust-lang.org/std/io/trait.Read.html) for reading the contents as bytes. So, I decided to handle everything (including the search) using bytes rather than strings.

[^3]: Honestly, Rust is also my first language where I felt *happy* about errors, because I knew (almost immediately) what I should do next to prevent it.

[^4]: Actually, memory safety won't be a problem in these trivial cases, but it's much more important while handling threads. For instance, you can't allow two threads to write over the same data. One of Rust's prime goals is to eliminate such [data races](https://en.wikipedia.org/wiki/Race_condition). If you're more interested in that, [Manish explains about it](http://manishearth.github.io/blog/2015/05/30/how-rust-achieves-thread-safety/) in one of his posts.
