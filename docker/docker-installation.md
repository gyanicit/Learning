# Docker Installation: Windows, WSL, and Linux

## Table of Contents

- [Windows with WSL 2](#windows-with-wsl-2)
  - [1. Install or update WSL](#1-install-or-update-wsl)
  - [2. Install Docker Desktop](#2-install-docker-desktop)
  - [3. Verify from WSL](#3-verify-from-wsl)
- [Linux: Install Docker Engine on Ubuntu](#linux-install-docker-engine-on-ubuntu)
  - [1. Set up Docker's APT repository](#1-set-up-dockers-apt-repository)
  - [2. Install Docker Engine and Compose](#2-install-docker-engine-and-compose)
  - [3. Optional: Run Docker without `sudo`](#3-optional-run-docker-without-sudo)
- [Official installation guides](#official-installation-guides)

## Windows with WSL 2

The recommended Windows setup for Linux containers is Docker Desktop using its WSL 2 backend.
Check Docker Desktop's current [Windows system requirements](https://docs.docker.com/desktop/setup/install/windows-install/) before installing; Docker's documentation currently requires WSL version 2.1.5 or later.

### 1. Install or update WSL

Open PowerShell as Administrator and install WSL with its default Ubuntu distribution:

```powershell
wsl --install
```

Restart Windows if prompted. If you already have WSL, update it and check its status:

```powershell
wsl --update
wsl --status
wsl --list --verbose
```

Confirm that your Linux distribution is version 2. If needed, convert it (replace `Ubuntu` with the distribution name shown by `wsl --list --verbose`):

```powershell
wsl --set-version Ubuntu 2
```

### 2. Install Docker Desktop

1. Download and install [Docker Desktop for Windows](https://docs.docker.com/desktop/setup/install/windows-install/).
2. Choose the WSL 2 backend when the installer offers a backend option.
3. Start Docker Desktop. In **Settings > General**, confirm **Use the WSL 2 based engine** is enabled, then apply the setting if changed.
4. In **Settings > Resources > WSL Integration**, enable integration for the distribution you use, then apply the setting.

Docker Desktop can be used from Windows terminals directly. WSL integration also makes the `docker` command available inside the selected Linux distribution.

### 3. Verify from WSL

Open your WSL distribution and run:

```bash
docker version
docker run --rm hello-world
```

The first command should show both the Docker client and server. The second downloads and runs a test container.

Keep Linux development projects in the WSL filesystem (for example, under `~/projects`) for better bind-mount performance. When using Docker Desktop integration, don't also install and run a separate Docker Engine inside that same WSL distribution; the two installations can conflict.

## Linux: Install Docker Engine on Ubuntu

These commands install Docker Engine and the Docker Compose plugin from Docker's official APT repository on a supported Ubuntu release. For other distributions, use the official [Docker Engine installation page](https://docs.docker.com/engine/install/) and select the matching distribution.

### 1. Set up Docker's APT repository

```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add the repository and refresh the package index:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

### 2. Install Docker Engine and Compose

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Verify the service and run Docker's test container:

```bash
sudo systemctl status docker
sudo docker run --rm hello-world
```

If the Docker service is not running, start it with `sudo systemctl start docker`.

### 3. Optional: Run Docker without `sudo`

Docker commands require `sudo` by default. To allow your user to use the Docker daemon without `sudo`, add the user to the `docker` group and refresh the group membership:

```bash
sudo usermod -aG docker "$USER"
newgrp docker
docker run --rm hello-world
```

Membership in the `docker` group grants root-level privileges. Keep using `sudo` if you do not want to grant that access.

## Official installation guides

- [Install Docker Desktop on Windows](https://docs.docker.com/desktop/setup/install/windows-install/)
- [Docker Desktop with the WSL 2 backend](https://docs.docker.com/desktop/features/wsl/)
- [Microsoft: Install WSL](https://learn.microsoft.com/en-us/windows/wsl/install)
- [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Install Docker Engine on Debian](https://docs.docker.com/engine/install/debian/)
- [Install Docker Engine on other supported Linux distributions](https://docs.docker.com/engine/install/)
- [Linux post-installation steps](https://docs.docker.com/engine/install/linux-postinstall/)
