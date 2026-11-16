# SeaORM 2.x — MSSQL-Specific Features (SeaORM-X)

> SeaORM-X is a **commercial, closed-beta** extension (SSH access to `SeaQL/sea-orm-x` required).
> It is still pinned to `2.0.0-rc` while open-source SeaORM is on stable 2.0.x — check API
> differences against the SeaORM-X docs (sea-ql.org/SeaORM-X) when something here does not compile.
> Setup: `01-cargo-setup.md`. Connection string: `02-connection.md`.

## Nested transactions via savepoints
Same API as other backends (`txn.begin()` inside a transaction — see `11-transactions.md`).
SeaORM-X emits:

| Depth | `begin` | `commit` | `rollback` |
|-------|---------|----------|------------|
| 0→1 | `BEGIN TRAN` | `COMMIT TRAN` | `ROLLBACK TRAN` |
| n→n+1 | `SAVE TRAN _sqlz_savepoint_n` | _(no-op)_ | `ROLLBACK TRAN _sqlz_savepoint_n` |

Dropping a transaction without `commit` rolls it back.

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
- `INSERT` → `OUTPUT INSERTED.*` (powers `exec_with_returning`)
- `DELETE` → `OUTPUT DELETED.*`
- UPDATE returning is not documented for SeaORM-X — re-select after `update_many` if needed.
- Constraint violations are classified like other backends:
  `err.sql_err()` → `SqlErr::UniqueConstraintViolation` / `SqlErr::ForeignKeyConstraintViolation`
  (see `12-error-handling.md`).

## Raw batches
`execute_unprepared` sends a raw TDS batch — use it for multi-statement T-SQL in migrations.

## Entity-first schema sync (`schema-sync` feature)
Works on MSSQL too — see `13-entity-first.md`.
```rust
db.get_schema_builder()
    .register(order::Entity)
    .register(customer::Entity)
    .sync(db)                    // creates tables in FK-topological order
    .await?;
```

## Known SQLz limitations
- No compile-time query verification (no equivalent of `sqlx::query!` macro)
- Fixed set of supported wire types — no user-defined `Encode`/`Decode`; custom types require changes inside SQLz
