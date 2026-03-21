---
name: http-actix-axum
description: >
  HTTP best practices for actix-web 4 and axum 0.7+ Rust backends.
  Use when: naming REST resources, choosing HTTP status codes, implementing RFC 9457 Problem Details
  error responses, configuring OWASP security headers, setting up CORS, enabling response
  compression, versioning APIs, or structuring the HTTP layer. Covers both actix-web and axum.
argument-hint: 'Specify the framework (actix-web or axum) and the topic, e.g. "CORS for axum" or "error handling with Problem Details for actix-web"'
---

# HTTP Best Practices — actix-web / axum

## 1. Resource Naming

| Rule | Good | Bad |
|------|------|-----|
| Plural nouns | `/orders`, `/users` | `/order`, `/getOrders` |
| Lowercase + hyphens | `/order-items` | `/orderItems`, `/Order_Items` |
| Hierarchical nesting | `/orders/{id}/items` | `/order-items?orderId={id}` |
| No verbs in path | `POST /orders` | `POST /createOrder` |
| Filter / sort in query | `/orders?status=pending&sort=created_at` | `/pending-orders` |

Max nesting depth: **2 levels** (`/resource/{id}/sub-resource`). Avoid deeper hierarchies.

---

## 2. API Versioning

Prefix at the router level. Handlers are version-agnostic.

**actix-web**
```rust
// inbound/src/http/router.rs
pub fn configure(cfg: &mut web::ServiceConfig) {
    cfg.service(
        web::scope("/api/v1")
            .service(web::scope("/orders")
                .route("",      web::get().to(list))
                .route("",      web::post().to(create))
                .route("/{id}", web::get().to(get_one))
                .route("/{id}", web::put().to(update))
                .route("/{id}", web::delete().to(delete)),
            ),
    );
}
```

**axum**
```rust
// inbound/src/http/router.rs
pub fn build_router() -> Router {
    Router::new()
        .nest("/api/v1", Router::new()
            .nest("/orders", Router::new()
                .route("/",    get(list).post(create))
                .route("/:id", get(get_one).put(update).delete(delete)),
            ),
        )
}
```

---

## 3. HTTP Status Codes

| Operation | Method | Success | Error cases |
|-----------|--------|---------|-------------|
| Fetch one | GET | 200 | 404 if not found |
| Fetch list | GET | 200 | Empty list → 200 `[]`, never 404 |
| Create | POST | 201 + `Location` header | 400, 422 |
| Full replace | PUT | 200 | 404, 422 |
| Partial update | PATCH | 200 | 404, 422 |
| Delete | DELETE | 204 No Content | 404 |
| Async action | POST | 202 Accepted | — |
| Bad input | — | 400 Bad Request | — |
| Unauthenticated | — | 401 Unauthorized | — |
| Forbidden | — | 403 Forbidden | — |
| Conflict (duplicate) | — | 409 Conflict | — |
| Business rule violated | — | 422 Unprocessable Entity | — |
| Server fault | — | 500 Internal Server Error | Never leak stack traces |

```rust
// 201 + Location (actix-web)
HttpResponse::Created()
    .insert_header(("Location", format!("/api/v1/orders/{}", order.id)))
    .json(OrderResponse::from(order))

// 201 + Location (axum)
(StatusCode::CREATED, [("Location", format!("/api/v1/orders/{}", order.id))], Json(body))
```

---

## 4. Error Responses — Problem Details (RFC 9457)

→ Read [`references/problem-details.md`](./references/problem-details.md) for the full struct,
`ResponseError` / `IntoResponse` impl, `From<DomainError>`, and validation error mapping.

Quick rules:
- `Content-Type: application/problem+json`
- `type` is a URI; use `"about:blank"` when no dedicated error page exists
- Never expose stack traces, internal IDs, or DB details in `detail`

---

## 5. Security Headers (OWASP)

→ Read [`references/security-headers.md`](./references/security-headers.md) for middleware
implementation (actix-web `Transform` + axum `tower-http` layer).

Mandatory headers on every response:

| Header | Value |
|--------|-------|
| `X-Content-Type-Options` | `nosniff` |
| `X-Frame-Options` | `DENY` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Content-Security-Policy` | `default-src 'self'` *(tune per app)* |
| `Permissions-Policy` | `geolocation=(), microphone=(), camera=()` |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` *(HTTPS only)* |

Remove `Server` and `X-Powered-By` response headers.

---

## 6. CORS

→ Read [`references/cors.md`](./references/cors.md) for full configuration.

Decision guide:

| Scenario | Strategy |
|----------|----------|
| Public API, no cookies | `allow_any_origin()` |
| Cookie-auth / credentialed | Explicit origin allowlist + `allow_credentials(true)` |

> **Never** combine `allow_any_origin()` with `allow_credentials(true)` — browsers reject it.

---

## 7. Response Compression

→ Read [`references/compression.md`](./references/compression.md) for middleware setup.

- Priority order: **Brotli → gzip → deflate** (auto-negotiated from `Accept-Encoding`)
- Skip already-compressed content: images (jpeg/png/gif/webp), video, `application/zip`, `application/pdf`
- Skip small responses: < 1 KB gains nothing
