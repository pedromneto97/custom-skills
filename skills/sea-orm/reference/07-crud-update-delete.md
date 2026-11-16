# SeaORM 2.x — CRUD: Update & Delete

## ActiveValue states
| State | Meaning | In UPDATE |
|-------|---------|-----------|
| `Set(v)` | changed value | written |
| `Unchanged(v)` | loaded from DB | skipped |
| `NotSet` | unknown / default | skipped |

```rust
let mut am: fruit::ActiveModel = model.into();      // all fields Unchanged
am.name = Set("Pear".into());
am.name.set_if_not_equals("Pear".into());           // stays Unchanged if equal → no-op update
am.name.set_if_unset("Pear".into());                // set only if currently NotSet
am.set(fruit::Column::Name, "Apple".into());        // dynamic, by column
assert!(am.is_changed());
let model: fruit::Model = am.try_into_model()?;     // fails if any field NotSet
```

## UPDATE

### Update one
```rust
let mut pear: fruit::ActiveModel = Fruit::find_by_id(28).one(db).await?.unwrap().into();
pear.name = Set("Sweet pear".to_owned());
let pear: fruit::Model = pear.update(db).await?;
```

### Force-update all columns
```rust
pear.reset_all();                           // mark all columns as Set
pear.reset(fruit::Column::CakeId);          // force a specific column
let pear = pear.update(db).await?;
```

### Building `Update::one` / `Delete::one` statements (v2)
`.validate()?` is required before `.build()`; it returns `DbErr::PrimaryKeyNotSet` if the PK is
`NotSet` (v1 panicked).

```rust
let stmt = Update::one(fruit::ActiveModel {
        id: Unchanged(1),
        name: NotSet,
        cake_id: Set(Some(2)),
    })
    .validate()?
    .build(DbBackend::Postgres);
```

### Update many (bulk)
```rust
// Column expressions
fruit::Entity::update_many()
    .col_expr(fruit::Column::CakeId, Expr::value(1))
    .filter(fruit::Column::Name.contains("Apple"))
    .exec(db)
    .await?;

// Or from an ActiveModel — only Set fields are written
fruit::Entity::update_many()
    .set(fruit::ActiveModel { cake_id: Set(Some(1)), ..Default::default() })
    .filter(fruit::Column::Name.contains("Apple"))
    .exec(db)
    .await?;
```

> `Expr` methods like `.add()` / `.eq()` come from `ExprTrait` in v2 —
> `use sea_orm::ExprTrait;` (or the prelude).

### Update with returning (Postgres / SQLite / MariaDB with `mariadb-use-returning`)
```rust
let updated: Vec<fruit::Model> = fruit::Entity::update_many()
    .col_expr(fruit::Column::CakeId, Expr::value(1))
    .filter(fruit::Column::Name.contains("Apple"))
    .exec_with_returning(db)
    .await?;
```

### Nested update (v2, `#[sea_orm::model]` only)
```rust
let mut bob = user::Entity::load()
    .filter_by_id(1)
    .with(post::Entity)
    .one(db)
    .await?
    .unwrap()
    .into_active_model();                                             // → user::ActiveModelEx
bob.posts.push(post::ActiveModel::builder().set_title("New post"));   // append
// bob.posts.replace_all([...]);                                       // replace whole set
bob.save(db).await?;                                                  // one transaction, FK order
```

> Nested ActiveModel relation types are marked semver-exempt (may still change in 2.x). Compare
> relations with `is_unloaded_or_not_found()` / `is_not_found()` / `as_ref()`, not `== None`.

---

## DELETE

### Delete one (from Model)
```rust
let orange: fruit::Model = Fruit::find_by_id(30).one(db).await?.unwrap();
let res: DeleteResult = orange.delete(db).await?;
assert_eq!(res.rows_affected, 1);
```

### Delete by PK (v2 — returns `ValidatedDeleteOne`)
```rust
let res: DeleteResult = Fruit::delete_by_id(38).exec(db).await?;
```

### Delete by unique key (v2 — auto-generated for `#[sea_orm(unique)]`)
```rust
user::Entity::delete_by_email("bob@example.com").exec(db).await?;
```

### Delete many
```rust
let res: DeleteResult = fruit::Entity::delete_many()
    .filter(fruit::Column::Name.contains("Orange"))
    .exec(db)
    .await?;
```

### Delete with returning (Postgres / SQLite / MSSQL `OUTPUT DELETED.*`)
```rust
// Single — v2 returns Option<Model> (was Vec<Model>)
let deleted: Option<fruit::Model> = fruit::Entity::delete_by_id(3)
    .exec_with_returning(db)
    .await?;

// Many — returns Vec<Model>
let deleted: Vec<order::Model> = order::Entity::delete_many()
    .filter(order::Column::CustomerId.eq(22))
    .exec_with_returning(db)
    .await?;
```

### Cascade delete (v2, `#[sea_orm::model]` only)
```rust
// Deletes the model and its related children in FK-safe order, in one transaction.
// Shorthand for user_model.into_ex().delete(db)
let res: DeleteResult = user_model.cascade_delete(db).await?;
```
Prefer DB-level `ON DELETE CASCADE` foreign keys when every caller should get the same behavior.
