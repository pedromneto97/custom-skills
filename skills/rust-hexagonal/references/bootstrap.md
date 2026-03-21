# Bootstrap / Composition Root Reference

`app/src/main.rs` is the **only** file that imports all concrete types. It constructs the full
dependency graph and starts the server. No other crate should know about more than one layer.

With static dispatch, all types are monomorphized at compile time — no `Arc<dyn Trait>` needed
unless you explicitly want runtime polymorphism.

The composition is split across three files in `app/src/`:

| File | Responsibility |
|------|---------------|
| `config.rs` | Read env vars into a typed config struct |
| `state.rs` | Build `web::Data<Service>` per bounded context from config |
| `main.rs` | Wire state into actix-web, bind, and serve |

---

## `app/src/config.rs`

```rust
pub struct AppConfig {
    pub database_url: String,
    pub host: String,
    pub port: u16,
}

impl AppConfig {
    pub fn from_env() -> anyhow::Result<Self> {
        Ok(Self {
            database_url: std::env::var("DATABASE_URL")?,
            host: std::env::var("HOST").unwrap_or_else(|_| "0.0.0.0".to_string()),
            port: std::env::var("PORT")
                .unwrap_or_else(|_| "8080".to_string())
                .parse()?,
        })
    }
}
```

---

## `app/src/state.rs`

Constructs `AppState<R>` (defined in `inbound`) by wiring outbound adapters into it.
`app` is the only crate that names the concrete repository type.

```rust
use inbound::state::AppState;
use outbound::db::SeaOrmOrderRepository;
use std::sync::Arc;
use super::config::AppConfig;

pub type AppStateImpl = AppState;

pub async fn build_state(cfg: &AppConfig) -> anyhow::Result<AppStateImpl> {
    let db = sea_orm::Database::connect(&cfg.database_url).await?;

    // Run pending migrations before accepting traffic
    migration::Migrator::up(&db, None).await?;

    Ok(AppState {
        repo: Arc::new(SeaOrmOrderRepository::new(db)),
    })
}
```

---

## `app/src/main.rs` (actix-web — preferred)

```rust
mod config;
mod state;

use actix_web::{web, App, HttpServer};
use config::AppConfig;
use outbound::db::SeaOrmOrderRepository;
use inbound::http::router::order_routes;

#[actix_web::main]
async fn main() -> anyhow::Result<()> {
    dotenvy::dotenv().ok();
    tracing_subscriber::fmt::init();

    let cfg        = AppConfig::from_env()?;
    let state      = state::build_state(&cfg).await?;
    let state_data = web::Data::new(state);

    HttpServer::new(move || {
        App::new()
            .app_data(state_data.clone())
            .configure(order_routes)
    })
    .bind((cfg.host.as_str(), cfg.port))?
    .run()
    .await?;

    Ok(())
}
```

Handlers in `inbound` remain generic and receive `web::Data<AppState<R>>`:
```rust
// inbound/src/http/orders.rs
pub async fn get_order<R: AppRepository>(
    path: web::Path<Uuid>,
    state: web::Data<AppState<R>>,   // ← extracted by actix-web
) -> impl Responder { ... }
```

---

## `app/src/main.rs` (axum — alternative)

```rust
mod config;
mod state;

use config::AppConfig;
use outbound::db::SeaOrmOrderRepository;
use inbound::http::router::order_router;
use inbound::state::AppState;
use std::sync::Arc;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    dotenvy::dotenv().ok();

    let cfg   = AppConfig::from_env()?;
    let state = Arc::new(state::build_state(&cfg).await?);

    let app      = order_router::<SeaOrmOrderRepository>(state);
    let listener = tokio::net::TcpListener::bind((cfg.host.as_str(), cfg.port)).await?;
    axum::serve(listener, app).await?;

    Ok(())
}
```

---

## `app/Cargo.toml`

```toml
[package]
name = "app"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "server"
path = "src/main.rs"

[dependencies]
domain.workspace    = true
inbound.workspace   = true
outbound.workspace  = true
migration.workspace = true
tokio.workspace     = true
anyhow.workspace    = true
# layer-specific — not in workspace
actix-web          = "4"          # or axum = "0.8"
sea-orm            = { version = "2", features = ["sqlx-postgres", "runtime-tokio-rustls"] }
dotenvy            = "0.15"
tracing-subscriber = "0.3"
```
