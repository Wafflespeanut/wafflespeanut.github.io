---
layout: post
title: "Experience Report: Swift from a Rustacean's perspective"
date: 2018-12-31 23:59:59 +0530
comments: true
categories: [Coding, Swift, Rust]
---

I've been using Swift for over a year now, mostly for the [decentralized e-commerce platform](https://naamio.cloud/projects/kauppa) I've been working on (and a bunch of side projects like OpenSSL bindings, WebID-TLS prototype, etc.). So, this is something I've been wanting to write for a long time, but never found time until now.[^1]

A few things about this write-up. Firstly, I love the language![^2] And second, even though I'll be comparing bits of both the languages, I'll never say that one is better than the other. Both are amazing languages and they aim to solve different things. Also, I began using Rust from around its 1.0 release (about 3.5 years back), and Swift only *after* its 4.0 release (last year), so I'll be talking about the features they currently have, not what they had / lacked ages ago!

> As a side-note, please don't expect this post to be a tutorial or have in-depth discussions about language internals or have some sort of order in comparisons.

<!-- more -->

Before we begin, here are a few basic differences between both the languages, just to show you why a direct comparison isn't fair.

Rust | Swift |
---- | ----- |
Low-level | Not that low-level
Embedded, kernels, browser/game engines, etc. | Web services, apps for iOS, macOS, etc.
No garbage collection | Reference-counted garbage collector
Supports pretty much all platforms | Only XCode and Ubuntu (as of now)
Supports static builds | Foundation (Swift's stdlib) in Linux [can't be statically linked yet](https://bugs.swift.org/browse/SR-2205)

With this in place, let's proceed.

## Basic syntax

In Rust (being expression-based), semicolons have meaning, whereas Swift doesn't need semicolons to separate statements (unless they're on the same line), like Python.

