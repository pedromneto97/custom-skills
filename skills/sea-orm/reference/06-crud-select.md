# SeaORM 2.x — CRUD: Select

> Example schema: `user` / `post` / `profile` / `tag` / `user_tag` / `order` / `customer` are the
> dense entities in `04-entity-structure.md`. `cake` / `fruit` / `filling` are the official docs'
> bakery sample, also declared with `#[sea_orm::model]`.

## Find by primary key
```rust
let cake: Option<cake::Model> = Cake::find_by_id(1).one(db).await?;
// Composite PK:
let row: Option<cake_filling::Model> = CakeFilling::find_by_id((6, 8)).one(db).await?;
```

## Filter + order
```rust
// #[sea_orm::model] / #[sea_orm::compact_model] — prefer COLUMN
let results: Vec<cake::Model> = Cake::find()
    .filter(cake::COLUMN.name.contains("chocolate"))
    .order_by_asc(cake::COLUMN.name)
    .all(db)
    .await?;

// Plain #[derive(DeriveEntityModel)] (v1 style) — Column enum only
let results = Cake::find()
    .filter(cake::Column::Name.contains("chocolate"))
    .order_by_asc(cake::Column::Name)
    .all(db)
    .await?;
```

## v2 — Strongly-typed COLUMN constants (requires `#[sea_orm::model]` or `#[sea_orm::compact_model]`)
```rust
// Old (Column enum — no compile-time type enforcement)
cake::Column::Name.contains("choc")

// New (type-aware, compile-time checked)
cake::COLUMN.name.contains("choc")          // StringColumn methods only
user::COLUMN.id.between(1, 100)             // NumericColumn methods only
order::COLUMN.created_at.gt(some_date)      // DateTimeLikeColumn methods only
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
let bob: user::Model = user::Entity::find_by_email("bob@example.com").require_one(db).await?;
let posts: Vec<post::Model> = bob.find_related(post::Entity).all(db).await?;
```

## Eager loading (JOIN)
```rust
// N-1 / 1-1
let pairs: Vec<(post::Model, Option<user::Model>)> =
    post::Entity::find().find_also_related(user::Entity).all(db).await?;

// 1-N and M-N (rows consolidated per left entity)
let with_posts: Vec<(user::Model, Vec<post::Model>)> =
    user::Entity::find().find_with_related(post::Entity).all(db).await?;
let with_tags: Vec<(user::Model, Vec<tag::Model>)> =
    user::Entity::find().find_with_related(tag::Entity).all(db).await?;   // via user_tag
```

## LoaderTrait (batched, avoids N+1)
```rust
let users: Vec<user::Model> = user::Entity::find().all(db).await?;

// 1-1 / N-1 — relation must be has_one or belongs_to (load_one on has_many errors at runtime)
let profiles: Vec<Option<profile::Model>> = users.load_one(profile::Entity, db).await?;

// 1-N
let posts: Vec<Vec<post::Model>> = users.load_many(post::Entity, db).await?;

// M-N (v2: no junction entity argument required)
let tags: Vec<Vec<tag::Model>> = users.load_many(tag::Entity, db).await?;
```

## v2 — Entity Loader (`#[sea_orm::model]` or `#[sea_orm::compact_model]`)
Returns nested `ModelEx` values whose relation fields are filled. Uses a JOIN for 1-1 and a
batched data loader for 1-N / M-N — no N+1. `filter_by_id` / `filter_by_<unique>` are available
on the loader. Relation fields need the dense `#[sea_orm::model]` format.

```rust
let bob: user::ModelEx = user::Entity::load()
    .filter_by_email("bob@example.com")
    .with(profile::Entity)     // has_one  → JOIN
    .with(post::Entity)        // has_many → data loader
    .with(tag::Entity)         // M-N via user_tag, no junction entity needed
    .one(db)
    .await?
    .unwrap();
// bob.profile → HasOne::Loaded(Some(profile::ModelEx { .. }))
// bob.posts   → HasMany::Loaded(vec![post::ModelEx { .. }])
// bob.tags    → HasMany::Loaded(vec![tag::ModelEx { .. }])
// relations not requested stay HasOne::Unloaded / HasMany::Unloaded

// Nested: user → posts → comments (post::Model must declare its own has_many comments field)
let smart = user::Entity::load()
    .filter_by_id(42)
    .with((post::Entity, comment::Entity))
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
let rows = user_tag::Entity::find()
    .cursor_by((user_tag::Column::UserId, user_tag::Column::TagId))
    .after((0, 1))
    .first(3)
    .all(db)
    .await?;
```

## Custom join projection (`FromQueryResult`)

When a query joins tables and you only need a subset of columns from multiple entities,
define a projection struct with `#[derive(FromQueryResult)]`:

```rust
use sea_orm::{FromQueryResult, QuerySelect};

#[derive(Debug, FromQueryResult)]
struct OrderSummaryRow {
    order_id:      i32,     // match the entity's Rust types (order.id is i32)
    status:        String,
    customer_name: String,  // from the joined table
}

let rows: Vec<OrderSummaryRow> = order::Entity::find()
    .inner_join(customer::Entity)          // join condition from the belongs_to relation
    .select_only()
    .column_as(order::Column::Id,      "order_id")
    .column(order::Column::Status)
    .column_as(customer::Column::Name, "customer_name")
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
// Select every column of the model (a missing column fails at runtime)
let user: Option<user::Model> = user::Entity::find()
    .from_raw_sql(raw_sql!(
        Postgres,
        r#"SELECT * FROM "user" WHERE "name" LIKE {name} AND "id" IN ({..ids})"#
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
