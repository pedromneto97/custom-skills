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
