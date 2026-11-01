---
created: 2026-11-01
topics:
  - "[[Penpot]]"
  - "[[Docker]]"
  - "[[Tailscale]]"
  - "[[Self-hosting]]"
---

> [!summary]
> Penpot 2.18.5 runs in Docker Desktop on my Windows 11 PC. It's reachable only over [[Tailscale]] with HTTPS, and sign-up is invite-only. Docker's data lives on G:, config in `G:\03 Program\Computer Program\Docker\penpot`, and backups on M:. Docker is started by hand when needed. Browser tests passed on two devices, and a backup restored correctly into a test copy.

> [!warning] This vault syncs to GitHub
> Secrets, the auth key and the real tailnet name are left out of this note on purpose. Below, `<tailnet>` stands for my tailnet name. Find the full address with `tailscale status`, or on the Machines page of the Tailscale admin console.

## Daily use

Open **https://penpot.`<tailnet>`.ts.net** with Tailscale connected. Chrome is the recommended browser.

Run commands from the config folder in PowerShell:

```powershell
cd "G:\03 Program\Computer Program\Docker\penpot"
```

| Task | Command |
|---|---|
| Start Docker Desktop | Open the app, or `docker desktop start` |
| Start Penpot | `docker compose -p penpot up -d` |
| Stop Penpot | `docker compose -p penpot stop` |
| Check status | `docker compose -p penpot ps` |
| See logs | `docker compose -p penpot logs --tail 50 penpot-backend` |
| Back up | `powershell -ExecutionPolicy Bypass -File backup-penpot.ps1` |
| Read Penpot's emails | `http://localhost:8025` (this PC only) |

Penpot comes up by itself when Docker starts, unless I stopped it with the stop command. To free memory, quit Docker Desktop from its tray icon, because closing the window can leave it running.

> [!danger] Rules that protect my data
> - **Never run `docker compose down -v`.** The `-v` deletes all designs. Plain `down` is safe.
> - **Always use `-p penpot`.** Another project name starts an empty Penpot with new volumes.
> - **Back up before** using Docker's *Reset to factory defaults* or uninstalling it. Either can wipe the disk.
> - **Don't sync** the G: disk image or `.env` to the cloud. Syncing a live disk file can corrupt it, and `.env` holds secrets.

## How it's set up

| Part | Where or what |
|---|---|
| Penpot | 2.18.5 (`PENPOT_VERSION` in `.env`) |
| Config | `G:\03 Program\Computer Program\Docker\penpot` |
| Docker disk (all containers and volumes) | `G:\03 Program\Computer Program\Docker\disk-image\DockerDesktopWSL` |
| Backups | `M:\03 Program\Computer Program\Docker Backup\penpot` |
| Volumes | `penpot_penpot_postgres_v15`, `penpot_penpot_assets`, `penpot_penpot_tailscale` |
| Tailscale device | `penpot`, tagged `tag:container`, no key expiry, tailnet only (Funnel off) |
| Ports on the PC | Only `127.0.0.1:8025` (test mailbox). Port 9001 isn't published |

Config files:

| File | Purpose |
|---|---|
| `docker-compose.yaml` | My edited Compose file, with 9 services |
| `docker-compose.original.yaml` | The untouched official file, for comparing at upgrades |
| `.env` | Version, secret key, database password, and the (now blank) `TS_AUTHKEY` |
| `tailscale/serve.json` | HTTPS proxy config for the sidecar |
| `backup-penpot.ps1` | Backup script |
| `RESTORE.md` | Restore and upgrade steps |

Traffic path: browser → Tailscale → `penpot` sidecar (HTTPS) → `penpot-frontend:8080` → backend, database, and so on.

## Decisions

