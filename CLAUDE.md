# RustDesk Server (Fork) Context & Instructions

## Architecture & Fork Relationship

- Primary repo: `quanla93/rustdesk-server` (branch `forapi`)
- Upstream server: `rustdesk/rustdesk-server`
- Companion API fork: `quanla93/rustdesk-api` (branch `master`)
  - Image published to GitHub Container Registry: `ghcr.io/quanla93/rustdesk-api:latest`
  - S6 all-in-one server Dockerfile consumes the API image via:
    `FROM ghcr.io/quanla93/rustdesk-api:latest` in `docker/Dockerfile`

## Key Ports & Services

- `21114`: RustDesk API web console, API server, and web client (`/_admin/`, `/webclient/`, `/api/`)
- `21116` (TCP/UDP): `hbbs` (ID / Rendezvous server)
- `21117` (TCP): `hbbr` (Relay server)
- `21118` / `21119`: WebRTC / websocket support

## Rules & Conventions

- In Rust code, avoid `unwrap()` or `expect()`, except in unit tests or lock acquisitions where unavoidable.
- In Docker images, always ensure secrets (like `RUSTDESK_API_JWT_KEY`) are passed via environment variables, not hardcoded.
- When working with Authentik/OIDC:
  - Callback URL: `https://<api-domain>/api/oidc/callback`
  - User claims required: `sub`, `email`, `preferred_username`
- Always verify Docker build and tests before publishing new releases.
