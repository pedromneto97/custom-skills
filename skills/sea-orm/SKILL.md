---
name: sea-orm
description: >
  SeaORM v2 (open-source, Postgres/MySQL/SQLite) and SeaORM-X (commercial MSSQL) patterns for Rust backends.
  Use when: adding SeaORM to a project, writing migrations, generating entities, performing CRUD
  (insert, select, update, delete), paginating results, loading relations, handling on-conflict,
  using the v2 `#[sea_orm::model]` macro, COLUMN constants, find_by_*, Entity Loader, or MSSQL
  OUTPUT/IDENTITY_INSERT/savepoint features.
argument-hint: 'Describe the task, e.g. "set up migrations for MSSQL", "insert many with returning", "cursor pagination", "entity with unique column"'
---

# SeaORM v2 — Skill Index

> Covers **sea-orm 2.0.0-rc** (open-source) and **SeaORM-X** (MSSQL commercial).
> Load only the reference file(s) relevant to the task. Do **not** load all files at once.

## Reference files

| Topic | File |
|-------|------|
| `Cargo.toml` dependencies, feature flags, workspace setup | `reference/01-cargo-setup.md` |
| `Database::connect`, `ConnectOptions`, pool, ping | `reference/02-connection.md` |
| Migrations — CLI, file layout, `MigratorTrait`, DDL helpers, raw SQL, seed data | `reference/03-migrations.md` |
| Entity structure — dense vs compact, attributes, CLI codegen, Rust↔DB type map | `reference/04-entity-structure.md` |
| CRUD Insert — insert one/many, `exec_with_returning`, on-conflict, IDENTITY_INSERT | `reference/05-crud-insert.md` |
| CRUD Select — find, filter, COLUMN constants, relations, pagination, partial model | `reference/06-crud-select.md` |
| CRUD Update & Delete — update one/many, delete one/many, `exec_with_returning` | `reference/07-crud-update-delete.md` |
| MSSQL-specific — savepoints, IDENTITY_INSERT, OUTPUT, schema rewrite, tuple IN | `reference/08-mssql-features.md` |
| v1 → v2 breaking changes | `reference/09-v1-v2-migration.md` |

## Quick decision guide

- **Starting a new project?** → load `01-cargo-setup.md` + `02-connection.md` + `03-migrations.md`
- **Defining entities?** → load `04-entity-structure.md`
- **Writing queries?** → load the relevant CRUD file(s): `05`, `06`, or `07`
- **Using MSSQL / SeaORM-X?** → always also load `08-mssql-features.md`
- **Upgrading from v1?** → load `09-v1-v2-migration.md`

## Key v2 concepts (quick reference)

- Use `#[sea_orm::model]` (dense format) to unlock `COLUMN` constants, auto `find_by_*`/`delete_by_*`, `ModelEx`, and Entity Loader.
- `exec_with_returning` is supported on INSERT, UPDATE, and DELETE for Postgres, SQLite, and MSSQL.
- M-N `load_many` no longer requires a junction entity in v2.
- MSSQL `IDENTITY_INSERT` is handled automatically by SeaORM-X when a PK column is `Set(...)`.
- `currentSchema` in the MSSQL connection string auto-prefixes all queries — no explicit schema annotations needed.
