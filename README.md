# TARS: a self-hosted media server built from spare parts

TARS is my home lab server. I built it from spare and second-hand PC parts, installed and hardened Ubuntu, and run a family media server (Jellyfin) on it. It is reachable from anywhere through a Cloudflare Tunnel, with no ports opened on my router. I administer it remotely over a Tailscale VPN.

This repo holds the infrastructure-as-code (Docker Compose) and the documentation for the build. It is a living document that I update as the server grows.

## Goals

- Learn self-hosting end to end: hardware, OS, networking, containers, security, and backups.
- Replace streaming subscriptions for my family with a private media library.
- Expose it to family safely, without port forwarding or a publicly reachable home IP.
- Keep the cost low by reusing parts I already had or could find second-hand.

## Hardware

Built into an Antec Fusion Remote micro-ATX HTPC case, mostly from spare and second-hand parts.

| Component | Part | Notes |
|---|---|---|
| Case | Antec Fusion Remote (black) | Low-profile HTPC, front LCD and IR receiver, 2 × 3.5" internal bays, 5.25" bay |
| CPU | Intel Core i7-7700 (4C/8T, up to 4.2 GHz) | Intel Quick Sync for hardware video transcoding; stock cooler with refreshed thermal paste |
| Motherboard | Gigabyte H270M-DS3H (micro-ATX) | 6 × SATA, 1 × M.2 NVMe slot, 4 × DDR4 slots |
| Memory | 16 GB DDR4 (2 × 8 GB) | |
| Boot / apps | Crucial P310 1 TB NVMe (PCIe Gen4) | OS, containers, Jellyfin database and transcode cache |
| Media storage | Seagate IronWolf 8 TB HDD (ST8000VN004, NAS-class) | NTFS; movies and TV shows |
| Optical | LG BD-RE WH08LS20 Blu-ray drive | 5.25" bay |
| Power | Corsair RM750 (750 W, 80 Plus Gold, fully modular) | Far more headroom than needed, so room to add drives |
| Cooling | 2 × 120 mm be quiet! PWM fans with magnetic dust filters | Tuned for quiet HTPC use |

**Build notes:** assembled by me, with fresh thermal paste on the CPU cooler and new SATA cables for the storage drive.

## Architecture

```
                       Internet
                          |
             Cloudflare (TLS, DNS, Tunnel)
                          |   outbound-only connection
  +-----------------------|----------------------------+
  | TARS (Ubuntu 26.04)   |                            |
  |   +-------------+     v      +-------------------+ |
  |   | cloudflared |----------->|     Jellyfin      | |
  |   +-------------+  Docker    | (Quick Sync via   | |
  |                    network   |  /dev/dri)        | |
  |                              +---------+---------+ |
  |                                        | read-only |
  |                              /mnt/media (8 TB HDD)| |
  +--------------------------------------------------+-+
        ^                    ^
        | LAN :8096          | Tailscale (SSH, remote desktop)
   Home TV              My laptop
```

- **Family access:** a Cloudflare Tunnel (`cloudflared` container) makes an outbound-only connection to Cloudflare, which serves Jellyfin on a public hostname over HTTPS. Nothing is port-forwarded on the router and my home IP is never exposed.
- **Home TV:** streams directly over the LAN on port 8096, bypassing the tunnel.
- **Administration:** SSH and remote desktop run over Tailscale, a WireGuard-based mesh VPN.
- **Tailnet access to Jellyfin:** `tailscale serve` also publishes Jellyfin over HTTPS to my own tailnet devices only (not Tailscale Funnel, so not public).

## Software stack

| Layer | Choice |
|---|---|
| OS | Ubuntu 26.04 LTS (desktop install, hardened) |
| Containers | Docker and Docker Compose |
| Media server | Jellyfin, with Intel Quick Sync hardware transcoding |
| Public access | Cloudflare Tunnel (`cloudflared`) |
| Remote admin | OpenSSH (key-only), Tailscale, GNOME Remote Desktop (RDP) |
| Firewall | ufw (iptables) |
| Source control | Git and GitHub (private deploy key, SSH) |

