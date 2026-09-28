# Satellite

> A personal VPS stack: services whose traffic should leave from a different
> server than [Apollo](https://github.com/yarimadam/apollo)'s, behind a single
> reverse proxy.

![Docker Compose](https://img.shields.io/badge/docker-compose-2496ED?logo=docker&logoColor=white)
![Caddy](https://img.shields.io/badge/reverse%20proxy-caddy-1F88C0?logo=caddy&logoColor=white)
![Self Hosted](https://img.shields.io/badge/self--hosted-yes-success)
![License](https://img.shields.io/badge/license-MIT-yellow)

Satellite runs [MediaFlow Proxy Light](https://github.com/mhdzumair/mediaflow-proxy-light),
the Rust rewrite of [MediaFlow Proxy](https://github.com/mhdzumair/mediaflow-proxy),
behind [Caddy](https://caddyserver.com/), with automated [restic](https://restic.net/)
backups, orchestrated with [Task](https://taskfile.dev). Apollo's AIOStreams
routes usenet playback through MediaFlow, so stream traffic leaves from this
server instead of Apollo's.

Same conventions as Apollo: opinionated, **Tailscale is mandatory**, and
nothing but Caddy is public. SSH and any admin access go over the tailnet.

## Features

- MediaFlow Proxy Light: API-compatible with the Python MediaFlow, a single
  Rust binary with flat memory use, stateless (no volume)
- Caddy in front, with real Let's Encrypt certs; MediaFlow's unauthenticated
  `/metrics` is not exposed
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
   Internet ─────────────▶    Caddy     │──────▶ MediaFlow (external)
                         │ (reverse     │
                         │  proxy)      │
                         └──────────────┘
                                │
                        satellite network
                                │
                         ┌─────────────┐
                         │   Backups   │
                         └─────────────┘
```

Every service lives in its own folder with its own `compose.yaml` and
`.env`, all attached to one external `satellite` Docker network.

Caddy additionally joins `satellite_edge`, an IPv6-enabled network that
carries its published ports, so it sees real client addresses on IPv6 too.
The apps stay IPv4-only, which keeps their outbound traffic on a single
address.

## Services

| Service     | Folder       | Description                        | Exposure  |
|-------------|--------------|------------------------------------|-----------|
| `caddy`     | `caddy/`     | Reverse proxy, automatic HTTPS     | External  |
| `mediaflow` | `mediaflow/` | Streaming proxy for AIOStreams     | External  |
| `backup`    | `backup/`    | restic backup / prune / check jobs | n/a       |

## Getting Started

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [Task](https://taskfile.dev/installation/)
- [Tailscale](https://tailscale.com/), installed and running on the host
- Provider firewall allowing 80/tcp, 443/tcp and 443/udp inbound, and
  nothing else public (on Oracle Cloud: the subnet's security list or the
  instance's NSG)

### Installation

```sh
git clone <this-repo>
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
- `caddy/.env` `MEDIAFLOW_DOMAIN`: MediaFlow's public hostname
- `caddy/.env` `TLS`: `tls internal` for local dev, empty in production
- `backup/.env` `RESTIC_PASSWORD` / `RESTIC_REPOSITORY`: as in Apollo

A few values connect Satellite to Apollo, so they must match there:

- AIOStreams' Proxy settings: MediaFlow URL `https://<MEDIAFLOW_DOMAIN>` and
  `mediaflow/.env` `API_PASSWORD`

`.env` files are git-ignored, never commit them.

See [RECOVERY.md](RECOVERY.md) for restoring onto a fresh server from backup.

## License

[MIT](LICENSE)

## Acknowledgements

- [MediaFlow Proxy Light](https://github.com/mhdzumair/mediaflow-proxy-light)
- [Caddy](https://github.com/caddyserver/caddy)
