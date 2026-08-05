# Portainer LXC deployment notes

This compose file is prepared for deploying the all-in-one RustDesk server from a Portainer **Repository** stack.

## Target host

- LXC IP: `192.168.101.113`
- Stack/container name: `rustdesk-server`
- Persistent data root on LXC: `/opt/rustdesk-server`

Prepare the data directories on the LXC before deploying:

```bash
sudo mkdir -p /opt/rustdesk-server/data/server
sudo mkdir -p /opt/rustdesk-server/data/api
sudo chown -R 1000:1000 /opt/rustdesk-server
```

The important persistent files are stored under `/opt/rustdesk-server/data/server`, especially `id_ed25519` and `id_ed25519.pub`.

## Portainer Repository stack

Use these Portainer fields:

- Name: `rustdesk-server`
- Build method: `Repository`
- Repository URL: `https://github.com/quanla93/rustdesk-server.git`
- Repository reference: `refs/heads/forapi`
- Compose path: `docker-compose.yml`

## Portainer environment variables

For first LAN deployment, set these variables in Portainer:

```env
RUSTDESK_IMAGE=ghcr.io/quanla93/rustdesk-server-s6:latest
CONTAINER_NAME=rustdesk-server
RUSTDESK_DATA_DIR=/opt/rustdesk-server

RUSTDESK_RELAY=192.168.101.113
RUSTDESK_ID_SERVER=192.168.101.113:21116
RUSTDESK_RELAY_SERVER=192.168.101.113:21117
RUSTDESK_API_SERVER=http://192.168.101.113:21114

ENCRYPTED_ONLY=1
MUST_LOGIN=N
TZ=Asia/Ho_Chi_Minh

# Fill after first startup from /opt/rustdesk-server/data/server/id_ed25519.pub
RUSTDESK_KEY=
```

After the first startup, read the generated public key:

```bash
sudo cat /opt/rustdesk-server/data/server/id_ed25519.pub
```

Then put that value into `RUSTDESK_KEY` in Portainer and redeploy/update the stack.

## Exposed ports

The compose file exposes:

| Port | Protocol | Purpose |
|---|---|---|
| `21114` | TCP | API / web console |
| `21115` | TCP | NAT test / hbbs console |
| `21116` | TCP + UDP | ID / rendezvous / hole punching |
| `21117` | TCP | relay |
| `21118` | TCP | websocket ID |
| `21119` | TCP | websocket relay |

## Cloudflare note

Cloudflare Tunnel should only be used for the HTTP API/web console:

```yaml
- hostname: rustdesk.quanla.org
  service: http://192.168.101.113:21114
```

Place that rule before the final catch-all rule in the cloudflared `ingress` list.

Do **not** put RustDesk native ID/relay traffic (`21116` TCP/UDP, `21117` TCP, `21118`, `21119`) behind the normal Cloudflare proxy/tunnel. For external clients, create a separate DNS-only hostname such as:

```text
rd.quanla.org
```

Point it to the public IP or DDNS target, keep it **DNS only**, and forward the RustDesk ports to `192.168.101.113`.

When using that external DNS name, change Portainer env vars to:

```env
RUSTDESK_RELAY=rd.quanla.org
RUSTDESK_ID_SERVER=rd.quanla.org:21116
RUSTDESK_RELAY_SERVER=rd.quanla.org:21117
RUSTDESK_API_SERVER=https://rustdesk.quanla.org
```
