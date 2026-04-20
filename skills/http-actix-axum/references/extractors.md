# Custom Request Extractors

Use custom extractors when request processing is cross-cutting and repeated across handlers:
- auth claims,
- tenant/account context,
- idempotency metadata,
- correlation/request IDs.

Keep extractors transport-focused. Business rules still belong in domain use cases.

## actix-web: `FromRequest`

```rust
use actix_web::{dev::Payload, FromRequest, HttpRequest, web};
use std::future::{ready, Ready};

#[derive(Debug)]
pub struct TenantId(pub String);

impl FromRequest for TenantId {
    type Error = actix_web::Error;
    type Future = Ready<Result<Self, Self::Error>>;

    fn from_request(req: &HttpRequest, _: &mut Payload) -> Self::Future {
        let value = req
            .headers()
            .get("x-tenant-id")
            .and_then(|h| h.to_str().ok())
            .map(str::to_owned);

        match value {
            Some(v) if !v.is_empty() => ready(Ok(TenantId(v))),
            _ => ready(Err(actix_web::error::ErrorUnauthorized("missing tenant header"))),
        }
    }
}
```

Usage:

```rust
pub async fn list_orders(
    tenant: TenantId,
    state: web::Data<AppState>,
) -> Result<impl Responder, ApiError> {
    // tenant.0 available here
    # todo!()
}
```

## axum: `FromRequestParts`

```rust
use axum::{
    async_trait,
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
};

#[derive(Debug)]
pub struct TenantId(pub String);

#[async_trait]
impl<S> FromRequestParts<S> for TenantId
where
    S: Send + Sync,
{
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(parts: &mut Parts, _: &S) -> Result<Self, Self::Rejection> {
        let value = parts
            .headers
            .get("x-tenant-id")
            .and_then(|h| h.to_str().ok())
            .map(str::to_owned);

        match value {
            Some(v) if !v.is_empty() => Ok(TenantId(v)),
            _ => Err((StatusCode::UNAUTHORIZED, "missing tenant header")),
        }
    }
}
```

## Best practices

- Keep rejection messages safe; avoid leaking internal parsing details.
- Convert extractor failures to Problem Details where your API standard requires it.
- Avoid DB/network I/O in extractors unless it is unavoidable.
- Keep extractors deterministic and fast.
- Prefer generic trait bounds in handlers and state; never bind handlers to concrete adapter types.