# custom-skills

A collection of GitHub Copilot skills providing domain-specific knowledge for Rust backend development.

## Skills

### `rust-hexagonal`
Hexagonal architecture (Ports & Adapters) for Rust backends. Covers structuring the domain layer, defining ports as traits, implementing adapters (HTTP, DB, gRPC), wiring dependency injection, and separating business logic from infrastructure.

**Reference files:** folder structure, domain, bootstrap, inbound/outbound adapters, HTTP practices, migrations, testing.

### `http-actix-axum`
HTTP best practices for **actix-web 4** and **axum 0.7+**. Covers REST resource naming, HTTP status codes, RFC 9457 Problem Details error responses, OWASP security headers, CORS configuration, response compression, and API versioning.

**Reference files:** compression, CORS, problem-details, security-headers.

### `sea-orm`
SeaORM v2 (Postgres/MySQL/SQLite) and SeaORM-X (commercial MSSQL) patterns. Covers project setup, database connections, migrations, entity generation, CRUD operations, pagination, relations, on-conflict handling, and MSSQL-specific features.

**Reference files:** cargo setup, connection, migrations, entity structure, CRUD insert/select/update/delete, MSSQL features, v1→v2 migration guide.
