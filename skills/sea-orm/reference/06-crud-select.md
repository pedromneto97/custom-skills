# SeaORM v2 — CRUD: Select

## Find by primary key
```rust
let cake: Option<cake::Model> = Cake::find_by_id(1).one(db).await?;
// Composite PK:
let row: Option<cake_filling::Model> = CakeFilling::find_by_id((6, 8)).one(db).await?;
```

## Filter + order
```rust
let results: Vec<cake::Model> = Cake::find()
    .filter(cake::Column::Name.contains("chocolate"))
    .order_by_asc(cake::Column::Name)
    .all(db)
    .await?;
```

## v2 — Strongly-typed COLUMN constants (requires `#[sea_orm::model]`)
```rust
// Old (Column enum — no compile-time type enforcement)
cake::Column::Name.contains("choc")

// New (type-aware, compile-time checked)
cake::COLUMN.name.contains("choc")          // StringColumn methods only
user::COLUMN.id.between(1, 100)             // NumericColumn methods only
user::COLUMN.created_at.gt(some_date)       // DateTimeLikeColumn methods only
// cake::COLUMN.name.like(2)  → compile error
```

## v2 — Find by unique key (auto-generated for `#[sea_orm(unique)]`)
```rust
let user: Option<user::Model> = user::Entity::find_by_email("bob@example.com").one(db).await?;
// Composite unique key with #[sea_orm(unique_key = "pair")]
let row = composite::Entity::find_by_pair((1, 2)).one(db).await?;
```

## Lazy loading
```rust
let cake = Cake::find_by_id(1).one(db).await?.unwrap();
let fruits: Vec<fruit::Model> = cake.find_related(Fruit).all(db).await?;
```

## Eager loading
```rust
// 1-to-1 join
let pairs: Vec<(fruit::Model, Option<cake::Model>)> =
    Fruit::find().find_also_related(Cake).all(db).await?;

// 1-to-many / many-to-many
let with_fruits: Vec<(cake::Model, Vec<fruit::Model>)> =
    Cake::find().find_with_related(Fruit).all(db).await?;
```

## LoaderTrait
```rust
let cakes: Vec<cake::Model> = Cake::find().all(db).await?;

// 1-to-many
let all_fruits: Vec<Vec<fruit::Model>> = cakes.load_many(Fruit, db).await?;

// many-to-many (v2: no junction entity required!)
let all_fillings: Vec<Vec<filling::Model>> = cakes.load_many(Filling, db).await?;
```

## v2 — Entity Loader (requires `#[sea_orm::model]`)
```rust
let super_cake = cake::Entity::load()
    .with(fruit::Entity)
    .with(filling::Entity)     // M-N: no junction entity needed in v2
    .one(db)
    .await?
    .unwrap();
// Returns cake::ModelEx { id, name, fruit: Option<fruit::ModelEx>, fillings: Vec<filling::ModelEx> }

// With unique key filter
let bob = user::Entity::load()
    .filter_by_email("bob@example.com")
    .with(profile::Entity)
    .one(db)
    .await?;
```

## Offset pagination
```rust
let mut pages = Cake::find()
    .order_by_asc(cake::Column::Id)
    .paginate(db, 50);

while let Some(batch) = pages.fetch_and_next().await? {
    // batch: Vec<cake::Model>
}
```

## Cursor pagination
```rust
// Single column
let mut cursor = Cake::find().cursor_by(cake::Column::Id);
cursor.after(1).before(100);
let first_10 = cursor.first(10).all(db).await?;
let last_10  = cursor.last(10).all(db).await?;   // returned in ASC order

// Composite cursor
let rows = cake_filling::Entity::find()
    .cursor_by((cake_filling::Column::CakeId, cake_filling::Column::FillingId))
    .after((0, 1))
    .first(3)
    .all(db)
    .await?;
```

## Custom join projection (`FromQueryResult`)

When a query joins tables and you only need a subset of columns from multiple entities,
define a projection struct with `#[derive(sea_orm::FromQueryResult)]`:

```rust
use sea_orm::{FromQueryResult, JoinType, QuerySelect, RelationTrait};

#[derive(Debug, sea_orm::FromQueryResult)]
struct OrderSummaryRow {
    order_id:    i64,
    customer_id: i64,
    store_name:  String, // from a joined table
}

let rows: Vec<OrderSummaryRow> = order::Entity::find()
    .join(JoinType::InnerJoin, order::Relation::Store.def())
    .select_only()
    .column_as(order::Column::Id,         "order_id")
    .column_as(order::Column::CustomerId,  "customer_id")
    .column_as(store::Column::Name,        "store_name")
    .into_model::<OrderSummaryRow>()
    .all(&db)
    .await?;

let domain_summaries: Vec<OrderSummary> = rows.into_iter().map(|r| r.into()).collect();
```

> Use `column_as(Column, "alias")` when the projection field name differs from the column name
> or when disambiguating columns from multiple tables.

---

## Partial model (select subset of columns)
```rust
#[derive(DerivePartialModel)]
#[sea_orm(entity = "cake::Entity")]
struct CakeSummary {
    name: String,
    #[sea_orm(nested)]
    fruit: Option<fruit::Model>,
}

let rows: Vec<CakeSummary> = Cake::find()
    .left_join(fruit::Entity)
    .into_partial_model()
    .all(db)
    .await?;
```
