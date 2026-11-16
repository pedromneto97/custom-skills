# SeaORM 2.x — Entity-First Workflow (schema sync)

Write entities by hand; SeaORM creates / evolves the schema from them. Alternative to writing
migrations first and generating entities with `sea-orm-cli`.

> `SchemaBuilder::sync` is marked **experimental / exempt from semver** — pin the exact
> `sea-orm` version if startup depends on it, and re-test sync after upgrades.

## Cargo features
```toml
sea-orm = { version = "2.0", features = [
  "sqlx-postgres", "runtime-tokio-rustls", "macros",
  "schema-sync",       # SchemaBuilder::sync (apply works without it)
  "entity-registry",   # auto-discover entities by module path (uses the `inventory` crate)
] }
```

## Sync at startup — all entities in a module
The registry keeps entities whose **module path starts with the given prefix**. The prefix must
begin with the crate name as Rust sees it (`my-app` → `my_app`). Entities that don't match are
skipped silently (debug log only) — `sync` still returns `Ok` and creates nothing.

```rust
// Creates missing tables (FK-topological order), columns, unique indexes, foreign keys
db.get_schema_registry("my_app::entity::*").sync(&db).await?;

// Entities in the *current* crate, from any module: take the crate name from module_path!()
db.get_schema_registry(module_path!().split("::").next().unwrap()).sync(&db).await?;

// Workspace with a separate `entity` crate (01-cargo-setup.md layout): prefix is that crate
db.get_schema_registry("entity::*").sync(&db).await?;
```

> Don't build the prefix with `concat!(module_path!(), "::entity::*")` — called from
> `my_app::startup` it becomes `my_app::startup::entity::*` and matches nothing.

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
| New `belongs_to` relation | foreign key created — **except SQLite** when the table already exists (no FK; relation queries still work, integrity not enforced) |
| Field renamed with `#[sea_orm(renamed_from = "old")]` | column renamed (instead of add + keep old) |
| Field / entity removed | **nothing** — no destructive drops of tables, columns or FKs |
| Column type changed | not migrated — write a migration |

New non-null columns on tables with existing rows need a default:
`#[sea_orm(default_value = "...")]` or `#[sea_orm(default_expr = "Expr::current_timestamp()")]`.

> SQLite: adding a column with `default_expr = "Expr::current_timestamp()"` does **not** work
> (SQLite rejects non-constant defaults in `ALTER TABLE ADD COLUMN`). Use a constant
> `default_value`, make the column `Option<T>`, or write a migration.

## When to use
- Prototypes, tests (`sqlite::memory:` + `apply`), internal tools, small services.
- For production systems with data, prefer versioned migrations (`03-migrations.md`) — reviewable,
  reversible, and they handle drops / type changes / data backfills. A hybrid works well: call
  `get_schema_builder().register(..).apply(db)` inside a migration to create new tables from entities.
