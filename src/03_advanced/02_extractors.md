# Extractors

As we learned from the [Handlers](../02_basics/03_handlers.md) chapter, we can get a request body by declaring `#[body] payload: Json<Name>` in the handler arguments. 

Extractors are a way to get a request body in a certain format.

While the framework provides `Json`, `Xml` and `Yaml` extractor implementations, you might need to implement your own. 

To do that, we need to implement the `FromRequestBody` or `FromRequestBodyWithContainer` trait, depending on whether we need to use the dependency injection container.


Lets start with `FromRequestBody`. The trait definition is as follows.
As you can see, we have 2 generic types.
If in any case, an error is returned from the trait methods, the request processing will immediately stop and an error response will be returned to the client.
This is the reason the `E` type has `ResponderError` bounds.


```rust
#[async_trait]
pub trait FromRequestBody<T, E>
where
    T: de::DeserializeOwned + 'static,
    E: ResponderError + 'static,
{
    async fn assert_content_type(_content_type: Option<&HeaderValue>) -> Result<(), E> {
        Ok(())
    }
    async fn extract(headers: &HeaderMap, b: Body) -> Result<T, E>;
}
```

As the implementor, we need to decide whether we want to require the content type header or not.
You can choose to not have it as a requirement from the client, then you can simply not implement it and use the default implementation.
And finally, the `extract` method is the thing that does most of the work.
You can very well imagine that a basic implementation of the would be as follows.

```rust
use darpi::hyper::body;
pub struct MyFormat<T>(pub T);

#[async_trait]
impl<T> FromRequestBody<MyFormat<T>, MyErr> for MyFormat<T>
    where
        T: DeserializeOwned + 'static,
{
    async fn assert_content_type(content_type: Option<&HeaderValue>) -> Result<(), MyErr> {
        if let Some(hv) = content_type {
            if hv != "application/my_format" {
                return Err(MyErr::InvalidContentType);
            }
            return Ok(());
        }
        Err(MyErr::MissingContentType)
    }
    async fn extract(_: &HeaderMap, b: Body) -> Result<MyFormat<T>, MyErr> {
        let full_body = body::to_bytes(b).await?;
        // here is where we need to deserialize the full_body bytes
        // for the example, serde_json is used.
        let ser: T = serde_json::from_slice(&full_body)?;
        Ok(MyFormat(ser))
    }
}

#[derive(Display)]
pub enum MyErr {
    ReadBody(hyper::Error),
    Serde(Error),
    InvalidContentType,
    MissingContentType,
}

impl From<Error> for MyErr {
    fn from(e: Error) -> Self {
        Self::Serde(e)
    }
}

impl From<hyper::Error> for MyErr {
    fn from(e: hyper::Error) -> Self {
        Self::ReadBody(e)
    }
}

impl ResponderError for MyErr {}
```


If you have dependencies that you need to access, you can instead implement `FromRequestBodyWithContainer`.
The implementation of the methods is exactly the same as above. The only difference is the `C` generic type that you need to specify the relevant bounds.


```rust
#[async_trait]
impl<F, T, E, C> FromRequestBodyWithContainer<T, E, C> for F
where
    F: FromRequestBody<T, E> + 'static,
    T: de::DeserializeOwned + 'static,
    E: ResponderError + 'static,
    C: std::any::Any + Sync + Send,
{
    async fn assert_content_type(content_type: Option<&HeaderValue>, _: Arc<C>) -> Result<(), E> {
        F::assert_content_type(content_type).await
    }
    async fn extract(headers: &HeaderMap<HeaderValue>, b: Body, _: Arc<C>) -> Result<T, E> {
        F::extract(headers, b).await
    }
}
```

As an example, we can see the `graphql` integration implements this trait.
As you can see the line `let opts = container.resolve().get();` is how you can fetch the specified dependency.

```rust
pub trait MultipartOptionsProvider: Interface {
    fn get(&self) -> MultipartOptions;
}

#[derive(Component)]
#[shaku(interface = MultipartOptionsProvider)]
pub struct MultipartOptionsProviderImpl {
    opts: MultipartOptions,
}

impl MultipartOptionsProvider for MultipartOptionsProviderImpl {
    fn get(&self) -> MultipartOptions {
        self.opts.clone()
    }
}

#[async_trait]
impl<C: 'static> FromRequestBodyWithContainer<GraphQLBody<BatchRequest>, GraphQLError, C>
for GraphQLBody<BatchRequest>
    where
        C: HasComponent<dyn MultipartOptionsProvider>,
{
    async fn extract(
        headers: &HeaderMap,
        mut body: darpi::Body,
        container: Arc<C>,
    ) -> Result<GraphQLBody<BatchRequest>, GraphQLError> {
        let content_type = headers
            .get(http::header::CONTENT_TYPE)
            .and_then(|value| value.to_str().ok())
            .map(|value| value.to_string());

        let (tx, rx): (
            Sender<std::result::Result<Bytes, _>>,
            Receiver<std::result::Result<Bytes, _>>,
        ) = bounded(16);

        darpi::job::FutureJob::from(async move {
            while let Some(item) = body.next().await {
                if tx.send(item).await.is_err() {
                    return;
                }
            }
        })
            .spawn()
            .map_err(|e| GraphQLError::Send(e.to_string()))?;

        let opts = container.resolve().get();
        Ok(GraphQLBody(BatchRequest(
            async_graphql::http::receive_batch_body(
                content_type,
                rx.map_err(|e| std::io::Error::new(std::io::ErrorKind::InvalidInput, e))
                    .into_async_read(),
                opts,
            )
                .await
                .map_err(|e| GraphQLError::ParseRequest(e))?,
        )))
    }
}
```