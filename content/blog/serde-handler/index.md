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

To avoid getting side-tracked by the underlying protocol, let's just say we have a few services, some generic `Requester` type for sending protocol requests to these services from clients, and a complimentary `Responder` type for receiving and answering requests from a `Requester` (conveniently, this also means I can cheat and not actually use any protocol for the examples).
Our basic interface thus looks like this:

```rust
pub struct Requester;
pub struct Responder;

impl Requester {
    pub fn send_request(&self, request: Message, to_service: &str) -> Result<()>;
    pub fn recv_response(&self) -> Result<Message>;
}

impl Responder {
    pub fn next_request(&self) -> Result<Message>;
    pub fn send_response(&self, message: Message) -> Result<()>;
}
```

We can send requests and receive responses in the underlying `Message` format from the protocol on one side, and receive those requests and send responses on the other.
Right now, both sides of the communication effectively do the same thing - except that the `Requester` has to specify whom to request from -, but we will add to their functionality over the course of this post.

Next, we want to "automate" the conversions to and from `Message` to whatever `to_service` actually expects the request to look like and what it produces as a response.
Let's start by tying together those two types (the request and the response) with a trait:

```rust
pub trait Api { 
    /// The request body.
    type Request;

    /// The data returned to answer a `Request`.
    type Reply;
}
```

For us, `Request` is always going to be some form of the implementing type.
So, for example, we could have a `struct FooRequest` representing a request to the `foo_service`.
If the `foo_service` returns a `String` as the reply to a `FooRequest`, we would represent that as

```rust
impl Api for FooRequest {
    type Request = FooRequest;
    type Reply = String;
}
```

This already allows us to write a much nicer interface for our `Requester`.
For the purpose of this post and the examples, we'll use `serde_json` to serialize and deserialize our Rust request and response types to and from bytes:[^bounding-the-request-type]<span id="fn-bounding-the-request-type"></span>

```rust
impl Requester {
    pub fn request<A: Api>(&self, request: A::Request, for_service: &str) -> Result<A::Reply>
    where
        A::Request: serde::Serialize,
        A::Reply: serde::DeserializeOwned,
    {
        let data = serde_json::to_vec_pretty(&request)
            .map_err(|e| format!("Serialize error: {e}"))?;
        let request = Message::from(data);
        self.send_request(request, for_service)
            .map_err(|e| format!("Failed to send request: {e}"))?;
        let response = self
            .recv_response()
            .map_err(|e| format!("Error receiving response: {e}"))?;
        serde_json::from_slice(&response.data())
            .map_err(|e| format!("Deserialize error: {e}"))
    }
}
```

As a bonus, we can utilize the fact that we are now dealing with `FooRequest`s instead of `Message`s, which we know is the request type of service `foo_service`, to automatically figure out that the request needs to be routed to `foo_service` in the `Requester`:

```rust,hl_lines=2-5 23-26
pub trait Api { 
    /// The service whose API is extended with this implementation.
    ///
    /// `Request`s of the implementing type will be sent to this service.
    const SERVICE: &'static str;

    /// The request body.
    type Request;

    /// The data returned to answer a `Request`.
    type Reply;
}

impl Requester {
    pub fn request<A: Api>(&self, request: A::Request) -> Result<A::Reply>
    where
        A::Request: serde::Serialize,
        A::Reply: serde::DeserializeOwned,
    {
        let data = serde_json::to_vec_pretty(&request)
            .map_err(|e| format!("Serialize error: {e}"))?;
        let request = Message::from(data);
        // Use `A::SERVICE` to route to the correct receipient
        //                         vvvvvvvvvv
        self.send_request(request, A::SERVICE)
            .map_err(|e| format!("Failed to send request: {e}"))?;
        let response = self
            .recv_response()
            .map_err(|e| format!("Error receiving response: {e}"))?;
        serde_json::from_slice(&response.data())
            .map_err(|e| format!("Deserialize error: {e}"))
    }
}
```

