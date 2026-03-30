# SeaORM v2 — Entity Structure

## MySQL / Postgres ENUM column

Map a database-native ENUM type to a Rust enum using `DeriveActiveEnum`:

```rust
use sea_orm::{DeriveActiveEnum, EnumIter};

#[derive(Debug, Clone, PartialEq, Eq, EnumIter, DeriveActiveEnum, Copy)]
#[sea_orm(rs_type = "String", db_type = "Enum", enum_name = "role")]
pub enum Role {
    #[sea_orm(string_value = "owner")]    Owner,
    #[sea_orm(string_value = "admin")]    Admin,
    #[sea_orm(string_value = "operator")] Operator,
}
```

- `rs_type = "String"` — the Rust representation used in serialization  
- `db_type = "Enum"` — maps to a native MySQL/Postgres ENUM column  
- `enum_name` must match the enum type name in the DB schema  
- For Postgres, use `db_type = "PgEnum"`

---

## Custom join projection (`FromQueryResult`)

When a query joins multiple tables and you need a partial row, define a projection struct:

```rust
use sea_orm::FromQueryResult;

#[derive(Debug, sea_orm::FromQueryResult)]
pub struct OrderSummaryRow {
    pub order_id:   i64,
    pub store_name: String, // from joined table
    pub role:       Role,   // projected column mapped via DeriveActiveEnum
}
```

Used with `.into_model::<OrderSummaryRow>()` in a select query — see `06-crud-select.md`.

---

## Dense model with inline relation (`has_many`)

In the v2 `#[sea_orm::model]` dense format, declare relations directly on the struct:

```rust
#[sea_orm::model]
#[derive(Clone, Debug, PartialEq, Eq, DeriveEntityModel)]
#[sea_orm(table_name = "stores")]
pub struct Model {
    #[sea_orm(primary_key)]
    pub id: i64,
    pub name: String,
    #[sea_orm(has_many)]
    pub orders: HasMany<super::order::Entity>, // declare here; no separate Relation enum needed
}
impl ActiveModelBehavior for ActiveModel {}
```

---

## v2 — Dense format (recommended — enables COLUMN, find_by_*, ModelEx)
```rust
use sea_orm::entity::prelude::*;

#[sea_orm::model]
#[derive(Clone, Debug, PartialEq, DeriveEntityModel, Eq)]
#[sea_orm(table_name = "order")]
pub struct Model {
    #[sea_orm(primary_key)]
    pub id: i32,
    pub customer_id: i32,
    pub status: String,
    #[sea_orm(unique)]            // generates find_by_reference / delete_by_reference
    pub reference: String,
    pub created_at: DateTime,
}
```

## Classic — Compact format
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
```rust
#[sea_orm(primary_key, auto_increment = false)]
pub cake_id: i32,
#[sea_orm(primary_key, auto_increment = false)]
pub filling_id: i32,
```

## Key entity attributes
```rust
#[sea_orm(primary_key)]
#[sea_orm(primary_key, auto_increment = false)]   // manual / composite PK
#[sea_orm(unique)]                                // unique — generates find_by_*/delete_by_*
#[sea_orm(unique_key = "pair")]                   // composite unique key
#[sea_orm(column_name = "DBColumnName")]
#[sea_orm(column_type = "Text")]
#[sea_orm(nullable)]
#[sea_orm(indexed)]
#[sea_orm(schema_name = "SalesLT")]               // MSSQL/Postgres schema prefix
```

## Generate entities via CLI
```shell
# Postgres
sea-orm-cli generate entity -u postgres://user:pass@localhost/db -o entity/src

# MSSQL with non-default schema
sea-orm-cli generate entity \
  --database-url "mssql://sa:YourStrong()Passw0rd@localhost/MyDb" \
  --database-schema "SalesLT" \
  --entity-format dense \        # dense (v2 recommended) | compact | expanded
  -o entity/src \
  -l \                           # emit lib.rs instead of mod.rs
  --with-serde both \            # none | serialize | deserialize | both
  --date-time-crate chrono \     # chrono | time
  --max-connections 1
```

## Rust → DB type mapping (MSSQL)
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
