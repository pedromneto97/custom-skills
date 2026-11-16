# SeaORM 2.x — Database Connection

## Simple connection
```rust
use sea_orm::{Database, DatabaseConnection};

let db: DatabaseConnection = Database::connect("postgres://user:pass@localhost/dbname").await?;
// MySQL / MariaDB: "mysql://user:pass@localhost/dbname"
// SQLite file:     "sqlite://./db.sqlite?mode=rwc"   (rwc = create if missing)
// SQLite read-only:"sqlite://./db.sqlite?mode=ro"
// SQLite memory:   "sqlite::memory:"
// MSSQL (SeaORM-X):"mssql://user:pass@localhost/dbname"
```

> v2: `DatabaseConnection` is a **struct** wrapping an inner `DatabaseConnectionType` enum
> (it was an enum in v1). Don't `match` on its variants; use `db.get_database_backend()`.

`DatabaseConnection` is cheap to clone (it holds a pool) — share it via `Clone` / app state.

## Postgres schema (search_path)
```rust
// Via URL
Database::connect("postgres://user:pass@localhost/db?options=--search_path=my_schema").await?;

// Via ConnectOptions (preferred)
let mut opt = ConnectOptions::new("postgres://user:pass@localhost/db");
opt.set_schema_search_path("my_schema");
```

## MSSQL connection string options (SeaORM-X)
```
mssql://user:pass@host/database                           # default schema dbo
mssql://user:pass@host/database?currentSchema=my_schema
mssql://user:pass@host:1433/database?currentSchema=my_schema&trustCertificate=true
```
> `currentSchema` causes SeaORM-X to auto-prefix every query with `[my_schema].[table]`.

## ConnectOptions (pool + logging)
```rust
use sea_orm::{ConnectOptions, Database};
use std::time::Duration;

let mut opt = ConnectOptions::new("postgres://user:pass@localhost/dbname");
opt.max_connections(100)
    .min_connections(5)
    .connect_timeout(Duration::from_secs(8))
    .acquire_timeout(Duration::from_secs(8))
    .idle_timeout(Duration::from_secs(8))
    .max_lifetime(Duration::from_secs(8))
    .sqlx_logging(true)
    .sqlx_logging_level(log::LevelFilter::Info);

let db = Database::connect(opt).await?;
```

### Practical defaults

For most APIs, start with conservative values and tune from metrics:

```rust
opt.max_connections(50)
    .min_connections(1)
    .connect_timeout(Duration::from_secs(8))
    .acquire_timeout(Duration::from_secs(8))
    .idle_timeout(Duration::from_secs(300))
    .max_lifetime(Duration::from_secs(1800));
```

Rules of thumb:
- Increase `max_connections` for read-heavy workloads after confirming DB server capacity.
- Keep `min_connections` low (1-5) unless cold-start latency is a problem.
- Prefer non-trivial `idle_timeout`/`max_lifetime` in long-running services to recycle stale connections.
- Enable SQL logging in development; reduce/noise-gate in production.
- SQLite in-memory: each pooled connection is a separate DB — use `max_connections(1)` in tests.

## Ping / close
```rust
db.ping().await?;
db.close().await?;      // also automatic on drop
```
