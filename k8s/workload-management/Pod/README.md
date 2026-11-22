# Kubernetes Pod

## Table of Contents

- [1. Create & Apply](#1-create--apply)
  - [What is a Pod?](#what-is-a-pod)
  - [Reading `pod.yaml`](#reading-podyaml)
  - [Resource identity and metadata](#resource-identity-and-metadata)
  - [What is inside `spec`?](#what-is-inside-spec)
  - [Common fields you can add under `spec`](#common-fields-you-can-add-under-spec)
  - [Namespace behavior](#namespace-behavior)
  - [Imperative method](#imperative-method)
  - [Create once from a manifest](#create-once-from-a-manifest)
  - [Declarative method](#declarative-method)
  - [Generate a manifest with `kubectl run`](#generate-a-manifest-with-kubectl-run)
- [2. Inspect & Monitor](#2-inspect--monitor)
  - [Pod lifecycle and component flow](#pod-lifecycle-and-component-flow)
  - [Numbered lifecycle steps](#numbered-lifecycle-steps)
  - [Components and their responsibilities](#components-and-their-responsibilities)
  - [Pod phases at a glance](#pod-phases-at-a-glance)
  - [Watch and inspect the Pod](#watch-and-inspect-the-pod)
  - [Show labels and node details](#show-labels-and-node-details)
  - [Follow container logs](#follow-container-logs)
  - [Open a shell in a container](#open-a-shell-in-a-container)
  - [Export a running Pod for inspection](#export-a-running-pod-for-inspection)
- [3. Scale Replicas](#3-scale-replicas)
- [4. Edit & Update](#4-edit--update)
  - [Edit a live Pod](#edit-a-live-pod)
  - [Replace a Pod from a manifest](#replace-a-pod-from-a-manifest)
- [5. Expose via a Service](#5-expose-via-a-service)
- [6. Delete & Orphan](#6-delete--orphan)
  - [Delete the Pod](#delete-the-pod)
  - [Deletion and graceful shutdown](#deletion-and-graceful-shutdown)
  - [What orphaning means](#what-orphaning-means)
- [References](#references)

> **About the outputs below:** These are representative examples. Names, ages, IP addresses, event counts, and status vary by cluster and timing; output excerpts omit unrelated fields. The current `pod.yaml` does not set `metadata.namespace`, so all commands below use the namespace configured for the current kubectl context.

## 1. Create & Apply

### What is a Pod?

A Pod is Kubernetes' smallest deployable unit. It wraps one or more containers that need to run together. Kubernetes schedules the whole Pod onto one cluster node.

Containers in the same Pod share the Pod's network address, so they can communicate through `localhost`. They can also share storage volumes. Most Pods run one application container. A helper container may be added when it needs to work closely with the main application.

Pods are temporary. If a Pod is deleted or replaced, its replacement is a new Pod with a new identity and possibly a different IP address. A Deployment or another controller can create replacement Pods; a standalone Pod does not recreate itself after it is deleted.

### Reading `pod.yaml`

The [pod.yaml](pod.yaml) file describes a Pod named `nginx`. It does not declare a namespace, so Kubernetes creates it in the namespace configured for the current kubectl context. Here is the current manifest:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: my-app
    env: dev
    type: frontend
spec:
  containers:
    - name: nginx
      image: nginx
```

### Resource identity and metadata

| Field | Meaning in this manifest |
| --- | --- |
| `apiVersion: v1` | Uses the core Kubernetes API version, which contains built-in resources such as Pods and Services. |
| `kind: Pod` | Declares the resource type. A Deployment uses `apps/v1`; this file creates a Pod directly. |
| `metadata.name: nginx` | Names this Pod. Names must be unique within a namespace. |
| `metadata.namespace` | Omitted in this manifest. Kubernetes uses the namespace configured for the current kubectl context. |
| `metadata.labels` | Adds `app=my-app`, `env=dev`, and `type=frontend`. Labels help identify and select objects; they are not environment variables and do not create a Deployment or Service by themselves. |

### What is inside `spec`?

`spec` is the desired Pod configuration. The scheduler selects a node, and that node's kubelet works to run the Pod according to this specification. Kubernetes reports observed results separately in `status`; you normally do not set `status` in a Pod manifest.

| Path | Meaning in this manifest |
| --- | --- |
| `spec.containers` | A list of application containers that belong to this Pod. Kubernetes schedules the Pod as a unit; the containers share the Pod network namespace and can share declared volumes. This manifest has one container. |
| `spec.containers[0].name: nginx` | Names the first container within the Pod. Use this name with commands such as `kubectl logs -c nginx` and `kubectl exec -c nginx`. |
| `spec.containers[0].image: nginx` | Tells the runtime which container image to run. No tag is written, so Kubernetes treats it as `nginx:latest`; the default `imagePullPolicy` for this image is `Always`. For repeatable deployments, choose a specific image version or digest and set an appropriate pull policy. |

This is a deliberately small spec. Fields that are not written are not necessarily absent from the object after creation: Kubernetes applies defaults. For example, the Pod's default `restartPolicy` is `Always`. The kubelet restarts a failed container in this same Pod; a standalone Pod does not get a replacement Pod just because a container restarts.

### Common fields you can add under `spec`

The following fields are not in the current `pod.yaml`; they are common ways to configure a Pod and its containers:

| Scope | Field | Responsibility |
| --- | --- | --- |
| Pod | `restartPolicy` | Controls how containers are restarted in this Pod: `Always`, `OnFailure`, or `Never`. The default is `Always`. It does not create a replacement Pod. |
| Pod | `initContainers` | Defines setup containers that run before regular `containers`. They commonly prepare files or wait for prerequisites. |
| Pod | `volumes` | Declares storage or other data sources available to containers in the Pod. A container uses a volume by adding a matching `volumeMounts` entry. |
| Pod | `nodeSelector`, `affinity`, `tolerations` | Add scheduling constraints or permissions that affect which nodes are eligible. The scheduler still chooses the node. |
| Pod | `serviceAccountName` | Selects the ServiceAccount identity used by the Pod when it needs to access the Kubernetes API. If omitted, Kubernetes uses the namespace's default ServiceAccount. |
| Pod or container | `securityContext` | Sets security-related execution options, such as the user/group or Linux capabilities, at Pod or container scope. |
| Container | `command`, `args` | Override the image's startup command (`command`) and default arguments (`args`). |
| Container | `env`, `envFrom` | Pass configuration into the container as environment variables; values can also come from ConfigMaps or Secrets. |
| Container | `ports` | Documents named container ports for other configuration, such as probes and Services. Declaring `containerPort` alone does not create a Service or publish a host port. |
| Container | `resources.requests`, `resources.limits` | Requests help the scheduler choose a node; limits constrain resource use for the running container. |
| Container | `startupProbe`, `readinessProbe`, `livenessProbe` | Check startup, ability to receive traffic, and whether the container should be restarted, respectively. |
| Container | `volumeMounts` | Mounts a volume declared under the Pod-level `spec.volumes` into the container's filesystem. |

For example, adding `resources` and a `readinessProbe` would look like this inside the existing container entry:

```yaml
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - name: http
          containerPort: 80
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
      readinessProbe:
        httpGet:
          path: /
          port: http
```

This is an illustrative fragment, not the complete current manifest. The resource requests are used when placing the Pod; the readiness probe keeps the Pod out of ready Service endpoints until the HTTP check succeeds. A declared port helps name/reference port 80, while a separate Service is still needed for a stable Service address.

A Pod is built into Kubernetes, so it does not require a CustomResourceDefinition (CRD). A CRD is used to add a new resource type to the Kubernetes API.

### Namespace behavior

These examples do not pass a namespace flag. `kubectl` uses the namespace configured for the current context (often `default`). The Pod manifest also omits `metadata.namespace`, so the file-based commands use that same context namespace. Ensure the current context points to the namespace where you want to create or inspect the Pod. See the official [Kubernetes namespace guide](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/).

### Imperative method

Create an nginx Pod directly with `kubectl run`:

```shell
kubectl run nginx-pod --image=nginx:latest
```

Example response:

```text
pod/nginx-pod created
```

Running the command again in the same namespace fails because a Pod with that name already exists. The imperative Pod is named `nginx-pod`; the manifest Pod is named `nginx`.

### Create once from a manifest

The manifest omits `metadata.namespace`, so this command uses the current context namespace:

```shell
kubectl create -f pod.yaml
```

Example response:

```text
pod/nginx created
```

If a Pod with that name already exists in the selected namespace, `kubectl create` returns an `AlreadyExists` error and does not update it.

Example error:

```text
Error from server (AlreadyExists): error when creating "pod.yaml": pods "nginx" already exists
```

### Declarative method

Use `kubectl apply -f` to create the Pod if needed and apply later changes from the manifest:

```shell
kubectl apply -f pod.yaml
```

First apply typically prints:

```text
pod/nginx created
```

Later applies may print `configured` or `unchanged`. These responses describe the API object operation; they do not mean the Pod is ready. Pod spec fields are often immutable, so changing the manifest may still require replacing the Pod as described in **4. Edit & Update**.

### Generate a manifest with `kubectl run`

Use client-side dry run to generate YAML or JSON without creating a Pod. Choose one output format; the filenames keep the outputs separate:

```shell
kubectl run nginx-pod --image=nginx:latest --dry-run=client -o yaml > new-pod.yaml
kubectl run nginx-pod --image=nginx:latest --dry-run=client -o json > new-pod.json
```

Because stdout is redirected, the commands normally print nothing in the terminal. The generated YAML file has content similar to this (fields vary by kubectl version and flags):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    run: nginx-pod
spec:
  containers:
    - name: nginx-pod
      image: nginx:latest
  restartPolicy: Always
```

The JSON file represents the same Pod object in JSON syntax. The dry-run output is a starting manifest; review it before applying. If you generated JSON, use the matching `.json` file.

For example, the generated JSON includes fields like these:

```json
{
  "apiVersion": "v1",
  "kind": "Pod",
  "metadata": {
    "name": "nginx-pod",
    "labels": {
      "run": "nginx-pod"
    }
  }
}
```

Apply one generated manifest:

```shell
kubectl apply -f new-pod.yaml
```

Example response when it creates the generated Pod:

```text
pod/nginx-pod created
```

Review the generated file before applying it. If it contains `metadata.namespace`, that value selects the namespace; if the field is omitted, `kubectl` uses the current context namespace.

## 2. Inspect & Monitor

### Pod lifecycle and component flow

This section follows a Pod created from `pod.yaml`. Applying a standalone Pod manifest creates only that Pod: if it is deleted or cannot run, Kubernetes does not automatically create a replacement. A Pod owned by a Deployment is different because the Deployment and ReplicaSet controllers maintain the requested number of Pods.

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

### Numbered lifecycle steps

1. **kubectl submits the manifest to the API server.** `kubectl` sends an API request to `kube-apiserver`. The API server authenticates the caller, checks authorization, runs applicable admission and validation, applies defaults, and persists the Pod object in the cluster's backing store, normally `etcd`. Other components read and update cluster objects through the API server rather than writing directly to etcd.
2. **The scheduler assigns a node.** `kube-scheduler` watches for Pods that have no node assigned. It filters and scores eligible nodes using resource requests, scheduling policies, affinity, taints and tolerations, and other constraints, then records the selected node through the API server. If a Pod already specifies `spec.nodeName`, the scheduler is bypassed. Scheduling only assigns a Pod once; a Pod is not moved to another node later.
3. **The node starts the Pod.** The `kubelet` on the selected node watches the API server for Pods assigned to that node. It makes sure the Pod's sandbox and containers match the Pod spec, using the Container Runtime Interface (CRI) to ask the node's container runtime (for example, containerd or CRI-O) to pull images and create/start containers. If the Pod has init containers, they must complete successfully before its application containers start. A CNI network plugin configures the Pod network when the runtime creates the Pod sandbox. A CSI storage driver (or another volume plugin) attaches or mounts requested storage. The kubelet coordinates Pod setup and monitors the result; these plugins and the runtime do the node-specific network, storage, and container work.
4. **The kubelet monitors containers and reports status.** The kubelet runs configured startup, readiness, and liveness probes and reports container state, Pod conditions, and events through the API server. The API server persists these updates. A readiness probe affects whether the Pod is considered ready; a liveness or startup probe can cause the kubelet to restart a container. If a container exits, the kubelet applies the Pod's `restartPolicy` (`Always` is the default). A container restart happens inside the same Pod and does not give it a new identity. Repeated failures may show `CrashLoopBackOff` in `kubectl`; that is a restart backoff indication, not a Pod phase. A Pod is `Pending` while it waits for scheduling or setup such as image download. It becomes `Running` when it is bound to a node, all containers have been created, and at least one is running, starting, or restarting. `Running` does not necessarily mean ready to receive traffic.
5. **Controllers react when they own the Pod.** The `kube-controller-manager` runs controllers that watch API objects and reconcile actual state toward desired state. A standalone Pod has no controller to replace it. For a Deployment, the Deployment controller manages a ReplicaSet, and the ReplicaSet controller creates replacement Pods when needed. A replacement is a new Pod with a new UID and may run on a different node.
6. **The Pod completes or is deleted.** A Pod whose containers are expected to finish usually ends in `Succeeded` if they all finish successfully or `Failed` if at least one fails. A long-running application Pod normally stays `Running` until deleted or otherwise terminated. On deletion, the API server records the request and grace period; the node kubelet runs any `preStop` hook, asks the runtime to stop containers (normally with `SIGTERM`), and force-stops remaining processes after the grace period. The control plane removes the Pod object after termination and any finalizers are handled. The kubelet/runtime and configured plugins clean up containers, networking, and mounted resources. If a controller owns the Pod, it may create a replacement to restore the desired replica count.

If a node stops reporting healthy heartbeats, the node controller (also run by `kube-controller-manager`) marks the node unhealthy and the control plane may eventually evict its Pods. A workload controller such as a ReplicaSet can then create replacement Pods. The original Pod is not rescheduled onto another node.

If a Pod is selected by a Service, the EndpointSlice controller updates Service endpoints based on Pod readiness. Traffic forwarding is then handled by the cluster's Service data plane, commonly `kube-proxy`, though some clusters use a CNI or other replacement. Services and their networking components are adjacent to Pod lifecycle; they are not required to create or run a Pod.

### Components and their responsibilities

| Component | Responsibility in the Pod lifecycle |
| --- | --- |
| `kubectl` | Sends create, update, read, and delete requests to the Kubernetes API. It is a client, not a cluster component. |
| `kube-apiserver` | Exposes the Kubernetes API; authenticates and authorizes requests, applies admission and validation, and serves reads and writes from the backing store. |
| `etcd` | Stores persisted Kubernetes API objects, including the Pod spec and reported status. |
| `kube-scheduler` | Selects and binds a node for each unscheduled Pod. |
| `kube-controller-manager` | Runs control loops such as Deployment, ReplicaSet, node, and EndpointSlice controllers; actions depend on which resources exist. |
| `cloud-controller-manager` (optional) | Integrates with a cloud provider for infrastructure tasks such as node and load balancer management; it does not schedule or start Pod containers. |
| `kubelet` | On each node, observes Pods assigned to that node, coordinates setup and shutdown, monitors containers and probes, and reports status. |
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

Phases are high-level summaries, not a guaranteed step-by-step sequence. `Terminating` and `CrashLoopBackOff` may appear in `kubectl` output, but they are not Pod phases. See the official [Kubernetes components](https://kubernetes.io/docs/concepts/overview/components/), [Pod lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/), and [controller](https://kubernetes.io/docs/concepts/architecture/controller/) documentation.

### Watch and inspect the Pod

Watch the Pod by name in the current context namespace:

```shell
kubectl get pod nginx --watch
```

Example output:

```text
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          30s
```

`--watch` keeps streaming updates; press `Ctrl+C` to stop watching. `READY` shows ready containers out of total containers; `STATUS` is the displayed state, not always the Pod phase.

Describe the Pod for conditions, node placement, events, and container details:

```shell
kubectl describe pod nginx
```

Representative output:

```text
Name:           nginx
Namespace:      <current-context-namespace>
Labels:         app=my-app,env=dev,type=frontend
Status:         Running
IP:             10.244.0.12
Node:           worker-1/192.168.1.20
Containers:
  nginx:
    Image:      nginx
    State:      Running
    Ready:      True
Events:
  Type    Reason   Message
  Normal  Pulled   Container image pulled
```

The namespace and node names are examples. Events can be empty after the Pod has been stable for a while. A standalone Pod typically has `Controlled By: <none>`; a controller-owned Pod names its owner.

### Show labels and node details

The current manifest labels the Pod `app=my-app`, `env=dev`, and `type=frontend`:

```shell
kubectl get pods nginx --show-labels
# If using the imperative Pod named nginx-pod:
kubectl get pods nginx-pod --show-labels
```

Example output:

```text
NAME   READY   STATUS    RESTARTS   AGE   LABELS
nginx  1/1     Running   0          30s   app=my-app,env=dev,type=frontend
```

`--show-labels` adds labels to the standard Pod columns.

The imperative Pod created with `kubectl run nginx-pod` has its own generated label, for example:

```text
NAME       READY   STATUS    RESTARTS   AGE   LABELS
nginx-pod  1/1     Running   0          30s   run=nginx-pod
```

Use `-o wide` to include node and network details:

```shell
kubectl get pods nginx -o wide
# If using the imperative Pod named nginx-pod:
kubectl get pods nginx-pod -o wide
```

Example output:

```text
NAME   READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE   READINESS GATES
nginx  1/1     Running   0          30s   10.244.0.12   worker-1   <none>           <none>
```

The Pod IP and node are assigned at runtime. They can change when a new Pod is created.

### Follow container logs

This manifest has one container named `nginx`. The `-c` option is optional for a single-container Pod, but explicit container selection is useful in multi-container Pods:

```shell
# Print existing logs and exit:
kubectl logs nginx
# Follow new log lines:
kubectl logs -f nginx
# Explicitly select the nginx container:
kubectl logs -f nginx -c nginx
```

Logs are the container's stdout/stderr, so their content depends on application activity. This nginx image may produce no access-log lines until it receives requests. `-f` streams new lines until you stop it with `Ctrl+C`. For a multi-container Pod, specify `-c <container-name>` or use `--all-containers=true` to include every container.

### Open a shell in a container

Run a command in the Pod's container:

```shell
# Let kubectl choose the container (the Pod has one container):
kubectl exec -it nginx -- /bin/bash
# Explicitly choose the container:
kubectl exec -it nginx -c nginx -- /bin/bash
```

A successful interactive command opens a prompt similar to:

```text
root@nginx:/#
```

With one container, `-c nginx` is optional; without it, kubectl uses the default-container annotation or the first container. This manifest names its container `nginx`; use `-c nginx1` only if the Pod actually has a container named `nginx1`. The command after `--` runs inside the container, and the image must include the requested shell. If `/bin/bash` is unavailable, try `/bin/sh`.

### Export a running Pod for inspection

Save the live Pod as YAML:

```shell
kubectl get pod nginx -o yaml > nginx-export.yaml
```

The shell writes the command output to the file, so there is normally no terminal output. The file contains a YAML object, including fields similar to:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: <current-context-namespace>
  uid: <generated-uid>
  resourceVersion: "<generated-version>"
  labels:
    app: my-app
    env: dev
    type: frontend
```

An exported object includes server-generated fields such as `metadata.uid`, `metadata.resourceVersion`, `metadata.creationTimestamp`, `metadata.managedFields`, and `status`. Remove those fields before treating an export as a clean source manifest.

## 3. Scale Replicas

A Pod has no `replicas` field and cannot scale itself. To run multiple copies and replace failed Pods automatically, scale the Deployment or standalone ReplicaSet that owns the Pods. A standalone Pod such as `nginx` is not replaced when deleted.

Scale a Deployment (replace `<deployment-name>` with its actual name):

```shell
kubectl scale deployment/<deployment-name> --replicas=3
```

Example response:

```text
deployment.apps/<deployment-name> scaled
```

Scale a standalone ReplicaSet only when you manage it directly:

```shell
kubectl scale rs/<replicaset-name> --replicas=3
```

Example response:

```text
replicaset.apps/<replicaset-name> scaled
```

These commands change the controller's desired Pod count, not a Pod object. If a ReplicaSet is managed by a Deployment, scale the Deployment instead. Monitor the controller and Pod counts using `kubectl get deployment`, `kubectl get rs`, and `kubectl get pods`. See the [ReplicaSet guide](../ReplicaSet/README.md) and [Deployment guide](../Deployment/README.md) for controller details.

## 4. Edit & Update

### Edit a live Pod

Edit the live Pod object in your configured editor. The command uses the current context namespace:

```shell
kubectl edit pod nginx
```

After a valid change is saved, kubectl typically prints:

```text
pod/nginx edited
```

Some Pod fields are immutable and cannot be changed in place. If the Pod belongs to a Deployment or ReplicaSet, edit the owning controller's Pod template instead; changing an individual managed Pod does not update the controller's desired configuration.

### Replace a Pod from a manifest

`replace --force` deletes the existing Pod and then creates it again from the manifest:

```shell
kubectl replace --force -f pod.yaml
```

Typical output:

```text
pod "nginx" deleted
pod/nginx replaced
```

This is a delete-and-recreate operation: the Pod gets a new UID and may get a different IP address, and there can be downtime. The manifest must contain the complete resource spec. Prefer `kubectl apply -f pod.yaml` for ordinary manifest changes; force replacement is useful when a change cannot be made to the existing Pod in place.

## 5. Expose via a Service

A Service provides a stable network endpoint and routes traffic to Pods selected by labels. It targets Pods, not the Pod manifest file. The current Pod has labels `app=my-app`, `env=dev`, and `type=frontend`. Declaring `containerPort: 80` alone would not create a Service or publish a host port.

Expose the nginx Pod on port 80 with a cluster-internal Service:

```shell
kubectl expose pod nginx --name=nginx-service --port=80 --target-port=80 --type=ClusterIP
```

Example response:

```text
service/nginx-service exposed
```

The Service selects the Pod using its labels. `port` is the port clients use to reach the Service; `target-port` is the port on the nginx container. `ClusterIP` is reachable only from inside the cluster. Use a suitable type such as `NodePort` or `LoadBalancer` only when external access is intended and supported by the cluster.

Check the Service:

```shell
kubectl get service nginx-service
```

Typical output:

```text
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-service   ClusterIP   10.96.120.35   <none>        80/TCP    10s
```

`CLUSTER-IP` is allocated by the cluster. `PORT(S)` shows the Service port and protocol.

Describe the Service to inspect its selector and endpoints:

```shell
kubectl describe service nginx-service
```

Relevant output:

```text
Name:       nginx-service
Type:       ClusterIP
IP:         10.96.120.35
Port:       <unset>  80/TCP
TargetPort: 80/TCP
Selector:   app=my-app,env=dev,type=frontend
Endpoints:  10.244.0.12:80
```

The selector must match the Pod labels. An `Endpoints: <none>` result usually means no ready Pod currently matches the Service selector.

A Service created from a standalone Pod only has that Pod as a backend. A controller-managed workload is usually preferable when the application should remain available as Pods are replaced or scaled.

## 6. Delete & Orphan

### Delete the Pod

Delete the Pod from the manifest. Since the current `pod.yaml` omits `metadata.namespace`, the command uses the current context namespace:

```shell
kubectl delete -f pod.yaml
```

Example response:

```text
pod "nginx" deleted
```

You can also delete by name:

```shell
kubectl delete pod nginx
```

Typical response:

```text
pod "nginx" deleted
```

A standalone Pod does not create a replacement for itself. If a Deployment or ReplicaSet owns the Pod, its controller may create a replacement Pod to restore the desired replica count.

### Deletion and graceful shutdown

When you delete a Pod, the API server records the deletion request and grace period. The kubelet on the node runs any `preStop` hook, asks the runtime to stop the containers (normally with `SIGTERM`), and force-stops remaining processes after the grace period. It reports terminal status; the control plane removes the Pod object after termination and any finalizers are handled. The kubelet/runtime and configured plugins clean up the containers, networking, and mounted resources.

### What orphaning means

A Pod is a workload instance, not a controller. Deleting a standalone Pod does not orphan or preserve other Pods. Orphan deletion applies to an owning controller: it deletes the controller object while leaving its dependent Pods running. Those Pods lose their controlling owner reference and may be adopted by a new matching ReplicaSet.

For a standalone ReplicaSet, the command looks like this (replace `<replicaset-name>` with its name):

```shell
kubectl delete rs <replicaset-name> --cascade=orphan
```

Example response:

```text
replicaset.apps "<replicaset-name>" deleted
```

The response confirms deletion of the ReplicaSet object; the Pods remain. A new ReplicaSet can adopt orphan Pods when they are in the same namespace and their labels match its selector. Do not orphan a ReplicaSet managed by a Deployment to perform an update; update the Deployment instead. See the [ReplicaSet guide](../ReplicaSet/README.md) for the adoption workflow.

## References

- [Pod API reference](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/)
- [Container image guide](https://kubernetes.io/docs/concepts/containers/images/)
- [Resource requests and limits](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Probe guide](https://kubernetes.io/docs/concepts/workloads/pods/probes/)
- [Kubernetes components](https://kubernetes.io/docs/concepts/overview/components/)
- [Pod lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Controller architecture](https://kubernetes.io/docs/concepts/architecture/controller/)
- [Namespace guide](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [kubectl apply](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
- [kubectl create](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/)
- [kubectl get](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
- [kubectl delete](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_delete/)
- [kubectl scale](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_scale/)
- [kubectl expose](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_expose/)
- [kubectl replace](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_replace/)
- [kubectl logs](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)
- [kubectl exec](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_exec/)
- [kubectl edit](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_edit/)
- [kubectl run](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_run/)

