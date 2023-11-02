+++
title = "Being Generic over Zero-Copy Request Handlers" 
date = 2023-11-02
draft = true
+++

Aka "I Just Want a Nice Request API" or a story about why it's sometimes difficult to have your cake and eat it too.

---

We all need APIs.
They're everywhere and what would we do if there _wasn't_ an interface to our favourite library, the ability to separate concerns and, most importantly, a way to [get a cow to tell us a joke in the terminal via  `curl`](https://cowsay.morecode.org/)?
And recently, I was working on a crate wrapping a communication protocol that we use at work.
I'd gotten everything set up nicely, implemented our own message flow on top and the only thing missing that I wanted to provide with the library was a nice abstraction for request-response endpoints to handle their requests.

See, while the protocol defines a `Message` datatype, it's not very nice to work with what basically amounts to a bag of bytes when implementing the API of a particular service.
It is inconvenient and cumbersome to write out the same code for checking a `Message`'s format at the protocol level, deserializing the `Message` into something more useful that the service understands, handling it and then serializing the response and constructing a protocol-conformant response in every service that wants to use the crate.
Not only is this repetitive and error prone, it forces the user of the library to think at a lower level conceptually than they would want to: instead of thinking about their service and its API, they have to think about the transport mechanism and message frames.

Hence, I set out to provide a nice interface that would allow a consumer of the library to define their API on a logical level.
Little did I know that what started with this simple, straightforward ambition would become a longer journey through `serde` and its lifetimes, `axum` routers, type erasure and being slapped in the face by compiler errors![^compiler-errors]<span id="fn-compiler-errors"></span>
In this post, I'll take a step back at the end of this implementation zig-zag and retrace my steps, trying to show where and why things can become complicated, what the compiler may be trying to tell you, and how you _can_ have your cake and eat it to.

## This Post
 - Is an (or, well, several) attempt(s) at writing a convenient request-response API. 
 - Discusses some of the intricacies involved in the quest for low overhead request processing, in particular why zero-copy deserialization of requests makes things more difficult. 
 - Will erase not just types but probably some of our brain cells as well as we try to figure out which type can be referred to from where and why they hell the compiler thinks it's ambiguous again?!?

## This Post is not
 - About a particular message format. While this post was triggered by my attempt to implement message routing for a particular format, I want this post to focus on the complexity of implementing handlers independent of the protocol in use.
 - An explanation of how `axum` is able to magically pass complex types to your web handlers. We'll briefly come across where this happens in `axum`, but we'll be more concerned about how `axum` can _accept and manage_ all these methods with different parameters as routes than how those parameters end up in the correct place at runtime. If you do want to read up some more on `axum`'s "magic handlers", check out [this post](https://lunatic.solutions/blog/magic-handler-functions-in-rust/) by Bernard Kolobara for an explanation and `@alexpusch`'s [`rust-magic-function-params`](https://github.com/alexpusch/rust-magic-function-params) example for a simplified version of the traits involved in making this happen.

---
## Blog Repository

The example code in this article is publicly available at [https://github.com/<wbr>domenicquirl/<wbr>blog/<wbr>tree/<wbr>master/<wbr>serde-<wbr>handler](https://github.com/domenicquirl/blog/tree/master/serde-handler).
The library crate there contains a submodule for each attempt and a specific attempt can be compiled with `cargo check --features <attempt>`, where `<attempt>` is given below for each variation.
For attempts where the library code compiles, there will also be an example that either demonstrates the code is working or shows issues that arise when trying to use it.
To run an example, run `cargo run --example <attempt> --features <attempt>`.[^auto-features]<span id="fn-auto-features"></span>

---

## The Setup

## The Problem: Zero-Copy Deserialization

## The Other Problem: Many Different Requests

imagine

```rust
struct Api1;
struct Api2;

impl Api for Api1 {
    type Request<'de> = String;
    type Reply = String;
}

impl Api for Api2 {
    type Request<'de> = String;
    type Reply = String;
}

// Should this implement `Handler<Api = Api1>` or `Handler<Api = Api2>`?
fn handler(request: String) -> String { /* ... */ }
```

```
error[E0223]: ambiguous associated type
  --> src\multiple_handlers3.rs:77:34
   |
77 | pub trait HandlerOn<'req>: FnMut(Self::Api::Request<'req>) -> Self::Api::Reply {
   |                                  ^^^^^^^^^^^^^^^^^^^^^^^^
   |
help: if there were a trait named `Example` with associated type `Request` implemented for `<Self as HandlerOn<'req>>::Api`, you could use the fully-qualified path
   |
77 | pub trait HandlerOn<'req>: FnMut(<<Self as HandlerOn<'req>>::Api as Example>::Request) -> Self::Api::Reply {
   |                                  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

error[E0223]: ambiguous associated type
  --> src\multiple_handlers3.rs:77:63
   |
77 | pub trait HandlerOn<'req>: FnMut(Self::Api::Request<'req>) -> Self::Api::Reply {
   |                                                               ^^^^^^^^^^^^^^^^
   |
help: if there were a trait named `Example` with associated type `Reply` implemented for `<Self as HandlerOn<'req>>::Api`, you could use the fully-qualified path
   |
77 | pub trait HandlerOn<'req>: FnMut(Self::Api::Request<'req>) -> <<Self as HandlerOn<'req>>::Api as Example>::Reply {
   |          
```

not object safe

```
error[E0038]: the trait `Handler` cannot be made into an object
  --> src\multiple_handlers4.rs:27:41
   |
27 |     handlers: HashMap<&'static str, Box<dyn Handler>>,
   |                                         ^^^^^^^^^^^ `Handler` cannot be made into an object
   |
note: for a trait to be "object safe" it needs to allow building a vtable to allow the call to be resolvable dynamically; for more information visit <https://doc.rust-lang.org/reference/items/traits.html#object-safety>
  --> src\multiple_handlers4.rs:78:5
   |
78 | /     FnMut(
79 | |     <<Self as HandlerOn<'req>>::Api as Api>::Request<'req>,
80 | | ) -> <<Self as HandlerOn<'req>>::Api as Api>::Reply
   | |      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   | |______|____________________________________________|
   |        |                                            ...because it uses `Self` as a type parameter
   |        ...because it uses `Self` as a type parameter
...
89 |   pub trait Handler: for<'req> HandlerOn<'req> {}
   |             ------- this trait cannot be made into an object...
```

even if would not help because

```
error[E0191]: the value of the associated type `Api` (from trait `HandlerOn`) must be specified
  --> src\multiple_handlers3.rs:27:45
   |
27 |     handlers: HashMap<&'static str, Box<dyn Handler>>,
   |                                             ^^^^^^^ help: specify the associated type: `Handler<Api = Type>`
...
78 |     type Api: Api;
   |     ------------- `Api` defined here
```

---
[^compiler-errors]: While I think some of the errors were some of the more cryptic ones I have seen so far, they still state fairly clearly what the problem is - if you know what they are talking about at all. Lifetimes and higher-ranked bounds are a tricky area to begin with, so I don't particularly fault the compiler and refrained from attributing them with things like "confusing", or "weird", or the above "cryptic" and I am not trying to summon Esteban.<a href="#fn-compiler-errors" class="footnote-backref" role="doc-backlink">↩︎</a>

[^auto-features]: It's unfortunate that the feature has to be specified even though we can (and I have) set the `required-features` for an example in `Cargo.toml`. You'll currently get an error from cargo that tells you to add the correct `--feature` flag if you try running the example without it, so clearly `cargo` knows what's up. But this is one of these issues that are a lot more difficult to actually change than it seems on the surface - if you're curious, you can check out the `cargo` issue for this at [`cargo#4663`](https://github.com/rust-lang/cargo/issues/4663). <a href="#fn-auto-features" class="footnote-backref" role="doc-backlink">↩︎</a>