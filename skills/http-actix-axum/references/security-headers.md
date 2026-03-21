# Security Headers (OWASP)

Reference: https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html

---

## Headers Reference

| Header | Recommended value | Purpose |
|--------|-------------------|---------|
| `X-Content-Type-Options` | `nosniff` | Prevent MIME-type sniffing |
| `X-Frame-Options` | `DENY` | Prevent clickjacking |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Limit referrer leakage |
| `Content-Security-Policy` | `default-src 'self'` *(tune per app)* | XSS / injection mitigation |
| `Permissions-Policy` | `geolocation=(), microphone=(), camera=()` | Restrict browser features |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | Force HTTPS *(omit on HTTP)* |
| `Cache-Control` | `no-store` | Sensitive endpoints only |

**Remove** `Server` and `X-Powered-By` — they reveal implementation details.

> `X-XSS-Protection` is **deprecated**; do not set it. Modern browsers ignore it and it can
> introduce vulnerabilities in older ones. Rely on CSP instead.

---

## actix-web — Custom Middleware

```rust
// inbound/src/http/middleware/security_headers.rs
use actix_web::{
    dev::{Service, ServiceRequest, ServiceResponse, Transform},
    Error,
};
use futures_util::future::{ok, LocalBoxFuture, Ready};
use std::task::{Context, Poll};

pub struct SecurityHeaders;

impl<S, B> Transform<S, ServiceRequest> for SecurityHeaders
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = SecurityHeadersMiddleware<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        ok(SecurityHeadersMiddleware { service })
    }
}

pub struct SecurityHeadersMiddleware<S> {
    service: S,
}

impl<S, B> Service<ServiceRequest> for SecurityHeadersMiddleware<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;

    fn poll_ready(&self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.service.poll_ready(cx)
    }

    fn call(&self, req: ServiceRequest) -> Self::Future {
        let fut = self.service.call(req);
        Box::pin(async move {
            let mut res = fut.await?;
            let h = res.headers_mut();

            use actix_web::http::header::{HeaderName, HeaderValue};
            macro_rules! set {
                ($name:expr, $val:expr) => {
                    h.insert(
                        HeaderName::from_static($name),
                        HeaderValue::from_static($val),
                    );
                };
            }

            set!("x-content-type-options", "nosniff");
            set!("x-frame-options", "DENY");
            set!("referrer-policy", "strict-origin-when-cross-origin");
            set!("content-security-policy", "default-src 'self'");
            set!("permissions-policy", "geolocation=(), microphone=(), camera=()");
            // Uncomment for HTTPS deployments:
            // set!("strict-transport-security", "max-age=31536000; includeSubDomains");

            h.remove("server");

            Ok(res)
        })
    }
}
```

### Wire in `HttpServer`

```rust
// app/src/main.rs  (or inbound/src/lib.rs)
App::new()
    .wrap(SecurityHeaders)
    // ... other middleware and routes
```

### Cargo.toml (actix-web)

```toml
actix-web      = "4"
futures-util   = "0.3"
```

---

## axum — `tower-http` SetResponseHeaderLayer

```rust
// inbound/src/http/middleware/security_headers.rs
use axum::http::{HeaderName, HeaderValue};
use tower::ServiceBuilder;
use tower_http::set_header::SetResponseHeaderLayer;

pub fn security_headers_layer() -> impl tower::Layer<
    tower::util::BoxCloneService<
        axum::http::Request<axum::body::Body>,
        axum::http::Response<axum::body::Body>,
        std::convert::Infallible,
    >,
    Service = impl tower::Service<
        axum::http::Request<axum::body::Body>,
        Response = axum::http::Response<axum::body::Body>,
        Error = std::convert::Infallible,
    >,
> + Clone {
    ServiceBuilder::new()
        .layer(header("x-content-type-options", "nosniff"))
        .layer(header("x-frame-options", "DENY"))
        .layer(header("referrer-policy", "strict-origin-when-cross-origin"))
        .layer(header("content-security-policy", "default-src 'self'"))
        .layer(header("permissions-policy", "geolocation=(), microphone=(), camera=()"))
        // Uncomment for HTTPS:
        // .layer(header("strict-transport-security", "max-age=31536000; includeSubDomains"))
}

fn header(name: &'static str, value: &'static str) -> SetResponseHeaderLayer<HeaderValue> {
    SetResponseHeaderLayer::overriding(
        HeaderName::from_static(name),
        HeaderValue::from_static(value),
    )
}
```

### Wire in Router

```rust
// inbound/src/http/router.rs
use tower_http::set_header::SetResponseHeaderLayer;
use axum::http::{HeaderName, HeaderValue};

pub fn build_router() -> Router {
    Router::new()
        .nest("/api/v1", api_routes())
        .layer(SetResponseHeaderLayer::overriding(
            HeaderName::from_static("x-content-type-options"),
            HeaderValue::from_static("nosniff"),
        ))
        .layer(SetResponseHeaderLayer::overriding(
            HeaderName::from_static("x-frame-options"),
            HeaderValue::from_static("DENY"),
        ))
        .layer(SetResponseHeaderLayer::overriding(
            HeaderName::from_static("referrer-policy"),
            HeaderValue::from_static("strict-origin-when-cross-origin"),
        ))
        .layer(SetResponseHeaderLayer::overriding(
            HeaderName::from_static("content-security-policy"),
            HeaderValue::from_static("default-src 'self'"),
        ))
        .layer(SetResponseHeaderLayer::overriding(
            HeaderName::from_static("permissions-policy"),
            HeaderValue::from_static("geolocation=(), microphone=(), camera=()"),
        ))
        // Uncomment for HTTPS:
        // .layer(SetResponseHeaderLayer::overriding(
        //     HeaderName::from_static("strict-transport-security"),
        //     HeaderValue::from_static("max-age=31536000; includeSubDomains"),
        // ))
}
```

### Cargo.toml (axum)

```toml
axum        = "0.7"
tower-http  = { version = "0.5", features = ["set-response-header"] }
tower       = "0.4"
```

---

## Content Security Policy — Tuning Guide

The `default-src 'self'` baseline blocks everything external. Common additions:

| Need | Directive to add |
|------|-----------------|
| Load fonts from Google | `font-src 'self' https://fonts.gstatic.com` |
| Load scripts from CDN | `script-src 'self' https://cdn.example.com` |
| Inline styles (avoid if possible) | `style-src 'self' 'unsafe-inline'` |
| Report violations | `report-uri /csp-report` *(or `report-to`)* |

Test your final CSP at https://csp-evaluator.withgoogle.com before shipping.
