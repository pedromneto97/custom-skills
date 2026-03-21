# Folder Structure Reference — Rust Hexagonal Architecture

## Full Annotated Tree (Workspace — Layout A)

```
my-app/
├── Cargo.toml                              # workspace root
│
├── domain/                                 # lib — ZERO infrastructure deps
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs                          # pub mod model; pub mod error; pub mod ports; pub mod use_cases;
│       ├── error.rs                        # DomainError (thiserror)
│       ├── model/
│       │   ├── mod.rs
│       │   ├── order.rs                    # Order, OrderId, OrderStatus
│       │   └── customer.rs
│       ├── ports/
│       │   ├── mod.rs
│       │   ├── inbound.rs                  # trait OrderUseCase
│       │   └── outbound.rs                 # trait OrderRepository
│       └── use_cases/
│           ├── mod.rs
│           └── order_service.rs            # OrderService<R: OrderRepository>
│
├── inbound/                                # lib — HTTP/gRPC/CLI adapters
│   ├── Cargo.toml                          # deps: domain, actix-web (or axum), serde
│   └── src/
│       ├── lib.rs                          # pub mod http;
│       └── http/
│           ├── mod.rs
│           ├── router.rs                   # fn order_routes<U: OrderUseCase + 'static>
│           ├── orders.rs                   # handler fns generic over U: OrderUseCase
│           └── dto.rs                      # request/response structs (serde) — never domain types
│
├── outbound/                               # lib — DB/cache adapters
│   ├── Cargo.toml                          # deps: domain, sea-orm
│   └── src/
│       ├── lib.rs                          # pub mod db;
│       └── db/
│           ├── mod.rs
│           ├── order_repository.rs         # SeaOrmOrderRepository implements OrderRepository
│           ├── entities/
│           │   └── order.rs               # sea_orm Entity, Model, ActiveModel
│           └── mappers.rs                  # entity Model ↔ domain::Order
│
└── app/                                    # bin — sole composition root
    ├── Cargo.toml                          # deps: all crates above
    └── src/
        └── main.rs                         # wire concrete types → serve
```

---

## Workspace `Cargo.toml`

Only list dependencies that are **shared across multiple crates**. Layer-specific deps belong in the crate's own `Cargo.toml`.

```toml
[workspace]
members  = ["domain", "inbound", "outbound", "app"]
resolver = "2"

[workspace.dependencies]
domain   = { path = "domain" }
inbound  = { path = "inbound" }
outbound = { path = "outbound" }

# shared across multiple crates
tokio     = { version = "1", features = ["full"] }
uuid      = { version = "1", features = ["v4"] }
thiserror = "2"
serde     = { version = "1", features = ["derive"] }
anyhow    = "1"

# actix-web  ← NOT here: only used in inbound
# sea-orm    ← NOT here: only used in outbound
```

---

## Per-Crate `Cargo.toml` Examples

### `domain/Cargo.toml`

```toml
[package]
name    = "domain"
version = "0.1.0"
edition = "2021"

[dependencies]
thiserror.workspace = true
uuid.workspace      = true
# tokio only needed in Layout A (use cases inside domain)
tokio = { workspace = true, optional = true }
# async-trait ONLY if you need dyn Trait object safety

[dev-dependencies]
mockall = "0.12"
tokio   = { workspace = true, features = ["macros", "rt"] }
```

### `inbound/Cargo.toml`

```toml
[package]
name    = "inbound"
version = "0.1.0"
edition = "2021"

[dependencies]
domain.workspace = true
serde.workspace  = true
uuid.workspace   = true
tokio.workspace  = true
# layer-specific — not in workspace
actix-web = "4"       # or: axum = "0.8"
```

### `outbound/Cargo.toml`

```toml
[package]
name    = "outbound"
version = "0.1.0"
edition = "2021"

[dependencies]
domain.workspace = true
uuid.workspace   = true
tokio.workspace  = true
# layer-specific — not in workspace
sea-orm = { version = "2", features = ["sqlx-postgres", "runtime-tokio-rustls", "macros"] }
```

---

## Dependency Matrix

| Crate | May depend on |
|-------|--------------|
| `domain` | nothing (std + thiserror + uuid) |
| `inbound` | `domain` only |
| `outbound` | `domain` only |
| `app` | all of the above |

**Forbidden edges (never list in `Cargo.toml`):**
- `inbound` → `outbound`
- `outbound` → `inbound`
- `domain` → any adapter or app crate

---

## Layout B variant

Add an `application` crate between `domain` and `app`:

```toml
# workspace Cargo.toml
members = ["domain", "application", "inbound", "outbound", "app"]
```

- `domain` has no `tokio` dependency
- `application` depends on `domain`; contains `use_cases/`
- `app` depends on `domain`, `application`, `inbound`, `outbound`
