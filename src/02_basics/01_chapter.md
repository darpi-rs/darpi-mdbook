# Basics

`darpi` provides various primitives to build web servers and applications with Rust. It provides routing, middleware, pre-processing of requests, post-processing of responses, etc.

`darpi` uses `shaku` for statically verifiable dependency injection, since it is alligned with the goals `darpi` has.

It is important to note that `darpi` does not store dynamic information about the application.
Everything is achieved by code generation. For example, the provided routes are represented by an enum variants used in a match statement.

`darpi` solves conflicting paths by sorting, based on different of factors such as number of arguments in
a path and their position within the string.

`/user/{name}` and `/user/article` are conflicting, if a user's name happens to be `article`. Therefore, the more generic path should be matched later.
