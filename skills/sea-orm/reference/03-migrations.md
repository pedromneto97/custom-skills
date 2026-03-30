# SeaORM v2 — Migrations

## Install CLI
```shell
# Open-source
cargo install sea-orm-cli@^2.0.0-rc

# SeaORM-X (from local path after cloning sea-orm-x)
cargo install --path "<SEA_ORM_X_ROOT>/sea-orm-x/sea-orm-cli"
```

## File layout
```
migration/
├── Cargo.toml
└── src/
    ├── lib.rs           ← MigratorTrait
    ├── main.rs          ← CLI entry
    └── m20220101_000001_create_post.rs
```

## `migration/Cargo.toml`
```toml
[dependencies.sea-orm-migration]
version = "2.0.0-rc"
features = ["runtime-tokio-rustls", "sqlx-postgres"]   # match main crate
# async-std is NOT needed — sea-orm-migration supports tokio natively
```

> **MySQL note:** change the feature flag to `"sqlx-mysql"` for MySQL/MariaDB.
> Use `"sqlx-sqlite"` for SQLite.

## `migration/src/lib.rs`
```rust
pub use sea_orm_migration::*;

mod m20220101_000001_create_post;

pub struct Migrator;

#[async_trait]
impl MigratorTrait for Migrator {
    fn migrations() -> Vec<Box<dyn MigrationTrait>> {
        vec![
            Box::new(m20220101_000001_create_post::Migration),
        ]
    }
}
```

## Migration file template
```rust
// migration/src/m20220101_000001_create_post.rs
use sea_orm_migration::{prelude::*, schema::*};

#[derive(DeriveMigrationName)]
pub struct Migration;

#[async_trait]
impl MigrationTrait for Migration {
    async fn up(&self, manager: &SchemaManager) -> Result<(), DbErr> {
        manager
            .create_table(
                Table::create()
                    .table(Post::Table)
                    .if_not_exists()
                    .col(pk_auto(Post::Id))         // INTEGER PRIMARY KEY AUTOINCREMENT / IDENTITY
                    .col(string(Post::Title))
                    .col(string_null(Post::Body))
                    .to_owned(),
            )
            .await?;

        manager
            .create_index(
                Index::create()
                    .if_not_exists()
                    .name("idx-post-title")
                    .table(Post::Table)
                    .col(Post::Title)
                    .to_owned(),
            )
            .await?;

        Ok(())
    }

    async fn down(&self, manager: &SchemaManager) -> Result<(), DbErr> {
        manager.drop_index(Index::drop().name("idx-post-title").to_owned()).await?;
        manager.drop_table(Table::drop().table(Post::Table).to_owned()).await?;
        Ok(())
    }
}

#[derive(DeriveIden)]
enum Post {
    Table,
    Id,
    Title,
    Body,
}
```

## Schema helper functions (`schema::*`)
```rust
pk_auto(col)                     // PK + auto-increment (IDENTITY on MSSQL)
big_integer(col)                 // BIGINT
big_integer_null(col)            // nullable BIGINT
string(col)                      // VARCHAR(255) / nvarchar(255) on MSSQL
string_len(col, 100)             // VARCHAR(100)
string_len_null(col, 100)        // nullable VARCHAR(100)
string_len_uniq(col, 320)        // VARCHAR(320) UNIQUE
integer(col)                     // INT
integer_null(col)                // nullable INT
boolean(col)                     // BOOLEAN / TINYINT(1)
timestamp(col)                   // TIMESTAMP (without time zone)

// MySQL ENUM column
enumeration(col, Alias::new("role"), [Role::Owner, Role::Admin, Role::Operator])

// Default value
string(Users::Name).default(Expr::value("anonymous"))
timestamp(Orders::CreatedAt).default(Expr::current_timestamp())
boolean(Stores::Active).default(Expr::value(true))
```

### Named foreign keys and indexes (recommended — aids debugging)

```rust
ForeignKey::create()
    .name("fk-order_items-order_id")
    .from(OrderItems::Table, OrderItems::OrderId)
    .to(Orders::Table, Orders::Id)
    .on_delete(ForeignKeyAction::Cascade)
    .to_owned()

Index::create()
    .name("idx-store_users-store_id-user_id")
    .table(StoreUsers::Table)
    .col(StoreUsers::StoreId)
    .col(StoreUsers::UserId)
    .unique()
    .if_not_exists()
    .to_owned()
```

## SchemaManager DDL API
```rust
manager.create_table(..)       manager.drop_table(..)
manager.alter_table(..)        manager.rename_table(..)
manager.truncate_table(..)
manager.create_index(..)       manager.drop_index(..)
manager.create_foreign_key(..) manager.drop_foreign_key(..)

// Inspection
manager.has_table("orders").await?
manager.has_column("orders", "status").await?
manager.has_index("orders", "idx-orders-status").await?
```

## Raw SQL in migrations
```rust
let db = manager.get_connection();

// DDL / no bindings
db.execute_unprepared("CREATE TABLE ...").await?;

// With values
db.execute_raw(Statement::from_sql_and_values(
    manager.get_database_backend(),
    r#"INSERT INTO "cake" ("name") VALUES ($1)"#,
    ["Cheese Cake".into()]
)).await?;
```

## Seed data inside a migration
```rust
cake::ActiveModel { name: Set("Cheesecake".to_owned()), ..Default::default() }
    .insert(manager.get_connection())
    .await?;
```

## CLI commands
```shell
sea-orm-cli migrate generate create_post  # new migration file
sea-orm-cli migrate up                    # run pending migrations
sea-orm-cli migrate down                  # rollback last
sea-orm-cli migrate down -n 3             # rollback 3
sea-orm-cli migrate status
sea-orm-cli migrate fresh                 # drop all + re-run all
sea-orm-cli migrate refresh               # down all + up all

# Override URL
sea-orm-cli migrate up -u mssql://sa:pass@localhost/db

# MSSQL non-default schema
sea-orm-cli migrate -u "mssql://sa:pass@localhost/db" -s my_schema
```

## Run programmatically
```rust
let db = Database::connect("mssql://...?currentSchema=my_schema").await?;
migration::Migrator::up(&db, None).await?;
```
