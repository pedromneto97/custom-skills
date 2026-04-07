# custom-skills

GitHub Copilot skills for Rust backend development, providing production-ready patterns and best practices.

## Skills

| Skill | Description | When |
|-------|-------------|------|
| `rust-hexagonal` | Hexagonal architecture (Ports & Adapters) for Rust backends. Covers workspace structure, domain modeling, port traits, dependency injection, composition root, testing strategies, and multi-bounded-context scaling. **Refs:** folder-structure, domain, bootstrap, inbound, outbound, http-practices, migrations, testing. | New Rust backend project |
| `http-actix-axum` | HTTP best practices for actix-web 4 and axum 0.7+. Covers REST naming conventions, HTTP status codes, error responses (RFC 9457 Problem Details), OWASP security headers, CORS, response compression, and API versioning. **Refs:** compression, cors, problem-details, security-headers. | Building Rust HTTP endpoints |
| `sea-orm` | SeaORM v2 database patterns for Rust (Postgres, MySQL, SQLite, with SeaORM-X for MSSQL). Covers setup, migrations, entity generation, CRUD operations, pagination, relationship handling, transaction management, and on-conflict/upsert strategies. **Refs:** 01-cargo-setup, 02-connection, 03-migrations, 04-entity-structure, 05-crud-insert, 06-crud-select, 07-crud-update-delete, 08-mssql-features, 09-v1-v2-migration. | SeaORM database work |
| `crypto-best-practices` | Cryptographic operations and security best practices for Rust: password hashing, encryption algorithms, key management, secure authentication patterns, and common pitfalls to avoid. | Authentication & encryption |
| `flutter-data-layer` | Flutter data-layer patterns and templates for repositories, datasources, models, and response mapping. **Refs:** datasource, model, templates. | Flutter app data layer |
| `flutter-domain-layer` | Domain/core layer guidance for Flutter Clean Architecture: entities, value objects, repository interfaces (ports), use cases, and domain exceptions. **Refs:** references, templates. | Flutter app domain layer |
| `flutter-presentation-layer` | Opinionated conventions for the presentation layer in Flutter Clean Architecture: per-page folders, single-responsibility cubits (one cubit per fetch/post/delete), close-to-use widget placement, clean `build()` patterns (extract methods/widgets), and enforced localization for strings. **Refs:** presentation-templates, checklist. | Building Flutter UI and state (pages & cubits) |



Skills are designed to work together—see the hexagonal architecture skill for how `http-actix-axum` and `sea-orm` integrate with the architecture.
