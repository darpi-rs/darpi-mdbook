# App

The `app` macro generates code that link all the other components together. 

```rust,ignore
#[tokio::main]
async fn main() -> Result<(), darpi::Error> {
    let address = format!("127.0.0.1:{}", 3000);
    app!({
        // if this the address is a literal
        // it will be checked, if it is a valid address at compiled time
        // in this case, it is a variable so that is impossible
        address: address,
        // here we set the function that builds the shaku container
        module: make_container => Container,
        // a set of global middleware that will be executed for every handler
        // middleware is executed in the order defined by the user
        middleware: [body_size_limit(128)],
        bind: [
            {
                route: "/login",
                method: Method::POST,
                // the POST method allows this handler to have
                // to have access to a request body
                handler: login
            },
            {
                route: "/hello_world/{name}",
                method: Method::GET,
                // GET request handlers are not allowed
                // to have request body as an argument
                // it is a compile time error
                handler: do_something
            },
        ],
    })
    .run()
    .await
}
```
