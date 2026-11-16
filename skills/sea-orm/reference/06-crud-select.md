# SeaORM 2.x — CRUD: Select

## Find by primary key
```rust
let cake: Option<cake::Model> = Cake::find_by_id(1).one(db).await?;
// Composite PK:
let row: Option<cake_filling::Model> = CakeFilling::find_by_id((6, 8)).one(db).await?;
```

## Filter + order
```rust
// Dense model (#[sea_orm::model]) — prefer COLUMN
let results: Vec<cake::Model> = Cake::find()
    .filter(cake::COLUMN.name.contains("chocolate"))
    .order_by_asc(cake::COLUMN.name)
    .all(db)
    .await?;

// Compact model / v1 style — Column enum
let results = Cake::find()
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
// Also reachable as cake::Entity::COLUMN
```

## v2 — Find by unique key (auto-generated for `#[sea_orm(unique)]`)
```rust
let user: Option<user::Model> = user::Entity::find_by_email("bob@example.com").one(db).await?;

// find_by_* returns a Select — chain more filters
let active_bob = user::Entity::find_by_email("bob@example.com")
    .filter(user::COLUMN.active.eq(true))
    .one(db)
    .await?;

// Composite unique key with #[sea_orm(unique_key = "pair")]
let row = composite::Entity::find_by_pair((1, 2)).one(db).await?;
```

## Exactly one row — `require_one` (2.0.2+)
```rust
// Returns Model directly; DbErr::RecordNotFound when no row matches
let user: user::Model = user::Entity::find_by_email("bob@example.com").require_one(db).await?;
```

## Postgres `= ANY(...)`
```rust
// WHERE id = ANY($1) — one bind param instead of IN (...) list
let rows = Cake::find().filter(cake::Column::Id.eq_any(vec![1, 2, 3])).all(db).await?;
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

## LoaderTrait (batched, avoids N+1)
```rust
let cakes: Vec<cake::Model> = Cake::find().all(db).await?;

// 1-to-1
let fruit: Vec<Option<fruit::Model>> = cakes.load_one(Fruit, db).await?;

// 1-to-many
let all_fruits: Vec<Vec<fruit::Model>> = cakes.load_many(Fruit, db).await?;

// many-to-many (v2: no junction entity argument required)
let all_fillings: Vec<Vec<filling::Model>> = cakes.load_many(Filling, db).await?;
```

## v2 — Entity Loader (requires `#[sea_orm::model]`)
Returns nested `ModelEx` values whose relation fields are filled. Uses a JOIN for 1-1 and a
batched data loader for 1-N / M-N — no N+1. `filter_by_<unique>` / `filter_by_id` are available
on the loader.

```rust
let super_cake: cake::ModelEx = cake::Entity::load()
    .filter_by_id(1)
    .with(fruit::Entity)
    .with(filling::Entity)     // M-N: no junction entity needed
    .one(db)
    .await?
    .unwrap();
// super_cake.fruit    → HasOne::Loaded(Some(fruit::ModelEx { .. }))
// super_cake.fillings → HasMany::Loaded(vec![filling::ModelEx { .. }])
// unrequested relations stay HasOne::Unloaded / HasMany::Unloaded
// (field names come from the relation fields on cake::Model)

// Nested: user → posts → tags
let smart_user = user::Entity::load()
    .filter_by_id(42)
    .with(profile::Entity)
    .with((post::Entity, tag::Entity))   // nested: post::Model must declare its tags relation
    .one(db)
    .await?;

// With unique key filter
let bob = user::Entity::load()
    .filter_by_email("bob@example.com")
    .with(profile::Entity)
    .with(post::Entity)
    .one(db)
    .await?;

// Many
let users: Vec<user::ModelEx> = user::Entity::load().with(post::Entity).all(db).await?;
```

## Offset pagination
```rust
let mut pages = Cake::find()
    .order_by_asc(cake::Column::Id)
    .paginate(db, 50);

let total = pages.num_items().await?;
let page_3: Vec<cake::Model> = pages.fetch_page(2).await?;   // 0-based

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
define a projection struct with `#[derive(FromQueryResult)]`:

```rust
use sea_orm::{FromQueryResult, JoinType, QuerySelect, RelationTrait};

#[derive(Debug, FromQueryResult)]
struct OrderSummaryRow {
    order_id:    i64,
    customer_id: i64,
    store_name:  String, // from a joined table
    role:        Role,   // DeriveActiveEnum columns map directly
}

let rows: Vec<OrderSummaryRow> = order::Entity::find()
    .join(JoinType::InnerJoin, order::Relation::Store.def())
    .select_only()
    .column_as(order::Column::Id,         "order_id")
    .column_as(order::Column::CustomerId,  "customer_id")
    .column_as(store::Column::Name,        "store_name")
    .column_as(order::Column::Role,        "role")
    .into_model::<OrderSummaryRow>()
    .all(db)
    .await?;

let domain_summaries: Vec<OrderSummary> = rows.into_iter().map(Into::into).collect();
```

> Use `column_as(Column, "alias")` when the projection field name differs from the column name
> or when disambiguating columns from multiple tables.

---

## Select all columns except some (2.0.4+)
```rust
let posts: Vec<PostPreview> = post::Entity::find()
    .select_except([post::Column::Body])
    .into_model::<PostPreview>()
    .all(db)
    .await?;
```

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

## Raw SQL (v2)
Prefer the `raw_sql!` macro — `format!`-like interpolation, but values are bound parameters
(no SQL injection). Supports field access and `..` list expansion:

```rust
use sea_orm::raw_sql;

let ids = [2, 3, 4];
let name = "Bob";
let user: Option<user::Model> = user::Entity::find()
    .from_raw_sql(raw_sql!(
        Postgres,
        r#"SELECT "id", "name" FROM "user" WHERE "name" LIKE {name} AND "id" IN ({..ids})"#
    ))
    .one(db)
    .await?;
```

Manual `Statement`:

```rust
use sea_orm::{FromQueryResult, Statement};

// SeaQuery statements → execute / query_one / query_all
// Raw Statement      → execute_raw / query_one_raw / query_all_raw
let rows = db.query_all_raw(Statement::from_sql_and_values(
    db.get_database_backend(),
    r#"SELECT "id", "name" FROM "cake" WHERE "id" > $1"#,
    [10.into()],
)).await?;

// Or straight into a model
let cakes: Vec<cake::Model> = cake::Entity::find()
    .from_raw_sql(Statement::from_sql_and_values(
        db.get_database_backend(),
        r#"SELECT * FROM "cake" WHERE "id" > $1"#,
        [10.into()],
    ))
    .all(db)
    .await?;
```
