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

This section follows a Pod created from `pod.yaml`. A Pod created directly with `kubectl apply` is a standalone Pod: if it is deleted or cannot run, Kubernetes does not automatically create a replacement. A Pod owned by a Deployment is different because the Deployment and ReplicaSet controllers maintain the requested number of Pods.

```text
kubectl apply -f pod.yaml
# Or make the manifest's namespace explicit: kubectl apply -f pod.yaml -n dev-ns
       |
       v
kube-apiserver: authenticate, authorize, admit/validate, and persist the Pod
       |                                      |
       |                                      +--> etcd: stores the API object
       v
kube-scheduler: selects a node and records the binding
       |
       v
Node kubelet: observes the assigned Pod and asks the runtime to make it run
       |                                      |
       |                                      +--> CRI runtime, CNI plugin, storage plugins (as needed)
       v
kubelet reports Pod/container status through kube-apiserver --> etcd
       |
       +--> containers keep running / restart according to restartPolicy
       +--> Succeeded or Failed (for completed Pods)
       +--> deletion: graceful stop, cleanup, and API object removal
```

The arrows represent asynchronous control loops, not one synchronous command chain. Components watch the Kubernetes API, compare desired state with observed state, and update the API as they act. The Pod's `status` is reported separately from its desired `spec`.

### 1. `kubectl` submits the manifest to the API server

`kubectl` is the client; it sends an API request to `kube-apiserver`. The API server authenticates the caller, checks authorization, runs applicable admission and validation, applies defaults, and persists the Pod object in the cluster's backing store, normally `etcd`. For this example, `apiVersion: v1` and `kind: Pod` identify the built-in Pod API resource. Other components read and update cluster objects through the API server rather than writing directly to etcd.

### 2. The scheduler assigns a node

The `kube-scheduler` watches for Pods that have no node assigned. It filters and scores eligible nodes using resource requests, scheduling policies, affinity, taints and tolerations, and other constraints, then records the selected node through the API server. If a Pod already specifies `spec.nodeName`, the scheduler is bypassed. Scheduling only assigns a Pod once; a Pod is not moved to another node later.

### 3. The node starts the Pod

The `kubelet` on the selected node watches the API server for Pods assigned to that node. It makes sure the Pod's sandbox and containers match the Pod spec, using the Container Runtime Interface (CRI) to ask the node's container runtime (for example, containerd or CRI-O) to pull images and create/start containers. If the Pod has init containers, they must complete successfully before its application containers start.

Depending on the Pod configuration and cluster setup, node-level integrations also take part:

- A CNI network plugin configures the Pod network when the runtime creates the Pod sandbox.
- A CSI storage driver (or another volume plugin) attaches or mounts requested storage.
- The kubelet coordinates Pod setup and monitors the result; these plugins and the runtime do the node-specific network, storage, and container work.

The Pod is `Pending` while it is accepted but not fully set up, including while it waits for scheduling or image download. It becomes `Running` when it is bound to a node, all containers have been created, and at least one container is running, starting, or restarting. `Running` does not mean the Pod is ready to receive traffic; readiness probes and Pod readiness conditions determine that.

### 4. The kubelet monitors containers and reports status

The kubelet runs the configured startup, readiness, and liveness probes and reports container state, Pod conditions, and events through the API server. The API server persists these updates. A readiness probe affects whether the Pod is considered ready; a liveness or startup probe can cause the kubelet to restart a container.

If a container exits, the kubelet applies the Pod's `restartPolicy` (`Always` is the default for Pods). A container restart happens inside the same Pod and does not give the Pod a new identity. Repeated failures may show `CrashLoopBackOff` in `kubectl`; this is a restart backoff indication, not a Pod phase.

### 5. Controllers react to changes (when a controller owns the Pod)

The `kube-controller-manager` runs controllers that watch API objects and reconcile actual state toward desired state. A standalone Pod has no controller to replace it. For a Deployment, the Deployment controller manages a ReplicaSet, and the ReplicaSet controller creates replacement Pods when needed to maintain the requested replica count. A replacement is a new Pod with a new UID and may be scheduled to a different node.

