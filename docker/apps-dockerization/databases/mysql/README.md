# MySQL with Docker Compose

## Table of Contents

- [Configuration](#configuration)
- [Start MySQL](#start-mysql)
- [Connect to the database](#connect-to-the-database)
- [Stop and remove the container](#stop-and-remove-the-container)
- [References](#references)

This folder contains a Compose file for starting a MySQL database for local learning and development.

## Configuration

The [Compose file](docker-compose-mysql.yaml) defines:

| Setting | Value |
| --- | --- |
| Compose service and container | `mysql` |
| Image | `mysql:latest` |
| Database created on first initialization | `demodb` |
| Root password for first initialization | `root` |
| Host port | `3306` |
| Persistent data volume | `mysql_data`, mounted at `/var/lib/mysql` |

These example credentials are plain text and intended only for a trusted local learning environment. Do not use them for a shared or production database.

## Start MySQL

Run these commands from this directory:

```shell
docker compose -f docker-compose-mysql.yaml up -d
docker compose -f docker-compose-mysql.yaml ps
docker compose -f docker-compose-mysql.yaml logs -f mysql
```

The first command creates and starts MySQL in the background. The status command shows whether the container is running. The logs command follows MySQL startup output; press `Ctrl+C` to stop following logs without stopping the database.

## Connect to the database

From a MySQL client on the host, use:

| Field | Value |
| --- | --- |
| Host | `127.0.0.1` or `localhost` |
| Port | `3306` |
| Database | `demodb` |
| User | `root` |
| Password | `root` |

You can also open the MySQL client inside the running container:

```shell
docker compose -f docker-compose-mysql.yaml exec mysql mysql -uroot -p demodb
```

Enter `root` when prompted for the password.

## Stop and remove the container

Stop and remove the container while keeping the database volume:

```shell
docker compose -f docker-compose-mysql.yaml down
```

Start it again with `up -d`; the database files remain in the `mysql_data` volume.

To also delete the volume and all data stored in it, run:

```shell
docker compose -f docker-compose-mysql.yaml down --volumes
```

This permanently removes the database data. The MySQL image's `MYSQL_DATABASE` and `MYSQL_ROOT_PASSWORD` initialization settings apply when the data directory is first initialized. Changing those values after the volume already contains a database does not reset or update the existing database; deleting the volume is required for a fresh initialization.

## References

- [MySQL official image documentation](https://hub.docker.com/_/mysql)
- [Docker Compose `down` reference](https://docs.docker.com/reference/cli/docker/compose/down/)
- [Repository table of contents](../../../../README.md)
