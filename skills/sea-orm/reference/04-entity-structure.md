# SeaORM v2 — Entity Structure

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
