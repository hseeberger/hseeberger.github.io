---
title: 'Printing errors in Rust'
date: 2024-01-01
permalink: 2024-01-01-printing-errors/
tags:
- rust
---

In Rust an [`Error`](https://doc.rust-lang.org/std/error/trait.Error.html) could have a [`cause`](https://doc.rust-lang.org/std/error/trait.Error.html#method.cause) which could also have a cause ... Put another way: there is an error chain.

Sometimes it might be useful to print the whole error chain; probably not when showing errors to users/clients, but most probably when logging in order to understand the root cause of an error.

In Rust we also have two very prominent "error libraries": [`anyhow`](https://docs.rs/anyhow/latest/anyhow) and [`thiserror`](https://docs.rs/thiserror/latest/thiserror). Simplified, the former is for binaries and the latter for libraries.

`anyhow` has two very nice features. First, we can attach "context" to almost any error via the `context` extension method, which essentially creates an `anyhow::Error` with the error at hand as source. Second, we can use the alternate way of formatting – `"{:#}"` – to print the whole error chain:

```rust
error!(error = format!("{error:#}"), "cannot do this");
```

This snippet is assuming the `tracing` library being used and some `error` of type `anyhow::Error` in scope.

All right, this works fine for binaries, i.e. when we can use `anyhow::Error`. But what if we want to print – using `tracing::error` or similar – errors in library code? There we typically do not use `anyhow::Error`, but instead some custom error types, probably derived by `thiserror`. Using the above snippet for such a custom error would just print the error and omit the sources.

To get the whole error chain printed, we could either write some custom `print_error_chain` function and invoke it every time we want to print an error in library code, or simply leverage `anyhow`. Not the full `anyhow` machinery, but just the `anyhow!` macro, which is able to transform any error into an `anyhow::Error`:

```rust
error!(error = format!("{:#}", anyhow!(error)), "cannot do this");
```

This way we do not use `anyhow::Error` or `anyhow::Result` in any signatures (parameters or return types) in our library code, but we get the error chains printed very easily.

What do you think about this approach? I am looking forward to your comments.
