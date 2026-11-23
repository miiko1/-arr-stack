# arr-stack

A Docker Compose stack for managing and playing a self-hosted media library. It includes Prowlarr, Radarr, Sonarr, Jellyfin, and Seerr, with immutable image references, health checks, persistent named volumes, log rotation, and a shared reverse-proxy network.

## Services

| Service | Purpose | Host port | Container port |
| --- | --- | ---: | ---: |
| Prowlarr | Indexer management for Radarr and Sonarr | `49152` | `9696` |
| Radarr | Movie management | `49153` | `7878` |
| Sonarr | TV series management | `49154` | `8989` |
| Seerr | Media requests and discovery | `49155` | `5055` |
| Jellyfin | Media server and player | `8096` | `8096` |

Prowlarr, Radarr, Sonarr, and Seerr bind to `127.0.0.1` by default. Jellyfin binds to all host interfaces so media clients can connect. See [Accessing the services](#accessing-the-services) before exposing the admin interfaces.

A download client such as qBittorrent or SABnzbd is **not included**. Configure one separately before expecting Radarr or Sonarr to download media.

## Stack defaults

- Images are pinned to immutable SHA-256 digests.
- Persistent application configuration uses named volumes.
- All services have HTTP health checks, restart policies, PID limits, and graceful shutdown timeouts.
- Logs use Docker's `local` driver with size-based rotation.
- All containers use `no-new-privileges`.
- Seerr additionally has a read-only root filesystem, dropped capabilities, and a small writable `/tmp`.
- A private `media` network connects the applications, while the external `proxy` network allows a reverse proxy such as Caddy to reach them by service name.
- CPU and memory limits are intentionally unset. Measure normal and peak usage with `docker stats` before adding limits appropriate for the host.

## Prerequisites

- Linux with Docker Engine and Docker Compose v2.
- A Synology NFS or CIFS share containing `movies`, `shows`, and `downloads` directories.
- An external Docker volume named `Synology` that mounts that share.
- An external Docker network named `proxy` if Caddy or another containerized reverse proxy is used.
- UID and GID `1000`, or alternative values with read/write access to the NAS share.
- Enough storage for application configuration, logs, metadata, and media.

The NAS volume should expose this layout inside Radarr, Sonarr, and Jellyfin:

```text
/data/
├── downloads/
├── movies/
└── shows/
```

Using one `/data` mount and identical paths in every application avoids cross-mount copies and supports hardlinks and atomic moves when the underlying filesystem permits them.

## Deployment

### 1. Clone the repository

```bash
git clone https://gitea.miikonet.net/miikonet/arr-stack.git
cd arr-stack
```

### 2. Create the Synology volume

The Compose file does not create the external NAS volume. Create it using the actual server address, export path, protocol, and permissions for your Synology.

Example for NFS:

```bash
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.0.2.10,rw,nfsvers=4 \
  --opt device=:/volume1/media \
  Synology
```

Replace `192.0.2.10` and `/volume1/media` with the real values. Ensure the NFS export allows the Docker host and grants UID/GID `1000` the required access.

```bash
docker volume inspect Synology
```

Do not substitute a plain `docker volume create Synology`: that creates an empty local volume rather than mounting the NAS.

### 3. Create the proxy network

If the reverse-proxy network does not already exist:

```bash
docker network inspect proxy >/dev/null 2>&1 || docker network create proxy
```

The reverse proxy must join this same network. If no containerized reverse proxy is needed, the external network declaration can be removed together with `proxy` from the shared service networks.

### 4. Configure optional overrides

Compose reads non-secret settings from a local `.env` file. All settings have defaults, so this file is optional.

```dotenv
PUID=1000
PGID=1000
TZ=Australia/Perth

ADMIN_BIND_ADDRESS=127.0.0.1
JELLYFIN_BIND_ADDRESS=0.0.0.0

SYNOLOGY_VOLUME=Synology
PROXY_NETWORK=proxy
SEERR_LOG_LEVEL=info
```

Individual host ports can be changed with `PROWLARR_PORT`, `RADARR_PORT`, `SONARR_PORT`, `SEERR_PORT`, and `JELLYFIN_PORT`.

Do not commit `.env`, application API keys, NAS credentials, or other secrets. This Compose file does not require secrets during initial container startup; credentials entered in the applications are stored in their configuration volumes.

### 5. Validate and start

```bash
docker compose config --quiet
docker compose pull
docker compose up -d --wait
docker compose ps
```

Inspect the resolved configuration before deployment with `docker compose config`. Avoid publishing resolved output if sensitive values were supplied through interpolation.

## Accessing the services

With the default bindings, use these addresses on the Docker host:

| Service | Address |
| --- | --- |
| Prowlarr | `http://127.0.0.1:49152` |
| Radarr | `http://127.0.0.1:49153` |
| Sonarr | `http://127.0.0.1:49154` |
| Seerr | `http://127.0.0.1:49155` |
| Jellyfin | `http://DOCKER_HOST_IP:8096` |

For a remote browser, access admin interfaces through the reverse proxy or an SSH tunnel. Example for Seerr:

```bash
ssh -L 49155:127.0.0.1:49155 user@docker-host
```

Then open `http://127.0.0.1:49155` locally.

To expose all admin ports directly on the LAN instead, set the following in `.env` and recreate the services:

```dotenv
ADMIN_BIND_ADDRESS=0.0.0.0
```

Only do this on a trusted network with authentication and firewall rules configured. A host port is not required for container-to-container or reverse-proxy access.

### Caddy

Because Caddy joins the external `proxy` network, it can use service names and container ports directly:

```caddyfile
seerr.example.com {
    reverse_proxy seerr:5055
}
```

The same pattern applies to `prowlarr:9696`, `radarr:7878`, `sonarr:8989`, and `jellyfin:8096`. Configure DNS, TLS, and access restrictions outside this Compose file.

## Initial application setup

Use container paths for libraries and downloads, not the host's NAS mount path.

| Setting | Value |
| --- | --- |
| Radarr movie root folder | `/data/movies` |
| Sonarr series root folder | `/data/shows` |
| Completed downloads | `/data/downloads` |
| Jellyfin movie library | `/data/movies` |
| Jellyfin TV library | `/data/shows` |

Use service names when connecting applications:

| Destination | Internal URL |
| --- | --- |
| Prowlarr | `http://prowlarr:9696` |
| Radarr | `http://radarr:7878` |
| Sonarr | `http://sonarr:8989` |
| Jellyfin | `http://jellyfin:8096` |
| Seerr | `http://seerr:5055` |

Recommended setup order:

1. Create strong administrator credentials in each application.
2. Add the movie and TV libraries in Jellyfin.
3. Configure the download client in Radarr and Sonarr using `/data/downloads` as the completed-download path.
4. Add indexers to Prowlarr and connect the Radarr and Sonarr applications.
5. Connect Seerr to Jellyfin, Radarr, and Sonarr.
6. Test a request, download, import, and Jellyfin playback before relying on automation.

If the download client runs outside this stack, make its paths match `/data/downloads` or configure Radarr/Sonarr remote path mappings. If it is added to this stack, mount the same `synology` volume at `/data`.

Jellyfin hardware transcoding is not enabled by this file. Where supported by the host, this commonly requires a device mapping such as `/dev/dri:/dev/dri` and appropriate group permissions.

## Persistent data

With the default project name, Compose creates these volumes:

| Volume | Container path | Contents |
| --- | --- | --- |
| `media-stack_prowlarr_config` | `/config` | Prowlarr configuration and database |
| `media-stack_radarr_config` | `/config` | Radarr configuration and database |
| `media-stack_sonarr_config` | `/config` | Sonarr configuration and database |
| `media-stack_jellyfin_config` | `/config` | Jellyfin configuration, metadata, and playback data |
| `media-stack_seerr_config` | `/app/config` | Seerr configuration and database |
| `Synology` | `/data` | External media and downloads |

A different Compose project name changes the prefix of managed volumes. The external NAS volume keeps its configured name.

### Migrating existing bind-mounted configuration

Named volumes start empty. Before replacing an older deployment that used `./data/...` or `./config`, stop the old containers and copy the existing data into empty target volumes. Back up the source directories first.

The following helper assumes the old paths below, default volume names, and UID/GID `1000`. Run it from the directory containing the old deployment and verify every source and target before proceeding.

```bash
copy_config() {
  docker run --rm \
    -v "$1:/source:ro" \
    -v "$2:/target" \
    alpine:3 sh -c 'cp -a /source/. /target/ && chown -R 1000:1000 /target'
}

copy_config "$PWD/data/prowlarr" media-stack_prowlarr_config
copy_config "$PWD/data/radarr" media-stack_radarr_config
copy_config "$PWD/data/jellyfin" media-stack_jellyfin_config
copy_config "$PWD/data/sonarr" media-stack_sonarr_config
copy_config "$PWD/config" media-stack_seerr_config
```

Adjust the ownership command if `PUID` or `PGID` differs. Do not run the copy into volumes that already contain a newer configuration. Media on the external `Synology` volume does not need migration.

## Updates and rollback

The pinned digests make deployments reproducible. Running `docker compose pull` alone does not select a newer application release.

To update:

1. Review the application's release notes and backup its configuration.
2. Replace the relevant digest in `compose.yaml`, or set `PROWLARR_IMAGE`, `RADARR_IMAGE`, `SONARR_IMAGE`, `JELLYFIN_IMAGE`, or `SEERR_IMAGE` to a reviewed image reference in `.env`.
3. Validate and deploy:

   ```bash
   docker compose config --quiet
   docker compose pull
   docker compose up -d --wait
   docker compose ps
   ```

4. Check logs and exercise the affected application.

To roll back, restore the previous image reference. Application database migrations are not always reversible, so a full rollback may also require restoring the matching configuration backup.

## Backup and restore

Back up every configuration volume and test restores before they are needed. Keep encrypted, off-host copies with an appropriate retention policy; these configurations contain application credentials and API keys.

- Prefer each application's built-in backup facility for application-consistent exports.
- Stop the relevant application before taking a filesystem-level copy of a live SQLite database or configuration volume.
- Back up media and download data according to the NAS's own snapshot and backup policy.
- Record the image digest alongside each backup so the matching application version is known.

`docker compose down` preserves named volumes. **Do not routinely run `docker compose down -v`**, because it deletes the stack's managed configuration volumes.

## Routine operations

```bash
# Show container and health status
docker compose ps

# Follow logs for all services or one service
docker compose logs -f --tail=100
docker compose logs -f --tail=100 seerr

# Restart one service
docker compose restart sonarr

# Stop containers while preserving volumes
docker compose stop

# Recreate after changing .env or compose.yaml
docker compose up -d --wait

# Remove containers and managed networks while preserving volumes
docker compose down

# Inspect resource usage before setting CPU or memory limits
docker stats
```

An unhealthy status alone does not make plain Docker Compose restart a container. The restart policy handles process exits; monitoring should alert on failed health checks and repeated restarts.

## Troubleshooting

### `no such volume: Synology`

Create the external volume with the correct NFS or CIFS options, then verify it with `docker volume inspect Synology`.

### `network proxy ... not found`

Create the network with `docker network create proxy`, or set `PROXY_NETWORK` to the existing reverse-proxy network.

### Admin UI refuses connections from another machine

The default admin binding is local-only. Use the reverse proxy or an SSH tunnel, or deliberately set `ADMIN_BIND_ADDRESS=0.0.0.0` and recreate the affected service.

### Seerr reports `EACCES` for `/app/config/logs`

A migrated configuration may have incorrect ownership. Assuming the default volume name and UID/GID `1000`:

```bash
docker run --rm \
  -v media-stack_seerr_config:/config \
  alpine:3 chown -R 1000:1000 /config
docker compose restart seerr
```

Do not work around this with world-writable permissions or by running Seerr as root.

### Imports copy slowly instead of using hardlinks

Confirm that the download client, Radarr, and Sonarr all see downloads and libraries beneath the same `/data` mount. Check UID/GID permissions and whether the NAS protocol and filesystem support hardlinks.

### A container is unhealthy or repeatedly restarting

Inspect its current state and logs:

```bash
docker compose ps
docker compose logs --tail=200 SERVICE_NAME
```

Replace `SERVICE_NAME` with `prowlarr`, `radarr`, `sonarr`, `jellyfin`, or `seerr`.

## License

See [LICENSE](LICENSE).
