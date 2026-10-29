# WordPress Docker Environment

A lightweight and reusable Docker environment for WordPress development and production deployments.

## Features

* WordPress with PHP 8.5 and Apache (`wordpress:php8.5-apache`), serving directly on port 80
* WP-CLI service running on PHP 8.5 (`wordpress:cli-php8.5`) and a helper script
* MariaDB
* Docker Compose based
* Custom PHP settings (memory limit, upload limits, OPcache) in a single `php/config.ini` file
* Easy environment configuration through a single `.env` file

---

## Requirements

Before getting started, make sure Docker and Docker Compose are installed.

For Ubuntu systems, an installation script is included in the root of this repository:

```bash
./install-docker.sh
```

The script follows the official Docker installation guide for Ubuntu.

---

## Configuration

Create your environment file by copying the example:

```bash
cp .env.example .env
```

Then edit the newly created `.env` file and update the following values:

| Variable                     | Description                                                                                                                                                     |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SERVICE_NAME`               | Name of your project. This is used as the Docker Compose project name.                                                                                          |
| `WORDPRESS_DB_HOST`          | Database host. Use the MariaDB service name, `mariadb`.                                                                                                         |
| `WORDPRESS_DB_USER`          | Database user used by WordPress.                                                                                                                                |
| `WORDPRESS_DB_PASSWORD`      | A strong password for the MariaDB root user.                                                                                                                    |
| `WORDPRESS_DB_NAME`          | *(Optional)* Database name. Default: `wordpress_db`.                                                                                                            |
| `WORDPRESS_DEBUG`            | *(Optional)* Set to `true` to enable WordPress debugging. Default: `false`.                                                                                     |
| `WORDPRESS_DEBUG_LOG`        | *(Optional)* Sets `WP_DEBUG_LOG` (`true` or `false`).                                                                                                           |
| `WORDPRESS_DEBUG_DISPLAY`    | *(Optional)* Sets `WP_DEBUG_DISPLAY` (`true` or `false`).                                                                                                       |
| `WORDPRESS_DISABLE_CRON`     | *(Optional)* Set to `false` to use built-in WP-Cron (not recommended for production). Default: `true`.                                                          |
| `WEBSITE_PROTOCOL`           | Set to `http` for local development or environments without SSL. Use `https` when SSL is handled externally (for example, by an AWS Application Load Balancer). |
| `WEBSITE_DOMAIN`             | Domain name of your website. For local development, you may use `localhost` or `127.0.0.1`.                                                                     |
| `WEBSITE_URL`                | Full site URL (protocol + domain). Used for `WP_HOME` and `WP_SITEURL`.                                                                                         |

---

## PHP Configuration

WordPress runs on the official Apache image, which already listens on port 80 and runs PHP as an Apache module. The container port 80 is published on the host's port 80, so no reverse proxy is needed in front of it.

Custom PHP settings live in `php/config.ini`, which is mounted into both the `wordpress` and `cli` containers as `/usr/local/etc/php/conf.d/zz-config.ini`. This way the web server and WP-CLI always run with the same configuration. The file currently defines:

| Setting                          | Value   |
| -------------------------------- | ------- |
| `memory_limit`                   | `512M`  |
| `upload_max_filesize`            | `64M`   |
| `post_max_size`                  | `64M`   |
| `max_execution_time`             | `120`   |
| `max_input_time`                 | `60`    |
| `max_input_vars`                 | `10000` |
| `opcache.memory_consumption`     | `2048`  |
| `opcache.interned_strings_buffer`| `128`   |
| `opcache.max_accelerated_files`  | `20000` |
| `opcache.revalidate_freq`        | `60`    |

OPcache is built into PHP 8.5, so it does not need to be installed or enabled separately. Edit `php/config.ini` to adjust these values, then restart the containers:

```bash
docker compose restart wordpress cli
```

---

## Database Initialization

On the first startup, the MariaDB container automatically executes any `.sql` files found in the `database/init` directory.

This is useful for restoring an existing WordPress database or providing an initial dataset for development.

Example:

```text
database/
├── data/
└── init/
    ├── 00_wordpress.sql
    └── 01_additional-data.sql
```

When the database volume is empty, all `.sql` files inside `database/init` are executed automatically in alphabetical order.

> **Note:** Initialization scripts are only executed during the first database creation. If the `database/data` directory already contains an initialized database, the scripts in `database/init` will be ignored.

---

## Starting the Environment

Start all services:

```bash
docker compose up -d
```

To stop the environment:

```bash
docker compose down
```

---

## WP-CLI

WP-CLI runs in a dedicated `cli` service defined in `docker-compose.yaml`. It shares the same environment as the `wordpress` service (database credentials, `WORDPRESS_CONFIG_EXTRA`, etc.), so any change made there is automatically seen by WP-CLI.

The service starts together with the rest of the environment (`docker compose up -d`). You can run commands directly through Docker Compose:

```bash
docker compose exec cli wp core update-db
docker compose exec cli wp plugin list
```

A helper script is also included as a shortcut for the command above:

```bash
./wp plugin list
./wp theme list
./wp core version
./wp cron event run --due-now
```

The script works from any directory and disables TTY allocation when not run interactively (for example, from `crontab`).

---

## WordPress Cron

By default, the internal WordPress cron is disabled.

In production, schedule the following command (using `crontab -e`) to run every minute:

```text
* * * * * /path/to/wp cron event run --due-now
```

If you prefer to use the built-in WP-Cron (not recommended for production), set:

```text
WORDPRESS_DISABLE_CRON=false
```

---

## Project Structure

```text
.
├── database/
│   ├── data/
│   └── init/
├── php/
│   └── config.ini
├── www/
├── docker-compose.yaml
├── .env.example
├── install-docker.sh
└── wp
```

---

## Notes

* This environment is designed to work with both local development and production deployments.
* HTTPS certificates are **not** managed by this project. If your infrastructure provides SSL termination (such as AWS Application Load Balancer, Cloudflare, or another reverse proxy), simply set `WEBSITE_PROTOCOL=https`.
* All project-specific configuration should be managed through the `.env` file whenever possible.
