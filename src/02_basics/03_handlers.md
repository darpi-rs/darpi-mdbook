# Handlers

The `handler` macro has 2 optional arguments.
1. the shaku container type, which is required if there is an `#[inject]` argument
in the handler or middleware used by the handler.   
2. a list of middleware eg `[....]`

in this order.

A request handler is an async function that accepts zero or more arguments.
The arguments can be provided in several ways, by macro attributes.
Handlers bound by a `GET` method do not have access to a request body.
The return type must `impl darpi::response::Responder`. For convenience, it is implemented for all common types.

##possible arguments

- shaku container
  - `#[inject] my_arg: Arc<dyn SomeTrait>`
- request    
  - `#[request_parts] rp: &darpi::RequestParts`
  - `#[query] q: MyStruct` where `MyStruct` has to implement
    `serde::Deserialize`. Furthermore, if wrapped in an `Option` it is not mandatory.
  - `#[path] path: MyStruct` where `MyStruct` has to implement
    `serde::Deserialize`. `/user/{id}/article/{id}` will deserialize both ids into `MyStruct`.
  - `#[body] body: &darpi::Body` if handler is not linked to a `GET` request
- middleware
  - `#[middleware(i) size: u64]` where `i` is a literal index of the middleware linked to the handler
    

```rust,ignore
use darpi_middleware::body_size_limit;
use darpi::{app, handler, response::Responder, Method, Path, Json, Query};
use serde::{Deserialize, Serialize};

#[derive(Deserialize, Serialize, Debug, Path, Query)]
pub struct Name {
    name: String,
}

// imagine body_size_limit is a middleware
// function that returns Result<u64, Error>
#[handler([body_size_limit(64)])]
async fn do_something(
    // the request query is deserialized into Name
    // if deseriliazation fails, it will result in an error response
    // to make it optional wrap it in an Option<Name>
    #[query] p: Name,
    // the request path is deserialized into Name
    #[path] p: Name,
    // the request body is deserialized into the struct Name 
    // it is important to mention that the wrapper around Name
    // should implement darpi::request::FromRequestBody
    // Common formats like Json, Xml and Yaml are supported out
    // of the box but users can implement their own
    #[body] payload: Json<Name>,
    // we can access the T from Ok(T) in the middleware result  
    #[middleware(0)] size: u64, 
    // returning a String works because darpi has implemented
    // the Responder trait for common types
) -> String {
    format!("{} sends hello to {} size {}", p.name, payload.name, size)
}

```
