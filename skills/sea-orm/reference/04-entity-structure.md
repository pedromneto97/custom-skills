# SeaORM 2.x — Entity Structure

## Dense format (recommended — `#[sea_orm::model]`)

Enables `COLUMN` constants, auto `find_by_*` / `delete_by_*` (+ `filter_by_*` on the loader), `ModelEx`,
Entity Loader and nested ActiveModel (`ActiveModel::builder()`).

```rust
use sea_orm::entity::prelude::*;

#[sea_orm::model]
#[derive(Clone, Debug, PartialEq, Eq, DeriveEntityModel)]
#[sea_orm(table_name = "order")]
pub struct Model {
    #[sea_orm(primary_key)]
    pub id: i32,
    pub customer_id: i32,
    pub status: String,
    #[sea_orm(unique)]            // generates find_by_reference / delete_by_reference
    pub reference: String,
    pub created_at: DateTime,
    #[sea_orm(belongs_to, from = "customer_id", to = "id")]
    pub customer: BelongsTo<super::customer::Entity>,
}

// Required — never delete, even if empty
impl ActiveModelBehavior for ActiveModel {}
```

> `#[sea_orm::compact_model]` generates the same COLUMN / `find_by_*` / `delete_by_*` / ModelEx /
> Entity Loader, but relations stay in a classic `Relation` enum + `Related` impls (no
> `HasOne`/`HasMany`/`BelongsTo` fields). A plain `#[derive(DeriveEntityModel)]` with neither
> attribute (the classic compact format below) gets none of the v2 helpers.
> Prefer `#[sea_orm::model]` for new code.

## Relations in the dense format

Relations are typed fields on `Model` — no separate `Relation` enum / `Related` impl needed.

```rust
#[sea_orm::model]
#[derive(Clone, Debug, PartialEq, Eq, DeriveEntityModel)]
#[sea_orm(table_name = "user")]
pub struct Model {
    #[sea_orm(primary_key)]
    pub id: i32,
    #[sea_orm(unique)]
    pub email: String,

    #[sea_orm(has_one)]
    pub profile: HasOne<super::profile::Entity>,      // 1-1

    #[sea_orm(has_many)]
    pub posts: HasMany<super::post::Entity>,          // 1-N

    #[sea_orm(has_many, via = "user_tag")]
    pub tags: HasMany<super::tag::Entity>,            // M-N through junction table
}
impl ActiveModelBehavior for ActiveModel {}
```

```rust
// post.rs — child side owns the FK
#[sea_orm::model]
#[derive(Clone, Debug, PartialEq, Eq, DeriveEntityModel)]
#[sea_orm(table_name = "post")]
pub struct Model {
    #[sea_orm(primary_key)]
    pub id: i32,
    pub user_id: i32,
    pub title: String,
    #[sea_orm(belongs_to, from = "user_id", to = "id")]
    pub author: BelongsTo<super::user::Entity>,
}
impl ActiveModelBehavior for ActiveModel {}
```

| Relation | Field type | Attribute |
|----------|-----------|-----------|
| 1-1 (parent side) | `HasOne<E>` | `#[sea_orm(has_one)]` |
| 1-N (parent side) | `HasMany<E>` | `#[sea_orm(has_many)]` |
| M-N | `HasMany<E>` | `#[sea_orm(has_many, via = "junction_table")]` |
| N-1 (child side, FK here) | `BelongsTo<E>` | `#[sea_orm(belongs_to, from = "fk_col", to = "pk_col")]` |
| Nullable FK | `BelongsTo<Option<E>>` | same as above |

- `BelongsTo<E>` requires every `from` column to be `NOT NULL`; `BelongsTo<Option<E>>` requires a
  nullable one. A mismatch is a **compile error**.
- `BelongsTo<E>` can't be detached on the active side (orphaning is a compile error);
  `BelongsTo<Option<E>>` can (nulls the FK on save).
- Junction entity for M-N: composite PK + two `BelongsTo` fields (see below).
- `sea-orm-cli` still generates `HasOne<E>` for `belongs_to` fields — that form keeps working;
  switch to `BelongsTo` by hand for the compile-time check.
- Legacy fields (`Option<super::x::Entity>`, `Vec<super::x::Entity>`) are still accepted, but
  prefer the typed wrappers.

```rust
// user_tag.rs — junction table for user ⇄ tag
#[sea_orm::model]
#[derive(Clone, Debug, PartialEq, Eq, DeriveEntityModel)]
#[sea_orm(table_name = "user_tag")]
pub struct Model {
    #[sea_orm(primary_key, auto_increment = false)]
    pub user_id: i32,
    #[sea_orm(primary_key, auto_increment = false)]
    pub tag_id: i32,
    #[sea_orm(belongs_to, from = "user_id", to = "id")]
    pub user: BelongsTo<super::user::Entity>,
    #[sea_orm(belongs_to, from = "tag_id", to = "id")]
    pub tag: BelongsTo<super::tag::Entity>,
}
impl ActiveModelBehavior for ActiveModel {}
```

