- Feature Name: `effect_generic_types`
- Start Date: 2024-08-12
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

This RFC introduces support for effect-generic types. These are types which are
are generic over effects such as `async`, and can change their implementation
based on which traits are being used. This makes it possible to for example
write a `TcpStream` implementation which can be constructed as either sync or
async:

```rust
// uses a single definition for both variants
use std::net::TcpStream;

// non-async
let stream = TcpStream::connect("localhost:8080")?;
let mut buf = [0; 1024];
let bytes_read = stream.read(&mut buf)?;

// async
let stream = TcpStream::connect("localhost:8080").await?;
let mut buf = [0; 1024];
let bytes_read = stream.read(&mut buf).await?;
```

This is made possible by enabling types to carry effect keyword when declared
(e.g. `async struct File {}`) . As well as a new `#[cfg(effect = <effect>)]`
attribute to make fields as conditional on the presence of effects.

# Motivation
[motivation]: #motivation

Previously we're introduced the ability to declare [traits which are generic over
effects][egt], as well as [function and method bodies which are generic over
effects][egf]. This RFC completes the core system for effect-generics, by also
enabling types to be generic over effects.

Just like we have one `File` type in the stdlib shared by Windows, macOS, and
Linux - that same type `File` should be able to work in both async and non-async
contexts. Effect-generics enables IO types like [`File`], [`TcpStream`], and
[`Stdout`] to work with different effects. As well as other types such as
[`Option`], [`Sink`], and [`Bytes`] which may contain IO types and expose
effect-sensitive methods and traits.

[`File`]: https://doc.rust-lang.org/std/fs/struct.File.html
[`TcpStream`]: https://doc.rust-lang.org/std/net/struct.TcpStream.html
[`Stdout`]: https://doc.rust-lang.org/std/io/struct.Stdout.html
[`Option`]: https://doc.rust-lang.org/std/option/enum.Option.html
[`Sink`]: https://doc.rust-lang.org/std/io/struct.Sink.html
[`Bytes`]:  https://doc.rust-lang.org/std/io/struct.Bytes.html

By enabling types to be generic over effects, it becomes possible for existing
libraries to be fully portable to different effect contexts. While also enabling
effect-specific libraries to be authored that make full use of the unique
capabilities provided by that context. This makes it possible to strike a
balance between portability of existing code and patterns, without inhibiting
the creation of effect-specific interfaces which are specifically tuned for a
target environment.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Declaring effect-generic types

Before we can create an effect-generic type, we have to define it. Just like
functions and traits now carry effect markers, this RFC introduces effect
markers for types too. We'll be using `TcpStream` and the `async` as our example
here. Today if you want to write an async and non-async version of `TcpStream`,
you have to write both versions separately. That will look something like this:

```rust
// non-async
struct TcpStream { … }
impl TcpStream {
    pub fn connect(addr: impl ToSocketAddrs) -> Result<Self> { … }
}

// async
struct AsyncTcpStream { … }
impl AsyncTcpStream {
    async pub fn connect(addr: impl AsyncToSocketAddrs) -> Result<Self> { … }
}
```

What we're seeing here is an async and non-async copy of the concrete type
`TcpStream`, a copy of the public constructor method `connect`, as well as a
copy of the trait `AsyncToSocketAddrs`. That last one might be surprising: since
`ToSocketAddrs` can take domains as well as IP addresses which may require DNS
lookups - which means networked IO.

The design in this RFC makes it possible to define a single type which is
abstract over effects. For our example here, we'll define a type `TcpStream`
which is generic over the `async` effect. The way we do this is by making our
type `TcpStream` take a `#[maybe(async)]` annotation on the declaration. Which
effectively works as a generic param that is shared by all methods, bounds,
and traits.

```rust
#[maybe(async)]
struct TcpStream { … }

impl #[maybe(async)] TcpStream {
    #[maybe(async)]
    pub fn connect(addr: impl #[maybe(async)] ToSocketAddrs) -> Result<Self> { … }
}
```

