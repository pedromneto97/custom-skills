# SeaORM v2 — CRUD: Update & Delete

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

### Update many (bulk)
```rust
fruit::Entity::update_many()
    .col_expr(fruit::Column::CakeId, Expr::value(1))
    .filter(fruit::Column::Name.contains("Apple"))
    .exec(db)
    .await?;
```

### Update with returning (Postgres / SQLite / MSSQL OUTPUT)
```rust
let updated: Vec<fruit::Model> = fruit::Entity::update_many()
    .col_expr(fruit::Column::CakeId, Expr::value(1))
    .filter(fruit::Column::Name.contains("Apple"))
    .exec_with_returning(db)
    .await?;
```

---

## DELETE

### Delete one (from Model)
```rust
let orange: fruit::Model = Fruit::find_by_id(30).one(db).await?.unwrap();
let res: DeleteResult = orange.delete(db).await?;
assert_eq!(res.rows_affected, 1);
```

### Delete by PK (v2 — ValidatedDeleteOne)
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

### Delete with returning (Postgres / SQLite / MSSQL OUTPUT DELETED.*)
```rust
// Single — returns Option<Model>
let deleted: Option<fruit::Model> =
    fruit::Entity::delete(ActiveModel { id: Set(3), ..Default::default() })
        .exec_with_returning(db)
        .await?;

// Many — returns Vec<Model>
let deleted: Vec<order::Model> = order::Entity::delete_many()
    .filter(order::Column::CustomerId.eq(22))
    .exec_with_returning(db)
    .await?;
```
