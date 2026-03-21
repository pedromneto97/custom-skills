# CORS Configuration

---

## Decision Guide

| Scenario | Strategy |
|----------|----------|
| Public API — no cookies, no `Authorization` header | `allow_any_origin()` |
| SPA / mobile with cookie-auth or `Authorization` | Explicit origin allowlist + `allow_credentials(true)` |
| Server-to-server (no browser) | No CORS middleware needed |

> **Critical:** Never combine `allow_any_origin()` with `allow_credentials(true)`.
> The CORS spec forbids it; browsers will refuse the response.

---

## actix-web — `actix-cors`

```toml
# inbound/Cargo.toml
actix-web  = "4"
actix-cors = "0.7"
```

### Permissive (public APIs)

```rust
use actix_cors::Cors;

let cors = Cors::default()
    .allow_any_origin()
    .allow_any_method()
    .allow_any_header()
    .max_age(3600);

App::new()
    .wrap(cors)
    // ...
```

### Restricted (credentialed / cookie-auth)

```rust
use actix_cors::Cors;
use actix_web::http::header;

let cors = Cors::default()
    .allowed_origins(&[
        "https://app.example.com",
        "https://admin.example.com",
    ])
    .allowed_methods(vec!["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"])
    .allowed_headers(vec![
        header::AUTHORIZATION,
        header::CONTENT_TYPE,
        header::ACCEPT,
    ])
    .expose_headers(vec![header::LOCATION])   // expose custom response headers if needed
    .supports_credentials()                    // allows cookies / Authorization header
    .max_age(3600);

App::new()
    .wrap(cors)
    // ...
```

> Wrap CORS **before** other middleware — actix-web middleware executes in reverse registration order.

---

## axum — `tower-http` CorsLayer

```toml
# inbound/Cargo.toml
axum       = "0.7"
tower-http = { version = "0.5", features = ["cors"] }
```

### Permissive (public APIs)

```rust
use std::time::Duration;
use tower_http::cors::{Any, CorsLayer};

let cors = CorsLayer::new()
    .allow_origin(Any)
    .allow_methods(Any)
    .allow_headers(Any)
    .max_age(Duration::from_secs(3600));

Router::new()
    // routes...
    .layer(cors)
```

### Restricted (credentialed / cookie-auth)

```rust
use std::time::Duration;
use axum::http::{HeaderName, HeaderValue, Method};
use tower_http::cors::CorsLayer;

let cors = CorsLayer::new()
    .allow_origin([
        "https://app.example.com".parse::<HeaderValue>().unwrap(),
        "https://admin.example.com".parse::<HeaderValue>().unwrap(),
    ])
    .allow_methods([
        Method::GET,
        Method::POST,
        Method::PUT,
        Method::PATCH,
        Method::DELETE,
        Method::OPTIONS,
    ])
    .allow_headers([
        HeaderName::from_static("authorization"),
        HeaderName::from_static("content-type"),
        HeaderName::from_static("accept"),
    ])
    .expose_headers([HeaderName::from_static("location")])
    .allow_credentials(true)
    .max_age(Duration::from_secs(3600));

Router::new()
    // routes...
    .layer(cors)
```

---

## Preflight Requests

Both `actix-cors` and `tower-http`'s `CorsLayer` handle `OPTIONS` preflight automatically.
You do **not** need to register explicit `OPTIONS` routes.

## Dynamic Origins (runtime allowlist)

When allowed origins are loaded from config rather than hardcoded:

**axum**
```rust
use tower_http::cors::AllowOrigin;

let allowed: Vec<HeaderValue> = config.cors_origins
    .iter()
    .map(|o| o.parse().expect("invalid origin in config"))
    .collect();

let cors = CorsLayer::new()
    .allow_origin(AllowOrigin::list(allowed))
    .allow_credentials(true)
    // ...
```
