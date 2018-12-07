---
layout: post
title: "The Swiss Army Knife of Hashmaps"
date: 2018-12-07 22:28:46 +0530
comments: true
categories: [Coding, Rust]
---

A while back, there was [a discussion](https://github.com/rust-lang/rust/issues/55514) comparing the performance of using the [hashbrown crate](https://github.com/Amanieu/hashbrown) (based on [Google's SwissTable](https://www.youtube.com/watch?v=ncHmEUmJZf4) implementation[^1]) in the Rust compiler. In the [last RustFest](https://rome.rustfest.eu/), Amanieu was [experimenting on integrating](https://github.com/rust-lang/rust/pull/56241) his crate into stdlib, which turned out to have [some really promising results](https://twitter.com/Gankro/status/1067816516652015616). As a result, it's being planned to move the crate into stdlib.

While the integration is still ongoing, there's currently no blog post out there explaining SwissTable at the moment. So, I thought I'd dig deeper into the Rust implementation to try and explain how its (almost) identical twin `hashbrown::HashMap` works.

<!-- more -->

## Hashing and Hashmaps

In order to establish some terminology, I'm gonna start from scratch. If you know about hashing, hashmaps, open-addressing and cache performance in general, feel free to [skip this section](#robin-hood-hashing).

While arrays and linked lists hold a sequence of items, hashmaps (or tables) hold key/value pairs i.e., they bind *values* to *keys*. So, you insert a value for a key and you can later address those values (fetch/remove/whatever) **using the same key**.

*Hashing* is basically computing a special number (the *hash*) for some *hashable* object, provided that if two objects are equal, their hashes *must* be equal. When a *hash function* produces the same hash for two different objects, we call it a *hash collision*. A perfect hash function doesn't result in collisions, but since we're not in an ideal world, collisions happen (the *rate* depends on the algorithm).

Hashmaps use arrays as their backend. In the context of hashmaps, the array elements are called *buckets* or *slots* (I'll be using this interchangeably). Let's take the following array:

```
 index |  0  |  1  |  2  |  3  |
-------|-----|-----|-----|-----|
 value |     |     |     |     |
```

> The *value* here is the array's element in that index (usually, it's the key/value pair as a whole!).

This array has 4 slots (it could be just one!). We wish to insert a key/value pair `(4, 8)`, for which we use a hash function `H(x)`. In order to find the index at which the key/value pair should be inserted, the **key is hashed** using the hash function `H(K)`, and the index is obtained by modulo'ing the hash using the length of the array.

```
H(4) = 12638164110811449565
i = H(4) % 4 = 3
```

For the sake of keeping this post simple, I've used an *unspecified* hash function[^2] (so don't worry about it) - only the actual hashes matter as far as we're concerned. And, note that the hash is 64 bits - we'll be sticking with this size throughout the post. Again for simplicity, we're only focusing on 64-bit machines.

Anyway, now that we've found the index of the bucket, let's insert the key/value pair. Our array now looks like this:

```
 index |  0  |  1  |  2  |  3  |
-------|-----|-----|-----|-----|
 value |     |     |     | 4,8 |
```

But, why did we have to add both the key and value `(4, 8)`, instead of just the value `8`?

### Dealing with collisions

Right now, for performing an operation on a key/value pair in the map, we *find* it by following the same steps - hash the key, modulo `N`, land on the index and perform the desired action.

Hashing functions, despite that they have a whole 8 bytes, encounter hash collisions. By doing the *modulo* operation, we've greatly reduced their range. Let's try inserting `(8, 2)` into our map;

```
H(8) = 12638161911788193143
i = H(8) % 4 = 3
```

But, we already have an element at index 3!

In order to deal with these collisions, we group similar data together. One way is to assign a linked list to **each bucket**. Now, our map will look like:

```
 index |  0  |  1  |  2  |  3  |
-------|-----|-----|-----|-----|
 value |     |     |     | |o| |
                          __|__
                         | 4,8 |
                         |-----|
                         | 8,2 |
                         -------
```

... and when we wish to find an element, we stop at the bucket, traverse through the linked list and **compare the keys** for locating the value. This method of using another data structure for storing the values in each bucket is called *separate chaining*. So, if we keep getting collisions for subsequent insertions, our linked list will get bigger and that will impact our performance, right?

