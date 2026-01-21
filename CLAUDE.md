# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Vaultwarden is an alternative server implementation of the Bitwarden Client API written in Rust. It's designed for self-hosted deployment and is compatible with official Bitwarden clients.

**Key Technology Stack:**
- **Language:** Rust (pinned to version 1.92.0)
- **Web Framework:** Rocket 0.5.1 (async)
- **ORM:** Diesel 2.3.5 with support for SQLite, MySQL/MariaDB, and PostgreSQL
- **Database Migrations:** Diesel migrations (managed per database type)
- **Memory Allocator:** Optional MiMalloc for Alpine builds
- **Container Support:** Multi-architecture Docker/Podman images (amd64, arm64, armv7, armv6)

## Build Commands

### Building with Cargo

**IMPORTANT:** You MUST specify at least one database backend feature when building:

```bash
# Build with SQLite (most common for development)
cargo build --features sqlite

# Build with all databases and additional features
cargo build --features sqlite,mysql,postgresql,enable_mimalloc,s3

# Release build
cargo build --release --features sqlite

# Development profile
cargo build --profile dev --features sqlite

# CI profile (used in GitHub Actions)
cargo build --profile ci --features sqlite,mysql,postgresql
```

### Running Tests

```bash
# Run tests with SQLite
cargo test --profile ci --features sqlite

# Run tests with all databases
cargo test --profile ci --features sqlite,mysql,postgresql

# Run tests with all features
cargo test --profile ci --features sqlite,mysql,postgresql,enable_mimalloc,s3
```

### Linting and Formatting

```bash
# Run clippy (fails on warnings in CI)
cargo clippy --features sqlite,mysql,postgresql,enable_mimalloc,s3

# Format check
cargo fmt --all -- --check

# Auto-format code
cargo fmt --all
```

### Running the Server

```bash
# Run with SQLite (development)
cargo run --features sqlite

# Set environment variables via .env file
cp .env.template .env
# Edit .env with your configuration
cargo run --features sqlite
```

## Container Building

The project uses `docker buildx bake` for multi-architecture builds. See `docker/README.md` for details.

### Quick Container Builds

```bash
# Build default Debian image for host architecture
docker/bake.sh

# Build specific architecture
docker/bake.sh debian-arm64
docker/bake.sh alpine-amd64

# Build with specific database features
DB="sqlite,enable_mimalloc" docker/podman-bake.sh alpine-arm64

# Multi-arch build (requires setup - see docker/README.md)
CONTAINER_REGISTRIES="localhost:5000/vaultwarden/server" docker/bake.sh alpine-multi
```

## Database Management

### Migrations

Migrations are stored per database type in `migrations/{sqlite,mysql,postgresql}/`.

```bash
# Install diesel CLI (if needed)
cargo install diesel_cli --no-default-features --features sqlite

# Run migrations (diesel.toml points to src/db/schema.rs)
diesel migration run --database-url "path/to/db.sqlite3"

# Create new migration
diesel migration generate migration_name
```

**Note:** The application automatically runs migrations on startup and includes automatic migration for U2F to WebAuthn and credential to passkey conversions.

## Architecture

### Core Structure

```
src/
├── main.rs           # Application entry point, server initialization
├── config.rs         # Configuration management (env vars + config.json)
├── error.rs          # Error handling and custom error types
├── auth.rs           # Authentication/authorization logic, JWT handling
├── crypto.rs         # Cryptographic utilities
├── db/               # Database layer
│   ├── mod.rs        # Connection pooling, DbConn abstraction
│   ├── schema.rs     # Diesel schema (auto-generated, don't edit manually)
│   └── models/       # Database models (User, Cipher, Organization, etc.)
├── api/              # API endpoints (Rocket routes)
│   ├── core/         # Core Bitwarden API
│   │   ├── accounts.rs       # Account management
│   │   ├── ciphers.rs        # Password vault items
│   │   ├── organizations.rs  # Organization/sharing features
│   │   ├── sends.rs          # Bitwarden Send (temporary sharing)
│   │   ├── folders.rs        # Folder management
│   │   ├── emergency_access.rs
│   │   └── two_factor/       # 2FA implementations
│   ├── identity.rs   # Authentication endpoints
│   ├── admin.rs      # Admin panel API
│   ├── icons.rs      # Website icon fetching
│   ├── notifications.rs # WebSocket notifications
│   ├── push.rs       # Push notifications
│   └── web.rs        # Web vault serving
├── mail.rs           # Email sending (lettre)
├── sso.rs            # SSO/OIDC implementation
├── util.rs           # Utility functions
└── static/           # Static resources (templates, images, scripts)
```

### Key Architecture Patterns

