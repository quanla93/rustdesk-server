# RustDesk Server with API Support

[![build](https://github.com/quanla93/rustdesk-server/actions/workflows/build.yaml/badge.svg?branch=forapi)](https://github.com/quanla93/rustdesk-server/actions/workflows/build.yaml)

This branch keeps the open-source RustDesk server and adds API/web-console integration for deployments that need account login validation.

## Branch features

- Fixes connection timeouts when a client is signed in with an API account.
- Adds API support to the S6 image. The API project is open source at <https://github.com/lejianwen/rustdesk-api>.
- Adds optional login enforcement. `MUST_LOGIN` defaults to `N`; set it to `Y` to require a valid login before a client can connect.
- Adds `RUSTDESK_API_JWT_KEY`. When this is set, the server validates client tokens with JWT.
- Supports client WebSocket connections for RustDesk client versions `>= 1.4.1`.

## Container images

This fork publishes images to GitHub Container Registry (GHCR):

- S6 all-in-one image: `ghcr.io/quanla93/rustdesk-server-s6`
- Classic image: `ghcr.io/quanla93/rustdesk-server`

Recommended tags:

| Tag | Description |
| --- | --- |
| `latest` | Latest build from this branch |
| `2` | Major-version tag for the current v2 release series |
| `x.y.z` | Exact release version tag |

## Quick start: S6 all-in-one image

The S6 image runs `hbbs`, `hbbr`, and the API service in a single container.

```yaml
networks:
  rustdesk-net:
    external: false

services:
  rustdesk:
    ports:
      - 21114:21114
      - 21115:21115
      - 21116:21116
      - 21116:21116/udp
      - 21117:21117
      - 21118:21118
      - 21119:21119
    image: ghcr.io/quanla93/rustdesk-server-s6:latest
    environment:
      - RELAY=<relay_server[:port]>
      - ENCRYPTED_ONLY=1
      - MUST_LOGIN=N
      - TZ=Asia/Shanghai
      - RUSTDESK_API_RUSTDESK_ID_SERVER=<id_server[:21116]>
      - RUSTDESK_API_RUSTDESK_RELAY_SERVER=<relay_server[:21117]>
      - RUSTDESK_API_RUSTDESK_API_SERVER=http://<api_server[:21114]>
      - RUSTDESK_API_KEY_FILE=/data/id_ed25519.pub
      - RUSTDESK_API_JWT_KEY=xxxxxx # JWT key
    volumes:
      - /data/rustdesk/server:/data
      - /data/rustdesk/api:/app/data # mount the API database
    networks:
      - rustdesk-net
    restart: unless-stopped
```

## API screenshots

![API dashboard](./readme/api.png)

![Command example](./readme/command_simple.png)

For more API details, see [RustDesk API](https://github.com/lejianwen/rustdesk-api).

## Authentik / OIDC login setup

This fork can use Authentik through the bundled RustDesk API service. Authentik is configured as a generic OIDC provider in the RustDesk API web console, while `hbbs` validates client login tokens with `RUSTDESK_API_JWT_KEY` when `MUST_LOGIN=Y`.

Recommended public endpoints:

| Endpoint | Purpose |
| --- | --- |
| `rustdesk.example.com` | RustDesk ID/relay host used by clients |
| `https://rustdesk-api.example.com` | RustDesk API web console and client API server |
| `https://auth.example.com` | Authentik |

### 1. Configure the RustDesk server

Set these environment variables for the S6 all-in-one container:

```env
MUST_LOGIN=Y
RUSTDESK_API_JWT_KEY=<long-random-secret>
RUSTDESK_API_RUSTDESK_ID_SERVER=rustdesk.example.com:21116
RUSTDESK_API_RUSTDESK_RELAY_SERVER=rustdesk.example.com:21117
RUSTDESK_API_RUSTDESK_API_SERVER=https://rustdesk-api.example.com
```

Generate a JWT key with:

```bash
openssl rand -base64 48
```

The same `RUSTDESK_API_JWT_KEY` must be used by the API service and `hbbs`; otherwise clients can log in but the ID server rejects their tokens.

### 2. Create an Authentik OAuth2/OIDC provider

In Authentik, create an **OAuth2/OpenID Provider** for RustDesk API:

| Field | Value |
| --- | --- |
| Client type | `Confidential` |
| Redirect URI | `https://rustdesk-api.example.com/api/oidc/callback` |
| Scopes | `openid profile email` |
| Issuer | Usually `https://auth.example.com/application/o/<provider-slug>/` |

Create an Authentik application that uses this provider and assign the users or groups that may log in to RustDesk.

### 3. Configure OIDC in RustDesk API

Open the RustDesk API web console at `https://rustdesk-api.example.com`, then add an OIDC login provider:

| Field | Value |
| --- | --- |
| Type | `OIDC` |
| Name | `Authentik` |
| Issuer | Authentik issuer URL, for example `https://auth.example.com/application/o/rustdesk-api/` |
| Client ID | Client ID from Authentik |
| Client Secret | Client secret from Authentik |
| Scopes | `openid profile email` |

The OIDC userinfo or ID token must include these claims:

- `sub`
- `email`
- `preferred_username`

If Authentik does not return `preferred_username`, add an Authentik scope/property mapping that returns the current user's username as `preferred_username`.

### 4. Configure RustDesk clients

In the RustDesk client network settings, set:

| Client setting | Value |
| --- | --- |
| ID server | `rustdesk.example.com` |
| Relay server | `rustdesk.example.com` or `rustdesk.example.com:21117` |
| API server | `https://rustdesk-api.example.com` |
| Key | The server public key, if encrypted/key enforcement is enabled |

With `MUST_LOGIN=Y`, clients must log in through the API server before they can connect.

### Troubleshooting

- `redirect_uri mismatch`: the Authentik redirect URI must exactly match `https://rustdesk-api.example.com/api/oidc/callback`.
- Login succeeds but clients cannot connect: verify `MUST_LOGIN=Y` and that `RUSTDESK_API_JWT_KEY` is identical for API token signing and `hbbs` validation.
- OIDC claim errors: make sure Authentik returns `sub`, `email`, and `preferred_username`.
- Client cannot open the login page: ensure `RUSTDESK_API_RUSTDESK_API_SERVER` is the public URL reachable from the client, not only an internal Docker URL.

---

<p align="center">
  <a href="#how-to-build-manually">Build</a> •
  <a href="#classic-image">Classic Docker</a> •
  <a href="#s6-overlay-image">S6-overlay</a> •
  <a href="#how-to-create-a-keypair">Keypair</a> •
  <a href="#deb-packages">Debian</a> •
  <a href="#environment-variables">Environment variables</a>
</p>

# RustDesk Server Program

[**Download releases**](https://github.com/quanla93/rustdesk-server/releases)

[**Self-hosting manual**](https://rustdesk.com/docs/en/self-host/)

[**FAQ**](https://github.com/rustdesk/rustdesk/wiki/FAQ)

Self-host your own RustDesk server. It is free and open source.

## How to build manually

```bash
cargo build --release
```

After the build completes, three executables are generated in `target/release`:

- `hbbs` - RustDesk ID/rendezvous server
- `hbbr` - RustDesk relay server
- `rustdesk-utils` - RustDesk command-line utilities

Updated binaries are available on the [Releases](https://github.com/quanla93/rustdesk-server/releases) page.

If you need extra features, [RustDesk Server Pro](https://rustdesk.com/pricing.html) may be a better fit.

If you want to develop your own server, [rustdesk-server-demo](https://github.com/rustdesk/rustdesk-server-demo) is usually a simpler starting point than this repository.

## Docker images

Images are automatically built when a release is published. This fork publishes to [GitHub Container Registry](https://ghcr.io).

Two image types are available:

1. **Classic image**: contains only the main server binaries, `hbbs` and `hbbr`.
2. **S6-overlay image**: runs `hbbs`, `hbbr`, and the API service in one container.

## Classic image

The classic image contains the two main RustDesk server binaries: `hbbs` and `hbbr`.

Supported architectures:

- `amd64`
- `arm64v8`
- `armv7`

Recommended tags:

| Version | Image tag |
| --- | --- |
| Latest | `ghcr.io/quanla93/rustdesk-server:latest` |
| Major version | `ghcr.io/quanla93/rustdesk-server:2` |

Start the classic images directly with `docker run`:

```bash
docker run --name hbbs --net=host -v "$PWD/data:/root" -d ghcr.io/quanla93/rustdesk-server:latest hbbs -r <relay-server-ip[:port]>
docker run --name hbbr --net=host -v "$PWD/data:/root" -d ghcr.io/quanla93/rustdesk-server:latest hbbr
```

You can also run without `--net=host`, but P2P direct connections will not work.

For systems using SELinux, replace `/root` with `/root:z` so the containers can run correctly. Alternatively, disable SELinux container separation by adding `--security-opt label=disable`.

```bash
docker run --name hbbs -p 21115:21115 -p 21116:21116 -p 21116:21116/udp -p 21118:21118 -v "$PWD/data:/root" -d ghcr.io/quanla93/rustdesk-server:latest hbbs -r <relay-server-ip[:port]>
docker run --name hbbr -p 21117:21117 -p 21119:21119 -v "$PWD/data:/root" -d ghcr.io/quanla93/rustdesk-server:latest hbbr
```

The `relay-server-ip` value is the IP address or DNS name of the server running `hbbr`. Use the optional `port` value if `hbbr` does not listen on the default port `21117`.

You can also use Docker Compose:

```yaml
version: '3'

networks:
  rustdesk-net:
    external: false

services:
  hbbs:
    container_name: hbbs
    ports:
      - 21115:21115
      - 21116:21116
      - 21116:21116/udp
      - 21118:21118
    image: ghcr.io/quanla93/rustdesk-server:latest
    command: hbbs -r rustdesk.example.com:21117
    volumes:
      - ./data:/root
    networks:
      - rustdesk-net
    depends_on:
      - hbbr
    restart: unless-stopped

  hbbr:
    container_name: hbbr
    ports:
      - 21117:21117
      - 21119:21119
    image: ghcr.io/quanla93/rustdesk-server:latest
    command: hbbr
    volumes:
      - ./data:/root
    networks:
      - rustdesk-net
    restart: unless-stopped
```

Set the `hbbs` command to point at your relay server, usually the host listening on port `21117`. Adjust the volume paths if needed.

Docker Compose example credit goes to @lukebarone and @QuiGonLeong.

## S6-overlay image

The S6-overlay image adds [S6-overlay](https://github.com/just-containers/s6-overlay) and runs the RustDesk server binaries plus the API service in a single container. You do not need to run separate `hbbs` and `hbbr` containers.

Supported architectures:

- `amd64`
- `i386`
- `arm64v8`
- `armv7`

Recommended tags:

| Version | Image tag |
| --- | --- |
| Latest | `ghcr.io/quanla93/rustdesk-server-s6:latest` |
| Major version | `ghcr.io/quanla93/rustdesk-server-s6:2` |

Start the S6 image directly with `docker run`:

```bash
docker run --name rustdesk-server \
  --net=host \
  -e "RELAY=rustdeskrelay.example.com" \
  -e "ENCRYPTED_ONLY=1" \
  -v "$PWD/data:/data" -d ghcr.io/quanla93/rustdesk-server-s6:latest
```

You can also run without `--net=host`, but P2P direct connections will not work.

```bash
docker run --name rustdesk-server \
  -p 21114:21114 -p 21115:21115 -p 21116:21116 -p 21116:21116/udp \
  -p 21117:21117 -p 21118:21118 -p 21119:21119 \
  -e "RELAY=rustdeskrelay.example.com" \
  -e "ENCRYPTED_ONLY=1" \
  -v "$PWD/data:/data" -d ghcr.io/quanla93/rustdesk-server-s6:latest
```

Or use Docker Compose:

```yaml
version: '3'

services:
  rustdesk-server:
    container_name: rustdesk-server
    ports:
      - 21114:21114
      - 21115:21115
      - 21116:21116
      - 21116:21116/udp
      - 21117:21117
      - 21118:21118
      - 21119:21119
    image: ghcr.io/quanla93/rustdesk-server-s6:latest
    environment:
      - "RELAY=rustdesk.example.com:21117"
      - "ENCRYPTED_ONLY=1"
    volumes:
      - ./data:/data
    restart: unless-stopped
```

For this container image, you can use these environment variables in addition to the variables listed in [Environment variables](#environment-variables):

| Variable | Optional | Description |
| --- | --- | --- |
| `RELAY` | No | IP address or DNS name of the host running this container |
| `ENCRYPTED_ONLY` | Yes | If set to `1`, unencrypted connections are rejected |
| `KEY_PUB` | Yes | Public key from the key pair |
| `KEY_PRIV` | Yes | Private key from the key pair |
| `MUST_LOGIN` | Yes | If set to `Y`, clients must log in before connecting |
| `RUSTDESK_API_JWT_KEY` | Yes | JWT key used to validate client tokens |

### Secret management in S6-overlay images

You can keep the key pair in a Docker volume, but best practice is to avoid writing private keys directly to the filesystem. This image supports environment variables and Docker secrets for key management.

On container startup, the image checks whether the key pair exists at `/data/id_ed25519.pub` and `/data/id_ed25519`. If either key is missing, it is recreated from environment variables or Docker secrets. The image then validates the key pair. If the public and private keys do not match, the container stops. If you provide no keys, `hbbs` generates a key pair in the default location.

#### Use environment variables to store the key pair

```bash
docker run --name rustdesk-server \
  --net=host \
  -e "RELAY=rustdeskrelay.example.com" \
  -e "ENCRYPTED_ONLY=1" \
  -e "DB_URL=/db/db_v2.sqlite3" \
  -e "KEY_PRIV=FR2j78IxfwJNR+HjLluQ2Nh7eEryEeIZCwiQDPVe+PaITKyShphHAsPLn7So0OqRs92nGvSRdFJnE2MSyrKTIQ==" \
  -e "KEY_PUB=iEyskoaYRwLDy5+0qNDqkbPdpxr0kXRSZxNjEsqykyE=" \
  -v "$PWD/db:/db" -d ghcr.io/quanla93/rustdesk-server-s6:latest
```

```yaml
version: '3'

services:
  rustdesk-server:
    container_name: rustdesk-server
    ports:
      - 21114:21114
      - 21115:21115
      - 21116:21116
      - 21116:21116/udp
      - 21117:21117
      - 21118:21118
      - 21119:21119
    image: ghcr.io/quanla93/rustdesk-server-s6:latest
    environment:
      - "RELAY=rustdesk.example.com:21117"
      - "ENCRYPTED_ONLY=1"
      - "DB_URL=/db/db_v2.sqlite3"
      - "KEY_PRIV=FR2j78IxfwJNR+HjLluQ2Nh7eEryEeIZCwiQDPVe+PaITKyShphHAsPLn7So0OqRs92nGvSRdFJnE2MSyrKTIQ=="
      - "KEY_PUB=iEyskoaYRwLDy5+0qNDqkbPdpxr0kXRSZxNjEsqykyE="
    volumes:
      - ./db:/db
    restart: unless-stopped
```

#### Use Docker secrets to store the key pair

Docker secrets are useful when using Docker Compose or Docker Swarm.

```bash
cat secrets/id_ed25519.pub | docker secret create key_pub -
cat secrets/id_ed25519 | docker secret create key_priv -
docker service create --name rustdesk-server \
  --secret key_priv --secret key_pub \
  --net=host \
  -e "RELAY=rustdeskrelay.example.com" \
  -e "ENCRYPTED_ONLY=1" \
  -e "DB_URL=/db/db_v2.sqlite3" \
  --mount "type=bind,source=$PWD/db,destination=/db" \
  ghcr.io/quanla93/rustdesk-server-s6:latest
```

```yaml
version: '3'

services:
  rustdesk-server:
    container_name: rustdesk-server
    ports:
      - 21114:21114
      - 21115:21115
      - 21116:21116
      - 21116:21116/udp
      - 21117:21117
      - 21118:21118
      - 21119:21119
    image: ghcr.io/quanla93/rustdesk-server-s6:latest
    environment:
      - "RELAY=rustdesk.example.com:21117"
      - "ENCRYPTED_ONLY=1"
      - "DB_URL=/db/db_v2.sqlite3"
    volumes:
      - ./db:/db
    restart: unless-stopped
    secrets:
      - key_pub
      - key_priv

secrets:
  key_pub:
    file: secrets/id_ed25519.pub
  key_priv:
    file: secrets/id_ed25519
```

## How to create a keypair

Encryption requires a key pair. You can provide one manually, but you need a tool to generate it.

Generate a key pair with:

```bash
/usr/bin/rustdesk-utils genkeypair
```

If `rustdesk-utils` is not installed on your system, run the same command with Docker:

```bash
docker run --rm --entrypoint /usr/bin/rustdesk-utils ghcr.io/quanla93/rustdesk-server-s6:latest genkeypair
```

Example output:

```text
Public Key:  8BLLhtzUBU/XKAH4mep3p+IX4DSApe7qbAwNH9nv4yA=
Secret Key:  egAVd44u33ZEUIDTtksGcHeVeAwywarEdHmf99KM5ajwEsuG3NQFT9coAfiZ6nen4hfgNICl7upsDA0f2e/jIA==
```

## .deb packages

Separate `.deb` packages are available for each binary on the [Releases](https://github.com/quanla93/rustdesk-server/releases) page.

Supported distributions:

- Ubuntu 24.04 LTS
- Ubuntu 22.04 LTS
- Ubuntu 20.04 LTS
- Ubuntu 18.04 LTS
- Debian 12 bookworm
- Debian 11 bullseye
- Debian 10 buster

## Environment variables

`hbbs` and `hbbr` can be configured with environment variables. Set them directly or through an `.env` file.

| Variable | Binary | Description |
| --- | --- | --- |
| `ALWAYS_USE_RELAY` | `hbbs` | If set to `Y`, direct peer-to-peer connections are disabled |
| `DB_URL` | `hbbs` | Database file path |
| `DOWNGRADE_START_CHECK` | `hbbr` | Delay before downgrade checks start, in seconds |
| `DOWNGRADE_THRESHOLD` | `hbbr` | Downgrade-check threshold, in bit/ms |
| `KEY` | `hbbs`/`hbbr` | Forces the use of a specific key. If set to `_`, any key is allowed |
| `LIMIT_SPEED` | `hbbr` | Speed limit, in Mb/s |
| `PORT` | `hbbs`/`hbbr` | Listening port (`21116` for `hbbs`, `21117` for `hbbr`) |
| `RELAY_SERVERS` | `hbbs` | IP addresses or DNS names of hosts running `hbbr`, separated by commas |
| `RUST_LOG` | all | Debug level: `error`, `warn`, `info`, `debug`, or `trace` |
| `SINGLE_BANDWIDTH` | `hbbr` | Maximum bandwidth for one connection, in Mb/s |
| `TOTAL_BANDWIDTH` | `hbbr` | Maximum total bandwidth, in Mb/s |
| `MUST_LOGIN` | `hbbs` | If set to `Y`, clients must log in before connecting |
| `RUSTDESK_API_JWT_KEY` | `hbbs` | JWT secret used to validate API login tokens |
