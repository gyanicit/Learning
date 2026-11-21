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
