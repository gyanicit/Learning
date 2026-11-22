# Minikube installation and cluster commands

## Table of Contents

- [Requirements](#requirements)
- [Install Minikube](#install-minikube)
  - [Windows](#windows)
  - [macOS](#macos)
  - [Linux](#linux)
- [Create the three-node cluster](#create-the-three-node-cluster)
- [Add workers to an existing single-node cluster](#add-workers-to-an-existing-single-node-cluster)
- [Check cluster and node status](#check-cluster-and-node-status)
- [Remove worker nodes](#remove-worker-nodes)
- [Stop, restart, or delete the cluster](#stop-restart-or-delete-the-cluster)
- [Official references](#official-references)

Install Minikube for your operating system, set up a supported container or virtual-machine driver, then create a cluster with one control-plane node and two workers. This guide covers Linux, macOS, and Windows. It contains Minikube setup and node-management commands only.

## Requirements

Minikube generally needs 2 CPUs or more, 2 GB of free memory, 20 GB of free disk space, an internet connection, and a supported container or VM manager. A three-node cluster may need more resources depending on what you run in it.

| Environment | Common supported drivers |
| --- | --- |
| Linux | Docker or KVM2; VirtualBox and QEMU are also supported. |
| macOS | Docker or VFkit; VirtualBox, Parallels, VMware Fusion, and QEMU are also supported. |
| Windows | Hyper-V or Docker; VirtualBox and VMware Workstation are also supported. |

See the [Minikube driver list](https://minikube.sigs.k8s.io/docs/drivers/) for driver requirements and setup. The Docker driver requires Docker to be installed and running.

## Install Minikube

Choose the instructions for your operating system. These install the latest stable release.

### Windows

Using Windows Package Manager:

```powershell
winget install Kubernetes.minikube
```

Or, if Chocolatey is installed:

```powershell
choco install minikube
```

For the standalone `.exe` installer or manual PATH setup, use the [official installation guide](https://minikube.sigs.k8s.io/docs/start/).

### macOS

With Homebrew (Intel and Apple silicon):

```bash
brew install minikube
```

### Linux

For Debian or Ubuntu on x86-64:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
```

For Debian or Ubuntu on ARM64, use the ARM64 package:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_arm64.deb
sudo dpkg -i minikube_latest_arm64.deb
```

For Fedora, RHEL, or another RPM-based distribution on x86-64:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-latest.x86_64.rpm
sudo rpm -Uvh minikube-latest.x86_64.rpm
```

For RPM-based distributions on ARM64, use the ARM64 package:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-latest.aarch64.rpm
sudo rpm -Uvh minikube-latest.aarch64.rpm
```

For other Linux architectures or installation methods, see the [official installation guide](https://minikube.sigs.k8s.io/docs/start/).

After installing, open a new terminal and confirm Minikube is available:

```text
minikube version
```

A separate `kubectl` installation is optional; Minikube can run a matching version with `minikube kubectl -- <kubectl arguments>`.

## Create the three-node cluster

Choose a driver installed on your machine. For Docker:

```text
minikube start --driver=docker --nodes 3
```

This creates one control-plane node and two worker nodes. Do not add `--ha`; that option creates multiple control-plane nodes. If using another driver, replace `docker` with its Minikube driver name.

## Add workers to an existing single-node cluster

Use this instead of creating a new three-node cluster if you already have a one-node cluster:

```text
minikube node add --worker
minikube node add --worker
```

## Check cluster and node status

```text
minikube status
minikube node list
minikube kubectl -- get nodes -o wide
```

## Remove worker nodes

List the nodes, then delete each worker using its exact name from `minikube node list`:

```text
minikube node list
minikube node delete <worker-node-name>
```

Replace `<worker-node-name>` with the listed worker's name and repeat for the other worker. Keep the control-plane node to keep the cluster running.

## Stop, restart, or delete the cluster

Stop preserves the cluster and its nodes; start resumes it:

```text
minikube stop
minikube start
```

To delete the entire cluster and all its nodes:

```text
minikube delete
```

## Official references

- [Installation and getting started](https://minikube.sigs.k8s.io/docs/start/)
- [Supported drivers](https://minikube.sigs.k8s.io/docs/drivers/)
- [Multi-node clusters](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
- [Node commands](https://minikube.sigs.k8s.io/docs/commands/node/)
