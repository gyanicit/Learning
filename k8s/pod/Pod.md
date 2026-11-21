# Kubernetes Pod

## What is a Pod?

A Pod is Kubernetes' smallest deployable unit. It wraps one or more containers that need to run together. Kubernetes schedules the whole Pod onto one cluster node.

Containers in the same Pod share the Pod's network address, so they can communicate through `localhost`. They can also share storage volumes. Most Pods run one application container. A helper container may be added when it needs to work closely with the main application.

Pods are temporary. If a Pod is deleted or replaced, its replacement is a new Pod with a new identity and possibly a different IP address. A Deployment or another controller can create replacement Pods; a standalone Pod does not recreate itself after it is deleted.

## Reading `pod.yaml`

The [`pod.yaml`](pod.yaml) file in this directory describes a Pod named `nginx` in the `dev-ns` namespace:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: dev-ns
spec:
  containers:
    - name: nginx
      image: nginx
```

- `apiVersion: v1` selects Kubernetes' core API, where built-in resources such as Pods and Services live.
- `kind: Pod` says that this manifest describes a Pod. `apps/v1` is used for resources such as a Deployment, not a Pod.
- `metadata` gives the Pod its name and namespace.
- `spec` describes the container Kubernetes should run.

A Pod is built into Kubernetes, so it does not require a CustomResourceDefinition (CRD). A CRD is used to add a new resource type to the Kubernetes API.

## Lifecycle of a Pod

The Pod lifecycle starts when Kubernetes accepts a Pod manifest and ends when the Pod is removed. Its phase is a summary of its overall status; it is not a detailed status for each container.

```text
Manifest submitted
       |
       v
API server validates and stores the Pod
       |
       v
Scheduler assigns the Pod to a node
       |
       v
Node starts the Pod's containers       (Pending -> Running)
       |
       +------> Succeeded
       +------> Failed
       +------> Unknown (node status cannot be determined)
```

### 1. The API server accepts the Pod

When you run `kubectl apply`, `kubectl` sends the manifest to the API server. The API server checks that the resource and its fields are valid, then stores the desired Pod state. For this example, `apiVersion: v1` and `kind: Pod` identify a built-in Kubernetes resource.

### 2. The scheduler chooses a node

If the manifest does not name a specific node, the scheduler selects a suitable node based on available resources and scheduling rules. Until a node is assigned and the containers are ready to run, the Pod is generally in the `Pending` phase.

### 3. The node prepares and starts the containers

The node's kubelet notices the assigned Pod. It prepares networking and requested storage, downloads the container image if needed, and asks the container runtime to start the containers. If the Pod has init containers, they must finish successfully before the application containers start.

The Pod enters the `Running` phase once it is bound to a node and its containers have been created; at least one container is running, starting, or restarting. A readiness probe, if configured, controls when a container is considered ready to receive traffic.

### 4. Kubernetes monitors the Pod

The kubelet monitors the containers. If a container exits, Kubernetes follows the Pod's `restartPolicy` (which defaults to `Always`) and may restart that container inside the same Pod. Restarting a container does not necessarily create a new Pod. Readiness and liveness probes can help Kubernetes determine whether a container should receive traffic or be restarted.

### 5. The Pod finishes or is deleted

For a Pod whose containers are expected to finish, the final phase is usually:

- `Succeeded` when all containers finish successfully and will not be restarted.
- `Failed` when the containers finish and at least one failed, or the Pod cannot run successfully.

When you delete a Pod, Kubernetes normally gives its containers a grace period to shut down. It then removes the Pod. A controller such as a Deployment may create a replacement Pod to maintain the requested number of replicas; the replacement has a different Pod identity.

### Pod phases at a glance

| Phase | Meaning |
| --- | --- |
| `Pending` | Kubernetes has accepted the Pod, but setup or container startup is not complete. |
| `Running` | The Pod is assigned to a node and at least one container is running, starting, or restarting. |
| `Succeeded` | All containers completed successfully and will not be restarted. |
| `Failed` | All containers have stopped and at least one failed, or the Pod encountered an unrecoverable error. |
| `Unknown` | Kubernetes cannot determine the Pod's state, usually because it cannot communicate with the node. |

These phases are summaries, not a guaranteed step-by-step sequence. For example, a long-running web server usually stays `Running` until deleted; it does not normally move to `Succeeded`.

## How to create a Pod

The manifest sets `namespace: dev-ns`, so that namespace must exist before applying it. Create it once if needed:

```shell
kubectl get namespace dev-ns
# Run this only if the namespace does not exist:
kubectl create namespace dev-ns
```

If `kubectl get namespace dev-ns` finds the namespace, skip the `create` command.

### Imperative method

Create an nginx Pod directly with `kubectl run`:

```shell
kubectl run nginx-pod --image=nginx:latest --namespace=dev-ns
```

### Declarative method

Create or update the Pod described by this directory's manifest:

```shell
kubectl apply -f pod.yaml
```

### Check and remove the Pod

```shell
kubectl get pod nginx -n dev-ns --watch
kubectl describe pod nginx -n dev-ns
kubectl logs nginx -n dev-ns
kubectl delete -f pod.yaml
```

The declarative example uses the Pod name `nginx`; the imperative example uses `nginx-pod`.