Here's an example demonstrating a function that simply fetches the value of the given env variable. It's a common practice - instead of dealing with importing [`Foundation`](https://developer.apple.com/documentation/foundation/) (stdlib) and calling `ProcessInfo` everywhere, we have this nice abstraction.

```swift
import Foundation

func getEnvVariable(name variable: String) -> String? {
    return ProcessInfo.processInfo.environment[variable]    // explicit return
}

let port = getEnvVariable(name: "LISTEN_PORT")!
```

If we were to write the same thing in Rust, then we'd do something like:

```rust
use std::env;

fn get_env_variable(name: &str) -> Option<String> {
    env::var(name).ok()
}

let port = get_env_variable("LISTEN_PORT").unwrap();
```

Similar, right?

### Imports, optionals and argument labels

`import` syntax was probably the first thing I felt weird about. It's like a "recursive glob import". In Rust, you need to be very specific about what you want in your module, but here you import a package at the top of the file, and **you get to use all** (public) stuff from that package! This means, if we'd imported multiple libs, then it's hard to know (without an IDE) where some item (type / function / whatever) is from - not to mention that we also need to `grep` the item in that lib to find wherever it's located.

In order to reduce the burden, Swift devs arrange their modules in such a way that they're self-explanatory. An example would be Vapor's [PostgreSQL driver lib](https://github.com/vapor/postgresql). There, we have [`PostgreSQLConnection` type in its own module](https://github.com/vapor/postgresql/blob/92881d8b29b7fef572b0d3d56b71527e8a4baeca/Sources/PostgreSQL/Connection/PostgreSQLConnection.swift), but then we also have [a number of `PostgreSQLConnection+Foo.swift` files](https://github.com/vapor/postgresql/tree/92881d8b29b7fef572b0d3d56b71527e8a4baeca/Sources/PostgreSQL/Connection) that contain additional implementations for that type related to some behavior "Foo" (in different modules).

Then, there are the optionals. In Rust, `Option` is [just like any other enum]((https://doc.rust-lang.org/std/option/enum.Option.html)), which means `None` is just another value. In Swift, even though [`Optional` is an enum](https://github.com/apple/swift/blob/0d4a5853bf665eb860ad19a16048664899c6cce3/stdlib/public/core/Optional.swift#L122), it's baked into the compiler such that `?` operator (in suffix) represents an optional type, which means `nil` is a special value to indicate nothing. As a result of this, unwrapping an optional can be as simple as using another (exclamation `!`) operator.

In the above example, `name` is the label and `variable` is the actual argument to be used in the function body. Argument labeling was weird at first, but nowadays, I wanna label everything! Here's a snippet from our platform:

```swift
let product = try products.getProductVariant(for: unit.product,
                                             from: order.shippingAddress)
try checkCurrency(for: product)
let inventoryItem = try inventory.getInventoryItem(for: product.inventoryId)
try updateConsumedInventory(for: inventoryItem, with: product, in: unit)
```

With labeling, it's possible to write some cool expressive code.

### try ... catch

In Swift, `Error` is a *protocol* (interface, if you want) which can be implemented for any type, just like Rust, where [it's a trait](https://doc.rust-lang.org/std/error/trait.Error.html) (another buzzword!). Only difference is that here, an error value can be *thrown* by some operation and can be *caught* elsewhere when it bubbles up.

To see this in action, let's write a function which throws 95% of the time:

```swift
enum Luck: Error {
    case worse
    case bad
}

func iFeelLucky() throws {  // mark explicitly
    let i = Int.random(in: 0 ..< 100)   // generate Int in [0, 100)

    // switch: not just ranges, but patterns (tuples and enums too!)
    switch i {
        case 0 ..< 50:
            throw Luck.worse
        case 51 ..< 95:  // biased rejection
            throw Luck.bad
        default:
            return
    }
}

do {
    try iFeelLucky()
} catch Luck.worse {
    print("Definitely not!")
} catch Luck.bad {
    print("Try again?")
} catch {
    // some other sorcery?
}
```

The `do { }` block represents your *trial* area, and you *catch* the error next to that block. It's worth mentioning that when you `throw` your error value, the actual error type is erased (since it gets casted to the `Error` interface), so you can throw any kind of error from the same block (if you've got good reason to do that). Later, when you `catch`, you can pattern match by casting it back to the actual error type.[^3]

## Types

All types (and protocols) in Swift can be *extended* - regardless of whether they're from a foreign package, like Foundation. For example, we could extend strings with an alphanumeric check like so:

```swift
import Foundation

extension String {
    /// Checks if the given string is alphanumeric.
    public func isAlphaNumeric() -> Bool {
        return self.range(of: "[^a-zA-Z0-9]", options: .regularExpression) == nil
    }
}
```

We've just added some functionality to a type that doesn't belong to us! It's an useful abstraction, yes, but Rust doesn't allow you to do this,[^4] and I think there's a good reason for it. When you start extending stuff you don't own, users will have trouble finding the implementation - whether it's from your package, or it's from a dependency, or whether this has existed in a core package all this time!

That said, I'm not against it either (I'm doing it myself!). I'm simply unsure about the downsides (if any) to not using / having this feature.

### Structs and Classes

In addition to enums and tuples, Swift supports structs and classes - both have static and stored properties (readable, writable or computed). Also, access to types, properties and methods can be [controlled with modifiers](https://docs.swift.org/swift-book/LanguageGuide/AccessControl.html).[^5]

Let's take a dumb struct:

```swift
import Foundation

public struct Customer {

    public let id: UUID
    public let createdAt: Date
    public var updatedAt: Date
    public var firstName = ""
    public var lastName = ""
    public var primaryEmail = ""

    public var isValid: Bool {
        return !firstName.isEmpty && !lastName.isEmpty // && validate email
    }

    public init(id: UUID, createdAt creation: Date, updatedAt updated: Date) {
        self.id = id
        createdAt = creation    // no name collision - can ignore `self`
        updatedAt = updated
    }

    public init() {
        let date = Date()
        self.init(id: UUID(), createdAt: date, updatedAt: date)
    }
}
```

I'm ambivalent about having methods as part of the type itself, but other than that, I like a number of things here:[^6]

 1. **Foundation has a lot of stuff!** So far, we've seen random number generation, regex, UUID and datetime. It's nice to have all these things in stdlib, so it's one less worry for us.
 2. Mutation is field-specific. In the above example, `id` cannot be changed for an instance (even if the instance itself is mutable).
 3. Functions could have the same names, as long as they have different signatures. `init` is special (in that it's the constructor), but it's no different from any other function.

### Values and References

All classes in Swift are "reference types" and all other types are "value types". The difference is that instances of **value types are copied**. Coming from Rust, this felt like *infidelity*, but well, that's what you pay for using languages with automatic memory management. Arrays and dictionaries are structs, so every time you assign them to some variable or pass them to another function, they get copied!

```swift
var a = [0, 2, 5, 10]   // array is a struct
var b = a       // copied
b.append(11)    // "a" still has 4 elements

class Foo {
    var inner = [0, 2, 5, 10]
}

let f = Foo()
let g = f               // "f" and "g" hold reference to same class
g.inner.append(15)      // "f.inner" and "g.inner" are same (5 elements).
```

### Protocols

Traits are one of those lovely things in Rust. With generics, they're simply *beautiful*. Having used to them, it wasn't hard for me to get into protocols (interfaces of Swift). All types in Swift can implement protocols.[^7]

That said, Swift has its limitations when it comes to protocols. Generics and traits in Rust are rather robust. For example, we can do this in Rust but not in Swift:

```rust
impl MyTrait for T where T: MyOtherTrait {
    // MyTrait impl
}
```

This translates to, "Implement `MyTrait` for **all types** that implement `MyOtherTrait`". This has some wonderful effects. [`From`](https://doc.rust-lang.org/std/convert/trait.From.html) and [`Into`](https://doc.rust-lang.org/std/convert/trait.Into.html) traits are my favorites. If your type implements `From`, then (because of this feature) it gets the `Into` implementation for free!

In Swift, you can add a protocol extension with such a constraint.

```swift
extension MyProtocol where Self: MyOtherProtocol {
    // MyProtocol impl
}
```

But, this doesn't automatically apply the implementation for all `MyOtherProtocol` implementors. You still need to extend your types *specifically* and mark them like:

```swift
extension MyStruct: MyProtocol {}
```

This hasn't become a big deal for me yet, just saying.

Otherwise, protocols are quite cool. There's a protocol for [hashing](https://developer.apple.com/documentation/swift/hashable), [equality](https://developer.apple.com/documentation/swift/equatable) and [iteration](https://developer.apple.com/documentation/swift/sequence) (just like Rust), and there are others like one [for types that could be raw values](https://developer.apple.com/documentation/swift/rawrepresentable), [encoding](https://developer.apple.com/documentation/swift/encodable) and [decoding](https://developer.apple.com/documentation/swift/decodable).

Then, there's [`Codable`](https://developer.apple.com/documentation/swift/codable) which unifies serialization and deserialization (again, built into Foundation). The problem is that it's not even close to [serde](https://serde.rs/), which is the commonly used encoding / decoding lib in Rust.

In serde, you can do almost anything with a bunch of attributes (you rarely need to write custom code), which is a great perk for using a statically typed language, whereas in Swift, let it be skipping a property, managing a particular property on your own or performing additional validation, **anything** that deviates even a little **requires custom code**. It's not hard to write, but it's difficult to maintain - whenever you alter the structs, you need to modify that custom implementation. It'd be nice if it could be done with less effort.

If there's one thing I like about Swift protocols, it's automatic box'ing (another perk of managed languages). In Rust, you need to specify the pointer which holds a particular [trait object](https://doc.rust-lang.org/book/ch17-02-trait-objects.html). I don't want this to change. It's always been and should always be that way in Rust (I need to know whether I'm using `Box`, `Arc` or a simple reference), it's just that it's sometimes annoying (depending on the use case) to box stuff on our own when you're dealing with trait objects.

## Packaging

[Swift Package Manager](https://github.com/apple/swift-package-manager/blob/ad40fc69276d5dafd213af9b3aafca1cccd6fe3c/Documentation/PackageDescriptionV4.md) reminded me of [build scripts](https://doc.rust-lang.org/cargo/reference/build-scripts.html) in Rust, because you write in Swift to build your Swift package, and I like it. But, there's no central registry upon which SPM relies on (like [crates.io](https://crates.io/) for cargo). Instead, it needs Git. In order to specify a package as a dependency, you have to specify the URL of a git repo, and versions are based on tags. That said, you can specify a branch / revision in that repo, or simply use your local path - everything works, so I haven't had any trouble with it.

I also liked SPM's model - a package has a name, a number of products (libraries and executables), dependencies and targets. Products depend on targets. Tests are part of targets (called test targets). This means, a package can output any number of executables and libraries, and tests can be located anywhere (typically they're inside `Tests/` in project root). In Rust, we can use [workspaces](https://doc.rust-lang.org/cargo/reference/manifest.html#the-workspace-section) to output multiple products, but unit tests cannot exist elsewhere.[^8]

## The Future

I think Swift and Rust have a number of similarities in their features (other than the obvious differences).[^9] I have a blind wish that it gets procedural macros from Rust at some point!

Anyway, it didn't require much effort for someone coming from Rust to get into Swift (but I guess that's the case for jumping from Rust into any other language, because well... we've learned from the master!). I like the way things are in Swift right now, and I'm looking forward to where it's headed.

If I were to write a web service today, then Swift will be my choice without any second thoughts.

---

<small>I may have left out some things along the way, but I'll update the post whenever something comes to mind.</small>

[^1]: I don't think I'll be able to write about my WebID-TLS work just yet. I've been diagnosed with [CTS](https://en.wikipedia.org/wiki/Carpal_tunnel_syndrome) lately, and it's ascending, so I'll probably be taking a break from my computer starting this February or something and I need to wrap up some work before that.

[^2]: Not as much as I love Rust though!

[^3]: I uhh... personally hate try-catching - I like Rust's way of dealing with fallible types using the [`Result`](https://doc.rust-lang.org/std/result/enum.Result.html) enum, but again that's just my preference.

[^4]: In Rust, you can only extend your own type or implement your own trait to other types.

[^5]: Rust 1.18 introduced [support for even fine-grained access control](https://blog.rust-lang.org/2017/06/08/Rust-1.18.html) like enabling access in a particular crate, in a module or even a specified path, etc. (besides public and private)

[^6]: We don't have any of this in Rust, although sometimes I wish we had argument labeling, differentiating functions based on signatures (not names), and some core crates getting stabilized.

[^7]: Although, protocols marked with `class` can only be implemented by classes.

[^8]: They need to exist in the same module. They can stay outside, but they won't have access to any internally used types, whereas in Swift, you can mark packages as `@testable` in imports just for testing.

[^9]: Speaking of the future, Swift has [NIO](https://github.com/apple/swift-nio) for event loops and futures (again, somewhat similar to Rust).
