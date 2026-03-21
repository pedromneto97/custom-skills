# Problem Details — RFC 9457

All error responses use `Content-Type: application/problem+json`.

RFC fields: `type` (URI), `title` (short summary), `status` (HTTP code), `detail` (explanation),
`instance` (URI of specific occurrence — optional). Add domain-specific extension fields as needed.

---

## Core Struct (shared between frameworks)

```rust
// inbound/src/http/errors.rs
use serde::Serialize;

#[derive(Debug, Serialize)]
pub struct ProblemDetail {
    #[serde(rename = "type")]
    pub problem_type: String,
    pub title: String,
    pub status: u16,
    pub detail: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub instance: Option<String>,
}

impl ProblemDetail {
    pub fn new(status: u16, title: &str, detail: impl Into<String>) -> Self {
        Self {
            problem_type: format!(
                "https://example.com/errors/{}",
                title.to_lowercase().replace(' ', "-")
            ),
            title: title.into(),
            status,
            detail: detail.into(),
            instance: None,
        }
    }

    /// Use when no dedicated error documentation page exists.
    pub fn blank(status: u16, title: &str, detail: impl Into<String>) -> Self {
        Self {
            problem_type: "about:blank".into(),
            title: title.into(),
            status,
            detail: detail.into(),
            instance: None,
        }
    }
}
```

---

## actix-web

### `ApiError` wrapper + `ResponseError`

```rust
use actix_web::{HttpResponse, ResponseError};
use std::fmt;

#[derive(Debug)]
pub struct ApiError(pub ProblemDetail);

impl fmt::Display for ApiError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.0.title)
    }
}

impl ResponseError for ApiError {
    fn status_code(&self) -> actix_web::http::StatusCode {
        actix_web::http::StatusCode::from_u16(self.0.status)
            .unwrap_or(actix_web::http::StatusCode::INTERNAL_SERVER_ERROR)
    }

    fn error_response(&self) -> HttpResponse {
        HttpResponse::build(self.status_code())
            .content_type("application/problem+json")
            .json(&self.0)
    }
}
```

### Mapping domain errors

```rust
use domain::error::DomainError;

impl From<DomainError> for ApiError {
    fn from(e: DomainError) -> Self {
        ApiError(match e {
            DomainError::NotFound(id) =>
                ProblemDetail::new(404, "Not Found", format!("Resource {id} does not exist")),
            DomainError::InvalidTransition =>
                ProblemDetail::new(422, "Invalid Transition", "State transition not allowed"),
            DomainError::Conflict(msg) =>
                ProblemDetail::new(409, "Conflict", msg),
            DomainError::Infrastructure(_) =>
                ProblemDetail::new(500, "Internal Server Error", "An unexpected error occurred"),
        })
    }
}
```

### Handler pattern

```rust
// Handlers return Result<impl Responder, ApiError>.
// The `?` operator calls From<DomainError> automatically.
pub async fn get_order<R: AppRepository>(
    path: web::Path<Uuid>,
    state: web::Data<AppState<R>>,
) -> Result<impl Responder, ApiError> {
    let order = orders::get_order(&state.repo, *path).await?;
    Ok(HttpResponse::Ok().json(OrderResponse::from(order)))
}
```

### Validation error (400)

```rust
use validator::Validate;

body.validate().map_err(|e| {
    ApiError(ProblemDetail {
        problem_type: "https://example.com/errors/validation-error".into(),
        title: "Validation Error".into(),
        status: 400,
        detail: e.to_string(),
        instance: None,
    })
})?;
```

---

## axum

### `ApiError` wrapper + `IntoResponse`

```rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};

pub struct ApiError(pub ProblemDetail);

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let status = StatusCode::from_u16(self.0.status)
            .unwrap_or(StatusCode::INTERNAL_SERVER_ERROR);
        (
            status,
            [("content-type", "application/problem+json")],
            Json(self.0),
        )
            .into_response()
    }
}
```

### Mapping domain errors

```rust
use domain::error::DomainError;

impl From<DomainError> for ApiError {
    fn from(e: DomainError) -> Self {
        ApiError(match e {
            DomainError::NotFound(id) =>
                ProblemDetail::new(404, "Not Found", format!("Resource {id} does not exist")),
            DomainError::InvalidTransition =>
                ProblemDetail::new(422, "Invalid Transition", "State transition not allowed"),
            DomainError::Conflict(msg) =>
                ProblemDetail::new(409, "Conflict", msg),
            DomainError::Infrastructure(_) =>
                ProblemDetail::new(500, "Internal Server Error", "An unexpected error occurred"),
        })
    }
}
```

### Handler pattern

```rust
// Handlers return Result<impl IntoResponse, ApiError>.
pub async fn get_order(
    Path(id): Path<Uuid>,
    State(state): State<AppState>,
) -> Result<impl IntoResponse, ApiError> {
    let order = orders::get_order(&state.repo, id).await?;
    Ok(Json(OrderResponse::from(order)))
}
```

### Validation error (400)

```rust
body.validate().map_err(|e| {
    ApiError(ProblemDetail {
        problem_type: "https://example.com/errors/validation-error".into(),
        title: "Validation Error".into(),
        status: 400,
        detail: e.to_string(),
        instance: None,
    })
})?;
```

---

## Cargo.toml

```toml
# shared
serde      = { version = "1", features = ["derive"] }
serde_json = "1"

# actix-web
actix-web = "4"
validator = { version = "0.18", features = ["derive"] }

# axum
axum = "0.7"
```