Everything is defined in [`jellyfin/compose.yaml`](jellyfin/compose.yaml).

## Security measures

Security was a design goal from the start. The layers:

**Network exposure**
- **No inbound ports on the router.** Public access goes through an outbound-only Cloudflare Tunnel, so there is nothing for a port scan of my home IP to find.
- **Default-deny firewall (ufw).** SSH is allowed only from the local subnet; all traffic arriving over the Tailscale interface is allowed, and everything else inbound is denied.
- **Docker and ufw:** Docker publishes ports by writing its own iptables rules, which bypass ufw. I bind published ports to specific addresses instead of `0.0.0.0` (Jellyfin is bound to localhost and the server's LAN address only) and audit with `ss -tlnp`.
- **Private remote access.** SSH and RDP are reachable only over the LAN or the Tailscale VPN, never the public internet.

**Authentication and access control**
- **SSH is key-only:** password authentication is disabled and root login is disabled. The private key never leaves my laptop.
- **sudo requires a password,** deliberately, as a pause before any privileged change. Day-to-day work is not done as root.
- **Per-purpose credentials:** the GitHub deploy key is scoped to this single repository.

**Containers and data**
- **Media is mounted read-only** into Jellyfin, so a bug or compromise in the app cannot alter or delete the library.
- **Jellyfin runs as an unprivileged user** (not root) inside its container, with only the GPU render device passed through for transcoding.
- **Secrets stay out of git.** The Cloudflare tunnel token lives in a `.env` file with `600` permissions and is excluded by `.gitignore`, as are Jellyfin's database and cache.

**Maintenance**
- **Automatic security updates** via `unattended-upgrades`.

**Process**
- Changes to SSH, the firewall, and networking follow a checklist: back up the config, validate it (`sshd -t`, `ufw show added`), keep a second session open, and confirm SSH is still permitted before applying anything.

## Skills demonstrated

- **Hardware:** component selection and compatibility, PC assembly, thermal paste, cable management, storage planning.
- **Linux administration:** Ubuntu install and hardening, systemd services, users and groups (`render` group for GPU access), filesystems and mounts (ext4, NTFS), package management, log inspection with `journalctl`.
- **Networking:** DHCP reservation, LAN/VPN/public access design, Cloudflare Tunnel and DNS, Tailscale and MagicDNS, firewall rules.
- **Containers:** Docker Compose, bind mounts, device passthrough, restart policies, container networking.
- **Security:** least privilege, defense in depth, attack-surface reduction, key-based auth, secrets management.
- **Troubleshooting:** diagnosed a remote desktop (RDP) client issue by isolating client, server, firewall, and VPN layers, reading server logs, and testing alternate clients (see below).
- **Documentation and version control:** Git/GitHub workflow, written runbooks and decisions.

## Lessons learned

- **Docker bypasses ufw.** A firewall that says "deny" does not cover container-published ports, so check what is actually listening.
- **Isolate the layer when debugging.** The macOS Windows App hung on RDP while Windows clients and FreeRDP worked fine, which proved the server, firewall, and VPN were healthy. I use FreeRDP instead.
- **Write down the single points of failure.** Backups are the next priority (see roadmap).

## Roadmap

- [ ] Back up Jellyfin's `config/` (database) on a schedule, using a consistent copy method
- [ ] Decide on a second copy of the media (offsite or a second drive)
- [ ] Pin container image versions instead of `:latest`
- [ ] Add intrusion prevention for SSH (fail2ban is not installed yet)
- [ ] Add monitoring and alerting (disk health, container health, uptime)
- [ ] Add more self-hosted services

## Repo layout

```
.
├── README.md
├── .gitignore               # excludes .env, Jellyfin config/ and cache/
└── jellyfin/
    └── compose.yaml         # Jellyfin + cloudflared
```

Secrets (`.env`) and runtime data are intentionally not in the repo. To deploy, create `jellyfin/.env` containing `TUNNEL_TOKEN=<your Cloudflare tunnel token>`, then run `docker compose up -d` from the `jellyfin/` directory.
