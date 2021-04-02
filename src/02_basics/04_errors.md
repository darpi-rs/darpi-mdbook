# errors

Any error implementing the `ResponderError` trait can be used in a handler return type.
In this case, we are using the default implementation of the trait.

```rust,ignore
use darpi::response::ResponderError;

#[derive(Display, Debug)]
pub enum Error {
    #[display(fmt = "wrong credentials")]
    WrongCredentialsError,
    #[display(fmt = "jwt token not valid")]
    JWTTokenError,
    #[display(fmt = "jwt token creation error")]
    JWTTokenCreationError,
    #[display(fmt = "no auth header")]
    NoAuthHeaderError,
    #[display(fmt = "invalid auth header")]
    InvalidAuthHeaderError,
    #[display(fmt = "no permission")]
    NoPermissionError,
}

impl ResponderError for Error {}

#[derive(Deserialize, Serialize, Debug)]
pub struct Login {
    email: String,
    password: String,
}

#[handler({
    container: Container
})]
async fn login(
    #[body] data: Json<Login>,
    #[inject] jwt_tok_creator: Arc<dyn JwtTokenCreator>,
) -> Result<Token, Error> {
    //verify user data
    let admin = Role::Admin; // hardcoded just for the example
    let uid = "uid"; // hardcoded just for the example
    let tok = jwt_tok_creator.create(uid, &admin).await?;
    Ok(tok)
}
```