# Media server Docker Compose

This Compose project runs Plex, media automation, download, request, monitoring, update, and Homebridge services on Synology DSM.

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

Watchtower runs every Saturday at 05:00 Europe/Berlin, removes superseded images, and only updates containers labeled `com.centurylinklabs.watchtower.enable=true`. Every service in this project is intentionally opted in and uses an explicit rolling image tag.

Rolling updates can include unattended database migrations. Keep tested backups of every `./config/<service>` directory and verify recovery before relying on automatic updates.

## Security and storage

- Published ports and host-networked services are not protected by Compose. DSM firewall rules and router policy must restrict access appropriately.
- Plex media mounts intentionally remain writable so Plex can manage media as currently configured.
- Watchtower requires the Docker socket to replace containers. Socket access is effectively root-equivalent on the Docker host; mounting it read-only would not remove that risk.
- `config/watchtower/config.json` can contain registry credentials. Keep it out of source control and mount it read-only.
- Each service uses bounded `json-file` logging: three files of 10 MB each.

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
