---
title: 'api-version: axum middleware for header based version selection'
date: 2025-03-20
permalink: 2025-03-20-api-version/
tags:
- rust
- axum
---

There are several ways to handle API versioning and two of the most prominent are path-based and header-based. Path-based means to include the version as a prefix segment, e.g. `https://api.example.com/v1/test` and header-based means to convey the desired version as a HTTP header. There are pros and cons for both and as always the choice/decision depends/is hard.

In [axum](https://docs.rs/axum/latest/axum/) it is quite simple to implement path-based versioning, e.g.:

```rust
let app = Router::new()
    .route("/", get(ready))
    .route("/v0/test", get(ok_0))
    .route("/v1/test", get(ok_1));
```

Here we define a readiness endpoint `/` without any version and a versioned `/test` endpoint for the versions "v0" and "v1". Of course the `/test` endpoints can be called like this:

```
http localhost:8080/v0/test
http localhost:8080/v1/test
```

But what if we (also) want to support header-based versioning, i.e. call like this:

```
http localhost:8080/test x-api-version:v0
http localhost:8080/test x-api-version:v1
```

Well, this is not too hard to implement on your own, but I created a little library that helps you: welcome `api-version`!

All you have to do is
- define the valid versions which usually are some contiguous range and 
- wrap your router with the `api-version` middleware:

```rust
const API_VERSIONS: ApiVersions<2> = ApiVersions::new([0, 1]);
let app = Router::new()
    .route("/", get(ready))
    .route("/v0/test", get(ok_0))
    .route("/v1/test", get(ok_1));
let app = ApiVersionLayer::new(API_VERSIONS, ReadyFilter).layer(app);
```

BTW, you do not have to use a `const` for the creation of the `ApiVersions`, but it will give you some compile time safety.

For more details take a look at [api-version on GitHub](https://github.com/hseeberger/api-version).
