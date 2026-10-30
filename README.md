# WordPress with Docker Compose

This repository runs WordPress 7.1.4 and MySQL 8.4.13 with Docker Compose. It uses official images, keeps WordPress and database data in named volumes, and restarts both containers after an unexpected exit. No custom Docker image or Dockerfile is needed.

## Table of contents

- [Prerequisites](#prerequisites)
- [Quickstart](#quickstart)
- [Project structure](#project-structure)
- [Usage](#usage)
  - [Configuration](#configuration)
  - [Images and startup](#images-and-startup)
  - [Run and manage WordPress](#run-and-manage-wordpress)
  - [Persistence and restarts](#persistence-and-restarts)
  - [Testing](#testing)
  - [Deployment](#deployment)
- [Troubleshooting](#troubleshooting)

## Prerequisites

- Docker with the Compose plugin (Docker Desktop works for local testing).
- An available TCP port `8080` on the host.
- For deployment, a cloud VM with Docker and access to port `8080` through its firewall and network rules.

## Quickstart

Clone the repository and change into it first. All commands use Bash (on Windows: Git Bash or WSL).

1. Clone the repository and change into it:

   ```bash
   git clone https://github.com/FriggemannMichael/wordpress.git
   cd wordpress
   ```

2. Copy the example environment file.

   ```bash
   cp .env.example .env
   ```

3. Open `.env` and replace **both** password placeholders with different, strong passwords. Keep `DB_NAME=wordpress` and `DB_USER=wordpress` unless you also change the matching MySQL settings in `docker-compose.yaml`.
4. Validate the Compose configuration, then start the services:

   ```bash
   docker compose config --quiet
   docker compose up -d
   docker compose ps
   ```

5. Open `http://<your_ip>:8080` and complete the WordPress installer. Create a strong administrator password and keep it private. The installer creates the administrator account in the database; it is separate from the database credentials in `.env`.

> [!CAUTION]
> `.env` contains database passwords and is ignored by Git. Never commit it, replace `.env.example` with real passwords, or share administrator credentials in an issue or log.

The first start downloads the images and initializes both named volumes. It can take a little longer than subsequent starts.

## Project structure

| Path                            | Purpose                                                                                                                        |
|---------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| `docker-compose.yaml`           | Defines the `wordpress` and `db` services, images, port, environment references, network, named volumes, and restart policies. |
| `.env.example`                  | Lists the required local variables with password placeholders. Copy it to `.env` before starting Compose.                      |
| `.gitignore`                    | Excludes the local `.env`, virtual environment, and IDE files from Git.                                                        |
| `docs/Wordpress Checkliste.pdf` | Project submission requirements and verification checklist.                                                                    |
| `README.md`                     | Setup, operation, configuration, testing, and deployment instructions.                                                         |

The local `.env` file is not tracked. Docker manages `wp-data` and `db-data` outside the repository; there is no generated WordPress directory to commit.

## Usage

### Configuration

Compose reads these variables from `.env` in the repository root:

| Variable           | Expected value                   | Effect                                                                                    |
|--------------------|----------------------------------|-------------------------------------------------------------------------------------------|
| `DB_NAME`          | `wordpress`                      | Database name used by WordPress. It must match `MYSQL_DATABASE` in `docker-compose.yaml`. |
| `DB_USER`          | `wordpress`                      | Database user used by WordPress. It must match `MYSQL_USER` in `docker-compose.yaml`.     |
| `DB_PASSWORD`      | A strong private value           | Password shared by the WordPress database connection and the MySQL application user.      |
| `DB_ROOT_PASSWORD` | A different strong private value | Password for the MySQL root account. WordPress does not use this account.                 |

`WORDPRESS_DB_HOST` is `db:3306`: `db` is the Compose service name, resolved on the shared `wordpress-network`. The database has no published host port. To publish WordPress on another host port, change the left side of `8080:80` under `wordpress.ports`, then use that port in the browser and VM firewall rules.

The MySQL service currently sets `MYSQL_DATABASE` and `MYSQL_USER` to `wordpress` directly. If you change `DB_NAME` or `DB_USER`, update those two MySQL settings to the same values **before the first database initialization**. MySQL initialization variables do not modify existing accounts or databases when `db-data` already contains data.

### Images and startup

Compose pulls `wordpress:7.1.4-php8.4-apache` and `mysql:8.4.13`; it does not build an image. The WordPress service is published on host port `8080`, and both services share the `wordpress-network`. The `depends_on` setting starts `db` first, but initial database setup can still take time.

The local WordPress image was checked with PHP **8.4.27**. The PHP version must be checked again in the image pulled on the VM before the site goes live:

```bash
docker compose exec wordpress php -v
```

For the October 2026 PHP security release, PHP 8.4 must be **8.4.27 or newer**. A WordPress image tag names the PHP minor version, not the exact patch version. See the [PHP security announcement](https://www.php.net/archive/2026.php#2026-10-21-5).

### Run and manage WordPress

Run each command separately from the repository root:

```bash
docker compose up -d
docker compose ps
docker compose logs -f wordpress
docker compose logs -f db
docker compose restart
docker compose down
```

Press `Ctrl+C` to stop following a log; the containers continue running. `docker compose down` removes the containers and network but keeps the named volumes. Start them again with `docker compose up -d`.

### Persistence and restarts

The `wp-data` volume stores WordPress files at `/var/www/html`. The `db-data` volume stores MySQL data at `/var/lib/mysql`. Both services use `restart: unless-stopped`, so Docker restarts them after an internal process exit while Docker is running. A deliberately stopped container stays stopped until you start it again.

To verify persistence, create a WordPress draft, run `docker compose down` followed by `docker compose up -d`, then confirm that you can still sign in and find the draft.

> [!CAUTION]
> Do not add `-v` to `docker compose down` when you need to keep the site. `down -v` removes the named volumes and can delete the WordPress files and database. Back up both volumes before any destructive change.

### Testing

Check the Compose structure and running services:

```bash
docker compose config --quiet
docker compose ps
```

Open `http://<your_ip>:8080`, sign in at `/wp-admin`, and navigate the dashboard. Confirm that the WordPress container reports PHP 8.4.27 or newer. Test a saved draft after recreating the containers as described above.

Locally, both services were also confirmed to restart automatically after their main processes exited inside the containers. A manual `docker kill` is not a valid test of `unless-stopped`, because Docker treats it as a manual stop.

### Deployment

Deploy manually from the repository on the cloud VM. Install Docker and Compose, copy `.env.example` to a new **VM-local** `.env`, and set new strong passwords there. Validate the Compose file, then run `docker compose up -d`. Check the PHP patch version inside the VM container before making the site publicly available. Then allow inbound TCP port `8080` in the VM firewall and cloud network rules.

Visit `http://<VM_HOST>:8080` and complete the WordPress installer on the VM. Verify the administrator login, navigation, persistence after a restart, and automatic container recovery. The local Docker volumes and local WordPress installation are **not** transferred to the VM by Git.

> [!CAUTION]
> Port `8080` serves HTTP. Keep real passwords, SSH keys, tokens, administrator usernames, and sensitive VM details out of this repository. Do not treat the local PHP check as proof of the version running on the VM.

There is no automated deployment workflow in this repository. If you add extra files or features, describe them in this README for the project reviewers.

## Troubleshooting

- **Compose reports missing variables:** Create `.env` beside `docker-compose.yaml` and replace both password placeholders. Run `docker compose config --quiet` again.
- **WordPress cannot reach MySQL:** Check that both services are running, `WORDPRESS_DB_HOST` is `db:3306`, and the database name, user, and application password match on both sides. MySQL may still be initializing on the first start.
- **Changing database credentials does not take effect:** MySQL reads the initialization variables only for an empty data volume. Do not delete `db-data` to apply a change unless you intend to discard the database and have a backup.
- **Port `8080` is unavailable:** Change the published host port in `docker-compose.yaml`, restart the setup, and use the new port in the browser and VM network rules.
- **WordPress is unreachable from the VM:** Check `docker compose ps`, the WordPress logs, the published port, and inbound firewall and cloud network rules.
- **A container stays stopped after `docker kill` or `docker compose stop`:** That is a manual stop. Start it again with `docker compose up -d`.
