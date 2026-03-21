# SeaORM v2 — MSSQL-Specific Features (SeaORM-X)

## Nested transactions via savepoints
```rust
let txn = db.begin().await?;
{
    let inner = txn.begin().await?;
    // ... work ...
    inner.commit().await?;   // commits to savepoint only
}
txn.commit().await?;         // commits outer transaction
```

| Depth | `begin` | `commit` | `rollback` |
|-------|---------|----------|------------|
| 0→1 | `BEGIN TRAN` | `COMMIT TRAN` | `ROLLBACK TRAN` |
| n→n+1 | `SAVE TRAN _sqlz_savepoint_n` | _(no-op)_ | `ROLLBACK TRAN _sqlz_savepoint_n` |

## Automatic IDENTITY_INSERT
SeaORM-X detects when an explicit PK value is `Set(...)` on an auto-increment column and wraps the INSERT in `SET IDENTITY_INSERT [table] ON/OFF` automatically. No manual intervention needed.

## Automatic schema rewriting (`currentSchema`)
When `?currentSchema=my_schema` is set in the connection string, every SQL statement (including subqueries, JOINs, CTEs) is automatically prefixed with `[my_schema].[table]`.

```rust
let db = Database::connect(
    "mssql://user:pass@localhost:1433/my_db?currentSchema=my_schema"
).await?;
// All queries automatically become: SELECT ... FROM [my_schema].[table] ...
```

## Tuple IN fallback (MSSQL doesn't support native tuple syntax)
```rust
cake::Entity::find()
    .filter(cake::Entity::column_tuple_in(
        [cake::Column::Id, cake::Column::Name],
        &[(1i32, "a").into_value_tuple(), (2i32, "b").into_value_tuple()],
        DbBackend::MsSql,
    ).unwrap())
    .all(db)
    .await?;
// MSSQL: ([id] = 1 AND [name] = 'a') OR ([id] = 2 AND [name] = 'b')
// MySQL/Postgres: native tuple syntax
```

## OUTPUT clause mapping
- `INSERT` → `OUTPUT INSERTED.*`
- `DELETE` → `OUTPUT DELETED.*`
- Constraint violations surfaced as typed `DbErr` variants: `UniqueConstraintViolation`, `ForeignKeyConstraintViolation`

## Entity-first schema sync (`schema-sync` feature)
```rust
db.get_schema_builder()
    .register(order::Entity)
    .register(customer::Entity)
    .sync(db)                    // creates tables in FK-topological order
    .await?;
```

## Known SQLz limitations
- No compile-time query verification (no equivalent of `sqlx::query!` macro)
- Fixed set of supported wire types — adding custom types requires changes inside SQLz