Not exactly. This is where we talk about the *load factor*. Just like all other dynamic data structures, hashmaps should be able to resize at will! Load factor is the ratio of the number of elements in the hashmap to the number of buckets. Once we reach a certain load factor (say, 50%, 70% or 90% depending on your configuration), hashmaps will resize and *rehash* all the key/value pairs.

Let's say that our hashmap *doubles* in size at a load factor of 60%. This means, once we add a third element `(5, 7)`, our new hashmap will resize (regardless of whether it's colliding). It will now look something like:

```
 index |  0  |  1  |  2  |  3  |  4  |  5  |  6  |  7  |
-------|-----|-----|-----|-----|-----|-----|-----|-----|
 value | |o| |     |     | |o| |     |     |     | |o| |
        __|__             __|__                   __|__
       | 5,7 |           | 4,8 |                 | 8,2 |
       -------           -------                 -------
```

The choice of the load factor depends entirely on the internals of your map i.e., how it hashes and determines buckets. For example, you have a weaker hash function which results in adding a number of elements to the same bucket, and in order to reduce traversal, you'd probably go for a resize / rehash when you hit a *relatively lesser* load factor (the choice is based on a gazillion performance tests!).

However, we have a major performance bottleneck. Firstly, the usage of external data structures require additional allocations / pointers which themselves consume some memory **per element**. And second, when it comes to linked lists, they have worse *processor* cache performance.

Okay, what does that mean?

### CPU Cache

Our RAM is faster than say, our SSD (like, a hundred times!), but it's also a hundred times slower than the CPU[^3]. That's why CPUs have their own caches (L1, L2, L3, etc.). Data flows from memory to the CPU cache in fixed sized blocks called *cache lines*. This fetch can take up to a 100 ns (~200-300 clock cycles). In contrast, L1 cache reference is ~1-2 ns, whereas for L2, it's ~8-12ns (CPU caches are hierarchical, think of L1 as close to the CPU and smaller in size compared to L2).

What this means is that, whenever the CPU needs to read/write to a memory location, it checks the cache(s), and if it's present, it's a *cache hit*, otherwise it's a *cache miss*. Whenever a cache miss occurs, we pay the cost of fetching the data from main memory (thereby losing a few hundred CPU cycles by **waiting**).

Coming back to linked lists, the pointers of subsequent nodes could be anywhere, which results in fetching cache lines randomly. This indirection leads to the poor cache performance of linked lists.

### The other way

The second way (for our hashmap) is to get rid of using external data structures completely, and use the same array for storing values alongside buckets. With another element `(12,9)`, our map will look like this:

```
 index |  0  |  1  |  2  |  3  |  4   |  5  |  6  |  7  |
-------|-----|-----|-----|-----|------|-----|-----|-----|
 value | 5,7 |     |     | 4,8 | 12,9 |     |     | 8,2 |
-------|-----|-----|-----|-----|------|-----|-----|-----|
 slot  |  0  |     |     |  3  |  3   |     |     |  7  |
```

> I've added another row to indicate the buckets/slots computed from the hashes.

Note that the new key/value pair is on index 4, even though its slot index is 3. This way, we sequentially fill the empty slots. Also, the ordering is unnecessary.

Let's try and add a few more elements...

```
 index |  0  |  1   |  2  |  3  |  4   |  5  |  6  |  7  |
-------|-----|------|-----|-----|------|-----|-----|-----|
 value | 5,7 | 29,7 | 6,6 | 4,8 | 12,9 | 3,4 |     | 8,2 |
-------|-----|------|-----|-----|------|-----|-----|-----|
 slot  |  0  |  0   |  1  |  3  |  3   |  2  |     |  7  |
```

As you can see, `(3, 4)` has a slot index of 2, but since that index already has an element, we're inserting it into the first empty element we find (which is at 5). During fetching, we land on a slot (based on the hash), compare the keys one by one, and traverse all the way until we either **find a matching pair or an empty slot** (*probing*).

Here's the catch. When you remove an element, you can't simply remove the key/value pair from a slot and be done with it. If you do that, then you've created an empty slot, which could result in the map complaining existing values to be non-existent (because the search has encountered an empty slot). So, you have two choices:

1. In addition to removing a pair, we also move the next pair to that slot (shift backwards).[^4] For example, if we removed `(4, 8)` in the above map, then we move `(3, 4)` to its position (which has the lowest slot index), and the table will now look like:

