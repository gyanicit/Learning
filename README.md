# Table of Contents

This file links to the documents and configuration files in this repository, grouped by topic.

## Docker

- [Docker Installation](./docker/docker-installation.md) — Docker setup on Windows with WSL 2 and on Ubuntu Linux.
- [Docker Commands](./docker/docker.md) — Common commands for images, containers, and Docker Compose.
- [Docker Build Guide](./docker/DOCKER_BUILD_GUIDE.md) — Build and run a Spring Boot demo with Docker or Compose.

### Application Dockerization

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

### Workload Management

- [Workload Management Overview](./k8s/workload-management/README.md) — How Pods, ReplicationControllers, ReplicaSets, and Deployments relate.
#### Pod
- [Pod Guide](k8s/workload-management/Pod/README.md) — Pod structure, lifecycle, components, creation, and common `kubectl` commands.
- [Pod Manifest](./k8s/workload-management/Pod/pod.yaml) — Manifest for the `nginx` Pod.

#### Replication Controller
- [ReplicationController Guide](./k8s/workload-management/ReplicationController/README.md) — Legacy controller behavior, commands, Pod adoption, and lifecycle.
- [ReplicationController Manifest](./k8s/workload-management/ReplicationController/replication-controller.yaml) — Manifest for the `nginx` Pod.

#### Replica Set
- [ReplicaSet Guide](./k8s/workload-management/ReplicaSet/README.md) — Selectors, replicas, scaling, Pod adoption, and lifecycle.
- [ReplicaSet Manifest](./k8s/workload-management/ReplicaSet/replica-set.yaml) — Example `apps/v1` ReplicaSet with three nginx Pods.

#### Deployment
- [Deployment Guide](./k8s/workload-management/Deployment/README.md) — Deployment behavior, rollout management, scaling, and rollback.
- [Deployment Manifest](./k8s/workload-management/Deployment/deployment.yaml) — Example Deployment manifest.

### Networking

- [Services](k8s/networking/k8s-Service/README.md) — Service types, ports, creation, connections, and troubleshooting.
- [Ingress](k8s/networking/Ingress/README.md) — Ingress creation, editing, and Minikube setup.

### kubectl Cheatsheet
- [Cheatsheet](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