If a node stops reporting healthy heartbeats, the node controller (also run by `kube-controller-manager`) marks the node unhealthy and the control plane may eventually evict its Pods. A workload controller such as a ReplicaSet can then create replacement Pods. The original Pod is not rescheduled onto another node.

If a Pod is selected by a Service, the EndpointSlice controller updates the Service's endpoints based on Pod readiness. Traffic forwarding is then handled by the cluster's Service data plane, commonly `kube-proxy`, though some clusters use a CNI or other replacement. Services and their networking components are adjacent to Pod lifecycle; they are not required to create or run a Pod.

### 6. The Pod completes or is deleted

For a Pod whose containers are expected to finish, the final phase is usually `Succeeded` when all containers finish successfully, or `Failed` when they finish and at least one failed. A long-running application Pod normally stays `Running` until it is deleted or otherwise terminated.

When you delete a Pod, the API server records the deletion request and grace period. The kubelet on the node runs any `preStop` hook, asks the runtime to stop the containers (normally with `SIGTERM`), and force-stops remaining processes after the grace period. It then reports the terminal status; the control plane removes the Pod object after termination and any finalizers are handled. The kubelet/runtime and configured plugins clean up the containers, networking, and mounted resources. If a controller owns the Pod, it may create a replacement to restore the desired replica count.

### Components and their responsibilities

| Component | Responsibility in the Pod lifecycle |
| --- | --- |
| `kubectl` | Sends create, update, read, and delete requests to the Kubernetes API. It is a client, not a cluster component. |
| `kube-apiserver` | Exposes the Kubernetes API; authenticates and authorizes requests, applies admission and validation, and serves reads and writes from the backing store. |
| `etcd` | Stores the persisted Kubernetes API objects, including the Pod spec and its reported status. |
| `kube-scheduler` | Selects and binds a node for each unscheduled Pod. |
| `kube-controller-manager` | Runs control loops such as Deployment, ReplicaSet, node, and EndpointSlice controllers; actions depend on which resources exist. |
| `cloud-controller-manager` (optional) | Integrates with a cloud provider for infrastructure tasks such as node and load balancer management; it does not schedule or start Pod containers. |
| `kubelet` | On each node, observes Pods assigned to that node, coordinates their setup and shutdown, monitors containers and probes, and reports status. |
| Container runtime | Runs the Pod sandbox and containers through CRI, including image handling and container process lifecycle. |
| CNI plugin / CSI driver | Optional node integrations for Pod networking and persistent storage, respectively. |
| `kube-proxy` or replacement data plane | Implements Service traffic forwarding on clusters that use it; some CNI implementations provide this function instead. It does not start Pods. |

### Pod phases at a glance

| Phase | Meaning |
| --- | --- |
| `Pending` | Kubernetes has accepted the Pod, but setup or container startup is not complete. |
| `Running` | The Pod is assigned to a node, all containers have been created, and at least one is running, starting, or restarting. It may not be ready. |
| `Succeeded` | All containers completed successfully and will not be restarted. |
| `Failed` | All containers have stopped and at least one failed. |
| `Unknown` | The control plane cannot determine the Pod's state, usually because it cannot communicate with the node. |

