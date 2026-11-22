# Kind cluster commands

Kind (Kubernetes IN Docker) creates local Kubernetes clusters with containerized nodes. This guide covers Kind installation and Kind cluster operations only. The configuration in `kind-k8s-cluster/cluster.yaml` creates one control-plane node and two worker nodes.

## Prerequisite

Install and start a container provider before creating a cluster. Kind can detect Docker, Podman, or nerdctl. The commands below rely on Kind to select the available provider.

## Install Kind

The current stable release is `v0.33.0`. Choose the instructions for your operating system and CPU architecture.

### Linux

For AMD64 / x86-64:

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.33.0/kind-linux-amd64
chmod +x ./kind
sudo install ./kind /usr/local/bin/kind
```

For ARM64, use `kind-linux-arm64` in the download URL.

### macOS

With Homebrew:

```bash
brew install kind
```

The Homebrew package is community-maintained. For the official binary, download `kind-darwin-amd64` for Intel or `kind-darwin-arm64` for Apple silicon from the [Kind quick start](https://kind.sigs.k8s.io/docs/user/quick-start/), then add it to your `PATH`.

### Windows

With Windows Package Manager:

```powershell
winget install Kubernetes.kind
```

Alternative community packages:

```powershell
choco install kind
```

Or:

```powershell
scoop bucket add main
scoop install main/kind
```

For the official Windows binary, download `kind-windows-amd64` from the [Kind quick start](https://kind.sigs.k8s.io/docs/user/quick-start/), rename it to `kind.exe`, and place it in a directory on your `PATH`.

Verify installation:

```text
kind version
kind --help
```

## Create this repository's cluster

Run this from the repository root. The config file defines one control-plane and two worker nodes:

```text
kind create cluster --name kind-multi-node --config kind-k8s-cluster/cluster.yaml --wait 5m
kind get clusters
```

Kind also creates a kubeconfig context named `kind-kind-multi-node`.

## Load local images into the cluster

After building an image with your container provider, load it into Kind so the cluster can use it:

```text
kind load docker-image <image-name>:<tag> --name kind-multi-node
```

Replace `<image-name>:<tag>` with the local image name and tag.

## Export Kind cluster logs

```text
kind export logs ./kind-logs --name kind-multi-node
```

## Delete the cluster

```text
kind delete cluster --name kind-multi-node
```

For a cluster created without `--name`, use `kind delete cluster` to delete the default cluster named `kind`.

Kind has no `kind stop` or `kind start` command. Use `kind delete cluster` to remove a cluster, or leave it created for later use.

## Official references

- [Kind quick start and installation](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [Kind cluster configuration](https://kind.sigs.k8s.io/docs/user/configuration/)
- [Kind v0.33.0 release](https://github.com/kubernetes-sigs/kind/releases/tag/v0.33.0)
