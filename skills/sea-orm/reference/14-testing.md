# SeaORM 2.x — Testing

Three levels, cheapest first. Repositories that take `&impl ConnectionTrait` /
`&DatabaseConnection` (see `10-repository-patterns.md`) work with all three unchanged.

| Level | Speed | Checks real SQL? | Use for |
|-------|-------|------------------|---------|
| `MockDatabase` | fastest | no — you script the results | mapping, error handling, which statements run |
| SQLite in-memory | fast | yes (SQLite dialect) | query logic, relations, constraints |
| Real DB (testcontainers) | slow | yes (production dialect) | Postgres/MySQL-specific SQL, migrations |

## MockDatabase (feature `mock`)
```toml
[dev-dependencies]
sea-orm = { version = "2.0", features = ["mock", "sqlx-postgres", "runtime-tokio-rustls", "macros"] }
```

```rust
use sea_orm::{DatabaseBackend, DbErr, MockDatabase, MockExecResult, Transaction, entity::*};

#[tokio::test]
async fn finds_user_and_inserts_post() -> Result<(), DbErr> {
    let db = MockDatabase::new(DatabaseBackend::Postgres)
        // one Vec per SELECT, in call order
        .append_query_results([vec![user::Model {
            id: 1,
            name: "Bob".into(),
            email: "bob@example.com".into(),
            active: true,
        }]])
        // INSERT ... RETURNING is a query too → also a query result
        .append_query_results([vec![post::Model { id: 10, user_id: 1, title: "Hi".into() }]])
        // one entry per INSERT/UPDATE/DELETE without RETURNING
        .append_exec_results([MockExecResult { last_insert_id: 0, rows_affected: 1 }])
        .into_connection();

    let bob = user::Entity::find_by_id(1).one(&db).await?.unwrap();
    assert_eq!(bob.name, "Bob");

    let post = post::ActiveModel { user_id: Set(1), title: Set("Hi".into()), ..Default::default() }
        .insert(&db)
        .await?;
    assert_eq!(post.id, 10);

    user::Entity::delete_by_id(1).exec(&db).await?;

    // Assert the exact SQL that ran
    let log = db.into_transaction_log();
    assert_eq!(log.len(), 3);
    assert_eq!(
        log[0],
        Transaction::from_sql_and_values(
            DatabaseBackend::Postgres,
            r#"SELECT "user"."id", "user"."name", "user"."email", "user"."active" FROM "user" WHERE "user"."id" = $1 LIMIT $2"#,
            [1i32.into(), 1u64.into()],
        )
    );
    Ok(())
}
```

- Results are consumed in order; a query with no scripted result returns an error.
- Script failures with `.append_query_errors([DbErr::Custom("boom".into())])` /
  `.append_exec_errors([...])` to test error mapping.
- Mock rows are returned as-is — `MockDatabase` never checks filters, joins or constraints.
- Exact SQL strings are brittle (column order, aliases change between versions); assert only
  where the statement shape matters.

## SQLite in-memory
```toml
[dev-dependencies]
sea-orm = { version = "2.0", features = ["sqlx-sqlite", "runtime-tokio", "macros"] }
```

```rust
use sea_orm::{ConnectOptions, Database, DatabaseConnection, DbErr};

async fn test_db() -> Result<DatabaseConnection, DbErr> {
    let mut opt = ConnectOptions::new("sqlite::memory:");
    opt.max_connections(1);   // each pooled connection would be a separate empty DB
    let db = Database::connect(opt).await?;

    // Create tables straight from the entities (no `schema-sync` feature needed for apply)
    db.get_schema_builder()
        .register(user::Entity)
        .register(post::Entity)
        .apply(&db)
        .await?;
    // ...or run the real migrations: migration::Migrator::up(&db, None).await?;
    Ok(db)
}
```

- One DB per test → tests are isolated and can run in parallel.
- SQLite differs from Postgres/MySQL (types, `RETURNING`, enum types, `= ANY`, non-constant
  defaults) — keep dialect-specific queries for the real-DB level.

## Real database with testcontainers
```toml
[dev-dependencies]
testcontainers-modules = { version = "0.15", features = ["postgres"] }
```

```rust
use testcontainers_modules::{postgres::Postgres, testcontainers::runners::AsyncRunner};

#[tokio::test]
async fn works_on_postgres() -> Result<(), Box<dyn std::error::Error>> {
    let node = Postgres::default().start().await?;          // keep `node` alive for the test
    let port = node.get_host_port_ipv4(5432).await?;
    let url = format!("postgres://postgres:postgres@127.0.0.1:{port}/postgres");

    let db = sea_orm::Database::connect(&url).await?;
    migration::Migrator::up(&db, None).await?;             // test the real migrations too
    // ... exercise repositories against `db`
    Ok(())
}
```

Needs a Docker daemon on the machine / CI runner. Start one container per test binary (e.g. a
shared `OnceCell`) if per-test startup is too slow.
