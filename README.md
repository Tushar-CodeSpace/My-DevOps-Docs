# DevSecOps Local Stack: Jenkins, SonarQube, PostgreSQL, and Trivy

A local DevSecOps environment managed with Docker Compose. Jenkins runs CI/CD pipelines, SonarQube analyzes code quality, PostgreSQL stores SonarQube data, and Trivy performs vulnerability and configuration scans.

## Services

| Service | Container | Local URL / access | Purpose |
|---|---|---|---|
| Jenkins | `jenkins-server` | http://localhost:8080 | CI/CD automation |
| SonarQube | `sonarqube-server` | http://localhost:9000 | Static analysis and quality gates |
| PostgreSQL | `sonarqube-db` | Docker network only | SonarQube database |
| Trivy | On-demand container | No web UI | Vulnerability, secret, and misconfiguration scanning |

The Jenkins and SonarQube web ports are bound to `127.0.0.1` and are not exposed to other machines by default. Jenkins and SonarQube share the `devSecOps_network` Docker network. From Jenkins, the SonarQube URL is:

```text
http://sonarqube-server:9000
```

Trivy is an on-demand Compose profile service, so it does not run continuously.

## Storage layout

Persistent data is intended to be stored on the larger `/data` partition:

```text
/data/docker/
├── jenkins/
└── sonarqube/
    ├── postgres/
    ├── sonarqube_data/
    ├── extensions/
    ├── logs/
    └── temp/

/data/docker/trivy/cache/
```

Verify actual mounts with `docker inspect` before assuming data is being written to these paths.

## Repository layout

```text
My-DevOps-Docs/
├── docker-compose.yml
├── .env                  # Local secret; never commit
├── .gitignore
├── README.md
├── AUTOMATION.md
└── SECURITY.md
```

## Prerequisites

- Linux host with Docker Engine and Docker Compose v2.
- Sufficient RAM for Jenkins, SonarQube, and PostgreSQL running together.
- Free disk space on `/data`.
- A strong, unique database password.

SonarQube uses Elasticsearch. Configure the host kernel settings:

```bash
sudo tee /etc/sysctl.d/99-sonarqube.conf <<'EOF'
vm.max_map_count=524288
fs.file-max=131072
EOF

sudo sysctl --system
sysctl vm.max_map_count
df -h / /data
```

## 1. Create persistent directories

Run from any directory:

```bash
sudo mkdir -p /data/docker/jenkins
sudo mkdir -p /data/docker/sonarqube/{postgres,sonarqube_data,extensions,logs,temp}
sudo mkdir -p /data/docker/trivy/cache
```

Jenkins commonly runs as UID/GID `1000:1000` in the configured image:

```bash
sudo chown -R 1000:1000 /data/docker/jenkins
```

The SonarQube image commonly runs as UID `1000`, GID `0`. If the directories are new and empty, set ownership for the SonarQube application directories:

```bash
sudo chown -R 1000:0 \
  /data/docker/sonarqube/sonarqube_data \
  /data/docker/sonarqube/extensions \
  /data/docker/sonarqube/logs \
  /data/docker/sonarqube/temp
```

Do not blindly change ownership of an existing PostgreSQL data directory. Check the database logs, runtime user, and permissions first. Do not recursively change database ownership while PostgreSQL is running.

## 2. Configure the environment

From the repository directory:

```bash
cd ~/DevOps/My-DevOps-Docs
nano .env
```

Add a strong, unique password:

```dotenv
SONAR_DB_PASSWORD=replace_with_a_strong_unique_password
```

Restrict access:

```bash
chmod 600 .env
```

Ensure `.env` is ignored by Git. Recommended `.gitignore` entries:

```gitignore
.env
.env.*
!.env.example
```

Never commit passwords, Jenkins tokens, SonarQube tokens, or scan credentials.

## 3. Validate and start the stack

From `~/DevOps/My-DevOps-Docs`:

```bash
docker compose config -q
docker compose pull
docker compose up -d
docker compose ps
```

Follow logs:

```bash
docker compose logs -f sonarqube
docker compose logs -f db
docker compose logs -f jenkins
```

Use `Ctrl+C` to stop following logs; this does not stop the containers.

## 4. Complete first-time setup

### Jenkins

Open http://localhost:8080.

Retrieve the initial administrator password:

```bash
sudo cat /data/docker/jenkins/secrets/initialAdminPassword
```

Install suggested plugins and create an administrator account. Change the initial password when prompted.

### SonarQube

Open http://localhost:9000. On a fresh instance, the initial credentials are commonly `admin` / `admin`; change the password when prompted.

SonarQube's database is PostgreSQL. The embedded database warning should not appear for a correctly configured new instance using PostgreSQL. This setup does not automatically migrate any data from an earlier embedded-database instance.

## 5. Use Trivy

The Compose file should define Trivy with an on-demand profile similar to:

```yaml
  trivy:
    image: aquasec/trivy:latest
    container_name: trivy-scanner
    profiles:
      - tools
    entrypoint: ["trivy"]
    volumes:
      - ./:/workspace:ro
      - /data/docker/trivy/cache:/root/.cache/
    networks:
      - devSecOps_network
```

Pull the image and check its version:

```bash
docker compose pull trivy
docker compose run --rm trivy --version
```

Run these commands from `~/DevOps/My-DevOps-Docs`. The repository directory is mounted read-only at `/workspace`.

### Scan the repository filesystem

