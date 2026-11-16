# SeaORM 2.x — CRUD: Insert

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
let res: InsertResult<fruit::ActiveModel> = fruit::Entity::insert(pear).exec(db).await?;
println!("{}", res.last_insert_id);
```

## Save (insert-or-update)
```rust
// PK NotSet → INSERT; PK Set/Unchanged → UPDATE. Saving an unchanged model is a no-op.
let am: fruit::ActiveModel = fruit::ActiveModel { name: Set("Kiwi".into()), ..Default::default() }
    .save(db)
    .await?;
```

## Insert many
```rust
// v2: returns InsertManyResult; last_insert_id is Option<_>
let res = fruit::Entity::insert_many([apple, orange]).exec(db).await?;
let last: Option<i32> = res.last_insert_id;
```

Empty input is **not** an error in v2 — it returns `Ok` with `last_insert_id == None`
(and `vec![]` for `exec_with_returning`). `on_empty_do_nothing()` is no longer needed
(deprecated since 2.0).

Useful for initialization flows where one transaction creates a root record and related defaults.

## Insert with returning
Supported on Postgres and SQLite (≥ 3.35, on by default), MariaDB with `mariadb-use-returning`,
and MSSQL via SeaORM-X (`OUTPUT INSERTED.*`). Not on MySQL.

```rust
// Single
let model: cake::Model = cake::Entity::insert(active_model).exec_with_returning(db).await?;

// Many
let models: Vec<cake::Model> = cake::Entity::insert_many([am1, am2]).exec_with_returning(db).await?;

// Primary keys only (e.g. composite PK)
let keys: Vec<(i32, i32)> = link::Entity::insert_many([am1, am2])
    .exec_with_returning_keys(db)
    .await?;   // MySQL → DbErr::BackendNotSupported
```

> Deprecated in 2.0: `exec_with_returning_many` (use `exec_with_returning`), `Insert::do_nothing()`
> and `on_empty_do_nothing()` (use `try_insert()` / `on_conflict_do_nothing*()`).

## On-conflict

A conflict that ends with **no row inserted or updated** returns `Err(DbErr::RecordNotInserted)`.
Call `.try_insert()` to get `TryInsertResult` instead of an error.

```rust
use sea_orm::sea_query::OnConflict;
use sea_orm::TryInsertResult;

// Upsert — update on conflict
entity::Entity::insert(am)
    .on_conflict(
        OnConflict::column(entity::Column::Name)
            .update_column(entity::Column::Price)
            .to_owned(),
    )
    .exec(db)
    .await?;

// Do nothing on a specific column — tolerate conflict
let res = entity::Entity::insert_many(items)
    .on_conflict(
        OnConflict::column(entity::Column::Name)
            .do_nothing()
            .to_owned(),
    )
    .try_insert()
    .exec(db)
    .await?;

match res {
    TryInsertResult::Inserted(r) => { /* r.last_insert_id */ }
    TryInsertResult::Conflicted  => { /* all rows skipped */ }
    TryInsertResult::Empty       => { /* no input rows */ }
}

// Shorthands — both return TryInsert (→ TryInsertResult), with a MySQL polyfill
// Postgres/SQLite: ON CONFLICT ("id") DO NOTHING
// MySQL:           ON DUPLICATE KEY UPDATE `id` = `id`
let res = entity::Entity::insert(am)
    .on_conflict_do_nothing()                          // conflict on PK
    .exec(db)
    .await?;

let res = entity::Entity::insert_many(items)
    .on_conflict_do_nothing_on([entity::Column::Name]) // conflict on given columns
    .exec(db)
    .await?;
```

## Nested insert with the ActiveModel builder (v2, `#[sea_orm::model]` only)

Insert a graph of related rows in one call. SeaORM runs it in a single transaction and inserts in
FK order (parents first).

```rust
let bob = user::ActiveModel::builder()
    .set_name("Bob")
    .set_email("bob@example.com")
    .set_profile(profile::ActiveModel::builder().set_picture("bob.jpg"))   // has_one
    .add_post(post::ActiveModel::builder().set_title("Hello"))             // has_many
    .add_post(post::ActiveModel::builder().set_title("World"))
    .add_tag(tag::ActiveModel::builder().set_name("rust"))                 // M-N via junction
    .save(db)
    .await?;
```

The builder produces an `ActiveModelEx` with `ActiveHasOne` / `ActiveHasMany` / `ActiveBelongsTo`
fields. Plain `ActiveModel` is unchanged — use the builder only where nested writes help.

## Transactions
Multi-statement inserts belong in a transaction — see `11-transactions.md`.

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

## Duplicate-key errors
Use `err.sql_err()` — don't string-match driver messages. See `12-error-handling.md`.

```rust
use sea_orm::{DbErr, SqlErr};

match user_am.insert(db).await {
    Ok(m) => Ok(m),
    Err(e) if matches!(e.sql_err(), Some(SqlErr::UniqueConstraintViolation(_))) => {
        Err(DomainError::Conflict("email already used".into()))
    }
    Err(e) => Err(DomainError::Infrastructure(e.to_string())),
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
