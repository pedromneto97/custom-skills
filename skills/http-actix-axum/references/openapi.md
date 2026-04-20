# OpenAPI / Swagger Wiring

This guide covers a pragmatic OpenAPI setup for Rust HTTP adapters while keeping hexagonal boundaries intact.

## Principles

- OpenAPI annotations belong in inbound transport code (handlers, payloads, responses).
- Domain models are mapped to transport DTOs; avoid exposing internal domain types directly.
- API docs route exposure can be environment-gated (debug/non-production only).

## actix-web + utoipa

```toml
# inbound/Cargo.toml
utoipa = "5"
utoipa-swagger-ui = { version = "8", features = ["actix-web"] }
```

```rust
use utoipa::OpenApi;

#[derive(OpenApi)]
#[openapi(
    paths(
        crate::http::orders::handlers::list_orders,
        crate::http::orders::handlers::create_order,
    ),
    components(
        schemas(
            crate::http::orders::payload::CreateOrderRequest,
            crate::http::orders::response::OrderResponse,
            crate::http::error::ProblemDetail,
        )
    ),
    tags(
        (name = "orders", description = "Order management")
    )
)]
pub struct ApiDoc;
```

Wire Swagger UI conditionally:

```rust
#[cfg(debug_assertions)]
app = app.service(
    utoipa_swagger_ui::SwaggerUi::new("/swagger-ui/{_:.*}")
        .url("/api-docs/openapi.json", ApiDoc::openapi()),
);
```

## Security scheme example (Bearer)

```rust
use utoipa::{Modify, openapi::security::{HttpAuthScheme, HttpBuilder, SecurityScheme}};

struct SecurityAddon;

impl Modify for SecurityAddon {
    fn modify(&self, openapi: &mut utoipa::openapi::OpenApi) {
        if let Some(components) = openapi.components.as_mut() {
            components.add_security_scheme(
                "bearer_auth",
                SecurityScheme::Http(
                    HttpBuilder::new()
                        .scheme(HttpAuthScheme::Bearer)
                        .bearer_format("JWT")
                        .build(),
                ),
            );
        }
    }
}
```

Then annotate protected routes with security metadata.

## Axum option

The same utoipa document can be served in axum with a docs route and static UI handler.
Keep docs router isolated from core business routes.

## Practical checklist

- Include `ProblemDetail` schema so error contracts are visible.
- Keep request/response DTOs annotated, not raw domain entities.
- Group routes by tags that mirror resource modules.
- Gate docs exposure by environment policy.
- Keep OpenAPI generation in inbound; never in domain/outbound.