```
 index |  0  |  1   |  2  |  3  |  4   |  5  |  6  |  7  |
-------|-----|------|-----|-----|------|-----|-----|-----|
 value | 5,7 | 29,7 | 6,6 | 3,4 | 12,9 |     |     | 8,2 |
-------|-----|------|-----|-----|------|-----|-----|-----|
 slot  |  0  |  0   |  1  |  2  |  3   |     |     |  7  |
```

2. Or, you add a special flag (*tombstone*) to the removed slots, and when you probe, you skip the slots containing that flag. But, this will also affect the load factor of the hashmap - removing a number of elements in a map followed by inserting a few could trigger a *rehash*.

There are other ways to resolve hash collisions in open addressing (*quadratic probing*, where we quadratically address the elements instead of sequentially, and *double hashing* where we use another hash function to land on the slot), but they're out of scope of this post.

The advantage of linear probing is that it has the best cache performance - if you think about it, linear probing sequentially visits the elements, which means, (most often, depending on the size of the object) the data is part of a cache line, which is great, because we don't waste CPU cycles (in contrast, quadratic probing doesn't offer such cache performance and double hashing uses another hash function, which by definition means that we're spending more work on computing another hash).

Although, we do have a problem with openly addressed maps (linear probing in particular) - *clustering*. Whenever a collision occurs, we start queueing the key/value pairs. For example, in the above table, because `(29, 7)` has the same slot as `(5, 7)`, it's at index 1, and we had to place `(6, 6)` at 2 (even though its slot index is 1), and other elements follow.

This way, as the table grows, more elements get pushed away from their actual index (which elongages the search/probe sequence, thereby increasing the cache misses). It also depends on the hash function, in that, a weaker hash function can lead to more collisions, and hence, more clusters.

## Robin Hood hashing

