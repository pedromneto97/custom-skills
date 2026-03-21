# Outbound Adapter Reference (SeaORM v2)

Outbound adapters implement repository port traits defined in `domain`. They import `domain`
and infrastructure crates — never `inbound`.

---

## SeaORM CLI: Generate Entities

```bash
cargo install sea-orm-cli
sea-orm-cli generate entity \
  -u postgres://user:pass@localhost/db \
  -o outbound/src/db/entities
```

- Generated files in `outbound/src/db/entities/` — **never edit by hand**, regenerate instead
- Entity types never cross crate boundaries

---

## SeaORM Migrations

See [migrations.md](./migrations.md) for full setup. Summary:
- Separate `migration/` crate at workspace root (`sea-orm-cli migrate init`)
- Only dependency: `sea-orm-migration` — no domain/app imports
- `Migrator::up(&db, None).await?` runs on startup in `app/src/main.rs`

---

## SeaORM Entity

Define the entity in `outbound/src/db/entities/`. These types must **never leave** the `outbound` crate.

```rust
// outbound/src/db/entities/order.rs
use sea_orm::entity::prelude::*;

#[derive(Clone, Debug, PartialEq, DeriveEntityModel)]
#[sea_orm(table_name = "orders")]
pub struct Model {
    #[sea_orm(primary_key, auto_increment = false)]
    pub id: Uuid,
    pub customer_id: Uuid,
    pub status: String,
}

#[derive(Copy, Clone, Debug, EnumIter, DeriveRelation)]
pub enum Relation {}

impl ActiveModelBehavior for ActiveModel {}
```

---

## Mappers

Translate immediately between SeaORM `Model` and domain types. Nothing leaks.

```rust
// outbound/src/db/mappers.rs
use domain::model::order::{CustomerId, Order, OrderId, OrderStatus};
use super::entities::order;

pub fn to_domain(m: order::Model) -> Order {
    Order {
        id: OrderId(m.id),
        customer_id: CustomerId(m.customer_id),
        status: match m.status.as_str() {
            "Confirmed" => OrderStatus::Confirmed,
            "Shipped"   => OrderStatus::Shipped,
            _           => OrderStatus::Pending,
        },
    }
}

pub fn to_active_model(o: &Order) -> order::ActiveModel {
    use sea_orm::ActiveValue::Set;
    order::ActiveModel {
        id:          Set(o.id.0),
        customer_id: Set(o.customer_id.0),
        status:      Set(format!("{:?}", o.status)),
    }
}
```

---

## Repository Implementation

```rust
// outbound/src/db/order_repository.rs
use sea_orm::{ActiveModelTrait, DatabaseConnection, EntityTrait};
use domain::{
    error::DomainError,
    model::order::Order,
    ports::outbound::OrderRepository,
};
use uuid::Uuid;
use super::{entities::order::Entity as OrderEntity, mappers};

pub struct SeaOrmOrderRepository {
    db: DatabaseConnection,
}

impl SeaOrmOrderRepository {
    pub fn new(db: DatabaseConnection) -> Self {
        Self { db }
    }
}

impl OrderRepository for SeaOrmOrderRepository {
    async fn find_by_id(&self, id: Uuid) -> Result<Order, DomainError> {
        OrderEntity::find_by_id(id)
            .one(&self.db)
            .await
            .map_err(|e| DomainError::Infrastructure(e.to_string()))?
            .map(mappers::to_domain)
            .ok_or(DomainError::OrderNotFound(id))
    }

    // Returns impl Iterator — callers decide whether to collect
    async fn find_all(&self) -> Result<impl Iterator<Item = Order>, DomainError> {
        let rows = OrderEntity::find()
            .all(&self.db)
            .await
            .map_err(|e| DomainError::Infrastructure(e.to_string()))?;
        Ok(rows.into_iter().map(mappers::to_domain))
    }

    async fn save(&self, order: &Order) -> Result<(), DomainError> {
        mappers::to_active_model(order)
            .insert(&self.db)
            .await
            .map_err(|e| DomainError::Infrastructure(e.to_string()))?;
        Ok(())
    }
}
```

---

## Module Layout

```
outbound/src/
├── lib.rs                      # pub mod db;
└── db/
    ├── mod.rs                  # pub mod order_repository; mod entities; mod mappers;
    ├── order_repository.rs
    ├── entities/
    │   ├── mod.rs
    │   └── order.rs
    └── mappers.rs
```

## `outbound/Cargo.toml`

```toml
[package]
name = "outbound"
version = "0.1.0"
edition = "2021"

[dependencies]
domain.workspace = true
uuid.workspace   = true
tokio.workspace  = true
# layer-specific — not in workspace
sea-orm = { version = "2", features = ["sqlx-postgres", "runtime-tokio-rustls", "macros"] }
```