**Multi-Database Support:**
- Uses `DbConnInner` enum with Diesel's `MultiConnection` derive
- Feature flags (`sqlite`, `mysql`, `postgresql`) control which backends are compiled
- Connection pooling via `r2d2` with custom `DbConnManager`
- Async database operations use `run_blocking` to wrap sync Diesel calls

**API Type Aliases:**
- `ApiResult<T>` - Standard result type for API endpoints
- `JsonResult` - API result returning JSON
- `EmptyResult` - API result with no return value

**Configuration System:**
- Environment variables override config.json
- Admin panel can modify config.json at runtime
- Uses `LazyLock` for static configuration initialization
- Sensitive values (passwords, tokens) are masked in config output

**WebSocket Notifications:**
- Real-time updates via `WS_USERS` and `WS_ANONYMOUS_SUBSCRIPTIONS`
- MessagePack protocol (rmpv) for efficient binary messaging
- Uses `dashmap` for concurrent connection management

**Authentication:**
- JWT-based auth with RSA key pairs (auto-generated on first run)
- Support for personal API keys
- Multi-factor authentication (TOTP, WebAuthn, Duo, YubiKey, Email)
- SSO via OpenID Connect

## Testing

### Unit Tests

Unit tests are inline with modules using `#[cfg(test)]`:
```rust
#[cfg(test)]
mod tests {
    use super::*;
    // tests here
}
```

Found in: `src/util.rs`, `src/db/models/organization.rs`, `src/api/core/two_factor/email.rs`, `src/api/admin.rs`

### Integration Tests

Playwright-based integration tests in `playwright/` directory.

```bash
# Run all integration tests (uses Docker)
cd playwright
DOCKER_BUILDKIT=1 docker compose --profile playwright --env-file test.env run Playwright

# Run tests for specific database
DOCKER_BUILDKIT=1 docker compose --profile playwright --env-file test.env run Playwright test --project=sqlite
DOCKER_BUILDKIT=1 docker compose --profile playwright --env-file test.env run Playwright test --project=postgres

# Run specific test file
DOCKER_BUILDKIT=1 docker compose --profile playwright --env-file test.env run Playwright test --project=sqlite tests/login.spec.ts

# Run with UI (requires local setup)
cd playwright
npm ci --ignore-scripts
npx playwright install firefox
npx playwright test --ui
```

## Development Workflow

### Local Development Setup

1. Install Rust toolchain (version managed by `rust-toolchain.toml`)
2. Copy `.env.template` to `.env` and configure
3. Build with desired database backend
4. Run migrations (auto-run on startup)
5. Access web vault at `http://localhost:8000` (or configured port)

**Important:** The web vault requires HTTPS or `http://localhost` for Web Crypto API.

### Database Feature Selection

The build system REQUIRES at least one database feature. The default `Cargo.toml` has all features commented out to force explicit selection:

```toml
[features]
default = [
    # "sqlite",
    # "mysql",
    # "postgresql",
]
```

This prevents accidental builds without database support.

### Web Vault

Vaultwarden uses a modified web vault from `dani-garcia/bw_web_builds`. The vault is:
- Bundled in container images
- Can be overridden via `WEB_VAULT_FOLDER` env var
- Served by the `web.rs` module

## Code Quality Standards

The project enforces strict linting rules (see `Cargo.toml` workspace lints):

**Forbidden:**
- `unsafe_code` - No unsafe code allowed
- `non_ascii_idents` - ASCII-only identifiers

**Key Denies:**
- Unused code, imports, lifetimes
- Deprecated APIs
- Future incompatibilities
- Clippy warnings for common issues

**Clippy Warnings:**
- `dbg_macro` - Debug macros should not be committed
- `todo` - TODO markers should be resolved

## Common Patterns

### Database Queries

```rust
use crate::db::{models::User, DbConn};

async fn get_user(uuid: &str, conn: &DbConn) -> Result<User, Error> {
    conn.run(move |c| {
        User::find_by_uuid(uuid, c)
    }).await
}
```

### API Endpoints

```rust
use rocket::serde::json::Json;
use crate::api::{JsonResult, EmptyResult};

#[get("/endpoint")]
async fn endpoint(conn: DbConn) -> JsonResult {
    Ok(Json(json!({ "status": "ok" })))
}
```

### Error Handling

```rust
use crate::error::Error;

// Use err! macro for quick error creation
err!("Something went wrong");

// Use MapResult for Option unwrapping
some_option.or_err("Not found")?;
```

## Important Notes

- **Do not edit `src/db/schema.rs` manually** - it's generated by Diesel migrations
- **Always specify database features** when building or testing
- **Migrations are database-specific** - changes may need to be replicated across sqlite/mysql/postgresql
- **Configuration precedence:** Admin UI config.json > Environment variables > Defaults
- **Container images** are the primary deployment method - binary builds are for development
- **Web vault must be served over HTTPS** or localhost for browser crypto APIs to work