I'm also going through Robin Hood hashing, because it's a nice optimization and at the time of writing of this post, Rust used this in `std::collections::HashMap`, but again if you know this, feel free to [skip this section](#hashbrown). It has nothing to do with hashbrown itself.

This is one of the few methods to improve the overall efficiency of openly addressed hashmaps. It offers a way to redistribute an existing key/value pair during the insertion of a new pair. For example, let's take this *openly addressed* map:

```
 index |  0  |  1   |  2  |  3  |  4   |  5  |  6  |  7  |
-------|-----|------|-----|-----|------|-----|-----|-----|
 value | 5,7 | 29,7 | 6,6 | 3,4 | 12,9 |     |     | 8,2 |
-------|-----|------|-----|-----|------|-----|-----|-----|
 slot  |  0  |  0   |  1  |  2  |  3   |     |     |  7  |
```

We're interested in a variable called *distance to the actual slot* - I'll call it `D(value)`. For example, the actual slot of `(5, 7)` is 0, and it's located at 0, so the distance is 0. For `(29, 7)` on the other hand, the distance is 1, because it's at 1, even though it should've been at 0.

In robin hood hashing, you follow one rule - if the distance to the actual slot of the current element in the slot is *less than* the distance to the actual slot of the element to be inserted, then we swap both the elements and proceed.

Okay, that was ugly. An example will really help us.

We'd like to insert `(13, 3)` into this map. The slot index for 13 is 0. So, let's start at 0. We already have `(5, 7)` located there, which also has a slot index 0.

```
 index |  0  |  1   |  2  |  3  |  4   |  5  |  6  |  7  |
-------|-----|------|-----|-----|------|-----|-----|-----|
 value | 5,7 | 29,7 | 6,6 | 3,4 | 12,9 |     |     | 8,2 |
-------|-----|------|-----|-----|------|-----|-----|-----|
 slot  |  0  |  0   |  1  |  2  |  3   |     |     |  7  |
          ^
          | D(13) == 0
        (13,3)
```

Both have the same distances i.e., `D(5) == D(13) == 0`. We move forward.

```
 index |  0  |  1   |  2  |  3  |  4   |  5  |  6  |  7  |
-------|-----|------|-----|-----|------|-----|-----|-----|
 value | 5,7 | 29,7 | 6,6 | 3,4 | 12,9 |     |     | 8,2 |
-------|-----|------|-----|-----|------|-----|-----|-----|
 slot  |  0  |  0   |  1  |  2  |  3   |     |     |  7  |
                ^
                | D(13) == 1
              (13,3)
```

Still the same. `D(29) == D(13) == 1` (both are 1 slot away from their actual slots). Moving on...

```
 index |  0  |  1   |  2  |  3  |  4   |  5  |  6  |  7  |
-------|-----|------|-----|-----|------|-----|-----|-----|
 value | 5,7 | 29,7 | 6,6 | 3,4 | 12,9 |     |     | 8,2 |
-------|-----|------|-----|-----|------|-----|-----|-----|
 slot  |  0  |  0   |  1  |  2  |  3   |     |     |  7  |
                       ^
                       | D(13) == 2
                     (13,3)
```

In the next stop, we see something. `D(13) == 2` whereas `D(6) == 1` and hence `D(6) < D(13)`, which means we should **swap the pairs** and proceed.

```
 index |  0  |  1   |  2   |  3  |  4   |  5  |  6  |  7  |
-------|-----|------|------|-----|------|-----|-----|-----|
 value | 5,7 | 29,7 | 13,3 | 3,4 | 12,9 |     |     | 8,2 |
-------|-----|------|------|-----|------|-----|-----|-----|
 slot  |  0  |  0   |  0   |  2  |  3   |     |     |  7  |
                       ^
                       | D(6) == 1
                     (6, 6)
```

Now, we go looking for a place to insert `(6, 6)`. This way, we compare and swap the key/value pairs based on their distances. In the end, we have this nicely distributed hashmap:

```
 index |  0  |  1   |  2   |  3  |  4  |  5   |  6  |  7  |
-------|-----|------|------|-----|-----|------|-----|-----|
 value | 5,7 | 29,7 | 13,3 | 6,6 | 3,4 | 12,9 |     | 8,2 |
-------|-----|------|------|-----|-----|------|-----|-----|
 slot  |  0  |  0   |  0   |  1  |  2  |  3   |     |  7  |
```

Since the hashmap has reached almost its capacity, you might be wondering whether this would've already triggered a *rehash*. But, the resize/rehash depends on the load factor, and because this method redistributes the key/value pairs regardless of when they get inserted (**takes away from the rich and gives it to the poor**, hence the name), the hashmaps could now have higher load factors of even 90-95%.

This also brings another improvement to searching. We don't have to probe all the way until we find an empty slot. We can stop when `D(slot value) < D(query value)`, since our rule guarantees that this shouldn't happen for the key we're looking for. For example, in the above table, if we wanna query for the key `21` (whose slot index is 0), then we can stop at index 3, because at that point `D(6) == 2` which is less than `D(21) == 3` which wouldn't have happened if the key/value pair were there. So, we can safely declare that the key doesn't exist.

Now that we've grazed over a lot of things associated with openly-addressed hashmaps[^5], let's proceed to hashbrown.

## Hashbrown

I'm not gonna call it "SwissTable" from here on because firstly, even though hashbrown was a port of SwissTable, the author has made a few changes to improve its performance, and second, I didn't read the C++ code at all - I followed the `hashbrown` crate.

An optimization for linear probing is storing some metadata for each key/value pair. The first thing that comes to mind is the check for *equality* - once we've landed on a slot, we probe by comparing the keys in the slots one by one. While this is easier for integers, it gets expensive for bigger objects. We could store the hash, but, that's another **8 bytes per slot**, which is a huge deal for memory-eating hashmaps!

Let's reset our map and make some changes to its behavior.

1. Let's make the initial size of the internal array to **16** (I'll get into why we're doing that in a bit, trust me for now). We call this group of 16 elements a *group*.[^6] So, a map is made of a number of consecutive groups.
2. For each slot in a group, let's assign **a byte for metadata** and call it *control byte*. Again, we'll see what it is soon.

It will now look something like this:

```
 index |   0    |   1    |   2    |   3    |   4    | ... |   16   |
-------|--------|--------|--------|--------|--------|     |--------|
 value |        |        |        |        |        | ... |        |
-------|--------|--------|--------|--------|--------|     |--------|
 ctrl  |00000000|00000000|00000000|00000000|00000000| ... |00000000|
```

I'm skipping a number of elements (irrelevant to us) so that it fits to the screen. I'm also representing the "control byte" in bits, because we'll be playing with bits. As a result, there are 128 bits in this group - a nice round number (we'll see why).

