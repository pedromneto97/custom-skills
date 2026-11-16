# SeaORM 2.x — Transactions

Inside a transaction, pass `txn` (a `&DatabaseTransaction`) wherever you would pass `db`.
Both implement `ConnectionTrait`.

## Closure API — commit on `Ok`, rollback on `Err`
```rust
use sea_orm::{TransactionError, TransactionTrait};

// Type params: <_, ReturnType, ErrorType>; map_db_err: 10-repository-patterns.md
// `reference: String` and `now: DateTime` are moved into the closure;
// the error type (DomainError) must implement std::error::Error
let (customer, order) = db
    .transaction::<_, (Customer, Order), DomainError>(|txn| {
        Box::pin(async move {
            // Set every NOT NULL column without a DB default — NotSet fields are left out of the INSERT
            let customer: Customer = customer::ActiveModel { name: Set("Alice".into()), ..Default::default() }
                .insert(txn)
                .await
                .map_err(map_db_err)?
                .into();

            let order: Order = order::ActiveModel {
                customer_id: Set(customer.id),
                status: Set("new".into()),
                reference: Set(reference),
                created_at: Set(now),
                ..Default::default()
            }
            .insert(txn)
            .await
            .map_err(map_db_err)?
            .into();

            Ok((customer, order))
        })
    })
    .await
    .map_err(|err| match err {
        TransactionError::Transaction(domain_err) => domain_err,       // error from the closure
        TransactionError::Connection(_) => DomainError::Infrastructure("db".into()), // begin/commit failed
    })?;
```

`Box::pin` is needed because `TransactionTrait::transaction` predates stable async closures.

## Async-closure API (2.0, no `Box::pin`)
Inherent method on `DatabaseConnection` / `DatabaseTransaction`:

```rust
db.transaction_async(async |txn| {
    bakery::Entity::insert(active_model).exec(txn).await?;
    Ok::<_, DbErr>(())
})
.await?;   // Result<T, TransactionError<E>>
```

## Manual begin / commit
```rust
let txn = db.begin().await?;

bakery::ActiveModel { name: Set("SeaSide".into()), ..Default::default() }
    .save(&txn)
    .await?;

txn.commit().await?;   // dropping `txn` without commit → rollback
// txn.rollback().await?;   // explicit rollback
```

## Nested transactions (savepoints)
```rust
let txn = db.begin().await?;
{
    let inner = txn.begin().await?;       // SAVEPOINT
    // ... work ...
    inner.commit().await?;                // RELEASE SAVEPOINT (only the savepoint)
}
txn.commit().await?;                      // commits everything

// closure form works the same: txn.transaction::<_, _, E>(|inner| Box::pin(async move { .. }))
```

## Isolation level & access mode (MySQL / Postgres)
```rust
use sea_orm::{AccessMode, IsolationLevel};

let txn = db
    .begin_with_config(Some(IsolationLevel::Serializable), Some(AccessMode::ReadOnly))
    .await?;

db.transaction_with_config::<_, (), DbErr>(
    |txn| Box::pin(async move { /* ... */ Ok(()) }),
    Some(IsolationLevel::RepeatableRead),
    None,
)
.await?;
```

## Guidelines
- Keep transactions short; never await external I/O (HTTP, queues) while holding one.
- Nested ActiveModel `save` / `cascade_delete` already open their own transaction (nested as a
  savepoint if you call them inside one).
- Postgres `SERIALIZABLE` can fail with serialization errors — retry the whole closure.
- With a pool of 1 (e.g. SQLite in-memory tests) a held `txn` blocks every other query on `db`
  → `DbErr::ConnectionAcquire(Timeout)`. Use `txn` for all queries inside the transaction.
- MSSQL (SeaORM-X) savepoint SQL: see `08-mssql-features.md`.
