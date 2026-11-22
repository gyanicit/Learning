# Docker Commands

## Table of Contents

- [Check Docker](#check-docker)
- [Work with Images](#work-with-images)
- [Run and Manage Containers](#run-and-manage-containers)
- [Docker Compose](#docker-compose)

## Check Docker

```bash
docker --version
```
Print the installed Docker CLI version.

```bash
docker info
```
Show information about the Docker client and daemon.

## Work with Images

```bash
docker pull nginx:latest
```
Download an image from a container registry.

```bash
docker images
```
List images available on the local machine.

```bash
docker build -t my-app:latest .
```
Build an image from the `Dockerfile` in the current directory and tag it `my-app:latest`.

For a complete commented Spring Boot example covering the Dockerfile and the end-to-end build cycle, see [DOCKER_BUILD_GUIDE.md](./DOCKER_BUILD_GUIDE.md).

```bash
docker tag my-app:latest username/my-app:latest
```
Add a registry-ready tag to a local image before pushing it.

```bash
docker push username/my-app:latest
```
Upload an image to the registry named in its tag. Sign in first with `docker login` if required.

## Run and Manage Containers

```bash
docker run --name web -d -p 8080:80 nginx:latest
```
Create and start a detached NGINX container named `web`. The `-p` option maps host port `8080` to container port `80`.

```bash
docker ps
```
List running containers.

```bash
docker ps -a
```
List all containers, including stopped ones.

```bash
docker logs -f web
```
Show the container's logs and continue following new output. Press `Ctrl+C` to stop following.

```bash
docker exec -it web sh
```
Open an interactive shell in a running container. Some images provide `bash` instead of `sh`.

```bash
docker stop web
docker start web
docker restart web
```
Stop, start, or restart the named container.

```bash
docker rm web
```
Remove a stopped container. Stop it first if it is still running.

```bash
docker inspect web
```
Display detailed configuration and state for a container or image.

## Docker Compose

```bash
docker compose up -d
```
Create and start the services defined in `compose.yaml` (or `docker-compose.yml`) in the background.

```bash
docker compose ps
```
Show the status of the Compose project's services.

```bash
docker compose logs -f
```
Follow logs from the Compose services.

```bash
docker compose down
```
Stop and remove the containers and network created for the Compose project.

