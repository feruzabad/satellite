# Satellite

> A small, hardened VPS stack for services whose traffic should leave from a
> separate server, behind a single reverse proxy.

![Docker Compose](https://img.shields.io/badge/docker-compose-2496ED?logo=docker&logoColor=white)
![Caddy](https://img.shields.io/badge/reverse%20proxy-caddy-1F88C0?logo=caddy&logoColor=white)
![Self Hosted](https://img.shields.io/badge/self--hosted-yes-success)
![License](https://img.shields.io/badge/license-MIT-yellow)

Satellite runs [MediaFlow Proxy Light](https://github.com/mhdzumair/mediaflow-proxy-light),
the Rust rewrite of [MediaFlow Proxy](https://github.com/mhdzumair/mediaflow-proxy),
behind [Caddy](https://caddyserver.com/), with automated [restic](https://restic.net/)
backups, orchestrated with [Task](https://taskfile.dev). Point
[AIOStreams](https://github.com/Viren070/AIOStreams) (or any MediaFlow client)
at it, and the streams it proxies are fetched and served from this server
instead of the one running your addons. It's a companion to
[Apollo](https://github.com/yarimadam/apollo), but doesn't depend on it.

This is an opinionated setup: only Caddy is public, and everything else binds
to `127.0.0.1` by default. How you reach the private parts is your choice: an
SSH tunnel, or a private network such as [Tailscale](https://tailscale.com/)
or WireGuard. See [Private access](#private-access).

## Features

- MediaFlow Proxy Light: API-compatible with the Python MediaFlow, a single
  Rust binary with flat memory use, stateless (no volume)
- Caddy in front, with real Let's Encrypt certs, exposing only MediaFlow's
  proxy endpoints. Its web UI and `/metrics` stay private unless you opt into
  serving them behind a Caddy login
- Automated restic backup/prune/check jobs, covering Caddy's data and every
  service's `.env`; local by default, optionally offsite (e.g. Cloudflare R2)
- One `compose.yaml` per service, sharing an external Docker network
- Pinned image versions everywhere, no floating `latest` tags
- Hardened containers: all capabilities dropped (only what each image needs
  is added back), `no-new-privileges`, PID limits, read-only root filesystem;
  MediaFlow runs as a non-root user
- `task up` / `task down` for the whole stack or a single service

## Architecture

```
                         ┌──────────────┐
   Internet ─────────────▶    Caddy     │──────▶ MediaFlow proxy endpoints
                         │ (reverse     │
                         │  proxy)      │
                         └──────────────┘
                                │
                        satellite network
                                │
                         ┌─────────────┐
                         │   Backups   │
                         └─────────────┘

   SSH tunnel / VPN ─────▶ MediaFlow :8888 (web UI, /metrics, API)
```

Every service lives in its own folder with its own `compose.yaml` and
`.env`, all attached to one external `satellite` Docker network.

Caddy additionally joins `satellite_edge`, an IPv6-enabled network that
carries its published ports, so it sees real client addresses on IPv6 too.
The apps stay IPv4-only, which keeps their outbound traffic on a single
address.

## Services

| Service     | Folder       | Description                        | Exposure                                  |
|-------------|--------------|------------------------------------|-------------------------------------------|
| `caddy`     | `caddy/`     | Reverse proxy, automatic HTTPS     | Public                                    |
| `mediaflow` | `mediaflow/` | Streaming proxy                    | Public (proxy), private (UI, metrics)     |
| `backup`    | `backup/`    | restic backup / prune / check jobs | n/a                                       |

## Getting Started

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [Task](https://taskfile.dev/installation/)
- A domain pointing at the server
- Provider firewall allowing 80/tcp, 443/tcp and 443/udp inbound, and nothing
  else public. On Oracle Cloud that means the subnet's security list or the
  instance's NSG, as well as the host's iptables.
- Optional: a VPN on the host (Tailscale, WireGuard, …), if you'd rather not
  use SSH tunnels

### Installation

```sh
git clone https://github.com/yarimadam/satellite.git
cd satellite
for d in */; do [ -f "$d.env.example" ] && cp "$d.env.example" "$d.env"; done
# fill in each */.env with your domain and secrets
task up
```

### Usage

```sh
task up                  # start everything
task down                # stop everything
task up:mediaflow        # start a single service
task down:caddy          # stop a single service
task --list              # see all available tasks
```

## Configuration

Each service reads only its own `<service>/.env` (see the `.env.example`
next to it for the full, documented template). Key things you'll want to set:

- `mediaflow/.env` `API_PASSWORD`: protects every proxy endpoint
- `mediaflow/.env` `INTERFACE`: where MediaFlow's own port binds (see
  [Private access](#private-access))
- `caddy/.env` `MEDIAFLOW_DOMAIN`: MediaFlow's public hostname
- `caddy/.env` `TLS`: `tls internal` for local dev, empty in production
- `caddy/.env` `MEDIAFLOW_ADMIN`: `off`, or `basic_auth` to serve the web UI
  publicly behind a login
- `backup/.env` `RESTIC_PASSWORD`: encrypts your backup repository
- `backup/.env` `RESTIC_REPOSITORY`: optional restic backend URL for offsite
  backups; leave empty for local-only

`.env` files are git-ignored, never commit them.

### Private access

MediaFlow's web UI, `/metrics` and API are on its own port 8888, published
only on `mediaflow/.env` `INTERFACE`. Pick one:

- **SSH tunnel** (default, `INTERFACE=127.0.0.1`): nothing else to install.
  ```sh
  ssh -L 8888:127.0.0.1:8888 user@server
  # then open http://localhost:8888
  ```
- **VPN** (`INTERFACE=<the host's VPN IP>`, e.g. `tailscale ip -4`): reach
  `http://<vpn-ip>:8888` from any device on that network. A server running
  AIOStreams on the same network can also use it for API calls (see below).
- **Caddy login** (`caddy/.env` `MEDIAFLOW_ADMIN=basic_auth`): the UI on the
  public domain, behind a password. For hosts where neither of the above is
  possible. Anyone who finds the domain sees the login prompt.

Never set `INTERFACE` to a public IP or `0.0.0.0`: that would expose port
8888 without Caddy's TLS and path filtering.

### AIOStreams

In AIOStreams' **Proxy** settings, choose MediaFlow and set:

| Field      | Value                                                                  |
|------------|------------------------------------------------------------------------|
| URL        | `https://<MEDIAFLOW_DOMAIN>`, or `http://<vpn-ip>:8888` over a VPN     |
| Public URL | `https://<MEDIAFLOW_DOMAIN>` when URL is the VPN address, else empty  |
| Credentials| `mediaflow/.env` `API_PASSWORD`                                        |

AIOStreams calls MediaFlow's API at **URL** and rewrites the stream links to
**Public URL**, so players always get the public domain. Streams from
AIOStreams' built-in usenet engine are never proxied (AIOStreams serves them
itself), so MediaFlow only carries the other streams you enable proxying for.

See [RECOVERY.md](RECOVERY.md) for restoring onto a fresh server from backup.

## License

[MIT](LICENSE)

## Acknowledgements

- [MediaFlow Proxy Light](https://github.com/mhdzumair/mediaflow-proxy-light)
- [Caddy](https://github.com/caddyserver/caddy)
- [restic](https://github.com/restic/restic)