Our first candidate for insertion is `(5, 7)`.

```
H(5) = 12638147618137026400
```

Taking the top (*most significant*) 7 bits of this hash[^7] and calling it `H2(x)`, we get:

```
H2(5) = H(5) >> 57 = 87 = 0b1010111
```

These will be the *bottom* 7 bits of our control byte.[^8] Then, we use a special bit for our own purposes (to indicate whether the slot value is empty, full or deleted) and this will be the top bit. The states are now represented as follows:[^9]

```
0b11111111  // EMPTY (all bits are set)
0b10000000  // DELETED (top bit is set)
0b0.......  // FULL (whenever the top bit isn't set)
```

In light of this information, all the slots are empty, so our map will look like:

```
 index |   0    |   1    |   2    |   3    |   4    | ... |   16   |
-------|--------|--------|--------|--------|--------|     |--------|
 value |        |        |        |        |        | ... |        |
-------|--------|--------|--------|--------|--------|     |--------|
 ctrl  |11111111|11111111|11111111|11111111|11111111| ... |11111111|
```

To recall what we've done so far, we're storing the top 7 bits of our key's hash in our control byte, and in addition to that, we use the top bit of the control byte to indicate whether the slot is full, empty or deleted.

Going back to our candidate `(5, 7)`, its slot index is 0 i.e., `H(5) % 16 == 0`.

```
 index |   0    |   1    |   2    |   3    |   4    | ... |   16   |
-------|--------|--------|--------|--------|--------|     |--------|
 value | (5,7)  |        |        |        |        | ... |        |
-------|--------|--------|--------|--------|--------|     |--------|
 ctrl  |01010111|11111111|11111111|11111111|11111111| ... |11111111|
```

Once the pair is inserted, we've also set its control byte to `H2(5)`, since the top bit is zero anyway (because it's now `FULL`). Now, let's try inserting `(39, 8)`.

```
H(39) = 17050702200253021726
i = H(39) % 16 = 2
H2(39) = H(39) >> 57 = 54 = 0b110110
```

And, we do the same thing.

```
 index |   0    |   1    |   2    |   3    |   4    | ... |   16   |
-------|--------|--------|--------|--------|--------|     |--------|
 value | (5,7)  |        | (39,8) |        |        | ... |        |
-------|--------|--------|--------|--------|--------|     |--------|
 ctrl  |01010111|11111111|00110110|11111111|11111111| ... |11111111|
```

Now, we're all set. Let's start addressing the reasons behind whatever we've done.

### Why the round number?

Each group contains 16 slots summing up to 8 control bytes. The first natural question is why we've restricted the group size to 128 bits.

When we query for a key in the map, we first use the hash to land on the group corresponding to a slot, find the offset of the slot inside that group and start probing by comparing against the 7-bit hashes in the control byte.

The boost here is that the control bytes (being 128 bits) can fit into an L1 cache line. This means, we can probe an *entire* group really quickly, before having to fetch another group from L2 or L3 or whatever. And, we don't have to worry about comparing the keys at all, until we encounter all 7 bits matching in a control byte.

There's one other cool optimization for modern processors. Modern CPUs support SIMD instructions, which basically means that we can do some operation (add or multiple or compare, etc.) on multiple values at the same time in a processor! [SSE](https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions) is a subset of that where we can work on different types such as two 64-bit floats, four 32-bit integers or **sixteen 8-bit integers**.

Now, our workflow will simply be:

