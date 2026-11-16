---
name: sea-orm
description: >
  SeaORM 2.x (open-source, Postgres/MySQL/MariaDB/SQLite) and SeaORM-X (commercial MSSQL) patterns for Rust backends.
  Use when: adding SeaORM to a project, writing migrations, generating entities, performing CRUD
  (insert, select, update, delete), paginating results, loading relations, handling on-conflict,
  transactions, mapping DbErr to domain errors, using the v2 `#[sea_orm::model]` macro, COLUMN constants,
  find_by_*, Entity Loader, nested ActiveModel / builder, entity-first schema sync, or MSSQL
  OUTPUT/IDENTITY_INSERT/savepoint features.
argument-hint: 'Describe the task, e.g. "set up migrations for MSSQL", "insert many with returning", "cursor pagination", "entity with unique column"'
---

# SeaORM 2.x — Skill Index

> Covers **sea-orm 2.0.x (stable)** and **SeaORM-X** (MSSQL, commercial, closed beta, still pinned to `2.0.0-rc`).
> Verified against crates.io `sea-orm 2.0.4`. Requires Rust edition 2024, MSRV **1.94** (async crate).
> Load only the reference file(s) relevant to the task. Do **not** load all files at once.

## Reference files

| Topic | File |
|-------|------|
| `Cargo.toml` dependencies, feature flags, workspace setup | `reference/01-cargo-setup.md` |
| `Database::connect`, `ConnectOptions`, pool, ping, schema search path | `reference/02-connection.md` |
| Migrations — CLI, file layout, `MigratorTrait`, DDL helpers, raw SQL, seed data | `reference/03-migrations.md` |
| Entity structure — dense vs compact, relation fields, attributes, CLI codegen, type map | `reference/04-entity-structure.md` |
| CRUD Insert — insert one/many, `save`, builder / nested insert, returning, on-conflict | `reference/05-crud-insert.md` |
| CRUD Select — find, filter, COLUMN constants, relations, Entity Loader, pagination, partial model | `reference/06-crud-select.md` |
| CRUD Update & Delete — ActiveValue states, update/delete one/many, returning, cascade | `reference/07-crud-update-delete.md` |
| MSSQL-specific (SeaORM-X) — savepoints, IDENTITY_INSERT, OUTPUT, schema rewrite, tuple IN | `reference/08-mssql-features.md` |
| v1 → v2 breaking changes | `reference/09-v1-v2-migration.md` |
| Repository boundary patterns — mappers, infra→domain error mapping | `reference/10-repository-patterns.md` |
| Transactions — closure, manual, nested (savepoints), isolation level | `reference/11-transactions.md` |
| Error handling — `DbErr`, `sql_err()` / `SqlErr`, driver error codes | `reference/12-error-handling.md` |
| Entity-first workflow — `schema-sync`, schema registry, `sync` / `apply` | `reference/13-entity-first.md` |
| Testing — `MockDatabase`, SQLite in-memory, testcontainers | `reference/14-testing.md` |

## Quick decision guide

- **Starting a new project?** → load `01-cargo-setup.md` + `02-connection.md` + `03-migrations.md` (or `13-entity-first.md` for entity-first)
- **Defining entities?** → load `04-entity-structure.md`
- **Writing queries?** → load the relevant CRUD file(s): `05`, `06`, or `07`
- **Multi-statement writes?** → load `11-transactions.md`
- **Implementing repositories?** → load `10-repository-patterns.md` + `12-error-handling.md` (+ relevant CRUD file)
- **Using MSSQL / SeaORM-X?** → always also load `08-mssql-features.md`
- **Writing tests?** → load `14-testing.md`
- **Upgrading from v1?** → load `09-v1-v2-migration.md`

## Key v2 concepts (quick reference)

- Use `#[sea_orm::model]` (dense format) to unlock `COLUMN` constants, auto `find_by_*`/`delete_by_*` (and `filter_by_*` on the Entity Loader), `ModelEx`, Entity Loader and nested ActiveModel (`ActiveModel::builder()`). `#[sea_orm::compact_model]` also gets COLUMN / `find_by_*` / ModelEx / loader, but keeps relations in a classic `Relation` enum (no relation fields). A plain `#[derive(DeriveEntityModel)]` without either attribute gets none of these.
- Relations are typed fields on the dense model: `HasOne<E>`, `HasMany<E>`, `BelongsTo<E>`.
- `exec_with_returning` works on INSERT, UPDATE, DELETE for Postgres and SQLite (MariaDB with `mariadb-use-returning`). MSSQL (SeaORM-X) maps INSERT/DELETE to `OUTPUT INSERTED.*` / `OUTPUT DELETED.*`.
- `insert_many` returns `InsertManyResult` (`last_insert_id: Option<_>`); empty input is `Ok`, not an error.
- A conflict that inserts nothing returns `DbErr::RecordNotInserted`; call `.try_insert()` (or `on_conflict_do_nothing*()`) to get `TryInsertResult::Conflicted` instead.
- Classify DB errors with `err.sql_err()` → `SqlErr::UniqueConstraintViolation` / `ForeignKeyConstraintViolation` — do not string-match messages.
- M-N `load_many` / Entity Loader no longer require a junction entity in v2.
- MSSQL `IDENTITY_INSERT` is handled automatically by SeaORM-X when a PK column is `Set(...)`.
- `currentSchema` in the MSSQL connection string auto-prefixes all queries — no explicit schema annotations needed.
