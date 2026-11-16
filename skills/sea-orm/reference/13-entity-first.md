# SeaORM 2.x — Entity-First Workflow (schema sync)

Write entities by hand; SeaORM creates / evolves the schema from them. Alternative to writing
migrations first and generating entities with `sea-orm-cli`.

## Cargo features
```toml
sea-orm = { version = "2.0", features = [
  "sqlx-postgres", "runtime-tokio-rustls", "macros",
  "schema-sync",       # SchemaBuilder: apply / sync
  "entity-registry",   # auto-discover entities by module path (uses the `inventory` crate)
] }
```

## Sync at startup — all entities in a module
```rust
// Creates missing tables (FK-topological order), columns, unique indexes, foreign keys
db.get_schema_registry("my_crate::entity::*").sync(&db).await?;

// Inside the crate itself, build the path from module_path!()
db.get_schema_registry(concat!(module_path!(), "::entity::*")).sync(&db).await?;
```

## Explicit entity list (no `entity-registry` needed)
```rust
db.get_schema_builder()
    .register(user::Entity)
    .register(post::Entity)
    .register(post_tag::Entity)
    .sync(&db)     // diff against the live schema and add what is missing
    .await?;

// .apply(&db) — create everything; for initializing an empty DB (e.g. tests, inside a migration)
```

## What `sync` does / does not do
| Change in entity | Result |
|------------------|--------|
| New entity | `CREATE TABLE` (+ FKs, unique indexes) |
| New field | `ALTER TABLE ... ADD COLUMN` |
| `#[sea_orm(unique)]` added / removed | unique index created / dropped |
| New `belongs_to` relation | foreign key created |
| Field renamed with `#[sea_orm(renamed_from = "old")]` | column renamed (instead of add + keep old) |
| Field / entity removed | **nothing** — no destructive drops of tables, columns or FKs |
| Column type changed | not migrated — write a migration |

New non-null columns on tables with existing rows need a default:
`#[sea_orm(default_value = "...")]` or `#[sea_orm(default_expr = "Expr::current_timestamp()")]`.

## When to use
- Prototypes, tests (`sqlite::memory:` + `apply`), internal tools, small services.
- For production systems with data, prefer versioned migrations (`03-migrations.md`) — reviewable,
  reversible, and they handle drops / type changes / data backfills. A hybrid works well: call
  `get_schema_builder().register(..).apply(db)` inside a migration to create new tables from entities.