```bash
docker compose run --rm trivy fs \
  --scanners vuln,misconfig,secret /workspace
```

### Scan a container image

For a local image that Trivy can access, the simplest approach is to use the Docker daemon socket. This gives the scanner powerful access to the host Docker daemon, so only do this when needed:

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /data/docker/trivy/cache:/root/.cache/ \
  aquasec/trivy:latest image nginx:latest
```

### Save a scan report

```bash
mkdir -p reports

docker compose run --rm trivy fs \
  --scanners vuln,misconfig,secret \
  --format json \
  --output /workspace/reports/trivy-fs-report.json \
  /workspace
```

The repository mount is read-only in the example Compose configuration, so this command cannot write the report to `/workspace/reports`. To save reports, either use a separate writable reports mount in the Compose service or use a one-off command with an explicit host directory mounted read-write. For example:

```bash
mkdir -p reports
docker run --rm \
  -v "$PWD:/workspace:ro" \
  -v "$PWD/reports:/reports" \
  -v /data/docker/trivy/cache:/root/.cache/ \
  aquasec/trivy:latest fs \
  --scanners vuln,misconfig,secret \
  --format json \
  --output /reports/trivy-fs-report.json \
  /workspace
```

Review reports before sharing them; secret scans can identify sensitive content.

## Useful Docker Compose commands

Run from the directory containing `docker-compose.yml`.

| Task | Command |
|---|---|
| Validate configuration | `docker compose config -q` |
| Start all services | `docker compose up -d` |
| Stop services without removing containers | `docker compose stop` |
| Start stopped services | `docker compose start` |
| Restart all services | `docker compose restart` |
| Restart Jenkins | `docker compose restart jenkins` |
| Restart SonarQube | `docker compose restart sonarqube` |
| Show service status | `docker compose ps` |
| Follow all logs | `docker compose logs -f` |
| Show recent logs | `docker compose logs --tail=100` |
| Pull newer configured images | `docker compose pull` |
| Recreate after image updates | `docker compose up -d` |
| Run Trivy version | `docker compose run --rm trivy --version` |
| Stop and remove containers/network | `docker compose down` |

**Avoid `docker compose down -v` unless you intentionally want to delete Compose-managed volumes.** The current storage configuration uses host bind mounts for persistent data, but always verify mounts and backups before destructive operations.

## Useful diagnostics

### Check containers

```bash
docker compose ps -a
docker ps -a
```

### Check disk usage

```bash
df -h / /data
sudo du -sh /data/docker/*
sudo du -sh /data/docker/sonarqube/*
```

### Verify storage mounts

```bash
docker inspect jenkins-server \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'

docker inspect sonarqube-server \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'

docker inspect sonarqube-db \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

### Check PostgreSQL readiness

```bash
docker exec sonarqube-db pg_isready -U sonar -d sonar
```

### Check listening ports

```bash
sudo ss -ltnp | grep -E ':(8080|9000)\b'
```

### Check Docker networks

```bash
docker network ls
docker network inspect devSecOps_network
```

## Jenkins-to-SonarQube integration

Because Jenkins and SonarQube share the `devSecOps_network` network, configure the SonarQube server URL in Jenkins as:

```text
http://sonarqube-server:9000
```

Create a SonarQube token in SonarQube and store it in Jenkins Credentials. Do not put the token directly in a Jenkinsfile or repository. Install the SonarQube Scanner for Jenkins plugin and configure the server connection in **Manage Jenkins → System**.

The Compose setup does not automatically install Docker CLI, Trivy inside Jenkins, build tools, or pipeline plugins. Add only the build tooling needed by your pipelines. Avoid mounting `/var/run/docker.sock` into Jenkins unless you understand that it grants highly privileged control over the host Docker daemon.

## Backups

Back up PostgreSQL with a logical dump:

```bash
mkdir -p ~/devsecops-backups

docker exec sonarqube-db \
  pg_dump -U sonar -d sonar \
  > ~/devsecops-backups/sonar-$(date +%F-%H%M%S).sql
```

Check that the backup is non-empty:

```bash
ls -lh ~/devsecops-backups/
```

For Jenkins, stop Jenkins before making a filesystem-level copy of its home directory to avoid an inconsistent backup:

```bash
docker compose stop jenkins
sudo tar -czf ~/devsecops-backups/jenkins-home-$(date +%F-%H%M%S).tar.gz \
  -C /data/docker jenkins
docker compose start jenkins
```

Back up to a separate disk or backup destination as well. A backup on the same disk does not protect against disk failure. Test restoration periodically.

## Updating images

Review release notes and compatibility requirements before upgrades. The `latest` tags can move forward and may introduce breaking changes. For controlled environments, pin tested versions and upgrade during a maintenance window.

```bash
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail=100
```

Back up Jenkins and PostgreSQL before upgrading. Do not assume that downgrading an image will safely downgrade persistent data or database schemas.

## Security notes

- Keep the `.env` file out of Git.
- Change default administrator credentials.
- Do not publish the PostgreSQL port.
- Keep Jenkins and SonarQube bound to localhost unless remote access is intentionally secured.
- Use least-privilege credentials and rotate tokens when needed.
- Treat Docker socket access as highly privileged.
- Restrict access to backups, logs, and scan reports.
- Scan reports can contain sensitive file paths and secret findings.
- Avoid using this local stack as a high-availability production deployment without additional hardening, monitoring, and tested backup/restore procedures.