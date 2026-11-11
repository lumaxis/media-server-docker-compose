# Media server Docker Compose

This Compose project runs Plex, media automation, download, request, Trakt sync, monitoring, update, Cloudflare Tunnel, and Homebridge services on Synology DSM.

## Validate

Validate the file before every deployment or synchronization:

```sh
docker compose config --quiet
git diff --check
```

On the Synology, validate from the live file's directory so relative bind mounts resolve correctly:

```sh
cd /volume1/homes/lukas
/usr/local/bin/docker-compose -f docker-compose.yml.candidate config --quiet
# Container Manager alternative:
/var/packages/ContainerManager/target/usr/bin/docker compose \
  -f docker-compose.yml.candidate config --quiet
```

## Update policy

Watchtower runs every Saturday at 05:00 Europe/Berlin, removes superseded images, and only updates containers labeled `com.centurylinklabs.watchtower.enable=true`. Every service in this project is intentionally opted in and uses an explicit rolling image tag. Recyclarr (`8`) and Seerr (`v3`) track a major-version tag, so Watchtower applies minor and patch releases but never a new major version. PlexTraktSync (`0.35`) is pre-1.0 and tracks a minor-version tag, so Watchtower applies only its patch releases. Bump those tags manually after reviewing the release notes.

Rolling updates can include unattended database migrations. Keep tested backups of every `./config/<service>` directory and verify recovery before relying on automatic updates.

Watchtower itself carries the same enable label so its weekly run can update the updater too.

## Security and storage

- Published ports and host-networked services are not protected by Compose. DSM firewall rules and router policy must restrict access appropriately.
- Seerr publishes port 5055 on `127.0.0.1` only. It is reachable publicly through the Cloudflare Tunnel, not on LAN addresses. Tailscale on DSM uses userspace networking, so tailnet devices still reach it through loopback.
- Plex media mounts intentionally remain writable so Plex can manage media as currently configured.
- Watchtower requires the Docker socket to replace containers. Socket access is effectively root-equivalent on the Docker host; mounting it read-only would not remove that risk.
- `config/watchtower/config.json` can contain registry credentials. Keep it out of source control and mount it read-only.
- Each service uses bounded `json-file` logging: three files of 10 MB each.

## Cloudflare Tunnel

The `cloudflared` service connects the remotely managed Cloudflare Tunnel `plex-only-synology`. It is the only public path to Plex and Seerr: the router forwards no ports. Ingress rules and DNS live in the Cloudflare dashboard, not in this repository.

- The connector uses host networking so it can reach Plex on `127.0.0.1:32400` and Seerr on `localhost:5055`.
- The connector token is read from `./config/cloudflared/token`, which is gitignored and must never be committed. It is passed as a file, so it never appears in the Compose file, the container environment or process listings. Keep the file owned by `lukas:users` (`1033:100`, the container user) with mode `0600`, and the directory with mode `0700`, so Synology ACLs inherited from `config/` do not grant other users read access.
- Metrics and the `/ready` endpoint listen on `127.0.0.1:20242`. The healthcheck runs `cloudflared tunnel ready` against it.
- With a single connector, Watchtower's weekly update briefly interrupts public access while the container is replaced.

## Trakt sync

Trakt's own Plex sync can't reach this server because Remote Access is off, so it only receives webhooks and misses watches. The `plextraktsync` service runs [PlexTraktSync](https://github.com/Taxel/PlexTraktSync) every two hours and pushes the server owner's watched status and ratings from Plex to Trakt. It never writes to Plex, and with `rating_priority: none` it only adds ratings that are missing on Trakt.

- The service runs as `lukas:users` (`1033:100`) without published ports or the Docker socket. It only connects out, to Plex on its LAN `plex.direct` URL and to the Trakt API.
- `./config/plextraktsync` is gitignored and must never be committed. `servers.yml` holds a Plex device token (listed as "PlexTraktSync" under Plex's authorized devices) and `.pytrakt.json` holds the Trakt OAuth tokens for the "PlexTraktSync NAS" Trakt app. Keep the directory at mode `0700` and the files at `0600`.
- Don't run PlexTraktSync's `watch` mode: Trakt's webhook already scrobbles live plays, so both would record them.
- To set up from scratch, write `servers.yml` and `config.yml`, then run `docker compose run --rm plextraktsync trakt-login` once.

## Synchronize without restarting containers

Copy the committed file to a temporary path beside the live file, validate it there, and atomically replace only the Compose definition:

```sh
scp -O docker-compose.yml \
  synology:/volume1/homes/lukas/docker-compose.yml.candidate
ssh synology '
  set -eu
  cd /volume1/homes/lukas
  /usr/local/bin/docker-compose -f docker-compose.yml.candidate config --quiet
  mv docker-compose.yml.candidate docker-compose.yml
'
```

This workflow does not pull images or recreate, start, stop, or restart containers. Apply the definition separately during an intentional maintenance window.
