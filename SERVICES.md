# Home Lab Docker Services

*Snapshot taken 2026-09-28. Host: Windows 11 PC running Docker Desktop (WSL2 backend).*

## Overview

This PC runs a small self-hosted setup: photo backup, password management, network-wide ad blocking, reporting for penetration tests, file backups and remote access. Everything runs as Docker containers. Each service is defined by a compose file in its own folder under `C:\Users\crual\docker\`, which lets you manage all of them from Dockge.

The host has one physical NIC, **Ethernet 2**, with the static primary address `192.168.0.3/24` (gateway `192.168.0.1`). Rather than put every service on a different port of one IP, the NIC carries several **secondary IP addresses**, one per service. They're all marked `SkipAsSource` so Windows never uses them for outgoing traffic. Each service binds only to its own IP, which gives it a clean, memorable address on the LAN (usually on the service's default port).

| IP | Service | URL |
|---|---|---|
| 192.168.0.3 | Host (Immich, SysReptor) | http://192.168.0.3:2244 |
| 192.168.0.4 | Pi-hole (primary) | http://192.168.0.4/admin |
| 192.168.0.5 | Pi-hole (secondary, relay) | http://192.168.0.5/admin |
| 192.168.0.7 | Duplicati | http://192.168.0.7:8200 |
| 192.168.0.8 | Vaultwarden | https://192.168.0.8 |
| 192.168.0.9 | Dockge | http://192.168.0.9:5001 |

Secrets aren't written in this document. Each stack keeps its own in the `.env` file in its folder.

---

## 1. Immich: photo and video backup

**Folder:** `immich\` · **URL:** http://192.168.0.3:2244 (also `immich.local`)

Immich is a self-hosted alternative to Google Photos. Phones upload photos and videos automatically through the Immich mobile app, and the web UI supports browsing, face recognition, smart search and shared albums. Immich was the top priority of this setup.

It runs as four containers:

| Container | Role |
|---|---|
| `immich_server` | Web UI and API (container port 2283, published on host port **2244** on all interfaces) |
| `immich_machine_learning` | Face detection, CLIP smart search, runs locally |
| `immich_postgres` | PostgreSQL with vector extensions for search |
| `immich_redis` | Valkey (Redis-compatible) job queue |

**Data:** uploaded media goes to `immich\library` on disk. The database is in the `pgdata` Docker volume.

---

## 2. SysReptor: pentest reporting

**Folder:** `sysreptor\deploy\` · **URL:** http://127.0.0.1:8000 (reachable only from this PC)

SysReptor is a report-writing platform for penetration testing. You write findings in Markdown and it renders them into PDF reports built from templates. On purpose, it's bound to **localhost only**, so it isn't exposed to the LAN.

| Container | Role |
|---|---|
| `sysreptor-app` | Django app and web UI |
| `sysreptor-db` | PostgreSQL 14 |
| `sysreptor-redis-1` | Redis cache |

Its configuration is in `sysreptor\deploy\app.env`.

---

## 3. Pi-hole: network-wide DNS ad blocking

**Folder:** `pihole\` · **Admin:** http://192.168.0.4/admin (also `pi-hole.local`)

Pi-hole is a DNS server that blocks ads and trackers for every device that uses it. Upstream resolvers are Cloudflare (`1.1.1.1`) and Google (`8.8.8.8`).

It's reachable on **two IPs** so that DHCP clients can be given a primary and a secondary DNS server:

- **192.168.0.4**: the `pihole` container itself (DNS 53 TCP/UDP, web 80, HTTPS 8443).
- **192.168.0.5**: the `pihole-relay` container, a small `socat` forwarder that relays the same ports to Pi-hole. The relay is needed because Docker Desktop can't publish the same container port on two host IPs.

Pi-hole also acts as **local DNS** for the lab, so services can be reached by name:

| Name | Resolves to |
|---|---|
| immich.local | 192.168.0.3 |
| sysreptor.local | 192.168.0.3 |
| pi-hole.local | 192.168.0.4 |
| vaultwarden.local | 192.168.0.8 |
| dockge.local | 192.168.0.9 |

> **Note:** the PC and router still hand out DNS `192.168.0.15, 8.8.8.8`. Pi-hole doesn't filter anything until devices or the router's DHCP are pointed at 192.168.0.4 and 192.168.0.5. There's no `duplicati.local` entry yet.

**Data:** the `pihole-etc` Docker volume.

---

## 4. Vaultwarden: password manager

**Folder:** `vaultwarden\` · **URL:** https://192.168.0.8 (also `https://vaultwarden.local`)

