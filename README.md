# Satellite

> A small, hardened VPS stack for services whose traffic should leave from a
> separate server, behind a single reverse proxy.

![Docker Compose](https://img.shields.io/badge/docker-compose-2496ED?logo=docker&logoColor=white)
![Caddy](https://img.shields.io/badge/reverse%20proxy-caddy-1F88C0?logo=caddy&logoColor=white)
![Self Hosted](https://img.shields.io/badge/self--hosted-yes-success)
![License](https://img.shields.io/badge/license-MIT-yellow)

Satellite runs [MediaFlow Proxy Light](https://github.com/mhdzumair/mediaflow-proxy-light),
the Rust rewrite of [MediaFlow Proxy](https://github.com/mhdzumair/mediaflow-proxy),
and [AltMount](https://github.com/javi11/altmount), a usenet streaming server,
behind [Caddy](https://caddyserver.com/), with automated [restic](https://restic.net/)
backups, orchestrated with [Task](https://taskfile.dev). Point
[AIOStreams](https://github.com/Viren070/AIOStreams) at it, and proxied
streams and usenet playback are fetched and served from this server instead
of the one running your addons. It's a companion to
[Apollo](https://github.com/yarimadam/apollo), but doesn't depend on it.

This is an opinionated setup, not a general-purpose template: **[Tailscale](https://tailscale.com/)
is mandatory**, not an optional extra. Only Caddy is public, and only for
what players need. Every web UI and API is bound to the host's Tailscale IP
and has no public exposure path; the server running AIOStreams reaches them
over the same tailnet. There's no fallback for running this without one.

## Features

- MediaFlow Proxy Light: API-compatible with the Python MediaFlow, a single
  Rust binary with flat memory use, stateless (no volume)
- AltMount: streams usenet releases straight from your provider, without
  downloading them first; AIOStreams hands it NZBs and players stream from it
- Caddy in front, with real Let's Encrypt certs, exposing only what players
  need: MediaFlow's proxy endpoints and AltMount's stream endpoint. Web UIs,
  APIs and `/metrics` are tailnet-only
- Automated restic backup/prune/check jobs, covering Caddy's and AltMount's
  data and every service's `.env`; local by default, optionally offsite (e.g. Cloudflare R2)
- One `compose.yaml` per service, sharing an external Docker network
- Pinned image versions everywhere, no floating `latest` tags
- Hardened containers: all capabilities dropped (only what each image needs
  is added back), `no-new-privileges`, PID limits, read-only root filesystem;
  MediaFlow runs as a non-root user, AltMount drops to one after startup
- `task up` / `task down` for the whole stack or a single service

## Architecture

```
                         ┌──────────────┐
   Internet ─────────────▶    Caddy     │──────▶ MediaFlow proxy endpoints
                         │ (reverse     │──────▶ AltMount stream endpoint
                         │  proxy)      │
                         └──────────────┘
                                │
                        satellite network
                                │
                         ┌─────────────┐
                         │   Backups   │
                         └─────────────┘

   Tailscale ────────────▶ MediaFlow :8888 (web UI, /metrics, API)
                    └────▶ AltMount :8080 (web UI, API, WebDAV)
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
| `mediaflow` | `mediaflow/` | Streaming proxy                    | Public (proxy), Tailscale (UI, metrics)   |
| `altmount`  | `altmount/`  | Usenet streaming (NZB to WebDAV)   | Public (streams), Tailscale (UI, API)     |
| `ballast`   | `ballast/`   | Holds RAM against idle reclamation | n/a (no network)                          |
| `backup`    | `backup/`    | restic backup / prune / check jobs | n/a                                       |

## Getting Started

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [Task](https://taskfile.dev/installation/)
- [Tailscale](https://tailscale.com/download) on the host, logged in to your
  tailnet, with MagicDNS on
- A domain pointing at the server
- Provider firewall allowing 80/tcp, 443/tcp and 443/udp inbound, and nothing
  else public. If it also filters outbound traffic, allow your usenet
  provider's NNTP port (usually 563/tcp). On Oracle Cloud that means the subnet's security list or the
  instance's NSG, as well as the host's iptables.

### Installation

```sh
git clone https://github.com/yarimadam/satellite.git
cd satellite
for d in */; do [ -f "$d.env.example" ] && cp "$d.env.example" "$d.env"; done
# fill in each */.env with your domain and secrets
sudo install -Dm644 host/wait-for-tailscale.conf /etc/systemd/system/docker.service.d/wait-for-tailscale.conf
sudo systemctl daemon-reload
task up
```

Every service restarts with Docker (`restart: unless-stopped`), so the stack
comes back on boot by itself. The `host/` drop-in makes Docker wait for
Tailscale first: services publishing ports on the Tailscale IP would fail to
start, and stay down, if Docker won the race.

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
- `mediaflow/.env` `INTERFACE`: the host's Tailscale IP (`tailscale ip -4`);
  MediaFlow's own port binds only there
- `altmount/.env` `JWT_SECRET`: signs AltMount's web UI sessions
- `altmount/.env` `INTERFACE`: the same Tailscale IP; AltMount's own port
  binds only there
- `altmount/.env` `UI_HOST`: the host name you open AltMount's web UI at
  (e.g. its MagicDNS name); the login cookie is tied to it
- `caddy/.env` `MEDIAFLOW_DOMAIN`: MediaFlow's public hostname
- `caddy/.env` `ALTMOUNT_DOMAIN`: AltMount's public hostname
- `caddy/.env` `TLS`: `tls internal` for local dev, empty in production
- `ballast/.env` `SIZE_MB`: RAM held by the ballast (see
  [Oracle Always Free](#oracle-always-free))
- `backup/.env` `RESTIC_PASSWORD`: encrypts your backup repository
- `backup/.env` `RESTIC_REPOSITORY`: optional restic backend URL for offsite
  backups; leave empty for local-only

`.env` files are git-ignored, never commit them.

### Tailscale

MediaFlow's web UI, `/metrics` and API are on its own port 8888, and
AltMount's web UI, API and WebDAV on port 8080, each published only on its
`.env` `INTERFACE`: the host's Tailscale IP. From any device on your tailnet:

- MediaFlow: `http://<host>.<tailnet>.ts.net:8888`
- AltMount: `http://<host>.<tailnet>.ts.net:8080`, with `altmount/.env`
  `UI_HOST` set to that name: it's the login cookie's domain, so logins only
  stick on that exact host (empty: the Tailscale IP)

AltMount's first start opens registration: the first account created in its
web UI becomes the admin, and registration closes after it. Create it right
after the first `task up`.

### AIOStreams

#### MediaFlow

In AIOStreams' **Proxy** settings, choose MediaFlow and set:

| Field      | Value                                                                  |
|------------|------------------------------------------------------------------------|
| URL        | `http://<host>.<tailnet>.ts.net:8888`                                  |
| Public URL | `https://<MEDIAFLOW_DOMAIN>`                                           |
| Credentials| `mediaflow/.env` `API_PASSWORD`                                        |

AIOStreams calls MediaFlow's API at **URL**, over the tailnet, and rewrites
the stream links to **Public URL**, so players always get the public domain. Streams from
AIOStreams' built-in usenet engine are never proxied (AIOStreams serves them
itself), so MediaFlow only carries the other streams you enable proxying for.

#### AltMount

AltMount replaces AIOStreams' built-in usenet engine: AIOStreams hands it
NZBs, and players stream from AltMount directly, so the server running
AIOStreams no longer downloads or serves the video.

In AltMount's web UI:
- **Providers**: add your usenet provider(s).
- **Stremio**: enable it, and set its base URL to `https://<ALTMOUNT_DOMAIN>`
  so the stream links it returns use the public domain.
- Copy the API key from your account page.

In AIOStreams' **Services**, enable AltMount and set:

| Field                 | Value                                                                  |
|-----------------------|------------------------------------------------------------------------|
| URL                   | `http://<host>.<tailnet>.ts.net:8080`                                  |
| Public URL            | `https://<ALTMOUNT_DOMAIN>`                                            |
| API key               | AltMount's API key                                                     |
| WebDAV user/password  | AltMount's WebDAV login (Configuration → WebDAV); change the default   |
| AIOStreams Auth Token | leave empty: it routes streams back through AIOStreams                 |

AltMount downloads each NZB itself, from the link AIOStreams passes on, so
your indexer (or NZBHydra2) must be reachable from this server under the
host name in its links. For NZBHydra2 on another tailnet host, point
AIOStreams at Hydra's MagicDNS name (e.g. Apollo's `aiostreams/.env`
`BUILTIN_NZBHYDRA_URL`), since Hydra builds its links from that address.

### Oracle Always Free

Oracle reclaims Always Free instances it deems idle: over 7 days, CPU
(95th percentile), network and, on A1 shapes, memory all below 20%. Staying
above 20% on any one is enough, and memory is the cheapest: the `ballast`
service fills an in-memory filesystem with `SIZE_MB` once at startup, then
sleeps, with no network and no CPU use. Size it so the host's total memory
use stays above 20% with a margin, and lower it as real services grow.
Oracle's view is the `MemoryUtilization` metric in the instance's
monitoring; the Oracle Cloud Agent's Compute Instance Monitoring plugin must
be enabled to report it. Not on Oracle? Leave it out: `task down:ballast`.

See [RECOVERY.md](RECOVERY.md) for restoring onto a fresh server from backup.

## License

[MIT](LICENSE)

## Acknowledgements

- [MediaFlow Proxy Light](https://github.com/mhdzumair/mediaflow-proxy-light)
- [Caddy](https://github.com/caddyserver/caddy)
- [restic](https://github.com/restic/restic)
