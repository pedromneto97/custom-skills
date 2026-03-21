# SeaORM v1 → v2 Migration Guide

| Area | v1 | v2 |
|------|----|----|
| Entity macro | `#[derive(DeriveEntityModel)]` | Add `#[sea_orm::model]` for dense format (enables COLUMN, find_by_*, ModelEx) |
| Type-safe columns | `Column` enum (no type enforcement) | `COLUMN` constant with per-type methods (compile-time checked) |
| `find_by_*` / `delete_by_*` | Manual implementation | Auto-generated for `#[sea_orm(unique)]` columns |
| Entity Loader | Not available | `Entity::load().with(RelatedEntity).one(db)` |
| `ModelEx` (nested models) | Not available | Returned by Entity Loader; fields typed per relation |
| `delete_by_id` return type | `DeleteMany` | `ValidatedDeleteOne` |
| `exec_with_returning` on `delete_by_id` | `Vec<Model>` | `Option<Model>` |
| M-N `load_many` | Requires junction entity | Junction entity **not required** |
| `eq_any` | Not available | `= ANY(...)` shorthand (Postgres only) |
| `--entity-format` CLI flag | `compact` / `expanded` | Added `dense` (recommended for v2) |
| Schema-first only | Yes | Entity-first also supported via `feature = "schema-sync"` |
