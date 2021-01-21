# middleware

darpi’s middleware system allows us to add additional behavior to
request/response processing. Middleware can hook into an incoming request
process, enabling us to halt request processing to return a response early
but it cannot modify the request.
Middleware can also hook into response processing.

#### middleware types

- global (app)
  - Request
  - Response
- local (handler)
  - Request
  - Response


### possible arguments
- shaku container
    - `#[inject] my_arg: Arc<dyn SomeTrait>`
- request
    - `#[request_parts] rp: &darpi::RequestParts`
      `serde::Deserialize`. `/user/{id}/article/{id}` will deserialize both ids into `MyStruct`.
    - `#[body] body: &mut darpi::Body` if handler is not linked to a `GET` request
- response
  - `#[response] r: &mut darpi::Response<Body>`
- handler
    - `#[handler] arg: T` where the value must be provided when the middleware is invoked

    
```rust,ignore
use darpi::{middleware, request::PayloadError, Body, HttpBody};

#[middleware(Request)]
async fn body_size_limit(#[body] b: &mut Body, #[handler] size: u64) -> Result<(), PayloadError> {
    if let Some(limit) = b.size_hint().upper() {
        if size < limit {
            return Err(PayloadError::Size(size, limit));
        }
    }
    Ok(())
}
```

a more complicated example

```rust, ignore
#[middleware(Request)]
pub async fn authorize(
    #[handler] role: impl UserRole,
    #[request_parts] rp: &RequestParts,
    #[inject] algo_provider: Arc<dyn JwtAlgorithmProvider>,
    #[inject] token_ext: Arc<dyn TokenExtractor>,
    #[inject] secret_provider: Arc<dyn JwtSecretProvider>,
) -> Result<Token, Error> {
    let token_res = token_ext.extract(&rp).await;
    match token_res {
        Ok(jwt) => {
            let decoded = decode::<Claims>(
                &jwt,
                &DecodingKey::from_secret(secret_provider.secret().await.as_ref()),
                &Validation::new(algo_provider.algorithm().await),
            )
            .map_err(|_| Error::JWTTokenError)?;

            if !role.is_authorized(&decoded.claims.role) {
                return Err(Error::NoPermissionError);
            }

            Ok(decoded.claims.sub)
        }
        Err(e) => return Err(e),
    }
}
```
