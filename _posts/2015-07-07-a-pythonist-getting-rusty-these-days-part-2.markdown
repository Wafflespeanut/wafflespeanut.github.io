---
layout: post
title: "A Pythonist getting Rusty these days... (Part 2)"
date: 2015-07-08 18:17:58 +0530
comments: true
categories: [Python, Rust]
---

Last time, I talked about [my experience with Rust as a newbie]({{site.url}}/2015/07/05/a-pythonist-getting-rusty-these-days-part-1/). Today, I'll try to explain the hard time I had with the [FFI](http://en.wikipedia.org/wiki/Foreign_function_interface). Well, I shouldn't have gone into it, but I needed that for communication with Python[^1].

FFI was *hard*, because I can't quite guess what happens along the way. Errors don't mean much, which means that I can get the results only during runtime (and then figure out whether it's the cake I want or not). Then, there's this *interesting* problem of memory safety. Rust is specifically designed to be memory safe, but most of the other languages (especially "C") aren't. Python, being a descendant of C, can only speak "C" at its low level. So, Rust *has* to speak C if it wants to communicate with Python.

<!-- more -->

This means that there's no simple way of sending a list from Python and getting it in Rust. Both have entirely different data structures. And, we don't have to say about Rust - its clockworks are *crazier* than C or Python. But, thanks to Rust's "libc" and Python's "ctypes" libraries, which enable both the languages to communicate in C.

# Establishing the Rust-Python FFI

My script was a bit *cranky*. In the Python side, we have a list of file paths, which can be sent as an array of strings. Things got somewhat complicated here. To send an array, I should create one using the `ctypes`, grab its address (a pointer) and send it along with its length.

In Rust, I already have a setup like this...

``` rust
extern crate libc;      // because, the standard library is unstable

use std::slice;
use libc::{size_t, c_char};

#[no_mangle]            // "Please don't mangle the names, my dear Rust!"
pub extern fn get_stuff(array: *const *const c_char, length: size_t) {
    let array = unsafe { slice::from_raw_parts(array, length as usize) };
    // do some stuff with the array
}
```

... so that, I could link it to Python like this,

``` python
import ctypes

lib = ctypes.cdll.LoadLibrary("test.so")    # load the compiled library
list_to_send = ['blah', 'blah', 'blah']
lib.get_stuff.argtypes = (ctypes.POINTER(ctypes.c_char_p), ctypes.c_size_t)

c_array = (ctypes.c_char_p * len(list_to_send))(*list_to_send)  # generate the array
lib.get_stuff(c_array, len(list_to_send))   # sending stuff to our Rust library
```

Python's side is pretty much self-explanatory - just take a list, declare the argument types, form an array from the list and send its reference, along with its length. But, there are some interesting things going on at Rust's side. I'm sure you're aware of `pub extern`, which allows a function to be publicly called and especially from "C".

Now, there's an `unsafe`. Like I said previously, Rust guarantees memory safety, but it doesn't know whether the data from an FFI is safe or not. So, we're supposed to use an `unsafe` block to tell Rust that everything outside this `unsafe` block is *truly* safe indeed, and that it should kindly lower its safety measures a little bit, so that we can put some *unsafe* data inside and take its responsibility.[^2]

The arguments indicate an array as a whole (which is understandable). `*const c_char` is a string in C, and so `*const *const c_char` is a pointer to an array of strings of length `size_t`, from which the data is extracted into a Rust-type slice using `from_raw_parts`.

Now that we've got a slice of **C-type** strings, we gotta convert them to Rust-type strings - either `&str` or `String`. For that, we can do some nice things with the `map`,

``` rust
use std::str;
use std::ffi::CStr;

let mut stuff: Vec<&str> = array.iter()
                                .map(|&p| {
                                    let c_str = unsafe { CStr::from_ptr(p) };
                                    let byte = c_str.to_bytes();
                                    str::from_utf8(byte).unwrap()
                                }).collect();
```

We simply iterate through the slice, get the C-type string, convert it to bytes, from which we can then extract our UTF-8 strings.[^3]

Okay, so we have a vector of `&str`, alright. Now, I do some stuff - in this case, I iterate through the filenames, grab the contents from them, decrypt the bytes, search through them, count the occurrences of a word in those files and collect the values. Then, I should send it back to Python.

Since the counts are just a number (an `u8`), I just collect them as a `String`, then `join` them, convert the Rust-type `String` to a `CString` pointer and send it to python.

In Rust, we now have...[^4]

``` rust
use std::mem;
use std::ffi::CString;

pub extern fn get_stuff(array: *const *const c_char, length: size_t) -> *const c_char {
    // some operation here that finally gives us a data stream `count_string`
    let c_string = CString::new(count_string).unwrap();
    let raw_ptr = c_string.as_ptr();    // get the pointer and forget the object
    mem::forget(c_string);              // wheeeee... leak everything away...
    raw_ptr
}
```

The `as_ptr` method takes an *immutable* reference and constructs a `CString` object. Then, I've used `mem::forget` because the `c_string` is owned by Rust. Even if we transfer the pointer, we can't be sure if it's been freed on Rust's side. So, we ask Rust to cut it loose, thereby *purposely* leak the memory, so that the FFI code (Python, in this case) now owns the thing.

Or, we could do something else. The nightly version offers an `into_raw` method, which consumes the `CString` without deallocating the memory. So, we could also use that...

``` rust
use std::ffi::CString;

pub extern fn get_stuff(array: *const *const c_char, length: size_t) -> *const c_char {
    // all the code here
    CString::new(count_string).unwrap().into_raw()
}
```

I personally think this is better, because there's another method which does the opposite (which we'll be needing soon). Or, if you just wanna stick to the stable version (*and* avoid memory leaks), then you've got no other choice, but to wait for a while, so that those methods become stable.

