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
This repository provides a containerized WordPress environment built with Docker Compose. It defines two services `wordpress` and `db` (MySQL) running on a shared Docker network. Both application files (`/var/www/html`) and database contents are persisted using named Docker volumes, so data is not lost on container restarts. The purpose of this repository is to offer a reproducible, easily configurable WordPress setup that can be deployed on a local machine or a cloud VM with minimal adjustments. Both services use official pre-built images from Docker Hub (`wordpress:7.1.0-apache` and `mysql:8.0`) — no custom Dockerfile or build step is required.

## Repository Contents
| File | Purpose |
|---|---|
| `docker-compose.yaml` | Defines the `wordpress` and `db` services, network, and volumes |
| `.env.example` | Template for required environment variables (copy to `.env`) |
| `.gitignore` | Excludes irrelevant files and sensitive data (e.g. `.env`) from version control |
| `README.md` | Project documentation (this file) |

## Quickstart
**Requirements:**
- Docker
- Docker Compose

**Steps:**
```bash
git clone https://github.com/PariKhavari/Wordpress.git
cd Wordpress
cp .env.example .env
# edit .env and set your own values
docker compose up -d
```
Then open `http://localhost:8080` (or `http://<vm-ip>:8080` on a VM) in your browser and complete the WordPress setup wizard.

> [!TIP]
> Keep a note of the admin username and password you set during the WordPress setup wizard — you'll need them for the login test.

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
| `PORT` | Host port WordPress is exposed on | `8080` |

No credentials are hardcoded in `docker-compose.yaml`; only variable references (`${VARIABLE_NAME}`) are used.

> [!IMPORTANT]
> Never commit your actual `.env` file to the repository. It is excluded via `.gitignore` — only `.env.example` (with placeholder values) should be tracked.

### Changing the Port
By default, WordPress is exposed on host port `8080`. To use a different port, set `PORT` in your `.env` file (e.g. `PORT=8081`) — no changes to `docker-compose.yaml` are required.

### Changing the Table Prefix
Set `WORDPRESS_TABLE_PREFIX` in `.env` before the first startup (e.g. `wp_` → `custom_`). This only takes effect on a fresh database; changing it afterward requires a manual database migration.

### Managing Data
WordPress files (themes, plugins, uploads) are stored in the named volume `wordpress_data`, and database contents in `db_data`. To fully reset the installation, stop the containers and remove both volumes:
```bash
docker compose down
docker volume rm <project-folder>_wordpress_data <project-folder>_db_data
docker compose up -d
```

> [!WARNING]
> Removing these volumes permanently deletes all WordPress content (posts, users, settings, uploaded files). This action cannot be undone.

### Restart Behavior
Both services are configured with `restart: unless-stopped`, so they automatically restart whenever their main process exits — whether due to a crash or a host reboot — but not after a manual stop. Two things to keep in mind when testing this:

- A manual `docker stop`/`docker kill` **on the container itself** is treated by Docker as an intentional stop and will **not** trigger an automatic restart.
- Sending `SIGKILL` to PID 1 *from inside* the container's own PID namespace (e.g. via `docker exec ... kill -9 1`) is ignored by the kernel — this is standard Linux behavior protecting PID 1. Use `SIGTERM` instead, which Apache handles and exits on cleanly.

> [!NOTE]
> This PID 1 signal-handling behavior is a general Linux/container characteristic, not specific to this project's setup.

To properly simulate a crash and verify the restart:
```bash
docker exec wordpress kill -TERM 1
docker inspect --format='{{.State.StartedAt}}' wordpress
docker ps
```
Compare the `StartedAt` timestamp before and after — a newer timestamp confirms the container was automatically restarted.

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
   git clone -b <feature-name> https://github.com/<dein-username>/<repo-name>.git
   cd <repository-folder>
   ```
3. **Create the environment file** from the template and fill in your own values:
   ```bash
   cp .env.example .env
   nano .env
   ```
4. **Open port 8080** (or your configured `PORT`) in the VM's firewall / cloud security group so it is reachable from outside (exact command depends on the provider, e.g. for `ufw`):
   ```bash
   sudo ufw allow 8080/tcp
   ```

   > [!CAUTION]
   > Opening a port to the public internet exposes the service to anyone who finds it. Restrict access (e.g. by source IP) where possible, especially before entering real credentials.

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
