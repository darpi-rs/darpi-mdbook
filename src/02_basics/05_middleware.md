# middleware

darpi’s middleware system allows us to add additional behavior to
request/response processing. Middleware can hook into an incoming request
process, enabling us to halt request processing to return a response early
and modify the request body.
Middleware can also hook into response processing.

#### middleware types

- global (app)
  - Request
  - Response
- local (handler)
  - Request
  - Response


### possible arguments
- Request middleware
  - dependency injection container
      - `#[inject] my_arg: Arc<dyn SomeTrait>`
  - request
      - `#[request] rp: &mut darpi::Request<darpi::Body>` supports shared and mutable reference
  - handler
    - `#[handler] arg: T` where the value must be provided when the middleware is invoked
  

- Response middleware
  - dependency injection container
    - `#[inject] my_arg: Arc<dyn SomeTrait>`
  - response
     - `#[response] r: &mut darpi::Response<Body>` supports shared and mutable reference
  - handler
    - `#[handler] arg: T` where the value must be provided when the middleware is invoked


While i recommend using the macros. If you really want to, you could implement the middleware request and response traits.

    
```rust
use darpi::{middleware, request::PayloadError, Body, HttpBody};

#[middleware(Request)]
pub async fn body_size_limit(
  #[request] r: &Request<Body>,
  #[handler] size: u64,
) -> Result<(), PayloadError> {
  if let Some(limit) = r.size_hint().upper() {
    if size < limit {
      return Err(PayloadError::Size(size, limit));
    }
  }
  Ok(())
}
```

a more complicated example

```rust
#[middleware(Request)]
pub async fn authorize(
  #[handler] role: impl UserRole,
  #[request] rp: &Request<Body>,
  #[inject] algo_provider: Arc<dyn JwtAlgorithmProvider>,
  #[inject] token_ext: Arc<dyn TokenExtractor>,
  #[inject] secret_provider: Arc<dyn JwtSecretProvider>,
) -> Result<Claims, Error> {
  let token_res = token_ext.extract(&rp).await;
  match token_res {
    Ok(jwt) => {
      let decoded = decode::<Claims>(
        &jwt,
        secret_provider.decoding_key().await,
        &Validation::new(algo_provider.algorithm().await),
      ).map_err(|_| Error::JWTTokenError)?;

      if !role.is_authorized(&decoded.claims) {
        return Err(Error::NoPermissionError);
      }

      Ok(decoded.claims)
    }
    Err(e) => return Err(e),
  }
}
```