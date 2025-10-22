# Awesome, tiny crates!

Rust has wonderful crates like `serde`, `tokio`, `axum`, `clap` and many others.

Everyone talks about these crates, and everyone knows them. But there are thousands of small utility crates that just do 1 thing well,
but most aren't aware of them. Let's change that!

The list focuses on crates that fit in the [Rust patterns](https://lib.rs/rust-patterns) category, hopefully you'll try some of these out!

# Non-empty collections

I like pushing invariants into the type system, and make invalid states impossible to represent.
That's one of the things that make Rust fun! (although, sometimes I can [overdo](https://arhan.sh/blog/the-generativity-pattern-in-rust/) it 👀)

I've long searched for a good crate that provides non-empty collections, notably `Vec`.
There is [`nonempty`](https://lib.rs/crates/nonempty), which is the most popular, but it uses a representation like this:

```rust
struct NonEmptyVec<T> {
  head: T,
  tail: Vec<T>
}
```

Sure, this makes it literally impossible to construct a non-empty `Vec`, **but at what cost?**

- `Vec<T>` to `NonEmptyVec<T>` is `O(n)`. **Not zero cost.**
- Since it doesn't deref to `&[T]` anymore, it's just much harder to compose with everything else...
- The only advantage of this layout is that you can store 1 element without allocation.
  Cool, but I've personally never needed that.

That's when I discovered [`mitsein`](https://lib.rs/crates/mitsein). This library:

- Contains dozens of non-empty collections: `Vec`, `Hash`, ...
- Includes feature flags for popular crates, like `indexmap`, `serde`, `smallvec`, ...
- Has extremely good documentation. Every API is extensively documented.
- Uses the zero-cost layout `#[repr(transparent)] struct NonEmptyVec<T>(Vec<T>)`

Here's a small example from the library:

```rust
use mitsein::NonEmpty;

fn first(vec: NonEmpty<Vec<u32>>) -> u32 {
    vec.first()
}
```

# Derive aliases

Rust's `derive` feature is incredibly powerful, with crates like [`serde`](https://lib.rs/crates/serde),
[`strum`](https://lib.rs/crates/strum), [`derive_more`](https://lib.rs/crates/derive_more),
[`num_traits`](https://lib.rs/crates/num-traits) and many others providing custom derive macros.

Especially for people who like newtypes, it's not uncommon to have 10+ derives applied to the same item.
At which point `rustfmt` will place every derive onto its own line. Then before you know it, the `#[derive]`
attribute is twice the size of whatever item you are applying it to!

Say no more. With [`derive_aliases`](https://lib.rs/crates/derive_aliases), you can define an alias that expands into a
bunch of derives, replacing code like this:

```rust
#[derive(Debug, ..Ord, ..Copy)]
struct User;
```

With this:

```rust
#[derive(Debug, PartialEq, Eq, PartialOrd, Ord, Copy, Clone)]
struct User;
```

# Tests, now with twice the fun

Ever wrote a test with a bunch of calls to `assert!` - and the test fails. But only the first panic is shown! Oh no!
You now need to manually re-run the test and comment out the first `assert!` to get information about the 2nd one.

This experience isn't the best. The [`assert2`](https://lib.rs/crates/assert2) crate is here to save us all, it provides the `check!` macro
which is like `assert!` but it doesn't fail the test immediately. Instead, all failures of `check!` are collected and then reported all at once!

The assertion macros in this crate provide more descriptive and helpful error messages. **🌈 It also have color! 🌈**,
and detailed information about what exactly failed!

```rust
assert!(6 + 1 <= 2 * 3);
```

With the error:

![colorful assertion](https://raw.githubusercontent.com/de-vri-es/assert2-rs/ba98984a32d6381e6710e34eb1fb83e65e851236/binary-operator.png)

`assert2` even has **diffs**!

```rust
check!((3, Some(4)) == [1, 2, 3].iter().size_hint());
```

With the error:

![diff](https://raw.githubusercontent.com/de-vri-es/assert2-rs/54ee3141e9b23a0d9038697d34f29f25ef7fe810/single-line-diff.png)

Where have you been my whole life, `assert2`??

# Configuration aliases

The [`cfg_aliases!`](https://lib.rs/crates/assert2) crate exports a macro that you invoke inside of
the [`build.rs`](https://doc.rust-lang.org/cargo/reference/build-scripts.html) file.
It allows you to have nice, rememberable names for configuration predicates.

```rust
cfg_aliases! {
    // Platforms
    wasm: { target_arch = "wasm32" },
    android: { target_os = "android" },
    macos: { target_os = "macos" },
    linux: { target_os = "linux" },

    // Backends
    surfman: { all(unix, feature = "surfman", not(wasm)) },
    glutin: { all(feature = "glutin", not(wasm)) },
    wgl: { all(windows, feature = "wgl", not(wasm)) },
    dummy: { not(any(wasm, glutin, wgl, surfman)) },
}
```

# Struct patching

Let's say your program first reads config from `~/.config/foo.toml`, and if the user's current working directory contains `.foo/config.toml` it also reads that.

When we read the first config file, we want to create a `Config` struct with all values set to the default if not specified.
Reading the 2nd config file, we want to record only the values the user has set.

The [`struct-patch`](https://lib.rs/crates/struct-patch) crate makes that process easy and fun, with a derive macro:

```rust
#[derive(Patch, Serialize, Deserialize)]
#[patch(attribute(derive(Serialize, Deserialize)))]
struct Config {
    timeout: Duration,
    clipboard_provider: String,
    editor_config: bool,
}

// The above generates:
#[derive(Serialize, Deserialize)]
struct ConfigPatch {
    timeout: Option<Duration>,
    clipboard_provider: Option<String>,
    editor_config: Option<bool>,
}
```

By deserializing the 2nd config file into `ConfigPatch`, you can then apply only the overrides that the user has set. Handy!

# Ranged integers

It's not uncommon to have integers that we want to limit to a certain range, for example a `Percentage` that can only hold
a value in the range `1..=100`. The crate [`deranged`](https://lib.rs/crates/deranged) contains special wrapper around every
integer kind from the standard library, and you can specify them like `RangedI8::<1, 100>`

# Easy and fun extension traits

Have you ever wished you can just create methods on types from other crates? Let's say you call `result.map_err(Into::into)` a lot,
and you *wish* the standard library had this functionality built-in.

You can do that! This pattern is called "Extension traits" -
you define a trait with the methods that you want (`err_into`), and then implement it for the type:

```rust
pub trait ResultExt<T, E> {
    fn err_into<U>(self) -> Result<T, U>
    where
        E: Into<U>;
}

impl<T, E> ResultExt<T, E> for Result<T, E> {
    fn err_into<U>(self) -> Result<T, U>
    where
        E: Into<U>,
    {
        self.map_err(Into::into)
    }
}
```

Needless to say, it's **really** verbose.
You have to repeat the signature **twice** (😱), and since generics are usually involved, it's common to amount to a lot of boilerplate.

[`easy-ext`](https://lib.rs/crates/easy-ext) promises to cut down this boilerplate in **half**:

```rust
use easy_ext::ext;

#[ext(ResultExt)]
pub impl<T, E> Result<T, E> {
    fn err_into<U>(self) -> Result<T, U>
    where
        E: Into<U>,
    {
        self.map_err(Into::into)
    }
}
```

# Bounded vector

While [non-empty](#non-empty-collections) collections are useful, sometimes we have a list of elements that can only ever be 2 or more.
Or, between 5 and 10. The [`bounded-vec`](https://lib.rs/crates/deranged) crate provides exactly this type - `BoundedVec<u8, 2, 4>` that holds 2, 3 or 4 `u8`s:

```rust
let data: BoundedVec<u8, 2, 4> = [1u8, 2].into();

assert_eq!(*data.first(), 1);
assert_eq!(*data.last(), 2);
assert_eq!(data.mapped(|x| x * 2), [2u8, 4].into());
```

# Implement `Display` from documentation comments

If you've ever written a library and want to have 100% documentation on everything, then you might have written errors that look like this:

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum DataStoreError {
    /// data store disconnected
    #[error("data store disconnected")]
    Disconnect(#[source] io::Error),
    /// the data for key is not available
    #[error("the data for key `{0}` is not available")]
    Redaction(String),
    /// invalid header
    #[error("invalid header (expected {expected:?}, found {found:?})")]
    InvalidHeader {
        expected: String,
        found: String,
    },
    /// unknown data store error
    #[error("unknown data store error")]
    Unknown,
}
```

We are repeating the same content! That's so much noise. We have an extra unnecessary line for every variant.
The [`displaydoc`](https://lib.rs/crates/displaydoc) crate allows us to avoid repeating ourselves - the documentation comments
are used to implement the `Display` trait!

```rust
use displaydoc::Display;
use thiserror::Error;

#[derive(Display, Error, Debug)]
pub enum DataStoreError {
    /// data store disconnected
    Disconnect(#[source] io::Error),
    /// the data for key `{0}` is not available
    Redaction(String),
    /// invalid header (expected {expected:?}, found {found:?})
    InvalidHeader {
        expected: String,
        found: String,
    },
    /// unknown data store error
    Unknown,
}
```

# Ergonomic multi-line string literals

Writing multi-line string literals in rust can be... a bit of a pain!o

On one hand, you have to sacrifice on indentation:

```rust
fn main() {
    println!(
        "\
create table student(
    id int primary key,
    name text
+)");
}
```

Another option is to increase syntax pollution levels to dangerous levels:

```rust
fn main() {
    println!(concat!(
        "create table student(\n",
        "    id int primary key,\n",
        "    name text,\n",
        ")\n",
    ));
}
```

What if we want the best of both world? We want:

- Gets formatted by rustfmt to match indentation levels of surrounding code
- It is readable, without syntactic noise
- Composes nicely with macros that use the formatting syntax

[`docstr!`](https://lib.rs/crates/docstr) is all of those things, and more!

```rust
fn main() {
    docstr!(println!
        /// create table student(
        ///     id int primary key,
        ///     name text,
        /// )
    );
}
```