Anyway, let's get back into business. Now, that `c_string` is sent back to Python as a pointer, Python receives it, gets the result from it, and resume doing whatever it's been doing. But, life's not that simple in FFI. There's a problem here. What happened to the pointer we've just received? It's just there (as a dump!). We can't simply free the pointer in Python (like I said, both Rust & Python have totally different implementations). So, that represents a terrible [**memory leak!**](https://en.wikipedia.org/wiki/Memory_leak)

The leak isn't a big problem for my application (the memory is gonna be flushed out by the OS sometime anyway), but it's still bad to have leaks. A good way is to return the pointer back to Rust, so that it can be freed properly. Only one (unstable) method offers this "proper freeing" of a pointer - `CString::from_raw`, which does the inverse of `CString::into_raw` (this is why I suggested using it in the first place).

Now, there's a little addition to the Rust code,

``` rust
use std::ffi::CString;

#[no_mangle]
pub extern fn kill_pointer(p: *const c_char) {
    unsafe { CString::from_raw(p) };
}
```

and my final Python code looks like this,

``` python
import ctypes

lib = ctypes.cdll.LoadLibrary("test.so")
list_to_send = ['blah', 'blah', 'blah']
# argument types should be mentioned while doing FFI in Python
# forgetting to do that has caused segfaults for me
lib.get_stuff.argtypes = (ctypes.POINTER(ctypes.c_char_p), ctypes.c_size_t)
lib.get_stuff.restype = ctypes.c_void_p
lib.kill_pointer.argtypes = [ctypes.c_void_p]

c_array = (ctypes.c_char_p * len(list_to_send))(*list_to_send)  # generate the array
# send the stuff to Rust library and get the pointer
c_pointer = lib.get_stuff(c_array, len(list_to_send))
count_string = ctypes.c_char_p(c_pointer).value     # get the string

lib.kill_pointer(c_pointer)     # send the pointer back for hack & slash!
```

In Python, I get the pointer, extract the string from it, and send it back to Rust. So, I (had to) create a function in Rust to just *kill* this ugly pointer. The `CString::from_raw` consumes the pointer and produces a `CString` for Rust, which then gets deallocated as the scope runs out of life, which means that the obtained `CString` should be destroyed.[^5] And, the processes are now memory safe!

## So much for the FFI. What are the perks?

Only one - *performance*. The decrypting & searching (of ~200 files) took about a minute in Python, whereas Rust did it in ~0.4 seconds!

When I got to see this *effect*, I decided (almost immediately) to use the Rust-Python FFI for almost all my computation-related works in the future. As an undergrad in aeronautics, I've got some ugly computations (for which I've always used proprietary software). Maybe it's time for me to start using open-source and do the works all by myself!

<small>**Sidenote:** I'm still entirely not sure whether this is efficient, or whether this is the proper way to do things in systems programming, so feel free to drop any suggestions. I welcome your *criticism* happily! Again, thanks to all the Rustaceans at the [#rust IRC channel](https://botbot.me/mozilla/rust/) for helping me out whenever I got stuck.</small>

<small>You can also check out the discussion on [HN](https://news.ycombinator.com/item?id=9853688) and [Reddit](https://www.reddit.com/r/rust/comments/3ckv6z/a_pythonist_getting_rusty_these_days_part_2/).</small>

[^1]: Of course, it's for that very same [diary](https://github.com/wafflespeanut/biographer) I'd talked about. I had to decrypt a lot of files and search through them (which Rust is pretty good at).

[^2]: Well, this also means that it's the coder's job to ensure that the code inside the `unsafe` block is actually safe.

[^3]: Thanks to [this wonderful answer](http://stackoverflow.com/a/31075375/2313792) from a Rustacean at Stackoverflow.

[^4]: I used `as_ptr` at first, but [it was suggested](http://stackoverflow.com/a/31083443/2313792) that `into_ptr` serves much better for that purpose (too bad that it's *unstable*).

[^5]: I had a chat about that in the #rust IRC channel, and some of them suggested that it's the only way, though I don't have a choice...