Look ma, no `for_service` anymore!
However, we're missing a counterpart on the `Responder` side.
We can implement a similar workflow to the `Requester` by running the same steps in reverse, in a loop so the `Responder` fetches and responds to one request after the other.
Instead of the name of the service that takes the request (which, on the service's side, is just the `Requester` itself), our endpoint takes the _logical_ message handler.
That is, you can pass a function that takes an `Api::Request` and returns an `Api::Reply` and the `Responder` will continuously feed it with new requests that it has already decoded and take care of serializing and wrapping the `Reply` as well:[^return-errors]<span id="fn-return-errors"></span>

```rust
impl Responder {
    /// Perpetually waits for incoming requests on `self` and handles them with 
    /// the given `handler`, sending back the computed reply.
    pub fn serve_forever<A: Api, H>(self, mut handler: H) -> Result<()>
    where
        H: FnMut(A::Request) -> A::Reply,
        A::Request: serde::DeserializeOwned,
        A::Reply: serde::Serialize,
    {
        loop {
            let request = self.next_request()?;
            let request: A::Request = serde_json::from_slice(&request.data())
                .map_err(|e| format!("Deserialize error: {e}"))?;
            let reply = handler(request);
            let data = serde_json::to_vec_pretty(&reply)
                .map_err(|e| format!("Serialize error: {e}"))?;
            let response = Message::from(data);
            self.send_response(response)?;
        }
    }
}
```

This code compiles, which you can verify by `cargo check`ing the code in the repo with the `start` feature, and this post could simply end here.
However.

## The Problem: Zero-Copy Deserialization

As it is written above, on the service side our `Responder` API forces the request data to be copied into the logical `A::Request` type.
When we say `let request: A::Request = serde_json::from_slice(x)`, `serde_json` will recursively construct whatever is inside `A::Request` and the request type itself from the JSON object represented by the bytes in `x` by copying those bytes into a new, **owned** Rust type.
This can be, for example, a `u32` or a `String`, but not a `&u32` or a `&str`, since those are references, not owned data.
We can see the requirement that we can deserialize into an owned `Request` in the `where` bounds of `serve_forever`, where we ask for `A::Request: serde::DeserializeOwned`.

### Aside: `serde` Lifetimes

Most of the time, we all probably primarily use `serde` by deriving `Serialize` and / or `Deserialize` for our types so they work with whatever other library we want to use that implements or uses a data format (such as `serde_json` for JSON or a webserver crate that uses `serde_json` internally to help you build a JSON Web API).
If that is the case for you too, you might not have come across `DeserializeOwned` before.
But the derive macro for the `Deserialize` trait actually hides the fact that the trait has a lifetime: the actual definition of `Deserialize` is `trait Deserialize<'de>: Sized`.[^serialize]<span id="fn-serialize"></span>

The `serde` documentation contains [its own section](https://serde.rs/lifetimes.html) about what this lifetime means and what it is for.
For our purposes, the most important feature that the `'de` lifetime enables is _zero-copy deserialization_.
"Zero-copy" refers to the fact that, as opposed to our current `serve_forever` implementation, a request can be deserialized into a type that holds a reference into the request instead of copying everything over into the struct.

Consider a service which provides an API to convert some text to uppercase.
With our current code, the request type looks like this:

```rust
#[derive(serde::Deserialize)]
struct UppercaseRequest {
    input: String
}
```

Where `input` is an owned `String`.
`serde`, however, also supports deriving `Deserialize` for a struct like this:

```rust
#[derive(serde::Deserialize)]
struct UppercaseRequest<'input> {
    input: &'input str,
}
```

Where `input` is a reference to the text stored in the request JSON: 

```json
{ 
    "input": "Foo" 
    //        ^^^
    //         |
    //         -- input from UppercaseRequest
}
```

The `serde` derive will make `UppercaseRequest<'input>` (with the lifetime) implement `Deserialize<'input>`, which means it's possible to deserialize some data that lives at least as long as some lifetime `'a` into an `UppercaseRequest<'a>` of the same lifetime (provided that the data is a string, of course).
For structs without a lifetime (like the first `UppercaseRequest` where `input` is a `String`), `Deserialize<'de>` will be implemented for _any_ lifetime `'de`, because all data in the struct is owned and you can use the Rust type independently of the input buffer of the request once deserialized.

Coincidentally, this is exactly what our current `DeserializeOwned` bound requires of the request type: it is equivalent to the bound `A::Request: for<'de> Deserialize<'de>`, which is the Rust syntax for "no matter the lifetime `'de` of the input, this type implements `Deserialize` with that lifetime (which means it can be deserialized from that input)".
But copying every single request is a lot of work, so could we support zero-copy deserialization in our API as well?

### Let's Try to be Zero-Copy!

We'll need to change the bound on `serve_forever` away from `DeserializeOwned` to `Deserialize<'de>` with a lifetime.
But... what lifetime? Where does it come from?
The request is deserialized _inside_ `serve_forever` so we can pass it to the handler, so the lifetime of the input buffer and the lifetime of the Rust request object are both some subset of the current function.
That's not something we can really explicitly refer to,[^named-block-lifetimes]<span id="fn-named-block-lifetimes"></span> so maybe this means `serve_forever` just has to be generic over `'de` and the compiler will figure out what lifetime that represents?

```rust
/// Perpetually waits for incoming requests on `self` and handles them with the
/// given `handler`, sending back the computed reply.
pub fn serve_forever<'de, A: Api, H>(self, mut handler: H) -> Result<()>
where
    H: FnMut(A::Request) -> A::Reply,
    A::Request: Deserialize<'de>,
    A::Reply: Serialize,
{
    loop {
        let request = self.next_request()?;
        let request: A::Request = serde_json::from_slice(&request.data())
            .map_err(|e| format!("Deserialize error: {e}"))?;
        let reply = handler(request);
        let data = serde_json::to_vec_pretty(&reply)
            .map_err(|e| format!("Serialize error: {e}"))?;
        let response = Message::from(data);
        self.send_response(response)?;
    }
}
```

If we try to compile this (which you can try with `--features zero_copy1` in the repo), we get an error:

```
error[E0597]: `data` does not live long enough
  --> src\zero_copy1.rs:60:40
   |
51 |     pub fn serve_forever<'de, A: Api, H>(self, mut handler: H) -> Result<()>
   |                          --- lifetime `'de` defined here
...
58 |             let data = self.next_request()?.data();
   |                 ---- binding `data` declared here
59 |             let data = serde_json::from_slice(&data)
   |                 ------------------------------^^^^^-
   |                 |                             |
   |                 |                             borrowed value does not 
   |                 |                             live long enough
   |                 argument requires that `data` is borrowed for `'de`
...
66 |         }
   |         - `data` dropped here while still borrowed
```

Seems like the compiler is not happy about us leaving the `'de` lifetime up in the air.
As there is nowhere else that takes a lifetime, we might try to use the `for<'de>` syntax from above to summon one out of thin air inside the `where` bound for `Deserialize<'de>` instead of the function generic parameters, but as we've learned this just means `DeserializedOwned`, which we had before.
So while it compiles, the `zero_copy2` example in the repo shows that we can't actually invoke `serve_forever` with that bound for our borrowing `UppercaseRequest`:

```
error: implementation of `Deserialize` is not general enough
  --> examples\zero_copy2.rs:28:14
   |
28 |             .serve_forever::<UppercaseRequest, _>(|request| 
   |              ^^^^^^^^^^^^^ implementation of `Deserialize` is not 
   |                            general enough
   |
   = note: `UppercaseRequest<'_>` must implement `Deserialize<'0>`, 
           for any lifetime `'0`...
   = note: ...but `UppercaseRequest<'_>` actually implements `Deserialize<'1>`, 
           for some specific lifetime `'1`
```

Don't be confused by the lifetime names `'0` and `'1` that the compiler uses.
The important information is that we "must implement `Deserialize` for any lifetime" (because we said so by requesting `for<'de> Deserialize<'de>`, just that the compiler names `'de` as `'0` instead), but `UppercaseRequest` contains a lifetime (the `input` reference) and so we can only `Deserialize` for this "specific lifetime" (which the compiler calls `'1`).

### Going Higher-Order

This doesn't work.
We won't get any further without a way to tie the `Deserialize` lifetime `'de` to the (potential) lifetime in our request type, so we need a way to pass `'de` through to `UppercaseRequest`.
One way to do this would be to make the `Api` trait generic over a lifetime as well, like `Deserialize` is:

```rust,hl_lines=10
// Now with lifetime!
//            vvv
pub trait Api<'de> {
    /// The service whose API is extended with this implementation.
    ///
    /// `Request`s of the implementing type will be sent to this service.
    const SERVICE: &'static str;

    /// The request body.
    type Request: Serialize + Deserialize<'de>;
    //                                    ^^^
    //       can now mention the same lifetime

    /// The data returned to answer a `Request`.
    type Reply: Serialize + DeserializeOwned;
}
```

Note that we've also moved the `Serialize` and `Deserialize` bounds from the respective functions into the trait definition so both `'de`s are in the same place.
But, since we have them now, let's use generic associated types (GATs) instead!
With GATs, we can move the lifetime for the requests onto the `Request` associated type itself: 

```rust,hl_lines=8
pub trait Api {
    /// The service whose API is extended with this implementation.
    ///
    /// `Request`s of the implementing type will be sent to this service.
    const SERVICE: &'static str;

    /// The request body.
    type Request<'de>: Serialize + Deserialize<'de>;

    /// The data returned to answer a `Request`.
    type Reply: Serialize + DeserializeOwned;
}
```

With either version, there is now a lifetime associated with `Api` that is passed on to the `Deserialize` bound on the `Request` type.
Therefore, we now have access to `'de` when implementing `Api` for our `UppercaseRequest` and can pass it on to our type:

```rust,hl_lines=2
impl Api for UppercaseRequest<'_> {
    type Request<'de> = UppercaseRequest<'de>;
    type Reply = String;

    const SERVICE: &'static str = TEXT_SERVICE;
}
```

Which, combined with the trait definition, means that we have established a "lifetime chain" `UppercaseRequest<'_>::Request<'de> = UppercaseRequest<'de>: Deserialize<'de>`.

When updating `serve_forever`, we have to concern ourselves with the type of the `handler` parameter:
So far, this has been `H: FnMut(A::Request) -> A::Reply` - a function from requests to replies.
This is still what we want, but now `A::Request` is actually `A::Request<'de>`, so we've got the inverse problem as before: now we _can_ name a lifetime, but that also means we _have_ to pass one and need to figure out which is correct.

The simplest possible solution would be to once again defer to the compiler and write `H: FnMut(A::Request<'_>) -> A::Reply`.
Surprisingly, this time it actually works!
To the compiler, `'_` means "any lifetime" (as opposed to "some specific lifetime `'1`"), so the above is equivalent to `H: for<'de> FnMut(A::Request<'de>) -> A::Reply`: no matter `'de`, if I have a request with that lifetime then I can pass it to the handler and it will compute a `Reply`.
This time, this is what we want and everything works and you can send as many `UppercaseRequests` as your heart desires by running the `zero_copy3` example and entering evil lowercase text on the terminal that must be converted to uppercase glory.

### Giving Our Bound A Name

For our single `serve_forever` function, it's fine to write out the rather convoluted bound for `H` and be done with it.
If there were multiple functions that accepted such a type, it's probably better to define a name for it in one place so you can refer to just the name wherever you might need it.

We'll define two new traits, `HandlerOn<'req>` and `Handler`, that represent handlers that can handle a request with a specific lifetime `'req` and requests for any lifetime, respectively:

```rust
/// A function that can handle [`A::Request<'de>`](Api::Request) 
/// for `'de == 'req`.
pub trait HandlerOn<'req, A: Api>: FnMut(A::Request<'req>) -> A::Reply {}
impl<'req, A: Api, F> HandlerOn<'req, A> for F 
    where F: FnMut(A::Request<'req>) -> A::Reply
{}

/// A function that can handle [`A::Request<'de>`](Api::Request) 
/// for any `'de`.
pub trait Handler<A: Api>: for<'req> HandlerOn<'req, A> {}
impl<A: Api, F: for<'req> HandlerOn<'req, A>> Handler<A> for F {}
```

The definition of `serve_forever` now becomes just

```rust
pub fn serve_forever<A: Api, H: Handler<A>>(self, mut handler: H) -> Result<()>;
```

To verify that this indeed still runs as before, run example `zero_copy4` in the repository.
You can read more on this entire construction in [this URLO thread](https://users.rust-lang.org/t/generic-parameter-bound-by-deserialize-de-is-this-pattern-possible-in-rust/82163/3) if you want to.

## The Other Problem: Many Different Requests

We've succeeded in writing an API for both clients and services that allows them to implement an API purely by working with a logical Rust request and response data structure.
So that's it! Time to head home.
Unless, of course, you want your services to serve more than a single kind of requests.

Currently, you can call `serve_forever` on a `Responder` and, well, serve one implementation of `Api`... forever. 
But what if we want our text service to be able to both convert text to uppercase as well as to lowercase? 
And maybe later we'll add an API for trimming whitespace, too!
`serve_forever` can only handle (with the given `handler`) one particular `A: Api`, though, so what do?

For this example, all requests go from text to text, so we could work around the issue by defining an

```rust
enum TextRequest<'a> {
    UppercaseRequest(&'a str),
    LowercaseRequest(&'a str),
    TrimRequest(&'a str),
}
```

implementing `Api` for it and providing a handler that takes a `TextRequest` and returns a `String`.
But what if we have one API for strings and one for numbers? 
Or, perhaps more realistically, if all our APIs take some data structure specific to this API as their input that we would like to represent as Rust `struct`s, and also return similarly different data?

Instead of falling back on two big `enum Request` and `enum Response`s where we can't even guarantee that the correct `Response` variant is sent for a given `Request`, let's try to come up with a library API that supports multiple types of requests.
Perhaps we can look at some other frameworks for inspiration?
Surely, we can't be the first developers to ever want to route different requests to different handlers... wait, isn't that what like every single web framework does?

Let's look at `axum` as an example. 
On a high level, it provides a `Router` type that you can use to build up different web endpoints for different URLs and HTTP methods like so (taking from their [documentation](https://docs.rs/axum/latest/axum/)):

```rust 
let app = Router::new()
    .route("/", get(root))
    .route("/foo", get(get_foo).post(post_foo))
    .route("/foo/bar", get(foo_bar));
```

where `get_foo` etc. are `async fn`s.
The `Router::route` method takes a `MethodRouter` as its second argument, which is basically a static `HashMap` that can hold one handler per HTTP method (`GET`, `POST`, etc.).
This makes sense for a web framework, but we'll only consider one handler per route for this post.

To start with, in addition to the `SERVICE` name we'll also give each `Api` its own unique name to differentiate the APIs implemented by a service:

```rust
pub trait Api {
    /// The service whose API is extended with this implementation.
    ///
    /// `Request`s of the implementing type will be sent to this service.
    const SERVICE: &'static str;

    /// The unique name of the API that identifies the kind of `Request` 
    /// to the `SERVICE`.
    const NAME: &'static str;

    /// The request body.
    type Request<'de>: Serialize + Deserialize<'de>;

    /// The data returned to answer a `Request`.
    type Reply: Serialize + DeserializeOwned;
}
```

We'll send the name as part of the underlying protocol `Message`.[^api-name]<span id="fn-api-name"></span>
Next, we want to have our own router type that maps the API names of a service to their correct handlers.

```rust
pub struct ApiRouter {
    handlers: HashMap<&'static str, dyn Handler<A>>,
}
```

We can then provide APIs to get an `ApiRouter`, add new handlers and run our `serve_forever` loop, except now we extract the API name from the request and look up the corresponding handler to invoke.

```rust
impl ApiRouter {
    /// Create a new `Router`.
    ///
    /// Unless you add additional routes via [`register_handler`], this
    /// will respond with `InvalidRequest` to all requests.
    pub fn new() -> Self {
        Self { handlers: HashMap::new() }
    }

    /// Add a new handler for API requests of type `A`.
    ///
    /// This will make the router route all requests of type `A` to the given 
    /// `handler` if the request data can be successfully deserialized into 
    /// [`A::Request`]. The `handler` may be a function name or a closure.
    pub fn register_handler<A: Api, H: Handler<A>>(mut self, handler: H) -> Self
    where
        H: 'static,
    {
        self.handlers.insert(A::NAME, handler);
        self
    }

    /// Perpetually waits for incoming requests on `socket` and handles them 
    /// with the handler registered for their route, sending back the computed
    /// reply.
    pub fn serve_on(mut self, socket: Responder) -> Result<()> {
        loop {
            let Message { api_name, data } = socket.next_request()?;

            let handler = self
                .handlers
                .get_mut(api_name.as_str())
                .ok_or_else(|| format!("No handler for '{api_name}'",))?;
            let reply = (handler.0)(&data)?;
            let response = Message {
                api_name,
                data: reply,
            };
            socket.send_response(response)?;
        }
    }
}
```

Except. 
This doesn't compile.
We're storing `dyn Handler`s in the router so we don't have to differentiate between the different types of different handler functions, but `Handler` is still generic over `A`, so the compiler (rightfully) complains:

```
error[E0412]: cannot find type `A` in this scope
  --> src\multiple_handlers1.rs:27:49
   |
27 |     handlers: HashMap<&'static str, dyn Handler<A>>,
   |                                                 ^ not found in this scope
   |
help: you might be missing a type parameter
   |
26 | pub struct ApiRouter<A> {
   |                     +++
```

It tries to help us by suggesting we make the router generic as well, but we don't want that:
then a service could only have an `ApiRouter<SomeRequest>` for a fixed request type `SomeRequest` - exactly what we're trying to fix right now!

How can we get rid of `Handler`'s generic parameter... maybe we can move the `A` _inside_ the trait, as an associated type?[^explicit-trait]<span id="fn-explicit-trait"></span>
That would look like this:

```rust
/// A function that can handle [`A::Request<'de>`](Api::Request) 
/// for `'de == 'req`.
pub trait HandlerOn<'req>: FnMut(Self::Api::Request<'req>) -> Self::Api::Reply
{
    type Api: Api;
}
impl<'req, A: Api, F> HandlerOn<'req> for F 
    where F: FnMut(A::Request<'req>) -> A::Reply 
{
    type Api = A;
}

/// A function that can handle [`A::Request<'de>`](Api::Request) 
/// for any `'de`.
pub trait Handler: for<'req> HandlerOn<'req> {}
impl<F: for<'req> HandlerOn<'req>> Handler for F {}
```

We're saying we want the handler to be a function that takes request of the `Api` associated type, which is the `A: Api` in the first `impl`.
The compiler isn't very happy:

```
error[E0223]: ambiguous associated type
  --> src\multiple_handlers2.rs:77:34
   |
77 | pub trait HandlerOn<'req>: FnMut(Self::Api::Request<'req>) 
   |     -> Self::Api::Reply {
   |                                  ^^^^^^^^^^^^^^^^^^^^^^^^
   |
help: if there were a trait named `Example` with associated type `Request` 
      implemented for `<Self as HandlerOn<'req>>::Api`, you could use the 
      fully-qualified path
   |
77 | pub trait HandlerOn<'req>: FnMut(<<Self as HandlerOn<'req>>::Api as Example>::Request) -> Self::Api::Reply {
   |                 
```

Because both `Self` and `Self::Api` may implement multiple traits, and any (or all) of these traits could have associated types named `Api` or `Request`, the compiler wants us to be specific and say that we mean the `Api` associated type from `HandlerOn` and the `Request` associated type from `Api` (which it doesn't know, so it gives us an arbitrary `Example` trait).
Fine, let's write what it says even though it is positively hideous:

```rust
pub trait HandlerOn<'req>:
    FnMut(<Self::Api as Api>::Request<'req>) -> <Self::Api as Api>::Reply
{
    type Api: Api;
}
```

That doesn't help, for multiple reasons.
First of all, it is _also_ ambiguous, just in a different way.
Imagine we have the following two `Api` implementations:

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

Here, `handler` fits the description of `FnMut(A::Request) -> A::Reply` for both `Api1` and `Api2`, as their `Request` and `Reply` types are the same.
But if we change `Handler` and `HandlerOn` to use an associated type, we are forced to choose one single value for the implementation of `HandlerOn` for the `fn handler`.
We don't, which the compiler reports as

```
error[E0207]: the type parameter `A` is not constrained by the impl trait, 
              self type, or predicates
  --> src\multiple_handlers2.rs:82:12
   |
82 | impl<'req, A: Api, F> HandlerOn<'req> for F 
   |            ^ unconstrained type parameter
```

We _do_ mention `A` in the bounds of the implementation (`where F: FnMut(A::Request<'req>) -> A::Reply`), but - as we've shown above - that's not enough to be able to uniquely infer what type `A` actually represents. 
If you `rustc --explain E0207` the error, the compiler will tell you the exact rules for what counts as "constraining" a generic parameter:

<blockquote>
Any type or const parameter of an `impl` must meet at least one of the
following criteria:

 - it appears in the _implementing type_ of the impl, e.g. `impl<T> Foo<T>`
 - for a trait impl, it appears in the _implemented trait_, e.g.
   `impl<T> SomeTrait<T> for Foo`
 - it is bound as an associated type, e.g. `impl<T, U> SomeTrait for T
   where T: AnotherTrait<AssocType=U>`

Any unconstrained lifetime parameter of an `impl` is not supported if the
lifetime parameter is used by an associated type.
</blockquote>

So while we _mention_ `A` in a `where` bound, we don't constrain anything to `A` itself, only to its `Request` and `Reply` types.

There's another problem as well: because we need these types to write the `where` bound, we need to go through the implementing type `Self` to get them.
We want a function that takes the `Request` type for the API it is a handler for (otherwise it wouldn't make sense), which is `Self::Api::Request` (same for `Reply`).
However, this makes the `Handler` and `HandlerOn` traits no longer object safe, which means we cannot have a `dyn Handler`, which we wanted to store in our router:

```
error[E0038]: the trait `Handler` cannot be made into an object
  --> src\multiple_handlers4.rs:27:41
   |
27 |     handlers: HashMap<&'static str, Box<dyn Handler>>,
   |                                         ^^^^^^^^^^^ 
   |              `Handler` cannot be made into an object
   |
note: for a trait to be "object safe" it needs to allow building a vtable to 
      allow the call to be resolvable dynamically; for more information visit 
      <https://doc.rust-lang.org/reference/items/traits.html#object-safety>
  --> src\multiple_handlers4.rs:78:5
   |
78 | /     FnMut(
79 | |     <<Self as HandlerOn<'req>>::Api as Api>::Request<'req>,
80 | | ) -> <<Self as HandlerOn<'req>>::Api as Api>::Reply
   | |      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   | |______|____________________________________________|
   |        |...because it uses `Self` as a type parameter
   |        |
   |        ...because it uses `Self` as a type parameter
...
89 |   pub trait Handler: for<'req> HandlerOn<'req> {}
   |             ------- this trait cannot be made into an object...
```

Even if we could somehow magic the trait to still support having a `dyn Handler`, it wouldn't help because...

```
error[E0191]: the value of the associated type `Api` (from trait `HandlerOn`) 
              must be specified
  --> src\multiple_handlers3.rs:27:45
   |
27 |     handlers: HashMap<&'static str, Box<dyn Handler>>,
   |                                             ^^^^^^^ 
   | help: specify the associated type: `Handler<Api = Type>`
...
78 |     type Api: Api;
   |     ------------- `Api` defined here
```

That's right! 
Associated types don't even solve our problem in the first place, because even when there is only one `Api` per `Handler` Rust requires us to name both.
It won't be enough to move the `Api` generic to somewhere else, we'll need to _erase_ it completely.

### Calling Upon The Dark Arts

## Bonus: I've Been Hiding Something
closure type annotations

---
[^compiler-errors]: While I think some of the errors were some of the more cryptic ones I have seen so far, they still state fairly clearly what the problem is - if you know what they are talking about at all. Lifetimes and higher-ranked bounds are a tricky area to begin with, so I don't particularly fault the compiler and refrained from attributing them with things like "confusing", or "weird", or the above "cryptic" and I am not trying to summon Esteban.<a href="#fn-compiler-errors" class="footnote-backref" role="doc-backlink">↩︎</a>

[^auto-features]: It's unfortunate that the feature has to be specified even though we can (and I have) set the `required-features` for an example in `Cargo.toml`. You'll currently get an error from cargo that tells you to add the correct `--feature` flag if you try running the example without it, so clearly `cargo` knows what's up. But this is one of these issues that are a lot more difficult to actually change than it seems on the surface - if you're curious, you can check out the `cargo` issue for this at [`cargo#4663`](https://github.com/rust-lang/cargo/issues/4663). <a href="#fn-auto-features" class="footnote-backref" role="doc-backlink">↩︎</a>

[^bounding-the-request-type]: In the example code, I've used a slightly different signature for `request`: there, it is written as `fn request<A: Api<Request = A>>` and takes the request as an `A` instead of an `A::Request`. Constraining the implementation of `Api` to be implemented on the request type itself is less flexible, but allows calling `request` without specifying `A` with a turbofish (as `requester.request::<FooRequest>(req)`) because the compiler is able to infer the generic parameter from the argument type. But it's also a bit ugly and requires bounding `Api` itself by `Serialize`, so I'm leaving it as a footnote. <a href="#fn-bounding-the-request-type" class="footnote-backref" role="doc-backlink">↩︎</a>

[^return-errors]: I'm presupposing that the logical handler is "infallible", which just means that if there _is_ an error with or while processing the request, this will be communicated to the requester via the `Reply` as opposed to having the handler fail (by allowing it to return a `Result<Reply, E>` with some error). Since the protocol format and also (de-)serialization are handled by our API, any remaining error can only be a logical one. When implementing `Api`, this can be represented by making `Reply` be a `Result<T>`, using an `enum FooReply` as the `Reply` type that contains an `InvalidRequest` variant, or similar. <a href="#fn-return-errors" class="footnote-backref" role="doc-backlink">↩︎</a>

[^serialize]: This is not the case for `Serialize`. When you serialize a type to some format, you probably intend to send the resulting bytes somewhere else, e.g. over the network to make a web request. It would be of little use to be able to borrow from the Rust type from within the serialized bytes, and besides you cannot serialize to a single byte buffer or string of which only a subsection is a reference to the original type and the rest is owned bytes, so `serde` doesn't support it. <a href="#fn-serialize" class="footnote-backref" role="doc-backlink">↩︎</a>

[^named-block-lifetimes]: No, [`label-break-value`](https://github.com/rust-lang/rfcs/pull/2046) doesn't count here. <a href="#fn-named-block-lifetimes" class="footnote-backref" role="doc-backlink">↩︎</a>

[^api-name]: If the underlying protocol supports it, the API (as well as the service) could also be identified by a numeric ID or some other, more compact identifier. I'll stick to string names here, though, since it makes the examples more readable. <a href="#fn-api-name" class="footnote-backref" role="doc-backlink">↩︎</a>

[^explicit-trait]: Note that there is no way to do this with the original bound of `H: for<'de> FnMut(A::Request<'de>) -> A::Reply`. There is simply no place in the syntax where we could even _state_ an associated type. This might already give you an idea of how well this is going to turn out... <a href="#fn-explicit-trait" class="footnote-backref" role="doc-backlink">↩︎</a>