## Classic — Compact format (v1 style, still supported)
```rust
use sea_orm::entity::prelude::*;

#[derive(Clone, Debug, PartialEq, DeriveEntityModel, Eq)]
#[sea_orm(table_name = "cake")]
pub struct Model {
    #[sea_orm(primary_key)]
    pub id: i32,
    pub name: String,
}

#[derive(Copy, Clone, Debug, EnumIter, DeriveRelation)]
pub enum Relation {
    #[sea_orm(has_many = "super::fruit::Entity")]
    Fruit,
}

impl Related<super::fruit::Entity> for Entity {
    fn to() -> RelationDef { Relation::Fruit.def() }
}

impl ActiveModelBehavior for ActiveModel {}
```

## Composite primary key
Up to 12 columns.
```rust
#[sea_orm(primary_key, auto_increment = false)]
pub cake_id: i32,
#[sea_orm(primary_key, auto_increment = false)]
pub filling_id: i32,
```

## Key entity attributes
```rust
// Struct-level
#[sea_orm(table_name = "order")]
#[sea_orm(schema_name = "sales")]                 // Postgres / MSSQL schema prefix
#[sea_orm(rename_all = "camelCase")]              // column naming for all fields

// Field-level
#[sea_orm(primary_key)]
#[sea_orm(primary_key, auto_increment = false)]   // manual / composite / UUID PK
#[sea_orm(unique)]                                // generates find_by_* / delete_by_*
#[sea_orm(unique_key = "pair")]                   // composite unique key → find_by_pair((a, b))
#[sea_orm(indexed)]
#[sea_orm(nullable)]                              // or just use Option<T>
#[sea_orm(column_name = "DBColumnName")]
#[sea_orm(column_type = "Text")]
#[sea_orm(column_type = "Decimal(Some((16, 4)))")]
#[sea_orm(default_value = "active")]              // literal default (schema-sync / codegen)
#[sea_orm(default_expr = "Expr::current_timestamp()")]
#[sea_orm(column_type = r#"custom("citext")"#, select_as = "text", save_as = "citext")] // cast on read / write
#[sea_orm(ignore)]                                // not a DB column
#[sea_orm(renamed_from = "old_name")]             // schema-sync renames instead of add+drop
```

## Database ENUM column (`DeriveActiveEnum`)

```rust
use sea_orm::entity::prelude::*;

#[derive(Debug, Clone, Copy, PartialEq, Eq, EnumIter, DeriveActiveEnum)]
#[sea_orm(rs_type = "String", db_type = "Enum", enum_name = "role")]
pub enum Role {
    #[sea_orm(string_value = "owner")]    Owner,
    #[sea_orm(string_value = "admin")]    Admin,
    #[sea_orm(string_value = "operator")] Operator,
}
```

- `db_type = "Enum"` — native ENUM on MySQL and Postgres (Postgres needs the type created in a
  migration, see `03-migrations.md`); stored as text on SQLite.
- `enum_name` must match the enum type name in the DB schema.
- For a plain string/int column use `db_type = "String(StringLen::None)"` / `db_type = "Integer"`
  with `rs_type = "String"` / `"i32"` and `num_value = N`.

## Newtype columns (`DeriveValueType`)
```rust
#[derive(Clone, Debug, PartialEq, Eq, DeriveValueType)]
pub struct UserId(pub i32);   // v2: usable as a primary key
```

## Generate entities via CLI
```shell
# Postgres
sea-orm-cli generate entity -u postgres://user:pass@localhost/db -o entity/src --entity-format dense

# Common flags
#   --entity-format   dense (v2 recommended) | compact | expanded
#   -l                emit lib.rs instead of mod.rs
#   --with-serde      none | serialize | deserialize | both
#   --date-time-crate chrono | time
sea-orm-cli generate entity \
  --database-url "postgres://user:pass@localhost/db" \
  --entity-format dense \
  -o entity/src \
  -l \
  --with-serde both \
  --date-time-crate chrono \
  --max-connections 1

# SQLite: INTEGER now maps to i64 in v2; force i32 with
sea-orm-cli generate entity -u sqlite://db.sqlite -o entity/src --big-integer-type=i32

# MSSQL (SeaORM-X) with non-default schema
sea-orm-cli generate entity \
  --database-url "mssql://sa:YourStrong()Passw0rd@localhost/MyDb" \
  --database-schema "SalesLT" \
  --entity-format dense \
  -o entity/src
```

## Custom projections
For join projections with `FromQueryResult` / `DerivePartialModel`, see `06-crud-select.md`.

## Rust → DB type mapping (MSSQL, SeaORM-X)
| Rust type | MSSQL |
|-----------|-------|
| `String` | `nvarchar(255)` |
| `i8`/`u8` | `tinyint` |
| `i16`/`u16` | `smallint` |
| `i32`/`u32` | `int` |
| `i64`/`u64` | `bigint` |
| `f32` | `real` |
| `f64` | `float` |
| `bool` | `bit` |
| `Vec<u8>` | `binary` |
| `chrono::NaiveDate` | `date` |
| `chrono::NaiveDateTime` | `datetime` |
| `chrono::DateTime<Utc>` | `datetime2` |
| `chrono::DateTime<FixedOffset>` | `datetimeoffset` |
| `uuid::Uuid` | `uniqueidentifier` |
| `serde_json::Value` | `nvarchar(max)` |
| `rust_decimal::Decimal` | `decimal` |
