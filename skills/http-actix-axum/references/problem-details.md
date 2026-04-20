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
use validator::{Validate, ValidationErrors};

// In the handler — validate before calling the use case:
body.validate().map_err(ApiError::from)?; // or body.validate()?
```

```rust
// Conversion — inbound/src/http/error.rs
use validator::ValidationErrors;

impl From<ValidationErrors> for ApiError {
    fn from(e: ValidationErrors) -> Self {
        ApiError(ProblemDetail {
            problem_type: "about:blank".into(),
            title: "Validation Error".into(),
            status: 400,
            detail: e.to_string(),
            instance: None,
        })
    }
}
```

### Domain-validation bridge

When the domain owns the validation rules, bridge them into the `validator` crate so the
same logic is reused at the HTTP boundary without duplicating rules:

```rust
// domain exposes a pure fn that returns Vec<String> errors:
// pub fn validate_password(pw: &str) -> Result<(), Vec<String>>

use validator::ValidationError;
use domain::use_cases::validate_password;

fn password_valid(val: &str) -> Result<(), ValidationError> {
    match validate_password(val) {
        Ok(_) => Ok(()),
        Err(errors) => {
            let mut e = ValidationError::new("password")
                .with_message("Password is too weak".into());
            e.add_param("errors".into(), &errors);
            Err(e)
        }
    }
}

#[derive(Deserialize, Validate)]
pub struct RegisterRequest {
    #[validate(custom(function = "password_valid"))]
    pub password: String,
}
```

> Trimming / normalising input (e.g. `name.trim().to_string()`) should happen in
> `From<RequestType> for UseCaseInput`, **not** in the validator function.

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

## Framework extractor errors → Problem Details

Validation and domain errors are only part of the picture. Parsing/extractor errors should also
be normalized to Problem Details with safe `detail` fields.

### actix-web: JSON and query extractor hooks

```rust
use actix_web::{error::{JsonPayloadError, QueryPayloadError}, HttpRequest};

pub fn json_payload_error_handler(err: JsonPayloadError, _req: &HttpRequest) -> actix_web::Error {
    let (status, title) = match err {
        JsonPayloadError::OverflowKnownLength { .. } | JsonPayloadError::Overflow { .. } => (413, "Payload Too Large"),
        JsonPayloadError::Serialize(_) => (500, "Internal Server Error"),
        _ => (400, "Bad Request"),
    };

    ApiError(ProblemDetail::blank(status, title, "Invalid JSON payload")).into()
}

pub fn query_payload_error_handler(err: QueryPayloadError, _req: &HttpRequest) -> actix_web::Error {
    let detail = match err {
        QueryPayloadError::Deserialize(_) => "Invalid query parameter",
        _ => "Bad request",
    };
    ApiError(ProblemDetail::blank(400, "Bad Request", detail)).into()
}
```

Register in app setup:

```rust
.app_data(JsonConfig::default().error_handler(json_payload_error_handler))
.app_data(QueryConfig::default().error_handler(query_payload_error_handler))
```

### axum: extractor rejection mapping

```rust
use axum::extract::rejection::{JsonRejection, QueryRejection};

fn map_json_rejection(err: JsonRejection) -> ApiError {
    match err {
        JsonRejection::JsonDataError(_) | JsonRejection::JsonSyntaxError(_) =>
            ApiError(ProblemDetail::blank(400, "Bad Request", "Invalid JSON payload")),
        JsonRejection::BytesRejection(_) =>
            ApiError(ProblemDetail::blank(413, "Payload Too Large", "Request body too large")),
        _ => ApiError(ProblemDetail::blank(400, "Bad Request", "Invalid request body")),
    }
}

fn map_query_rejection(_: QueryRejection) -> ApiError {
    ApiError(ProblemDetail::blank(400, "Bad Request", "Invalid query parameter"))
}
```

Return these mapped errors from handlers (or central middleware/layer) so clients always get
`application/problem+json` responses.

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