| Decision | Choice | Why |
|---|---|---|
| Keep Windows as the host | Docker Desktop on WSL 2 | Already installed and the least work. It doesn't need to be always on |
| Starting | By hand, not at Windows sign-in | I only run it when needed. Docker Desktop's `AutoStart` stays off |
| Tailscale method | Sidecar container, not `tailscale serve` on Windows | Sharing the Windows device would expose the whole PC, including Adminer and Portainer. The sidecar is its own device |
| Tagged device | `tag:container` | Key expiry is off automatically, and the device doesn't depend on my user account |
| Data on G: | Move Docker's whole disk image | Docker advises named volumes rather than bind mounts for databases. The catch is that every container moves with it |
| Email | Built-in test mailbox (Mailpit) | Enough for now. Real SMTP is a small change later |
| Telemetry | Off | The official Compose file turns it on |
| Restart policy | `unless-stopped` | The default `always` restarts Penpot with Docker even after I stopped it |
| Sizing | Assume up to 7 people, probably just me | See [[#Space]] |

## What was done

### 1. Updated the software

| Program | Before | After |
|---|---|---|
| WSL | 2.5.9.0 | 3.0.3.0 |
| Docker Desktop | 4.42.1 | 4.97.0 |
| Docker Engine | 28.2.2 (CLI) | 29.9.2 |
| Docker Compose | v2.37.1 | v5.6.1 |
| Ubuntu in WSL | 24.04.2 | 24.04.5 |

Two Ubuntu packages were held back (`libgl1-amber-dri`, `libglapi-mesa`). They don't affect Docker or Penpot.

> [!bug] Update problems and fixes
> - **WSL:** `winget upgrade` failed because it needed admin rights (`0x80073d28`). `wsl --update` worked, and it ran for about 5 minutes.
> - **Docker Desktop:** the first install failed with exit code 1 because `com.docker.service` was still running and its file was locked. It rolled back safely. Stopping the service first fixed it:
>   ```powershell
>   Start-Process powershell -Verb RunAs -Wait -ArgumentList '-NoProfile -Command Stop-Service com.docker.service -Force'
>   winget upgrade --id Docker.DockerDesktop --exact --interactive
>   ```
> - **Note:** winget's silent Docker install passes `--accept-license`.

### 2. Moved Docker's disk to G:

1. Quit Docker, copied `%LOCALAPPDATA%\Docker\wsl\disk\docker_data.vhdx` to M:, and checked the SHA-256 hash matched.
2. In Docker Desktop: **Settings → Resources → Advanced → Disk image location**, chose the G: folder, and applied.
3. Checked that nothing was left on C:, and that Portainer and Adminer still worked.

The Docker program itself stays on C:.

### 3. Changed the official Compose file

Downloaded from `https://raw.githubusercontent.com/penpot/penpot/main/docker/images/docker-compose.yaml`.

| Setting | Official default | Mine |
|---|---|---|
| `PENPOT_PUBLIC_URI` | `http://localhost:9001` | `https://penpot.<tailnet>.ts.net` |
| Secret key | `change-this-insecure-key` | Random, from `.env` |
| Database password | `penpot` | Random, from `.env` |
| `disable-secure-session-cookies` | Set | Removed |
| `disable-registration` | Not set | Added after my account existed |
| `PENPOT_TELEMETRY_ENABLED` | `"true"` | `"false"` |
| Frontend port | `9001:8080` | Removed |
| Mailpit port | `8025:8025` | `127.0.0.1:8025:8025` |
| `restart` | `always` | `unless-stopped` |
| Image version | Falls back to 2.18 | 2.18.5 |

`disable-email-verification` stays on because there's no real email. It's only acceptable while Penpot isn't on the public internet.

### 4. Added the Tailscale sidecar

In `docker-compose.yaml`, plus `penpot_tailscale:` under `volumes:`:

```yaml
penpot-tailscale:
  image: tailscale/tailscale:v1.102.2
  hostname: penpot
  restart: unless-stopped
  depends_on:
    - penpot-frontend
  volumes:
    - penpot_tailscale:/var/lib/tailscale
    - ./tailscale:/config:ro
  devices:
    - /dev/net/tun:/dev/net/tun
  cap_add:
    - net_admin
    - net_raw
  networks:
    - penpot
  environment:
    - TS_AUTHKEY=${TS_AUTHKEY:-}
    - TS_AUTH_ONCE=true
    - TS_HOSTNAME=penpot
    - TS_STATE_DIR=/var/lib/tailscale
    - TS_SERVE_CONFIG=/config/serve.json
    - TS_USERSPACE=false
```

`tailscale/serve.json`:

```json
{
  "TCP": { "443": { "HTTPS": true } },
  "Web": {
    "${TS_CERT_DOMAIN}:443": {
      "Handlers": { "/": { "Proxy": "http://penpot-frontend:8080" } }
    }
  },
  "AllowFunnel": { "${TS_CERT_DOMAIN}:443": false }
}
```

Tailscale admin console setup:

1. In **Access controls**, added:
   ```json
   "tagOwners": {
     "tag:container": ["autogroup:admin"]
   },
   ```
2. In **Settings → Keys**, generated an auth key: reusable off, ephemeral off, tag `tag:container`.
3. Pasted it into `.env` for the first start, then cleared it. The device stays logged in using its saved state.

> [!tip] Why these settings matter
> - `TS_AUTH_ONCE=true`: without it, the container logs in again on every start, which fails once the key is gone.
> - The state must persist in its volume, or every restart creates a new device.
> - Mount the serve config as a folder, not a single file, so Tailscale notices changes.
> - The `penpot.<tailnet>.ts.net` name appears in public certificate logs. That reveals the name, not access.

### 5. Started it and locked it down

1. `docker compose -p penpot up -d`, then checked the containers and certificate.
2. Created my account in the browser.
3. Added `disable-registration` and ran `up -d` again. A test sign-up was refused with `registration-disabled`.

### 6. Tested

- **Browser, this PC:** projects, saving, copy and paste, image upload, PNG export.
- **Second device, outside Wi-Fi:** certificate, login, live updates between devices.
- **Tailscale off:** the page doesn't load, so it isn't on the public internet.

## Backups

The script saves `database.dump`, `assets.tar.gz`, `tailscale-state.tar.gz` and `config\` to a dated folder, and keeps the newest 5.

- It only runs when I start it, so nothing is automatic. Penpot must be running.
- Back up before upgrades and after important work.
- Backup folders contain `.env` and Tailscale login state, so keep them private.

**Restore test (2026-11-01):** restored into a temporary, isolated database and volume. Accounts, teams, projects, files and stored objects matched live, and uploaded files matched by checksum. Penpot had very little data at the time.

> [!warning] Not tested
> The exact `RESTORE.md` steps have never been run on live Penpot. Take a fresh backup first if I ever need them.

Restore and upgrade steps are in `RESTORE.md`. The `.env` must be the one from the same backup, because its secret key and database password have to match.

## Upgrading

1. Back up and read the release notes. Go in small version steps.
2. Download the new official Compose file and compare it with `docker-compose.original.yaml`, because new versions can add services.
3. Carry over the changes and set `PENPOT_VERSION` in `.env`.
4. Run `docker compose -p penpot pull`, then `docker compose -p penpot up -d`.

Self-hosted releases arrive a little after Penpot's cloud version.

## Inviting people

1. They need Tailscale. Options are joining my tailnet, or me sharing just the `penpot` device.
2. Invite them from Penpot, then copy the invite link from `http://localhost:8025` and send it to them.
3. Password resets also land in that mailbox, so I'd have to pass those on too.

- The free Tailscale plan allows 6 users and is **non-commercial only**. Figures came from third-party pricing pages in 2026, so check before relying on them.
- Shared-device guests must use the full `penpot.<tailnet>.ts.net` address.
- Without custom access rules, every device on my tailnet can reach Penpot's login page.

## Space

Penpot's docs suggest 50–100 GB of database disk for up to 10 editors.

| Drive | Free on 2026-11-01 |
|---|---|
| G: (Docker) | 11.2 GB |
| M: (backups) | 22.2 GB |

G: is the main limit for real use. The Docker disk file grows with use, generally doesn't shrink by itself, and holds my other containers too. Check with `docker system df`.

## Troubleshooting

| Problem | Check |
|---|---|
| `failed to connect to the docker API` | Docker Desktop isn't running. `docker desktop start` |
| Site won't load | Tailscale connected on this device? `docker compose -p penpot ps` |
| Site loads, but login or saving fails | `docker compose -p penpot logs --tail 50 penpot-backend` |
| Tailscale problems | `docker compose -p penpot exec penpot-tailscale tailscale status` |
| Device shows as `penpot-1` | The old device still exists, or the state was lost. Remove it in the admin console and set a new auth key |

Other containers on this PC: Portainer (`https://localhost:9443`) and Adminer (`http://localhost:8080`, reachable on my network).

## To do

- [ ] Free up space on G: before adding much real work
- [ ] Delete `M:\03 Program\Computer Program\Docker Backup\docker-disk-before-move` (3.85 GB, no longer needed)
- [ ] Confirm Google Drive doesn't sync the G: and M: Docker folders
- [ ] Test sharing the tagged `penpot` device if I add guests
- [ ] Set up real email (SMTP) if passing on invite links gets annoying
- [ ] Find out why Penpot's log says `2.18.5-RC1` when the image tag is 2.18.5

## Sources

- [Penpot Docker guide](https://help.penpot.app/technical-guide/getting-started/docker/)
- [Penpot recommended settings](https://help.penpot.app/technical-guide/getting-started/recommended-settings/)
- [Penpot configuration](https://help.penpot.app/technical-guide/configuration/)
- [Tailscale Docker guide](https://tailscale.com/blog/docker-tailscale-guide)
- [Tailscale Docker parameters](https://tailscale.com/docs/features/containers/docker/docker-params)
- [Tailscale tags](https://tailscale.com/kb/1068/tags)
- [Tailscale auth keys](https://tailscale.com/kb/1085/auth-keys)
- [Tailscale node sharing](https://tailscale.com/kb/1084/sharing)
- [Docker Desktop settings](https://docs.docker.com/desktop/settings-and-maintenance/settings/)
