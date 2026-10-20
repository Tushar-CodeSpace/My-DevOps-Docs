# SonarQube with PostgreSQL — Docker Compose

A persistent, containerized SonarQube Community Build setup backed by PostgreSQL. The configuration stores application and database data under `/data/docker/sonarqube` to keep persistent data off the root filesystem.

## Architecture

- **SonarQube:** Web UI and code analysis server.
- **PostgreSQL 17:** Persistent database for SonarQube.
- **Docker Compose:** Runs and manages both services.
- **Persistent storage:** Host bind mounts under `/data/docker/sonarqube`.
- **Network:** Dedicated Docker bridge network named `devSecOps_network`.

```text
Browser
   |
   | http://localhost:9000
   v
SonarQube container (sonarqube-server)
   |
   | JDBC: jdbc:postgresql://db:5432/sonar
   v
PostgreSQL container (sonarqube-db)

Persistent host data:
  /data/docker/sonarqube/sonarqube_data
  /data/docker/sonarqube/extensions
  /data/docker/sonarqube/logs
  /data/docker/sonarqube/temp
  /data/docker/sonarqube/postgres
```

## Prerequisites

- Linux host with Docker Engine installed.
- Docker Compose v2 (`docker compose`).
- Sufficient memory and disk space for SonarQube and PostgreSQL.
- A writable `/data` filesystem with enough free space.

SonarQube uses Elasticsearch. Configure the required Linux kernel setting on the host:

```bash
sudo tee /etc/sysctl.d/99-sonarqube.conf <<'EOF'
vm.max_map_count=524288
fs.file-max=131072
EOF

sudo sysctl --system
```

Check available storage:

```bash
df -h / /data
```

## Repository layout

```text
My-DevOps-Docs/
├── docker-compose.yml
├── .env                  # Local secret; do not commit
├── .gitignore
├── README.md
├── AUTOMATION.md
└── SECURITY.md
```

## 1. Create the host data directories

The Compose file expects these directories to exist:

```bash
sudo mkdir -p /data/docker/sonarqube/{postgres,sonarqube_data,extensions,logs,temp}
```

The directories must be writable by the users inside the relevant containers. The SonarQube Community image commonly runs as UID `1000`, GID `0`. Set ownership for its directories:

```bash
sudo chown -R 1000:0 \
  /data/docker/sonarqube/sonarqube_data \
  /data/docker/sonarqube/extensions \
  /data/docker/sonarqube/logs \
  /data/docker/sonarqube/temp
```

Do not guess the PostgreSQL UID/GID. Check the runtime user and directory permissions before changing ownership:

```bash
docker compose up -d db
docker compose exec db id
docker compose exec db sh -c 'id postgres; ls -ld /var/lib/postgresql/data'
```

If PostgreSQL fails before `exec` is available, inspect its logs and the official image's expected ownership. Do not recursively change ownership of the database directory while PostgreSQL is running.

## 2. Configure the database password

Create a local `.env` file in the same directory as `docker-compose.yml`:

```bash
cd ~/DevOps/My-DevOps-Docs
nano .env
```

Add a strong, unique password:

```dotenv
SONAR_DB_PASSWORD=replace_with_a_strong_unique_password
```

Restrict permissions:

```bash
chmod 600 .env
```

Make sure `.env` is ignored by Git. Add this to `.gitignore` if it is not already present:

```gitignore
.env
.env.*
!.env.example
```

Never commit real database passwords, SonarQube tokens, or other credentials.

## 3. Validate and start

From the directory containing `docker-compose.yml`:

```bash
docker compose config -q
docker compose pull
docker compose up -d
```

Check service status:

```bash
docker compose ps
```

Follow logs:

```bash
docker compose logs -f db
docker compose logs -f sonarqube
```

The first startup can take several minutes. Wait for SonarQube to finish initialization before opening the UI.

## 4. Access SonarQube

Open:

