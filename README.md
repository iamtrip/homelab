# tystation homelab

Single-node Proxmox homelab. Internal DNS, a reverse proxy with trusted HTTPS on a real domain, a password manager (not yet in use), and an ArchiSteamFarm instance. Built from scratch, documented so it can be rebuilt.

> No secrets live in this repo. Keys, tokens and passwords are stored elsewhere (see [Secrets](#secrets)).

## Hardware

| | |
|---|---|
| Host | Proxmox VE, node name `tystation` |
| CPU | Intel Core i5-10500T (6 cores / 12 threads) |
| RAM | 64 GB |
| Storage | Two separate NVMe drives: `local` (boot + backups) and `vmpool` (guest disks) |

## Containers

| ID | Name | IP | Purpose |
|---|---|---|---|
| 100 | adguard | 192.168.1.10 | AdGuard Home: network-wide DNS and ad blocking, DNS rewrites for internal names |
| 101 | nginxproxy | 192.168.1.11 | Nginx Proxy Manager: reverse proxy, TLS termination, wildcard cert |
| 102 | vaultwarden | 192.168.1.12 | Vaultwarden (set up, not in use) |
| 103 | asf | 192.168.1.13 | ArchiSteamFarm: idles hours on alt Steam accounts, web GUI on port 1242 |

All containers use static IPs. Start-at-boot is on, with AdGuard first in the boot order so DNS is up before anything that depends on it.

## How it fits together

```
Device
  │  1. "where is *.tystation.com?"
  ▼
AdGuard Home (192.168.1.10)      <- router hands this out as the DNS server via DHCP
  │  2. DNS rewrite: *.tystation.com -> 192.168.1.11
  ▼
Nginx Proxy Manager (192.168.1.11)
  │  3. terminates HTTPS with the wildcard cert, forwards to the service
  ▼
Service LXC (IP:port)
```

- **Domain:** `tystation.com`, with DNS hosted on Cloudflare. Nothing is published publicly for internal names; resolution is handled entirely by the AdGuard rewrite.
- **Certificate:** one wildcard Let's Encrypt cert (`tystation.com` and `*.tystation.com`) issued by NPM using the Cloudflare DNS challenge. Auto-renews. Current expiry is February 2027; check the NPM SSL Certificates page occasionally to confirm renewal is working.
- **Cloudflare API token:** scoped to Zone > DNS > Edit on this one zone only.

## Reverse proxy hosts

| Hostname | Backend |
|---|---|
| adguard.tystation.com | http://192.168.1.10:80 |
| asf.tystation.com | http://192.168.1.13:1242 |
| proxmox.tystation.com | https://192.168.1.9:8006 |
| proxy.tystation.com | http://192.168.1.11:81 (NPM admin UI) |
| vault.tystation.com | https://192.168.1.12:8000 |

**Break-glass access:** if AdGuard or NPM is down, the names stop working. Reach Proxmox directly at `https://192.168.1.9:8006` (self-signed cert warning expected) and fix the container from there.

## ArchiSteamFarm

- Runs as a systemd service under a dedicated unprivileged `asf` user in `/opt/asf`.
- Bot passwords are stored AES-encrypted (`PasswordFormat: 1`) using a custom `--cryptkey`.
- The web GUI is bound to the LAN and protected by an SCrypt-hashed `IPCPassword`.
- First login for each bot is done once in the foreground (Steam Guard QR), after which ASF reuses its saved login token.

## Backups

- Proxmox backup job (Datacenter > Backup) covering the containers, snapshot mode, ZSTD compression.
- Schedule: First day of each month 03:00. Retention: 3.
- Target: `local`, which is a different physical NVMe drive from the guest disks.
- All backups together are about 1.2 GB.
- **Restore test:** restored a backup to a spare container ID with "Start after restore" off (avoids an IP clash with the live container), confirmed it booted, then deleted it.
- Limitation: both drives are in one machine, so this does not cover theft, fire or a power fault. Copy a recent backup off the box occasionally.

## Secrets

Not stored in this repo. Where they live:

- ASF crypt key is also present in the `asf.service` unit file (root-only, mode 600)

## Problems hit and how they were fixed

- **Proxmox login returned 401 through the proxy over plain HTTP.** Proxmox sets its session cookie with the `Secure` flag, so a browser on `http://` discards it. Fix: serve it over HTTPS with a trusted cert.
- **`SSL_ERROR_BAD_CERT_DOMAIN` on a proxied host.** The cert covered `tystation.com` only; the wildcard `*.tystation.com` was missing from the certificate's domain list. Fix: reissue with both names. A wildcard does not cover the bare apex domain.
- **ASF would not start: "global config could not be loaded".** A missing quote in `ASF.json` made it invalid JSON. Fix: correct the JSON; the error named the file.
- **`encrypt` and `hash` commands refused: "no bots defined".** ASF needs at least one bot config present before it will run commands. Fix: create a disabled placeholder bot config.

## Maintenance checklist

- [ ] Monthly: `apt update && apt upgrade` in each container, then on the Proxmox host (take a manual backup first)
- [ ] Check NPM wildcard cert expiry and renewal status
- [ ] Verify the latest backup exists and occasionally test a restore
- [ ] Copy a recent backup off the machine

## Ideas / next steps

- VLAN segmentation
- Decide whether Vaultwarden gets used or retired