1. [Load up](https://doc.rust-lang.org/nightly/core/arch/x86/fn._mm_set1_epi8.html?search=_mm_load%20si128) these bytes from an array.
2. [Set the byte](https://doc.rust-lang.org/nightly/core/arch/x86_64/fn._mm_set1_epi8.html) to compare.
3. [Compare](https://doc.rust-lang.org/nightly/core/arch/x86_64/fn._mm_cmpeq_epi8.html) both the values.
4. [Mask](https://doc.rust-lang.org/nightly/core/arch/x86_64/fn._mm_movemask_epi8.html) the values from comparison to true or false.

And, that's finding the results from 16 slots in **four CPU instructions!**

In the above example, if we wish to find `39`, then all we have to do is find the position of the group using the hash, find the 7-bit value `0b110110` from the hash, and do this:

```
1. Load group (A)

---------------------------------------------     ------------
| 01010111 | 11111111 | 00110110 | 11111111 | ... | 11111111 |
---------------------------------------------     ------------

2. Set comparable 0b110110 (B)

---------------------------------------------     ------------
| 00110110 | 00110110 | 00110110 | 00110110 | ... | 00110110 |
---------------------------------------------     ------------

3. Compare A and B

---------------------------------------------     ------------
| 00000000 | 00000000 | 11111111 | 00000000 | ... | 00000000 |
---------------------------------------------     ------------
                       (success!)
4. Mask values

---------------------------------------------     ------------
|    0     |    0     |    1     |    0     | ... |    0     |
---------------------------------------------     ------------
                         (true)
```

After masking, we actually get an integer, because the final result for each group is either `0` or `1`, which could all be accumulated into an integer. In other words, the value and position of each bit in the returned integer corresponds to a match of a slot in the group.

So, we have the results! Now, all we need to do is check the indices of bits that are set in the final integer, compare the key(s) in those slots against the querying key (for equality), find the corresponding value, and we'll land on `(39, 8)`.

### Hints to compiler!

Futher optimizations can be done on this implementation. If we've used a good hash function that distributes the bits reasonably well, then we can hint the compiler that the final equality check (for the key) will almost always be true. This directly aids [branch prediction](https://en.wikipedia.org/wiki/Branch_predictor). In Rust, we have [`likely`](https://doc.rust-lang.org/nightly/core/intrinsics/fn.likely.html) and [`unlikely`](https://doc.rust-lang.org/nightly/core/intrinsics/fn.unlikely.html) to achieve this. So, we can [tell the compiler](https://github.com/Amanieu/hashbrown/blob/6b9cc4e01090553c5928ccc0ee4568319ee0ed33/src/raw/mod.rs#L666) that the equality is *likely* to be true.

The next hint is on whether we should probe to the next group looking for that key. Again, if our hash function is good enough, then the odds of that happening is very *very* low.[^10] So, we can [hint the compiler again](https://github.com/Amanieu/hashbrown/blob/6b9cc4e01090553c5928ccc0ee4568319ee0ed33/src/raw/mod.rs#L670) that it's *likely* to stop probing.

When we remove a key/value pair, we can follow the tombstone method - we query the key as usual, find the slot (in some group), set a tombstone (by marking the control byte as `DELETED`) and later mark it back as `EMPTY` (when we resize, for example). But, we could take advantage of the previous fact and say that if the group had at least one empty element, then we don't have to add a tombstone. We could simply set it to `EMPTY`, because the probing is gonna stop with this group anyway (because the probing would've encountered an `EMPTY`).

Amanieu has added more stuff like making the hashmap efficient for 32-bit platforms, supporting maps smaller than a group, and a ton of other features to make it compatible with `std`, which is pretty cool!

I hope you found this post interesting. As always, feel free to drop any comment if you have anything to add.

---

<small>A huge thanks to [Ana](https://github.com/Hoverbear) and [Amanieu](https://github.com/Amanieu) for reviewing this post!</small>

[^1]: I insist on watching this talk when you have some free time!

[^2]: Although, hashbrown uses FX hash.

[^3]: [What Every Programmer Should Know About Memory](https://akkadia.org/drepper/cpumemory.pdf).

[^4]: This is what Rust used in its Robin hood hashing implementation.

[^5]: I haven't talked about a number of improvements that could be made to Robin Hood hashing - backward shifting entries during deletion (instead of tombstones), caching hashes, slot indices or "distances" to improve probing, etc. all improve the performance of the hashmap.

[^6]: This doesn't mean that a hashmap that contains, say 2 elements *should* have a 16 element array (for the values) - only the group has 16 elements, the actual array containing the key/value pairs will still be 2 elements.

[^7]: In the SwissTable implementation, the bottom (*least significant*) 7 bits were used for `H2`, but Amanieu has claimed that his choice lead to slightly more efficient code.

[^8]: Again, this is how hashbrown is implemented. SwissTable used the top 7 bits.

[^9]: One more Rust-specific enhancement - SwissTable had `kSentinel` to deal with C++ iterators, which wasn't required in Rust.

[^10]: According to the talk, it'd take a load factor of 94% for an ideal map to reach that situation.
