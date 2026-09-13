# WordPress Docker Setup

A containerized WordPress + MySQL environment, orchestrated with Docker Compose for easy, reproducible deployment.

## Table of Contents
- [Description](#description)
- [Repository Contents](#repository-contents)
- [Quickstart](#quickstart)
- [Usage](#usage)
  - [Configuration](#configuration)
  - [Changing the Port](#changing-the-port)
  - [Changing the Table Prefix](#changing-the-table-prefix)
  - [Managing Data](#managing-data)
  - [Restart Behavior](#restart-behavior)
- [Deployment](#deployment)

## Description
This repository provides a containerized WordPress environment built with Docker Compose. It defines two services `wordpress` and `db` (MySQL) running on a shared Docker network. Database contents are persisted using a named Docker volume, so data is not lost on container restarts. The purpose of this repository is to offer a reproducible, easily configurable WordPress setup that can be deployed on a local machine or a cloud VM with minimal adjustments. Both services use official pre-built images from Docker Hub (`wordpress:latest` and `mysql:8.0`) — no custom Dockerfile or build step is required.

## Repository Contents
| File | Purpose |
|---|---|
| `docker-compose.yaml` | Defines the `wordpress` and `db` services, network, and volume |
| `.env.example` | Template for required environment variables (copy to `.env`) |
| `.gitignore` | Excludes irrelevant files and sensitive data (e.g. `.env`) from version control |
| `README.md` | Project documentation (this file) |

## Quickstart
**Requirements:**
- Docker
- Docker Compose

**Steps:**
```bash
git clone <repository-url>
cd <repository-folder>
cp .env.example .env
# edit .env and set your own values
docker compose up -d
```
Then open `http://localhost:8080` (or `http://<vm-ip>:8080` on a cloud VM) in your browser and complete the WordPress setup wizard.

## Usage

### Configuration
All configurable values are passed via environment variables defined in `.env` (based on `.env.example`):

| Variable | Description | Default |
|---|---|---|
| `WORDPRESS_DB_NAME` | Name of the WordPress database | `wordpress` |
| `WORDPRESS_DB_USER` | Database username | `wordpress_user` |
| `WORDPRESS_DB_PASSWORD` | Database password | — |
| `WORDPRESS_TABLE_PREFIX` | Prefix used for WordPress DB tables | `wp_` |
| `MYSQL_ROOT_PASSWORD` | Root password for the MySQL container | — |

No credentials are hardcoded in `docker-compose.yaml`; only variable references (`${VARIABLE_NAME}`) are used.

### Changing the Port
By default, WordPress is exposed on host port `8080`. To use a different port, edit the `ports` section under the `wordpress` service in `docker-compose.yaml`:
```yaml
ports:
  - "8081:80"
```

### Changing the Table Prefix
Set `WORDPRESS_TABLE_PREFIX` in `.env` before the first startup (e.g. `wp_` → `custom_`). This only takes effect on a fresh database; changing it afterward requires a manual database migration.

### Managing Data
Database contents are stored in the named volume `db_data`. To fully reset the WordPress installation, stop the containers and remove the volume:
```bash
docker compose down
docker volume rm <project-folder>_db_data
docker compose up -d
```

### Restart Behavior
Both services are configured with `restart: always`, so they automatically restart after an unexpected crash or a host reboot. Note: a manual `docker stop` or `docker kill` on the container itself is treated by Docker as an intentional stop and will **not** trigger an automatic restart — this only applies to crashes.

To properly simulate a crash (not a manual stop), kill the main process *inside* the container instead of stopping the container itself:
```bash
docker exec wordpress kill -9 1
docker ps
```
The `wordpress` container should show a new `CREATED`/`STATUS` timestamp shortly after, confirming it was automatically restarted.

**Useful commands:**
```bash
docker compose up -d        # start containers in the background
docker compose ps           # check container status
docker compose logs -f      # follow logs
docker compose down         # stop and remove containers
docker compose restart      # restart both services
```

## Deployment
Steps to deploy this setup on a cloud VM (e.g. a fresh Ubuntu instance):

1. **Install Docker and Docker Compose** on the VM (e.g. via the official Docker install script or your cloud provider's package manager):
   ```bash
   curl -fsSL https://get.docker.com | sh
   ```
2. **Clone the repository** onto the VM:
   ```bash
   git clone <repository-url>
   cd <repository-folder>
   ```
3. **Create the environment file** from the template and fill in your own values:
   ```bash
   cp .env.example .env
   nano .env
   ```
4. **Open port 8080** in the VM's firewall / cloud security group so it is reachable from outside (exact command depends on the provider, e.g. for `ufw`):
   ```bash
   sudo ufw allow 8080/tcp
   ```
5. **Start the containers** in the background:
   ```bash
   docker compose up -d
   ```
6. **Verify both containers are running:**
   ```bash
   docker compose ps
   ```
7. **Access WordPress** in a browser at `http://<vm-public-ip>:8080` and complete the setup wizard.

**Updating the deployment** (after pulling new changes):
```bash
git pull
docker compose down
docker compose up -d
```