## Using effect-generic types

Now that we've defined a type `TcpStream` which is generic over the `async`
effect, it's time to look at how we can create an instance of it. Let's start
again with what it would look like today if we had two separate types `TcpStream` and `AsyncTcpStream`:

```rust
use std::net::{TcpStream, AsyncTcpStream};

// non-async
let tcp_stream = TcpStream::connect("localhost:8080")?;

// async
let tcp_stream = AsyncTcpStream::connect("localhost:8080").await?;
```

Both uses look remarkably similar, except for the extra "async" and "await"
notations sprinkled throughout. It also requires two separate imports to work.
Now let's compare that with the effect-generic design from this RFC:

```rust
use std::net::TcpStream;

// non-async
let tcp_stream = TcpStream::connect("localhost:8080")?;

// async
let tcp_stream = TcpStream::connect("localhost:8080").await?;
```

With this design we no longer need two separate types. Because the async effect
is now a generic param on the type, and we know whether we're calling `.await`
or not, we can infer whether we're trying to use the async or non-async version
of `TcpStream`.

Of course sometimes inference will fail, and it needs to be possible to
explicitly declare which variant we want. For that we can use the explicit
turbofish notation we've introduced in the effect-generic functions RFC. Because
the effect is shared by both the constructor and the type, we can either
hard-code the effect on the constructor method or on the type itself:

```rust
use std::net::TcpStream;

// explicit async via constructor
let tcp_stream = TcpStream::connect::<async>("localhost:8080")?;

// explicit async via type
let tcp_stream: TcpStream<async> = TcpStream::connect("localhost:8080")?;
```

Finally, it's also possible to *not* directly declare the effect, and instead
make it depend on the function it's called in. This works the same as calling
effect-generic functions in other effect-generic functions: the concrete effect
of the type will depend on the effect of the calling function. That means that
despite writing `connect(..).await` in the function body, it may be turned into
a no-op depending on whether the calling function is async or not:

```rust
#[maybe(async)]
fn maybe_async_function() -> io::Result<()> {
    let tcp_stream = TcpStream::connect("localhost:8080").await?;
}
```

## Effect-specific fields

IO types will typically use different syscalls and hold different types
depending on what platform they are constructed on. Internally a `TcpStream` on
Linux or macOS will typically hold onto an [`OwnedFd`]. While that same
`TcpStream` on Windows will hold onto an [`OwnedSocket`]. Depending on what
platform we're compiling to, we'll `#[cfg]` different internals for our types.

[`OwnedFd`]: https://doc.rust-lang.org/std/os/fd/struct.OwnedFd.html#impl-From%3CTcpStream%3E-for-OwnedFd
[`OwnedSocket`]: https://doc.rust-lang.org/std/os/windows/io/struct.OwnedSocket.html

Similarly when building types for different effects, we may want to use
different system calls and internals as well. While for example blocking network
calls on Linux block until they complete, in order to perform those same system
calls without blocking we typically need some handle to an [`epoll`
instance][epoll]. Here is what the fields for both an async and non-async
version of `TcpStream` would look like on Linux today:

[epoll]: https://www.man7.org/linux/man-pages/man2/epoll_create.2.html

```rust
// non-async
struct TcpStream {
    fd: OwnedFd,
}

// async
struct TcpStream {
    fd: OwnedFd,
    reactor: EpollHandle,
}
```

The field `reactor` is specific to the `async` effect only. If we want to define
a version of `TcpStream` which is generic over the `async` effect, that field
needs to be optional. To enable that, this RFC introduces a new `effect`
parameter to the `#[cfg]` attribute. If we apply that to our `TcpStream`
example, that would look like this:

```rust
#[maybe(async)]
struct TcpStream {
    fd: OwnedFd,
    #[cfg(effect = async)]
    reactor: EpollHandle,
}
```

