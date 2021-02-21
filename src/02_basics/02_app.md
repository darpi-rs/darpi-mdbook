# App

The `app` macro generates code that link all the other components together. 

```rust
#[tokio::main]
async fn main() -> Result<(), darpi::Error> {
    let address = format!("127.0.0.1:{}", 3000);
    app!({
        address: address,
        container: {
            factory: make_container(),
            type: Container
        },
        middleware: {
            request: [body_size_limit(128), decompress()],
            response: []
        },
        jobs: {
            request: [],
            response: [first_sync_job, first_sync_job1, first_sync_io_job]
        },
        handlers: [
            {
                route: "/",
                method: Method::GET,
                handler: home
            },
        ]
    })
    .run()
    .await
}
```

Lets break it down.

`address` can be either a `String` or a `&'static str`

`container` has a `factory` function, which is used to create a `shaku` container
and a `type` (supports arguments too), which is the return type of the `factory`.

`middleware`, `jobs` and `handlers` we will tackle in the next chapters.
