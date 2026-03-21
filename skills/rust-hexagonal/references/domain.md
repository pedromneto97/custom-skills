# Domain Layer Reference

The `domain` crate must list **no infrastructure crates** in its `Cargo.toml`. Pure Rust only.

---

## Step 1: Domain Model

```rust
// domain/src/model/order.rs
#[derive(Debug, Clone, PartialEq)]
pub struct Order {
    pub id: OrderId,
    pub customer_id: CustomerId,
    pub status: OrderStatus,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct OrderId(pub uuid::Uuid);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct CustomerId(pub uuid::Uuid);

#[derive(Debug, Clone, PartialEq)]
pub enum OrderStatus {
    Pending,
    Confirmed,
    Shipped,
}
```

Rules:
- Entities are plain Rust structs/enums — no derives from infrastructure crates
- Value objects (e.g. `OrderId`) wrap primitives in newtypes for type safety
- Add test fixtures inside `#[cfg(test)]` blocks:

```rust
#[cfg(test)]
impl Order {
    pub fn fixture() -> Self {
        Order {
            id: OrderId(uuid::Uuid::new_v4()),
            customer_id: CustomerId(uuid::Uuid::new_v4()),
            status: OrderStatus::Pending,
        }
    }
}
```

---

## Step 2: Domain Errors

```rust
// domain/src/error.rs
#[derive(Debug, thiserror::Error)]
pub enum DomainError {
    #[error("Order not found: {0}")]
    OrderNotFound(uuid::Uuid),
    #[error("Invalid status transition")]
    InvalidStatusTransition,
    #[error("Infrastructure error: {0}")]
    Infrastructure(String),
}
```

---

## Step 3: Outbound Port (Repository Trait)

The domain *defines* what it needs from infrastructure. Adapters *implement* this.

No `#[async_trait]` required with Rust ≥ 1.75 and static dispatch.

```rust
// domain/src/ports/outbound.rs
use crate::{error::DomainError, model::order::Order};
use uuid::Uuid;

// Requires mockall ≥ 0.12 for impl Trait + async fn support
#[cfg_attr(test, mockall::automock)]
pub trait OrderRepository {
    async fn find_by_id(&self, id: Uuid) -> Result<Order, DomainError>;
    async fn save(&self, order: &Order) -> Result<(), DomainError>;
    async fn find_all(&self) -> Result<impl Iterator<Item = Order>, DomainError>;
}
```

> **If you need `dyn OrderRepository`** (runtime polymorphism), add `#[async_trait::async_trait]`
> and replace `impl Iterator<Item = Order>` with `Box<dyn Iterator<Item = Order>>`.
> The trait then becomes object-safe at the cost of heap allocation.

---

## Step 4: Use Case Functions

Each use case is a plain exported async function in `domain/src/use_cases/`. The repository
is received as a generic parameter — static dispatch, no struct, no trait impl needed.

```rust
// domain/src/use_cases/orders.rs
use crate::{
    error::DomainError,
    model::order::{Order, OrderStatus},
    ports::outbound::OrderRepository,
};
use uuid::Uuid;

pub async fn get_order<R: OrderRepository>(repo: &R, id: Uuid) -> Result<Order, DomainError> {
    repo.find_by_id(id).await
}

pub async fn confirm_order<R: OrderRepository>(repo: &R, id: Uuid) -> Result<Order, DomainError> {
    let mut order = repo.find_by_id(id).await?;
    if order.status != OrderStatus::Pending {
        return Err(DomainError::InvalidStatusTransition);
    }
    order.status = OrderStatus::Confirmed;
    repo.save(&order).await?;
    Ok(order)
}

pub async fn list_orders<R: OrderRepository>(
    repo: &R,
) -> Result<impl Iterator<Item = Order>, DomainError> {
    repo.find_all().await
}
```

---

## `domain/src/lib.rs`

```rust
pub mod error;
pub mod model;
pub mod ports; // only contains outbound now
pub mod use_cases;
```

## `domain/Cargo.toml`

```toml
[package]
name = "domain"
version = "0.1.0"
edition = "2021"

[dependencies]
thiserror.workspace = true
uuid.workspace = true
tokio.workspace = true

[dev-dependencies]
mockall = "0.12"
tokio = { workspace = true, features = ["macros", "rt"] }
```
