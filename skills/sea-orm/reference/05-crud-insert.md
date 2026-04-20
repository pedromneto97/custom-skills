# SeaORM v2 — CRUD: Insert

## Insert one — get Model back
```rust
let pear = fruit::ActiveModel {
    name: Set("Pear".to_owned()),
    ..Default::default()
};
let pear: fruit::Model = pear.insert(db).await?;
```

## Insert one — get last insert id
```rust
let res: InsertResult = fruit::Entity::insert(pear).exec(db).await?;
println!("{}", res.last_insert_id);
```

## Insert many
```rust
let res: InsertResult = fruit::Entity::insert_many([apple, orange]).exec(db).await?;
```

Useful for initialization flows where one transaction creates a root record and related defaults.

## Insert many — empty guard
```rust
let res = Bakery::insert_many(std::iter::empty())
    .on_empty_do_nothing()
    .exec(db)
    .await;
assert!(matches!(res, Ok(TryInsertResult::Empty)));
```

## Insert with returning (Postgres / SQLite / MSSQL OUTPUT)
```rust
// Single
let model = cake::Entity::insert(active_model).exec_with_returning(db).await?;
// -> cake::Model

// Many
let models = cake::Entity::insert_many([am1, am2]).exec_with_returning(db).await?;
// -> Vec<cake::Model>

// Composite PK keys only
let keys = link::Entity::insert_many([am1, am2])
    .exec_with_returning_keys(db)
    .await?;
// -> Vec<(i32, i32)>
```

## On-conflict
```rust
use sea_query::OnConflict;

// Do nothing (shorthand)
entity::Entity::insert(am)
    .on_conflict_do_nothing()
    .exec(db)
    .await?;

// Do nothing on specific column conflict
entity::Entity::insert(am)
    .on_conflict(
        OnConflict::column(entity::Column::Name)
            .do_nothing()
            .to_owned()
    )
    .exec(db)
    .await?;

// Upsert
entity::Entity::insert(am)
    .on_conflict(
        OnConflict::column(entity::Column::Name)
            .update_column(entity::Column::Name)
            .to_owned()
    )
    .exec(db)
    .await?;

// insert_many + do_nothing() → TryInsertResult::Conflicted instead of error
let res = entity::Entity::insert_many(items)
    .on_conflict(on_conflict)
    .do_nothing()
    .exec(db)
    .await;
assert!(matches!(res, Ok(TryInsertResult::Conflicted)));
```

## Transactions

```rust
use sea_orm::{TransactionTrait, TransactionError};

// Closure receives &txn — use it in place of &db for all queries inside the transaction.
// Type params: <_, ReturnType, ErrorType>
let result = db.transaction::<_, (User, Order), DomainError>(|txn| {
    Box::pin(async move {
        let user: User = user::ActiveModel { name: Set("Alice".into()), ..Default::default() }
            .insert(txn).await
            .map_err(|e| DomainError::Infrastructure(e.to_string()))?
            .into();

        let order: Order = order::ActiveModel {
            user_id: Set(user.id),
            ..Default::default()
        }
        .insert(txn).await
        .map_err(|e| DomainError::Infrastructure(e.to_string()))?
        .into();

        Ok((user, order))
    })
})
.await
.map_err(|err| match err {
    TransactionError::Transaction(domain_err) => domain_err,        // error from inside the closure
    TransactionError::Connection(_)           => DomainError::Infrastructure("db".into()),
})?;
```

### Batch initialization inside a transaction

```rust
let user = users::ActiveModel { email: Set(email), ..Default::default() }
    .insert(txn)
    .await?;

let defaults = vec![
    settings::ActiveModel { user_id: Set(user.id), key: Set("theme".into()), value: Set("light".into()) },
    settings::ActiveModel { user_id: Set(user.id), key: Set("locale".into()), value: Set("en".into()) },
];

settings::Entity::insert_many(defaults).exec(txn).await?;
```

> **Duplicate-key errors inside a transaction:** map the `DbErr` to the appropriate domain
> error *before* returning from the closure — e.g. inspect the error message for
> `"Duplicate entry"` (MySQL) or `"duplicate key value"` (Postgres) and map to a
> conflict variant.

```rust
fn is_unique_violation(err: &sea_orm::DbErr) -> bool {
    let msg = err.to_string();
    msg.contains("duplicate key")
        || msg.contains("Duplicate entry")
        || msg.contains("UNIQUE constraint failed")
}
```

---

## MSSQL — Explicit PK (IDENTITY_INSERT automatic in SeaORM-X)
```rust
// SeaORM-X wraps in SET IDENTITY_INSERT ON/OFF automatically when id is Set(...)
let model = bakery::ActiveModel {
    id: Set(42),
    name: Set("My Bakery".to_owned()),
    ..Default::default()
};
Bakery::insert(model).exec(db).await?;
```
