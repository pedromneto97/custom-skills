# SeaORM v2 — Database Connection

## Simple connection
```rust
use sea_orm::{Database, DatabaseConnection};

let db: DatabaseConnection = Database::connect("postgres://user:pass@localhost/dbname").await?;
// MySQL:  "mysql://user:pass@localhost/dbname"
// SQLite: "sqlite://./db.sqlite?mode=rwc"
// MSSQL:  "mssql://user:pass@localhost/dbname"
```

## MSSQL connection string options
```
mssql://user:pass@host/database
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

## Ping / close
```rust
db.ping().await?;
db.close().await?;      // also automatic on drop
```
