# Response Compression

---

## Algorithm Priority

Prefer **Brotli → gzip → deflate** (auto-negotiated from the `Accept-Encoding` request header).
Both frameworks handle negotiation automatically — no manual detection needed.

## What to Compress

| Compress | Skip |
|----------|------|
| JSON / XML (`application/json`, `application/xml`) | Images (`image/*`) |
| HTML / CSS / JS | Video (`video/*`) |
| Plain text (`text/*`) | Audio (`audio/*`) |
| SVG (`image/svg+xml`) | `application/zip`, `application/gzip` |
| | `application/pdf` |

Also skip responses smaller than **1 KB** — overhead exceeds the saving.

---

## actix-web — Built-in `Compress` Middleware

No extra crate required; shipped with actix-web.

### Enable (all encodings, auto-negotiated)

```toml
# inbound/Cargo.toml
actix-web = { version = "4", features = ["compress-brotli", "compress-gzip", "compress-zstd"] }
```

```rust
use actix_web::middleware::Compress;

App::new()
    .wrap(Compress::default())   // Brotli → gzip → deflate, chosen by Accept-Encoding
    // ...
```

### Prefer a specific encoding

```rust
use actix_web::{middleware::Compress, http::header::ContentEncoding};

App::new()
    .wrap(Compress::new(ContentEncoding::Br))   // Brotli only; falls back if client doesn't support it
    // ...
```

### Exclude endpoints from compression

Compression is applied globally; to skip specific routes, wrap them with
`actix_web::middleware::DefaultHeaders` or handle it via a custom condition. For most APIs,
global compression on all JSON routes is correct.

---

## axum — `tower-http` CompressionLayer

```toml
# inbound/Cargo.toml
tower-http = { version = "0.5", features = ["compression-full"] }
# Or pick individual algorithms:
# features = ["compression-br", "compression-gzip", "compression-deflate", "compression-zstd"]
```

### Basic (all content, auto-negotiated)

```rust
use tower_http::compression::CompressionLayer;

Router::new()
    // routes...
    .layer(CompressionLayer::new())
```

### With size threshold and content-type exclusions

```rust
use tower_http::compression::{
    CompressionLayer,
    predicate::{NotForContentType, SizeAbove},
};

let compression = CompressionLayer::new().compress_when(
    SizeAbove::new(1024)                                        // skip < 1 KB
        .and(NotForContentType::const_new("image/jpeg"))
        .and(NotForContentType::const_new("image/png"))
        .and(NotForContentType::const_new("image/gif"))
        .and(NotForContentType::const_new("image/webp"))
        .and(NotForContentType::const_new("video/"))
        .and(NotForContentType::const_new("audio/"))
        .and(NotForContentType::const_new("application/zip"))
        .and(NotForContentType::const_new("application/pdf")),
);

Router::new()
    // routes...
    .layer(compression)
```

> `NotForContentType::const_new` matches as a prefix, so `"video/"` covers all video subtypes.

### Decompressing request bodies (optional)

If clients send compressed request bodies (rare for REST APIs):

```toml
tower-http = { version = "0.5", features = ["decompression-full"] }
```

```rust
use tower_http::decompression::RequestDecompressionLayer;

Router::new()
    // routes...
    .layer(RequestDecompressionLayer::new())
```

---

## Middleware Order

**actix-web** — register `Compress` before `SecurityHeaders` (actix-web wraps in reverse order,
so the outermost wrapper is the last `.wrap()` call):

```rust
App::new()
    .wrap(SecurityHeaders)   // applied last (outermost)
    .wrap(Compress::default()) // applied first (innermost)
    .wrap(cors)
```

**axum** — `.layer()` calls are applied bottom-up; place `CompressionLayer` after route layers,
before CORS:

```rust
Router::new()
    .nest("/api/v1", api_routes())
    .layer(compression)        // innermost — compresses the response body
    .layer(cors)               // outermost — adds CORS headers after compression
```