[http://localhost:9000](http://localhost:9000)

The Compose configuration binds port `9000` to `127.0.0.1`, so the web UI is accessible only from the host by default. If remote access is required, prefer a secured reverse proxy and firewall rules rather than exposing the service publicly without protection.

On a fresh SonarQube instance, the initial credentials are typically:

- Username: `admin`
- Password: `admin`

Change the password when prompted. If this is not a fresh instance, use the credentials already configured for that instance.

## 5. Verify persistent storage

Inspect the host-to-container mounts:

```bash
docker inspect sonarqube-db \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'

docker inspect sonarqube-server \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

Expected database mapping:

```text
/data/docker/sonarqube/postgres -> /var/lib/postgresql/data
```

Expected SonarQube mappings:

```text
/data/docker/sonarqube/sonarqube_data -> /opt/sonarqube/data
/data/docker/sonarqube/extensions -> /opt/sonarqube/extensions
/data/docker/sonarqube/logs -> /opt/sonarqube/logs
/data/docker/sonarqube/temp -> /opt/sonarqube/temp
```

Check disk usage:

```bash
sudo du -sh /data/docker/sonarqube/*
df -h /data
```

## Common commands

Run these commands from the directory containing `docker-compose.yml`.

| Task | Command |
|---|---|
| Start services | `docker compose up -d` |
| Stop services | `docker compose stop` |
| Start stopped services | `docker compose start` |
| Restart services | `docker compose restart` |
| Restart SonarQube only | `docker compose restart sonarqube` |
| View status | `docker compose ps` |
| Follow all logs | `docker compose logs -f` |
| View recent logs | `docker compose logs --tail=100` |
| Pull configured images | `docker compose pull` |
| Validate Compose file | `docker compose config -q` |
| Stop and remove containers/network | `docker compose down` |

`docker compose down` does not remove host data in the configured bind-mounted directories.

**Avoid `docker compose down -v` unless you intentionally want to remove Compose-managed volumes.** Although the intended data paths are bind mounts, deleting volumes can still remove volume-managed data or make recovery harder if the configuration changed. Always verify mounts and back up data before destructive operations.

## Backups

Back up both the PostgreSQL database and the SonarQube data/configuration before upgrades or significant changes. A filesystem copy of a live PostgreSQL data directory is not a reliable database backup by itself.

Example logical database backup:

```bash
mkdir -p ~/sonarqube-backups
docker compose exec -T db pg_dump -U sonar -d sonar \
  > ~/sonarqube-backups/sonar-$(date +%F-%H%M%S).sql
```

Verify the dump file exists and is non-empty:

```bash
ls -lh ~/sonarqube-backups/
```

Protect backup files because they may contain sensitive project and account metadata. Store copies on a separate disk or backup destination. Test restoration periodically.

For a consistent recovery plan, also back up the SonarQube data and extensions directories while the service is stopped, or follow the backup procedure recommended for your SonarQube version.

## Upgrades

Do not use `latest` for a controlled environment. Pin a tested SonarQube image version in `docker-compose.yml`, read its upgrade notes, and verify database compatibility before upgrading.

Before upgrading:

1. Create and verify a PostgreSQL backup.
2. Back up the SonarQube data and extensions as appropriate.
3. Review the version-specific upgrade path and requirements.
4. Pull the chosen image version and upgrade during a maintenance window.
5. Check logs, sign in, and run a test analysis.

Do not assume that downgrading the container image will safely downgrade the database schema.

## Troubleshooting

### SonarQube does not start

```bash
docker compose ps
docker compose logs --tail=200 sonarqube
docker compose logs --tail=200 db
```

Confirm that PostgreSQL is healthy and the database password matches in the `.env` file.

### PostgreSQL permission errors

```bash
docker compose logs --tail=200 db
ls -ld /data/docker/sonarqube/postgres
```

Check the actual runtime user and required directory ownership. Avoid guessing numeric UIDs or changing ownership while the database is running.

### SonarQube reports Elasticsearch bootstrap or kernel errors

Confirm the host setting:

```bash
sysctl vm.max_map_count
```

It should report `524288` or a higher supported value. Apply the host configuration from the prerequisites section if needed.

### Port 9000 is already in use

```bash
sudo ss -ltnp | grep ':9000'
```

Stop the conflicting service or change the host-side port mapping in Compose. For example, `"127.0.0.1:9001:9000"` would expose the UI locally on port `9001`.

### Environment variable is missing

```bash
docker compose config
```

Confirm `.env` is in the same directory as `docker-compose.yml` and contains `SONAR_DB_PASSWORD`. Do not paste the secret into issue trackers or public logs.

## Security notes

- Keep `.env` out of version control.
- Change the default SonarQube administrator password.
- Do not expose PostgreSQL's port publicly; it is intentionally not published to the host.
- Keep Docker images and the host OS updated using a planned maintenance process.
- Use least privilege for tokens and rotate credentials when necessary.
- Restrict access to backups and persistent data directories.
- For remote use, configure TLS and appropriate authentication through a secured reverse proxy.

## Important notes

- This configuration is intended for a persistent development/DevSecOps environment, not a complete high-availability production architecture.
- PostgreSQL persistence does not automatically migrate data from SonarQube's embedded database. A newly configured PostgreSQL instance starts as a separate instance unless a supported migration is performed.
- Data is intended to live under `/data/docker/sonarqube`. Verify the actual mounts with `docker inspect` before assuming storage is configured correctly.