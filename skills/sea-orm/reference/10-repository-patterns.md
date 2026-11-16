# SeaORM 2.x — Repository Boundary Patterns

This reference focuses on adapter boundaries in hexagonal architectures:
- keep SeaORM entities inside outbound,
- map models at the repository edge,
- convert infrastructure errors to domain errors.

## Recommended repository layout

```text
outbound/src/database/
├── connection.rs
├── models/
├── repositories/
│   ├── order.rs
│   └── user.rs
└── mappers.rs
```

`repositories/` owns query logic.
`mappers.rs` owns persistence-to-domain conversion rules.

## Mapper strategy

Use `From`/`Into` for direct conversions:

```rust
impl From<order::Model> for Order {
    fn from(model: order::Model) -> Self {
        Self {
            id: model.id,
            status: model.status.into(),
        }
    }
}
```

Use explicit mapper functions when conversion can fail or needs defaults.
Don't leak `ModelEx` / `HasMany` / `ActiveModel` types through ports — map them to domain types.

## Error translation

Translate `DbErr` immediately at the adapter boundary. Use the portable `sql_err()` classifier
instead of matching error message strings (details in `12-error-handling.md`):

```rust
use sea_orm::{DbErr, SqlErr};

fn map_db_err(err: DbErr) -> DomainError {
    match err.sql_err() {
        Some(SqlErr::UniqueConstraintViolation(_)) => {
            return DomainError::Conflict("resource already exists".into());
        }
        Some(SqlErr::ForeignKeyConstraintViolation(_)) => {
            return DomainError::InvalidReference("related resource not found".into());
        }
        _ => {}
    }
    match err {
        DbErr::RecordNotFound(_) | DbErr::RecordNotUpdated => DomainError::NotFound,
        _ => DomainError::Infrastructure("database error".into()),
    }
}
```

`SqlErr` is `#[non_exhaustive]` — always keep a fallback arm.

Deletes never error on a missing row — map `rows_affected == 0` to `NotFound` explicitly:

```rust
let res = order::Entity::delete_by_id(id).exec(db).await.map_err(map_db_err)?;
if res.rows_affected == 0 {
    return Err(DomainError::NotFound);
}
```

## Transaction pattern

Wrap multi-step writes in `db.transaction(..)` and map `TransactionError` back to the domain
error — full examples (closure, async-closure, manual, savepoints, isolation) in
`11-transactions.md`. Inside the closure, call `map_db_err` on every query.

### Repositories that work inside or outside a transaction
Take `&impl ConnectionTrait` (or a generic `C: ConnectionTrait`) so the same method accepts
`&DatabaseConnection` and `&DatabaseTransaction`:

```rust
pub async fn create_order<C: ConnectionTrait>(db: &C, order: NewOrder) -> Result<Order, DomainError> {
    let model = order::ActiveModel::from(order).insert(db).await.map_err(map_db_err)?;
    Ok(model.into())
}
```

## Collection ports and allocation

When domain ports return collections, prefer iterator outputs over forcing `Vec<T>` in the port
signature. Repositories can still collect DB rows internally before returning an iterator.