Fields in types can be conditional over multiple effects using `#[cfg(any(..))]`
or apply other boolean logic such as `#[cfg(not(..))]`. This just makes it
possible to configure fields in types using the same system people are already
used to.

## Effect-generic trait impls

In our example so far we've shown how to declare an effect-generic type
`TcpStream` which has a single effect-generic method `connect`. But what if we
wanted to implement a trait that was generic over effects too? Let's say we
wanted to implement a trait `Read` which is generic over the `async` effect
today. That would look like this:

```rust
// effect-generic trait declaration
use std::io::Read;

// non-async
struct TcpStream { … }
impl Read for TcpStream {
    fn read(&mut self, buf: &mut [u8]) -> Result<usize> { … }
}

// async
struct AsyncTcpStream { … }
impl async Read for TcpStream {
    async fn read(&mut self, buf: &mut [u8]) -> Result<usize> { … }
}
```

Like we described in the [effect-generic trait declarations RFC][egt], using
effect-generics we can define a single trait which can be implemented here as
either async or non-async on effect-specific types. With effect-generic type
declarations we can unify these types into a single type, and define a single
`TcpStream` type whose `Read` trait is generic over the `async` effect:

```rust
#[maybe(async)]
struct TcpStream { … }

impl #[maybe(async)] Read for #[maybe(async)] TcpStream {
    #[maybe(async)]
    fn read(&mut self, buf: &mut [u8]) -> Result<usize> { … }
}
```

Just like with inherent methods, the `async` effect is shared by the type,
trait, and trait methods. If the type is constructed as `async`, the trait's
methods will be `async` too. This direct connection between type, traits, and
methods should end up feeling very natural to use:

```rust
// non-async
let stream = TcpStream::connect("localhost:8080")?;
let mut buf = [0; 1024];
let bytes_read = stream.read(&mut buf)?;

// async
let stream = TcpStream::connect("localhost:8080").await?;
let mut buf = [0; 1024];
let bytes_read = stream.read(&mut buf).await?;
```

Just like types may need different fields depending on effects, types such as
`TcpStream` may want to perform different system calls depending on which
effects are present. To support this, functions can use the `async_eval_select`
built-in introduced in the [effect-generic bounds and functions RFC][egf]. This
allows functions to specialize their function bodies depending on the `async`
effect is present or not.

```rust
#[maybe(async)]
struct TcpStream { … }

impl #[maybe(async)] Read for #[maybe(async)] TcpStream {
    #[maybe(async)]
    fn read(&mut self, buf: &mut [u8]) -> Result<usize> {
        unsafe {
            let args = (self, &mut buf);
            async_eval_select(args, read, async_read).await
        }
    }
}

fn read(&mut TcpStream, buf: &mut [u8]) { … }
async fn async_read(&mut TcpStream, buf: &mut [u8]) { … }
```

