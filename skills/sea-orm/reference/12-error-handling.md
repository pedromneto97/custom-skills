# SeaORM 2.x — Error Handling

All SeaORM operations return `Result<_, DbErr>`.

## Portable constraint classification — `sql_err()`
```rust
use sea_orm::{DbErr, SqlErr};

match user_am.insert(db).await {
    Ok(model) => Ok(model),
    Err(err) => match err.sql_err() {
        Some(SqlErr::UniqueConstraintViolation(msg)) => Err(DomainError::Conflict(msg)),
        Some(SqlErr::ForeignKeyConstraintViolation(msg)) => Err(DomainError::InvalidReference(msg)),
        _ => Err(DomainError::Infrastructure(err.to_string())),
    },
}
```

- Works across MySQL, Postgres, SQLite (and MSSQL via SeaORM-X).
- Only unique-key and foreign-key violations are recognized; `SqlErr` is `#[non_exhaustive]`.
- Do **not** string-match driver messages (`"duplicate key"`, `"Duplicate entry"`, ...) — they
  differ per backend and locale.

## Common `DbErr` variants
| Variant | When |
|---------|------|
| `RecordNotFound(String)` | `require_one`, update/delete of a missing row |
| `RecordNotInserted` | insert with `ON CONFLICT DO NOTHING` inserted nothing (use `try_insert()` to get `Ok`) |
| `RecordNotUpdated` | `ActiveModel::update` matched no row |
| `PrimaryKeyNotSet { .. }` | `Update::one` / `Delete::one` `.validate()` with PK `NotSet` |
| `BackendNotSupported { .. }` | feature not available on this backend (e.g. `exec_with_returning_keys` on MySQL) |
| `ConnectionAcquire(..)` | pool exhausted / timeout |
| `Conn(..)` | cannot connect |
| `Exec(RuntimeErr)` / `Query(RuntimeErr)` | statement failed in the driver |
| `Custom(String)` | your own errors (e.g. from `ActiveModelBehavior::before_save`) |

`DbErr` is `#[non_exhaustive]` — always keep a `_` arm.

## Backend-specific details (SQLSTATE, check constraints, ...)
```rust
use sea_orm::{DbErr, RuntimeErr};
use std::ops::Deref;

if let Some(sql_err) = err.sql_err() {
    // portable path above
} else if let DbErr::Query(RuntimeErr::SqlxError(e)) | DbErr::Exec(RuntimeErr::SqlxError(e)) = &err
    && let sea_orm::sqlx::Error::Database(db_err) = e.deref()
{
    // Postgres: "23514" = check_violation, "40001" = serialization_failure (retry)
    match db_err.code().as_deref() {
        Some("40001") => { /* retry the transaction */ }
        _ => {}
    }
}
```

## Mapping at the repository boundary
Convert `DbErr` to a domain error as soon as it leaves the adapter — see
`10-repository-patterns.md`. Never expose `DbErr` (or its message) in HTTP responses.