Vaultwarden is a lightweight server that's compatible with Bitwarden, so the official Bitwarden apps and browser extensions work with it. Point them at `https://192.168.0.8` as a self-hosted server.

| Container | Role |
|---|---|
| `vaultwarden` | The password vault server (internal only) |
| `vaultwarden-caddy` | Caddy reverse proxy that provides HTTPS on 192.168.0.8:443 |

Bitwarden clients require HTTPS, so Caddy serves the vault with a certificate from its own **internal CA**. To avoid certificate warnings, install the exported root CA (`vaultwarden\root.crt` / `root.cer`) as a trusted root on every device that uses the vault.

> **Security to-do:** `SIGNUPS_ALLOWED` is still `true`. Once your accounts exist, set it to `false` and redeploy.

**Data:** the `vw-data` volume (vault) and the `caddy-data` volume (CA and certificates).

---

## 5. Dockge: stack manager

**Folder:** `dockge\` · **URL:** http://192.168.0.9:5001 (also `dockge.local`)

Dockge is a web UI for managing Docker Compose stacks. It reads every stack in `C:\Users\crual\docker\`, so you can start, stop, update and edit all of the services above from the browser. It controls Docker through the Docker socket, so treat access to it as admin access to the whole host.

---

## 6. Duplicati: backups

**Folder:** `duplicati\` · **URL:** http://192.168.0.7:8200

Duplicati makes encrypted, deduplicated, scheduled backups to local disks, NAS shares or cloud storage (S3, B2, Google Drive, OneDrive and more).

- **Source:** the whole user folder `C:\Users\crual` is mounted **read-only** at `/source/crual`.
- **Local destination:** `/backups`, which saves to `duplicati\backups`.
- The UI password and settings encryption key are in `duplicati\.env`. **Back up that file**, because Duplicati can't read its saved settings without the key.

> **Recommendation:** `duplicati\backups` is on the same disk as the source files, so it doesn't protect against a disk failure. Send important jobs to a different drive or to the cloud. Data stored in Docker volumes (Vaultwarden, Pi-hole, the Immich database) isn't visible to Duplicati yet, so those volumes would need to be mounted before they can be backed up.

---

## 7. Tailscale: remote access

**Folder:** `tailscale\` · **Tailnet IP:** 100.120.144.58 · **Hostname:** `docker-pc`

Tailscale connects this PC to a private WireGuard mesh VPN, so every service above can be reached from anywhere without opening ports on the router. The container runs as a **subnet router** that advertises `192.168.0.0/24`. Other tailnet devices, such as the Android phone and the laptop, can use the LAN IPs in this document directly while away from home. The subnet route must be approved once in the Tailscale admin console.

---

## Container inventory

| Container | Image | Published on |
|---|---|---|
| immich_server | ghcr.io/immich-app/immich-server:v3 | 0.0.0.0:2244 |
| immich_machine_learning | ghcr.io/immich-app/immich-machine-learning:v3 | none |
| immich_postgres | ghcr.io/immich-app/postgres:14-vectorchord | none |
| immich_redis | valkey/valkey:9 | none |
| sysreptor-app | syslifters/sysreptor:2026.82 | 127.0.0.1:8000 |
| sysreptor-db | postgres:14 | none |
| sysreptor-redis-1 | redis:8.0-alpine | none |
| pihole | pihole/pihole:latest | 192.168.0.4:53, 80, 8443 |
| pihole-relay | alpine/socat:latest | 192.168.0.5:53, 80, 8443 |
| vaultwarden | vaultwarden/server:latest | none (behind Caddy) |
| vaultwarden-caddy | caddy:2 | 192.168.0.8:80, 443 |
| dockge | louislam/dockge:1 | 192.168.0.9:5001 |
| duplicati | lscr.io/linuxserver/duplicati:latest | 192.168.0.7:8200 |
| tailscale | tailscale/tailscale:latest | none (VPN) |

## Open items

1. Point devices or the router's DHCP DNS at Pi-hole (192.168.0.4 and 192.168.0.5).
2. Set Vaultwarden `SIGNUPS_ALLOWED=false` once your accounts are created.
3. Set up Duplicati jobs to an off-disk destination, and mount the Docker volumes you want protected.
4. Optionally add `duplicati.local` to Pi-hole's local DNS.
5. Approve the Tailscale subnet route in the admin console if you haven't already.
