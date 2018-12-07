---
layout: post
title: "An easy bug in Stylo..."
date: 2016-10-05 08:30:34 +0530
comments: true
categories: [Servo, Coding, Rust]
---

While my [new job]({{site.url}}/2016/07/12/new-job-new-field/) demands writing backend tools in Rust, I get a lot of free time every once in a while, when I fiddle around Servo's code. Lately, I got interested in [Stylo](https://wiki.mozilla.org/Stylo).

Stylo is interesting enough for it to need a whole writeup about itself, but this post is just about an easy stylo bug, which then turned slightly *ugly*. Well, it's no big deal, since developers usually deal with this kind of thing every day, but since it's an easy bug, I thought it might give some ideas to the newcomers (to stylo) about where to look when hacking on stylo, and to keep pushing and **not give up** if an easy issue becomes less easy...

<!-- more -->

## "Stylo" in a nutshell!

There's parallel style code in Servo and sequential C++ code in Gecko. In stylo, we isolate Servo's style libraries and hook it up to Gecko (with a sleek FFI) and make it use that instead. Now, that's easier said than done, but once we have this integration, we can focus on pure style stuff, without having to worry about unimplemented layout/rendering stuff in Servo (since a Firefox build will provide feedback on how things are going).

Stylo contains [both Gecko and Servo](https://hg.mozilla.org/incubator/stylo) code. The workflow is somewhat difficult for a newcomer, because sometimes it demands submitting patches to both Gecko ([hg repo](https://hg.mozilla.org/mozilla-central)) and Servo ([git repo](http://github.com/servo/servo/)), dealing with codegen ([Mako](http://www.makotemplates.org/) for the glue code, and a version of [rust-bindgen](https://github.com/servo/rust-bindgen) for translating numerous C++ stuff to Rust), and finally testing them (when you build stylo and *manually* check whether your changes work).

## How it began...

A "good first bug" in stylo usually goes about changing (or adding) something in the glue code. Manish had scraped a few pages and put up a [list of CSS properties](https://manishearth.github.io/css-properties-list/), which comes in rather handy. As we can see, there are some stuff that are implemented in Servo, but not in Stylo. For those properties, the changes reside in the glue code (mostly)[^1], where we only have to get the computed values from Servo and **set** it on Gecko.

I'd done `border-spacing` a few days back. It was pretty easy, as it required nothing more than [copying some values](https://github.com/servo/servo/pull/13450/files) from Servo to Gecko. The next thing in my queue was `font-stretch`, which "looked" pretty similar.

The core principle behind style code is that each property belongs to a particular type! It's always a `struct` field (or a bunch of fields) in Gecko, whereas it'd be an `enum` or a `struct` in Servo.

`font-stretch` turned out to be [an enum](http://doc.servo.org/style/properties/longhands/font_stretch/computed_value/enum.T.html) in Servo, whereas it's a [16-bit signed integer](https://dxr.mozilla.org/mozilla-central/rev/ea104eeb14cc54da9a06c3766da63f73117723a0/gfx/src/nsFont.h#78) in Gecko. And, it wasn't straight-forward (like I thought it'd be).[^2] Whenever we encounter an enum, we can easily *cast it away* to an integer primitive. But, we can't do that here, because some of the [constants were negative](https://dxr.mozilla.org/mozilla-central/rev/ea104eeb14cc54da9a06c3766da63f73117723a0/gfx/thebes/gfxFontConstants.h#24-27). In order to keep things future-proof, we need the constants in Servo before we can do anything.

While we already have most of the types and values, importing a few more is pretty easy. Emilio had done a great job with bindgen, that we now have a [bunch of tools](https://github.com/servo/servo/tree/7914f14caabaa557c9f88130443ab77162c7072b/components/style/binding_tools) for generating the necessary bindings required for the glue code. So, simply including the file and adding the constants' pattern [should do it](https://dxr.mozilla.org/servo/rev/1a28907a8f3792b92cfbba9505d345c5ae796535/components/style/binding_tools/regen.py#51,77).

... or so I'd thought.

There was a [slight trouble with the bindgen](https://github.com/servo/servo/issues/13540) along the way, but once it got fixed, I could generate the bindings in no time. Surprisingly enough, bindgen then seemed to ignore constants with negative values. So, it was time to get into bindgen code.

## A bug within a bug...

Parsers never cease to impress me.[^3] As complicated as they look, they can never be perfect, and always have bugs! Rust bindgen is something that translates C++ code units to Rust (with support from `clang` libraries). It also has a parser. So, with the current scenario on hand, it's natural to assume that we're ignoring the negative values whenever we parse `#define` directives in C++ code.

Initial digging showed that [this](https://github.com/servo/rust-bindgen/blob/cfdf15f5d04d4fbca3e7fcb46a1dd658ade973cd/src/codegen/mod.rs#L1706) is where we filter the collected Rust constants (translated from `#define`) with respect to the whitelist of patterns, but throwing some `println!` there showed that those set of constants never even get there in the first place!

After more digging, [it turned](https://github.com/servo/rust-bindgen/blob/cfdf15f5d04d4fbca3e7fcb46a1dd658ade973cd/src/lib.rs#L333) [out that](https://github.com/servo/rust-bindgen/blob/cfdf15f5d04d4fbca3e7fcb46a1dd658ade973cd/src/ir/item.rs#L406) [we skip](https://github.com/servo/rust-bindgen/blob/cfdf15f5d04d4fbca3e7fcb46a1dd658ade973cd/src/ir/var.rs#L67) [parsing](https://github.com/servo/rust-bindgen/blob/cfdf15f5d04d4fbca3e7fcb46a1dd658ade973cd/src/ir/var.rs#L149) if we don't find an integer literal in a code unit. In our case, since parentheses and unary minus don't count as literals, they'd been neglected by the parser.

Finally, a patch [to bindgen](https://github.com/servo/rust-bindgen/pull/74) followed by another patch [for bindings regeneration](https://github.com/servo/servo/pull/13566) was enough for making [my actual glue code patch](https://github.com/servo/servo/pull/13570) to work.

This is what I like about Stylo. It's hard, interesting, new and definitely **not** straight-forward.[^4] So, I'm planning to keep fiddling around it for a while...

[^1]: Mostly... but not necessarily. It could also mean that we can't write the glue for that particular property *easily*, because either the Gecko code is complicated, or transforming the values from Servo to Gecko has some complication.

[^2]: There are tons of unimplemented *easy* properties in stylo (like this one). Some are pretty straight, while others require a bit of hacking, and spending time with both the codebases.

[^3]: They *attract* me so much that whenever I get to work with them, I tend to spend more time admiring the existing code rather than concentrating on the particular problem on hand.

[^4]: I admit. This consumed a few hours of mine. If I'd asked around, then maybe I'd have fixed this within an hour or so. But, where's the fun in that?
