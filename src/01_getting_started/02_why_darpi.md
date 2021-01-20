# Why darpi?

We all love how Rust allows us to write fast, safe software.
`darpi` attempts at extending that and having stronger invariants proven that are specific to the web.

Performance, safety and simplicity are the main goals the framework is trying to achieve.


Applications built with `darpi` have the potential to be much faster and
use fewer resources than other frameworks. To achieve its goals, `darpi` utilizes macros, heavily.

```rust,ignore
[dependencies]
darpi = {git = "https://github.com/petar-dambovaliev/darpi.git", branch = "master"}
tokio = {version = "0.2.11", features = ["full"]}
shaku = {version = "0.5.0", features = ["thread_safe"]}
```

```rust,ignore
use darpi::{app, handler, Error, Method};
use shaku::module;

fn make_container() -> Container {
    let module = Container::builder().build();
    module
}

module! {
    Container {
        components = [],
        providers = [],
    }
}

#[handler]
async fn hello_world() -> String {
    format!("hello world")
}

#[tokio::main]
async fn main() -> Result<(), Error> {
    app!({
        address: "127.0.0.1:3000",
        module: make_container => Container,
        middleware: [],
        bind: [
            {
                route: "/hello_world",
                method: Method::GET,
                handler: hello_world
            },
        ],
    })
    .run()
    .await
}

```
