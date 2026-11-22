# Table of Contents

This file links to the documents and configuration files in this repository, grouped by folder.

## Docker

- [Docker Installation](./docker/docker-installation.md) — Docker setup on Windows with WSL 2 and on Ubuntu Linux.
- [Docker Commands](./docker/docker.md) — Common commands for images, containers, and Docker Compose.
- [Docker Build Guide](./docker/DOCKER_BUILD_GUIDE.md) — Build and run a Spring Boot demo with Docker or Compose.

### Apps Dockerization

#### Databases
- MySQL
  - [MySQL Setup Guide](./docker/apps-dockerization/databases/mysql/README.md) — Configure, start, connect to, and stop the local MySQL database.
  - [MySQL Compose Configuration](./docker/apps-dockerization/databases/mysql/docker-compose-mysql.yaml) — MySQL service configuration and persistent data volume.
- PostgreSQL
  - [PostgreSQL, pgvector, and pgAdmin Setup](./docker/apps-dockerization/databases/PostgreSQL/POSTGRES_PGADMIN_SETUP.md) — Run PostgreSQL with pgvector in Docker and connect with pgAdmin on Windows.

#### Kafka

- [Kafka Commands](./docker/apps-dockerization/kafka/KAFKA_COMMANDS.md) — CLI setup and common topic and consumer group commands.
- [Kafka with Zookeeper Setup](./docker/apps-dockerization/kafka/README.md) — Kafka and Zookeeper with Kafka UI setup.
- [Kafka and Zookeeper Compose Configuration](./docker/apps-dockerization/kafka/docker-compose.yaml) — Kafka and Zookeeper services.
- [Kafka UI Compose Configuration](./docker/apps-dockerization/kafka/docker-compose-kafka-ui.yaml) — Kafka UI service configuration.

## Kubernetes

### Architecture

- [Kubernetes Cluster Architecture](./k8s/README.md) — Control-plane and data-plane components, responsibilities, and common topology combinations.

### Cluster Management

- [Minikube Cluster Guide](./k8s/Cluster-Management/minikube-k8s-cluster/README.md) — Install Minikube and create or manage a multi-node local cluster.
- [Kind Cluster Guide](./k8s/Cluster-Management/kind-k8s-cluster/README.md) — Install Kind and create or manage a multi-node local cluster.
  - [Kind Cluster Configuration](./k8s/Cluster-Management/kind-k8s-cluster/cluster.yaml) — One control-plane node and two worker nodes.

### Pods

- [Pod Guide](./k8s/pod/Pod.md) — Pod structure, lifecycle, components, creation, and common `kubectl` commands.
- [Example Pod Manifest](./k8s/pod/pod.yaml) — Manifest for the `nginx` Pod.


### Networking

- [Services](./k8s/networking/Service.md) — Service types, ports, creation, connections, and troubleshooting.
- [Ingress](./k8s/networking/Ingress.md) — Ingress creation, editing, and Minikube setup.

### Workload Management

_The `k8s/Workload Management` folder is currently empty._
