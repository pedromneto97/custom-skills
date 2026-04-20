# SeaORM v2 — Repository Boundary Patterns

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

## Error translation

Translate `DbErr` immediately at the adapter boundary:

```rust
fn is_unique_violation(err: &sea_orm::DbErr) -> bool {
    let msg = err.to_string();
    msg.contains("duplicate key")
        || msg.contains("Duplicate entry")
        || msg.contains("UNIQUE constraint failed")
}

fn map_db_err(err: sea_orm::DbErr) -> DomainError {
    if is_unique_violation(&err) {
        return DomainError::Conflict("resource already exists".into());
    }
    DomainError::Infrastructure("database error".into())
}
```

Prefer typed `DbErr` variants where the backend exposes them clearly; keep message fallback for
cross-database portability.

## Transaction pattern

```rust
use sea_orm::{TransactionError, TransactionTrait};

let value = db
    .transaction::<_, ResultType, DomainError>(|txn| {
        Box::pin(async move {
            let root = root::ActiveModel { ..Default::default() }
                .insert(txn)
                .await
                .map_err(map_db_err)?;

            related::Entity::insert_many(build_related(root.id))
                .exec(txn)
                .await
                .map_err(map_db_err)?;

            Ok(ResultType::from(root))
        })
    })
    .await
    .map_err(|e| match e {
        TransactionError::Transaction(domain) => domain,
        TransactionError::Connection(_) => DomainError::Infrastructure("database connection".into()),
    })?;
```

## Collection ports and allocation

When domain ports return collections, prefer iterator outputs over forcing `Vec<T>` in the port
signature. Repositories can still collect DB rows internally before returning an iterator.