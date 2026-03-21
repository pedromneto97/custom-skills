# SeaORM v2 — Cargo Setup

## Open-source (Postgres / MySQL / SQLite)
```toml
[dependencies.sea-orm]
version = "2.0.0-rc"
features = [
  "sqlx-postgres",          # or sqlx-mysql, sqlx-sqlite
  "runtime-tokio-rustls",   # or runtime-async-std-rustls / *-native-tls
  "macros",
  "with-uuid",
  "with-chrono",            # or with-time
]

[dependencies.sea-orm-migration]
version = "2.0.0-rc"
features = ["runtime-tokio-rustls", "sqlx-postgres"]
```

## SeaORM-X (MSSQL — commercial, via Git SSH)
```toml
[dependencies.sea-orm]
git = "ssh://git@github.com/SeaQL/sea-orm-x.git"
features = ["runtime-tokio-rustls", "sqlz-mssql", "macros", "with-uuid", "with-chrono"]

[dependencies.sea-orm-migration]
git = "ssh://git@github.com/SeaQL/sea-orm-x.git"
```

## Feature flag reference
| Flag | Purpose |
|------|---------|
| `sqlx-postgres` / `sqlx-mysql` / `sqlx-sqlite` | Open-source DB backends |
| `sqlz-mssql` | MSSQL backend (SeaORM-X only) |
| `runtime-tokio-rustls` | Tokio + rustls TLS (recommended) |
| `runtime-tokio-native-tls` | Tokio + native-tls |
| `runtime-async-std-rustls` | async-std + rustls |
| `macros` | Required for entity derives |
| `with-chrono` | `chrono` date/time types |
| `with-time` | `time` crate types |
| `with-uuid` | `uuid::Uuid` |
| `with-json` | `serde_json::Value` |
| `with-rust_decimal` | `rust_decimal::Decimal` |
| `mock` | Mock DB for unit tests |
| `debug-print` | Print every SQL to logger |
| `schema-sync` | Entity-first schema creation |

## Workspace layout
```toml
# root Cargo.toml
[workspace]
members = [".", "entity", "migration"]

[dependencies]
entity    = { path = "entity" }
migration = { path = "migration" }
```