While this works, the ergonomics of this aren't great right now. Improvements
like for example using  an in-line `if cfg!(effect = async) {}` notation are
discussed in the Future Possibilities section of the [effect-generic bounds and
functions RFC][egf]. See the [Future Possibilities][#if-cfg-effect]  section of
this RFC for what this example could look like if we had `cfg!(effect)` in
function bodies.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Desugaring bare types

*TODO: this needs to be updated to use the new effects desugaring in the
compiler. This still refers to the outdated const-bool design, where it should
use the updated const-enum design. This is definitely the weakest part of this
draft RFC, and we should update this once Oli is back from PTO.*

We can keep building on our earlier `TcpStream` example to show off how it
desugars in the compiler.  We will do that incrementally, starting with just the
bare type definition which looks like this:

```rust
#[maybe(async)]
struct TcpStream {
    fd: OwnedFd,
    #[cfg(effect = async)]
    reactor: EpollHandle,
}

// non-async construction
let stream: TcpStream<!async> = TcpStream::connect("localhost:8080")?;

// async construction
let stream: TcpStream<async> = TcpStream::connect("localhost:8080").await?;
```

Just like effect-generic functions and traits, the compiler will lower effects
to const-bools. The most challenging bit here will be to correctly lower the
optional field, but we can achieve that by making the type generic not just over
a const generic - but a regular generic too.

```rust
struct TcpStream<const bool IS_ASYNC, F: Fields> {
    fd: OwnedFd,
    fields: F,
}

struct AsyncFields {
    reactor: EpollHandle,
}
trait Fields {}
impl Fields for () {}               // zst for non-async variant
impl Fields for AsyncFields {}  // actual data for async variant

// non-async construction
let stream: TcpStream<false, ()> = TcpStream::connect(…)?;

// async construction 
let stream: TcpStream<true, AsyncFields> = TcpStream::connect(…).await?;
```

In this case the type `TcpStream<false, ()>` will have the same layout as
`std::net::TcpStream` does today. But in the case of `TcpStream<true,
AsyncFields>`, the extra fields will be present enabling them to be accessed
from async-specific code.

## Desugaring constructors

Now that we have seen how the base type desugars, it's time to include our
constructor method `connect` as well. This method is generic over the `async`
effect, and needs to choose between an async and non-async variant. As a
reminder, here is our definition again:

```rust
// non-async
#[maybe(async)]
struct TcpStream {
    fd: OwnedFd,
    #[cfg(effect = async)]
    reactor: EpollHandle,
}

#[maybe(async)]
impl TcpStream {
    #[maybe(async)]
    pub fn connect(addr: impl #[maybe(async)] ToSocketAddrs) -> Result<Self> { … }
}
```

We can lower this as follows, continuing to thread the const-bool and fields
param through the trait's methods too. The inference mechanism for selecting
between effectful and non-effectful functions we've introduced previously would
be applied here to select between the async and non-async version of the method:

```rust
struct TcpStream<const bool IS_ASYNC, F: Fields> {
    fd: OwnedFd,
    fields: F,
}

struct AsyncFields { … }
trait Fields {}
impl Fields for () {}             
impl Fields for AsyncFields {}

// non-async
impl TcpStream<false, ()> {
    pub fn connect(addr: impl ToSocketAddrs<false>) -> Result<Self> { … }
}

// async
impl TcpStream<true, Fields> {
    pub async fn connect(addr: impl ToSocketAddrs<true>) -> Result<Self> { … }
}
```

The trait `ToSocketAddrs` is generic over the `async` effect here too - and in
turn also inherits the "asyncness" on the type by threading through the
const-generic `IS_ASYNC` param. And because the constructor is monomorphized
into two concrete implementations, it provides an opportunity to also hard-code
the additional fields.

## Desugaring types with trait impls

Now that we have desugared the bare type and constructor, it's time to also
desugar the `Read` trait. Just like the `impl ToSocketAddrs` param on `connect`,
the asyncness of the type is threaded through the trait as well. As a reminder,
here is the high-level definition again:

```rust
#[maybe(async)]
struct TcpStream { … }

impl #[maybe(async)] Read for #[maybe(async)] TcpStream {
    #[maybe(async)]
    fn read(&mut self, buf: &mut [u8]) -> Result<usize> { … }
}
```

This uses a trait `Read` which has the following definition:

```rust
#[maybe(async)]
trait Read {
    #[maybe(async)]
    fn read(&mut self, buf: &mut [u8]) -> Result<usize> { … }
}
```

This trait would desugar the same way as we've covered in [the effect-generic bounds and functions RFC][egf]:

```rust
trait Read<const IS_ASYNC: bool = false> {
    type Ret = Result<usize>;
    fn read(&mut self, buf: &mut [u8]) -> Self::Ret { … }
}
```

Which when implemented on our type `TcpStream` would thread through like this,
where the async effect of the trait is again dependent on the async effect of
the type it's implemented on:

```rust
struct TcpStream<const bool IS_ASYNC, F: Fields> { … }

// non-async
impl Read<false> for TcpStream<false, ()> {
    fn read(&mut self, buf: &mut [u8]) -> Self::Ret { … }
}

// async
impl Read<true> for TcpStream<true, AsyncFields> {
    type Ret = impl Future<Output = Result<usize>;
    fn read(&mut self, buf: &mut [u8]) -> Self::Ret { … }
}
```

## TODO: Effect-specific implementations

- `impl AsyncOnlyTrait for MyType`
- this should be legal, just like we can implement traits exclusively for type-
and const generics

Notation for methods:
```rust
#[maybe(async)]
struct TcpStream { … }

impl TcpStream {
    pub fn non_async_method() {}
}

impl async TcpStream {
    pub fn async_only_method() {}
}
```

Desugaring for methods:

```rust
struct TcpStream<const bool IS_ASYNC, F: Fields> { … }


impl TcpStream<false, ()> {
    pub fn non_async_method() {}
}

impl TcpStream<true, AsyncFields> {
    pub fn async_only_method() {}
}
```

Notation for traits:
```rust
#[maybe(async)]
struct TcpStream { … }

trait NonAsyncTrait {}
impl NonAsyncTrait for TcpStream {}

trait AsyncOnlyTrait {}
impl AsyncOnlyTrait for async TcpStream {}
```

Desugaring for traits:

```rust
struct TcpStream<const bool IS_ASYNC, F: Fields> { … }

trait NonAsyncTrait {}
impl NonAsyncTrait for TcpStream<false, ()> {}

trait AsyncOnlyTrait {}
impl AsyncOnlyTrait for TcpStream<true, AsyncFields> {}
```

### TODO: Adding effect-specific fields to ABI-stable types

- The lens through which to view this is: "backwards-compat"
- The constraint is that we don't change the layout of code which already compiles.
- Effect-generic types add new generic params to existing types: the new variants don't inherit the same ABI stability guarantees.

Example:
```rust
// this
struct TcpStream {
    fd: OwnedFd,
}

// is the same size as this:
struct TcpStream {
    fd: OwnedFd,
    fields: (), // because this has no size
}
```

# Drawbacks
[drawbacks]: #drawbacks

None so far.

# TODO: Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## TODO: Null hypothesis - duplicate all types

- rather than making effects polymorphic, we could duplicate types across effect
  boundaries
- practically that would mean having e.g. `std::fs::File`, and
  `std::fs::AsyncFile`

# TODO: Prior art
[prior-art]: #prior-art

# Unresolved questions
[unresolved-questions]: #unresolved-questions

# Future possibilities
[future-possibilities]: #future-possibilities

## `if cfg!(effect) {}` 
[if-cfg-effect]: #if-cfg-effect

Earlier in this RFC we showed an example using `async_eval_select` to choose
between an async and non-async function body. This is necessary because code,
especially in IO types, may want to diverge depending on which effects are
present. In this RFC we've shown the `#[cfg(effect = <effect>)]` notation to
conditionally enable fields in types. It's not hard to imagine how the same
attribute system could be generalized to work in function bodies too. Here is
the same `TcpStream` example implementing the `Read` trait from earlier in this
RFC, translated to use `if cfg!()` instead:

```rust
#[maybe(async)]
struct TcpStream { … }

impl #[maybe(async)] Read for #[maybe(async)] TcpStream {
    #[maybe(async)]
    fn read(&mut self, buf: &mut [u8]) -> Result<usize> {
        if cfg!(effect = async) {
            … // async
        } else {
            … // non-async
        }
    }
}
```

This would desugar the same way `async_eval_select` would, but with the
arguments mapped to the parent arguments. But this would have the distinct
benefit of being significantly easier to write and use.

[egt]: https://rust-lang.github.io/keyword-generics-initiative/explainer/effect-generic-trait-declarations.html
[egf]: https://rust-lang.github.io/keyword-generics-initiative/explainer/effect-generic-bounds-and-functions.html