Phases are high-level summaries, not a guaranteed step-by-step sequence. `Terminating` and `CrashLoopBackOff` may appear in `kubectl` output, but they are not Pod phases. See the official [Kubernetes components](https://kubernetes.io/docs/concepts/overview/components/), [Pod lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/), and [controller](https://kubernetes.io/docs/concepts/architecture/controller/) documentation for more detail.

## How to create a Pod

### Namespace selection

For a namespaced resource such as a Pod, `-n dev-ns` explicitly selects `dev-ns`. If `-n` is omitted, `kubectl` uses the namespace configured for the current context (often `default`); it does not automatically choose `dev-ns`. The `pod.yaml` manifest already declares `metadata.namespace: dev-ns`, so file-based commands can use that namespace from the manifest, with or without a matching `-n dev-ns` flag. Namespace objects themselves are cluster-scoped, so commands that create or look them up do not take `-n`. See the official [Kubernetes namespace guide](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) for more details.

To make `dev-ns` the default for future commands in the current context, you can set the context preference:

```shell
kubectl config set-context --current --namespace=dev-ns
```

The manifest sets `namespace: dev-ns`, so that namespace must exist before applying it. Create it once if needed:

```shell
# Namespace resources are cluster-scoped; do not add -n here.
kubectl get namespace dev-ns
# Run this only if the namespace does not exist:
kubectl create namespace dev-ns
```

If `kubectl get namespace dev-ns` finds the namespace, skip the `create` command.

### Imperative method

Create an nginx Pod directly with `kubectl run`:

```shell
# Explicit namespace:
kubectl run nginx-pod --image=nginx:latest -n dev-ns
# Omit -n to use the current context's namespace:
kubectl run nginx-pod --image=nginx:latest
```

Choose one command. If both are run with the same namespace selected, the second command will fail because the Pod already exists.

### Create once from a manifest

Create the Pod described by this directory's manifest with `kubectl create -f`:

```shell
# The manifest declares metadata.namespace: dev-ns, so -n is optional here.
kubectl create -f pod.yaml
# Explicitly select the same namespace as the manifest:
kubectl create -f pod.yaml -n dev-ns
```

This creates the resource if it does not already exist. If a Pod with the same name and namespace already exists, the command returns an `AlreadyExists` error; it does not update that Pod. Use `kubectl apply -f` when you want to create the resource if needed and apply later manifest changes to it.

### Declarative method

Create or update the Pod described by this directory's manifest:

```shell
# The manifest declares metadata.namespace: dev-ns, so -n is optional here.
kubectl apply -f pod.yaml
# Explicitly select the same namespace as the manifest:
kubectl apply -f pod.yaml -n dev-ns
```

### Check and remove the Pod

```shell
# Explicit namespace:
kubectl get pod nginx -n dev-ns --watch
# Current context namespace:
kubectl get pod nginx --watch
# Explicit namespace:
kubectl describe pod nginx -n dev-ns
# Current context namespace:
kubectl describe pod nginx
# Explicit namespace:
kubectl logs nginx -n dev-ns
# Current context namespace:
kubectl logs nginx
# The manifest's metadata.namespace is dev-ns; -n is optional for this file.
kubectl delete -f pod.yaml
kubectl delete -f pod.yaml -n dev-ns
```

For name-based commands without `-n`, make sure the current context points to the namespace where the Pod exists.

The declarative example uses the Pod name `nginx`; the imperative example uses `nginx-pod`.

## Inspecting and managing a Pod

The examples below use the Pod `nginx` from `pod.yaml` in namespace `dev-ns`, unless stated otherwise.

### Replace a Pod from a manifest

```shell
# Manifest namespace is dev-ns:
kubectl replace --force -f pod.yaml
# Same namespace, selected explicitly:
kubectl replace --force -f pod.yaml -n dev-ns
```

`replace --force` deletes the existing resource and then creates it again from the manifest. This is a delete-and-recreate operation: the Pod gets a new UID and may get a different IP address, and there can be downtime. The manifest must contain the complete resource spec. Prefer `kubectl apply -f pod.yaml` for ordinary manifest changes; force replacement is useful when a change cannot be made to the existing Pod in place.

### Follow container logs

```shell
# Explicit namespace:
kubectl logs -f nginx -n dev-ns
# Current context namespace:
kubectl logs -f nginx
# Explicit container, with and without -n:
kubectl logs -f nginx -n dev-ns -c nginx
kubectl logs -f nginx -c nginx
```

The `-c nginx` flag is optional for this manifest because it has one container named `nginx`; it explicitly selects that container. `-f` streams new log output until you stop the command (usually with `Ctrl+C`). For a multi-container Pod, specify the desired container with `-c <container-name>` or use `--all-containers=true` to include all containers. Commands without `-n` use the current context namespace.

### Open a shell in a container

```shell
# Without explicitly choosing a container:
kubectl exec -it nginx -n dev-ns -- /bin/bash
# Explicitly choose the nginx container:
kubectl exec -it nginx -n dev-ns -c nginx -- /bin/bash
# The same forms using the current context namespace:
kubectl exec -it nginx -- /bin/bash
kubectl exec -it nginx -c nginx -- /bin/bash
```

With one container, `-c nginx` is optional; without it, `kubectl` uses the default-container annotation or the first container. In a multi-container Pod, use `-c` to choose the target. This manifest names its container `nginx`; use `-c nginx1` only if the Pod actually has a container named `nginx1`. The command after `--` runs inside the container, and the image must include the requested shell. If `/bin/bash` is unavailable, try `/bin/sh`.

### Edit a live Pod

```shell
# Explicit namespace:
kubectl edit pod nginx -n dev-ns
# Current context namespace:
kubectl edit pod nginx
```

This opens the live Pod object in your configured editor and submits valid changes to the API server when you save and close it. Some Pod fields are immutable and cannot be changed in place. If a Pod belongs to a Deployment, edit the Deployment's Pod template instead; changing an individual managed Pod will not update the Deployment's desired configuration.

### Generate a manifest with `kubectl run`

Use client-side dry run to generate a YAML or JSON manifest without creating a Pod. Choose a namespace variant and an output format; the filenames keep the outputs separate:

```shell
# YAML, explicit namespace:
kubectl run nginx-pod --image=nginx:latest -n dev-ns --dry-run=client -o yaml > new-pod-dev-ns.yaml
# YAML, current context namespace:
kubectl run nginx-pod --image=nginx:latest --dry-run=client -o yaml > new-pod-current-namespace.yaml
# JSON, explicit namespace:
kubectl run nginx-pod --image=nginx:latest -n dev-ns --dry-run=client -o json > new-pod-dev-ns.json
# JSON, current context namespace:
kubectl run nginx-pod --image=nginx:latest --dry-run=client -o json > new-pod-current-namespace.json
```

Choose one manifest, review it, then apply it. The generated manifest includes the selected namespace when available; `-n` can also be supplied on the apply request. If you generated JSON, use the matching `.json` filename instead:

```shell
# Manifest generated for dev-ns:
kubectl apply -f new-pod-dev-ns.yaml
kubectl apply -f new-pod-dev-ns.yaml -n dev-ns
# Manifest generated for the current context namespace:
kubectl apply -f new-pod-current-namespace.yaml
# Or pass -n YOUR_CONTEXT_NAMESPACE to select that context namespace explicitly.
```

The dry-run commands generate a starting manifest; they do not export an already-running Pod. To export a live Pod for inspection, use:

```shell
# Explicit namespace:
kubectl get pod nginx -n dev-ns -o yaml > nginx-export.yaml
# Current context namespace:
kubectl get pod nginx -o yaml > nginx-export-current-namespace.yaml
```

An exported object includes server-generated fields such as `metadata.uid`, `metadata.resourceVersion`, `metadata.creationTimestamp`, `metadata.managedFields`, and `status`. Remove those fields before treating the export as a clean source manifest.

### Show labels and node details

```shell
# Explicit namespace:
kubectl get pods nginx-pod -n dev-ns --show-labels
# Current context namespace:
kubectl get pods nginx-pod --show-labels
# Explicit namespace, with extra node and network columns:
kubectl get pods nginx-pod -n dev-ns -o wide
# Current context namespace, with extra node and network columns:
kubectl get pods nginx-pod -o wide
```

`--show-labels` adds the Pod's labels to the output. `-o wide` adds node and network details such as the assigned node and Pod IP.

For full command options, see the official [`kubectl replace`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_replace/), [`kubectl logs`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/), [`kubectl exec`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_exec/), [`kubectl edit`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_edit/), [`kubectl run`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_run/), and [`kubectl get`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/) references.
