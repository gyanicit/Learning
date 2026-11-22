# Kubernetes Deployment

## Table of Contents

- [1. Create & Apply](#1-create--apply)
  - [What a Deployment manages](#what-a-deployment-manages)
  - [Reading `deployment.yaml`](#reading-deploymentyaml)
  - [Important fields in this manifest](#important-fields-in-this-manifest)
  - [Imperative creation](#imperative-creation)
  - [Create once from the manifest](#create-once-from-the-manifest)
  - [Apply the manifest declaratively](#apply-the-manifest-declaratively)
  - [Generate a manifest with `kubectl create deployment`](#generate-a-manifest-with-kubectl-create-deployment)
- [2. Inspect & Monitor](#2-inspect--monitor)
  - [Explain the API fields](#explain-the-api-fields)
  - [Deployment lifecycle and controller flow](#deployment-lifecycle-and-controller-flow)
  - [Numbered rollout steps](#numbered-rollout-steps)
  - [Component responsibilities](#component-responsibilities)
  - [Inspect the Deployment, ReplicaSet, and Pods](#inspect-the-deployment-replicaset-and-pods)
  - [Check the ownership chain](#check-the-ownership-chain)
  - [Watch rollout status and history](#watch-rollout-status-and-history)
- [3. Scale Replicas](#3-scale-replicas)
- [4. Edit & Update](#4-edit--update)
  - [Edit the live Deployment](#edit-the-live-deployment)
  - [Change the container image](#change-the-container-image)
  - [Pause, resume, and roll back a rollout](#pause-resume-and-roll-back-a-rollout)
- [5. Expose via a Service](#5-expose-via-a-service)
- [6. Delete & Orphan](#6-delete--orphan)
  - [Delete the Deployment and its workload](#delete-the-deployment-and-its-workload)
  - [Delete the Deployment but keep its ReplicaSets](#delete-the-deployment-but-keep-its-replicasets)
- [References](#references)

> **Use a Deployment for most stateless applications.** A Deployment manages ReplicaSets and provides declarative updates, rollout status, history, and rollback. The ReplicaSet controller maintains the requested Pod count; kubelets and container runtimes run the containers in those Pods.

This guide describes [deployment.yaml](deployment.yaml), which defines `nginx-deployment` with three desired Pods. Commands omit namespace flags and use the namespace selected by the current kubeconfig context. The manifest also omits `metadata.namespace`.

> **About the outputs below:** These are representative examples. Generated ReplicaSet and Pod names, ages, IP addresses, event counts, and status vary by cluster and timing; output excerpts omit unrelated fields.

## 1. Create & Apply

### What a Deployment manages

A Deployment is a controller for an application release. It manages one or more ReplicaSets; each ReplicaSet manages Pods from a Pod template:

```text
Deployment
  └── ReplicaSet for a Pod-template revision
        └── Pods
              └── Containers
```

The Deployment controller creates a new ReplicaSet when the Pod template changes and scales ReplicaSets to carry out a rollout. The ReplicaSet controller creates or deletes Pods to meet its replica count. A Deployment does not start containers directly; the scheduler, node kubelet, and container runtime participate after Pod objects are created.

### Reading `deployment.yaml`

The manifest creates a Deployment named `nginx-deployment`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: my-app
    env: dev
    type: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
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

The Deployment selector matches Pods labeled `app: my-app`. The template contains the Pod metadata and Pod spec used for each replica. The Deployment controller creates a ReplicaSet from that template; the ReplicaSet then creates the Pods.

### Important fields in this manifest

| Field | Meaning |
| --- | --- |
| `apiVersion: apps/v1` and `kind: Deployment` | Select the stable `apps/v1` Deployment API resource. |
| `metadata.name: nginx-deployment` | Names this Deployment. Names must be unique within a namespace. |
| `metadata.labels` | Labels the Deployment object itself. These labels do not select the Pods it manages. |
| `spec.replicas: 3` | Desired number of Pods. The Deployment passes replica counts to its ReplicaSets; observed counts can temporarily differ while reconciliation is in progress. |
| `spec.selector.matchLabels` | Selects Pods labeled `app: my-app`. It must match the labels in `spec.template.metadata.labels`. The selector is immutable after the Deployment is created. |
| `spec.template` | The Pod template copied into a managed ReplicaSet. Changes to this template trigger a rollout. |
| `spec.template.metadata.name: nginx` | A name value in the template. The ReplicaSet creates Pods with generated names such as `nginx-deployment-<hash>-<suffix>`; do not expect every Pod to be named `nginx`. |
| `spec.template.metadata.labels` | Labels placed on created Pods. The `app: my-app` label satisfies the selector; the other labels are additional metadata. |
| `spec.template.spec.containers` | Container definitions for each new Pod. This template has one container named `nginx` using image `nginx`. |
| `spec.strategy` | Not set here. The default is `RollingUpdate`, which replaces old Pods gradually. |

The image omits a tag, so it resolves to `nginx:latest`; Kubernetes defaults its pull policy to `Always`. For repeatable releases, use a specific version or digest and update it through the Deployment template.

The default rolling update parameters are `maxSurge: 25%` and `maxUnavailable: 25%`. They control how many extra or unavailable Pods Kubernetes permits during an update. Percentage values are rounded up for `maxSurge` and down for `maxUnavailable`; with three desired replicas, the defaults permit one extra Pod and zero unavailable Pods. These defaults are not written in this manifest. You can set `spec.strategy.rollingUpdate.maxSurge` and `spec.strategy.rollingUpdate.maxUnavailable` explicitly. `Recreate` is another strategy, but it stops old Pods before starting new ones and can cause downtime.

The selector and template labels must remain aligned. Avoid reusing `app: my-app` for unrelated workloads or another controller in the same namespace: overlapping selectors can cause controllers to manage the same Pods.

### Imperative creation

For a quick Deployment without writing a manifest first, use `kubectl create deployment`:

```shell
kubectl create deployment nginx-demo --image=nginx:1.27 --replicas=3
```

Example response:

```text
deployment.apps/nginx-demo created
```

The command creates a Deployment object. Its controller creates a ReplicaSet and Pods asynchronously. The generated Deployment uses labels based on `nginx-demo`; it does not use the `app: my-app` selector in this guide's manifest. Do not apply both examples as if they were the same Deployment: they have different names and selectors.

`kubectl run` creates a Pod, not a Deployment. Use `kubectl create deployment` when you want this imperative workflow.

### Create once from the manifest

Use `kubectl create -f` for a one-time create from the checked-in manifest:

```shell
kubectl create -f deployment.yaml
```

Example response:

```text
deployment.apps/nginx-deployment created
```

If the Deployment already exists, `kubectl create` reports `AlreadyExists` and does not update it:

```text
Error from server (AlreadyExists): error when creating "deployment.yaml": deployments.apps "nginx-deployment" already exists
```

Use `kubectl apply -f` when you want to create the object if needed and apply later changes from the manifest.

### Apply the manifest declaratively

Apply the manifest to create or update the Deployment:

```shell
kubectl apply -f deployment.yaml
```

Example output on first apply:

```text
deployment.apps/nginx-deployment created
```

Later applies typically print `configured` when the Deployment object changes or `unchanged` when it does not. This response describes the API object operation; it does not mean the rollout has completed or that all Pods are ready. Use `kubectl rollout status deployment/nginx-deployment` to check progress.

### Generate a manifest with `kubectl create deployment`

Client-side dry run prints a Deployment manifest without creating the Deployment. Redirect the output to a file:

```shell
kubectl create deployment nginx-demo --image=nginx:1.27 --replicas=3 --dry-run=client -o yaml > deployment-generated.yaml
```

Because stdout is redirected, the command normally prints nothing in the terminal. Review the generated file before using it. Its generated labels and names can differ from [deployment.yaml](deployment.yaml).

Apply the generated file only if you intend to create or update the resource it describes:

```shell
kubectl apply -f deployment-generated.yaml
```

On first apply, kubectl typically prints `deployment.apps/nginx-demo created`; later applies print `configured` or `unchanged`.

## 2. Inspect & Monitor

### Explain the API fields

`deploy` is the short name for `deployment`. `kubectl explain` reads the schema published by the API server:

```shell
kubectl explain deployment
kubectl explain deploy.spec --recursive
kubectl explain deployment.spec.selector
kubectl explain deployment.spec.strategy
kubectl explain deployment.spec.template
```

Representative output for `kubectl explain deployment`:

```text
KIND:       Deployment
VERSION:    apps/v1

DESCRIPTION:
    Deployment enables declarative updates for Pods and ReplicaSets.

FIELDS:
  apiVersion    <string>
  kind          <string>
  metadata      <Object>
  spec          <Object>
  status        <Object>
```

The recursive spec output includes fields such as `replicas`, `selector`, `strategy`, `template`, and `revisionHistoryLimit`. Exact descriptions can vary by Kubernetes version.

For example, `kubectl explain deployment.spec.strategy` describes `type` and the `rollingUpdate` settings:

```text
FIELDS:
  rollingUpdate <Object>
  type          <string>
```

### Deployment lifecycle and controller flow

Applying `deployment.yaml` creates a Deployment, not three Pods directly. The controllers and node components work asynchronously to make observed state match the requested state:

```text
kubectl apply -f deployment.yaml
       |
       v
kube-apiserver: authenticates, validates, admits, and persists the Deployment
       |                                      |
       |                                      +--> etcd: stores API objects and status
       v
Deployment controller (in kube-controller-manager)
       | creates or updates the active ReplicaSet
       v
ReplicaSet controller (in kube-controller-manager)
       | creates or removes Pod objects to meet the replica count
       v
kube-scheduler: assigns each new Pod to a node
       |
       v
Node kubelet --> CRI container runtime: creates the Pod sandbox and containers
       |                |
       |                +--> CNI networking / CSI storage, when configured
       v
Pod readiness and status are reported through kube-apiserver
       |
       +--> Deployment controller updates rollout and availability status
       +--> Service endpoint controllers include ready matching Pods, if selected
```

The arrows show reconciliation relationships, not a synchronous command chain. Controllers watch API objects, compare desired and observed state, then write updates through the API server. Components do not write directly to etcd.

### Numbered rollout steps

1. **The client submits the desired Deployment.** `kubectl` sends the manifest to `kube-apiserver`. The API server authenticates and authorizes the request, runs admission and validation, applies defaults, and stores the Deployment object in `etcd`.
2. **The Deployment controller creates or reconciles a ReplicaSet.** The controller running in `kube-controller-manager` compares the Deployment's Pod template and replica count with its ReplicaSets. For the first create, it creates a ReplicaSet. A change to `spec.template` creates a new revision and ReplicaSet; a replica-count-only change scales the existing ReplicaSet without creating a new revision.
3. **The ReplicaSet controller creates Pod objects.** It compares the count of Pods matching its selector with its desired count and creates or removes Pods to converge. During a rolling update, the Deployment controller adjusts the old and new ReplicaSets according to the update strategy.
4. **The scheduler assigns each unscheduled Pod to a node.** `kube-scheduler` filters and scores eligible nodes using resource requests, affinity, taints and tolerations, and other policies, then records a binding through the API server.
5. **The node kubelet starts and monitors containers.** The kubelet asks the node's CRI runtime (for example, containerd or CRI-O) to pull images and create the Pod sandbox and containers. CNI networking and CSI storage integrations participate when the Pod requires them. The kubelet runs probes and reports status.
6. **Readiness and rollout status advance.** The Deployment controller observes available Pods and continues scaling the new ReplicaSet up and old ReplicaSet down until the rollout satisfies its desired state or reports a problem. If a Service selects the Pod labels, EndpointSlice controllers update its endpoints based on Pod readiness; the cluster data plane forwards Service traffic.

Readiness is not the same as a running container. A Pod can be `Running` but not ready, and an image pull or scheduling problem can leave it `Pending`. Inspect Pod events and Deployment conditions when rollout progress stalls.

### Component responsibilities

| Component | Responsibility in this Deployment lifecycle |
| --- | --- |
| `kubectl` | Sends requests to the Kubernetes API; it is the client, not a cluster controller. |
| `kube-apiserver` | Authenticates and authorizes requests, validates and admits objects, and serves cluster state to components. |
| `etcd` | Persists the Deployment, ReplicaSets, Pods, and their status as Kubernetes API objects. |
| Deployment controller | Runs in `kube-controller-manager`; creates and scales ReplicaSets to reconcile the Deployment and its rollout strategy. |
| ReplicaSet controller | Runs in `kube-controller-manager`; creates or removes Pods to match a ReplicaSet's desired count. |
| `kube-scheduler` | Assigns each unscheduled Pod to an eligible node. |
| Node `kubelet` | Ensures Pods assigned to its node are running, performs probes, coordinates shutdown, and reports status. |
| Container runtime | Pulls images and runs the Pod sandbox and containers through CRI. |
| CNI plugin / CSI driver | Configures Pod networking and provides requested storage, when the cluster and Pod use them. |
| EndpointSlice controller | Updates the endpoints for a Service based on matching Pods and their readiness. |
| `kube-proxy` or replacement data plane | Implements Service traffic forwarding on clusters that use it; some CNI implementations provide this function instead. |

The Deployment controller does not schedule Pods or start containers. It manages ReplicaSets; the scheduler, kubelet, runtime, and configured node plugins handle the later stages.

### Inspect the Deployment, ReplicaSet, and Pods

Use these commands to follow the resource chain. The label selector in the checked-in manifest is `app=my-app`:

| Task | Command | What it shows |
| --- | --- | --- |
| Show the Deployment summary | `kubectl get deployment nginx-deployment` | Desired readiness, updated replicas, available replicas, and age. |
| Show ReplicaSets | `kubectl get rs` | Desired, current, and ready Pod counts for each ReplicaSet in the current namespace. |
| Show this Deployment's Pods | `kubectl get pods -l app=my-app --show-labels` | Pod readiness, status, restarts, age, and labels. |
| Show node and IP details | `kubectl get pods -l app=my-app -o wide` | Assigned node and Pod IP for each matching Pod. |
| Read the live Deployment object | `kubectl get deployment nginx-deployment -o yaml` | Spec, generated metadata, conditions, and status. |
| Describe the Deployment | `kubectl describe deployment nginx-deployment` | Selector, strategy, Pod template, counts, conditions, and events. |

Typical `kubectl get deployment nginx-deployment` output after the Pods become available:

```text
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           2m
```

- `READY` is the number of ready replicas out of the desired count.
- `UP-TO-DATE` is the number running the latest Pod template.
- `AVAILABLE` is the number considered available to serve the workload.

The counts can temporarily differ while the Deployment is scaling or rolling out a change.

Typical `kubectl get rs` output for this Deployment:

```text
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-7d9c8f6b5c   3         3         3       2m
```

The generated hash identifies a Pod-template revision. A later template update normally creates another ReplicaSet with a different hash.

Typical `kubectl get pods -l app=my-app --show-labels` output:

```text
NAME                            READY   STATUS    RESTARTS   AGE   LABELS
nginx-deployment-7d9c8f6b5c-a1b2c  1/1     Running   0          2m    app=my-app,env=dev,type=frontend
nginx-deployment-7d9c8f6b5c-d3e4f  1/1     Running   0          2m    app=my-app,env=dev,type=frontend
nginx-deployment-7d9c8f6b5c-g5h6i  1/1     Running   0          2m    app=my-app,env=dev,type=frontend
```

The actual suffixes are generated by Kubernetes and will differ. The label selector filters by labels, not by Deployment name.

`kubectl get pods -l app=my-app -o wide` adds node and network columns:

```text
NAME                              READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE   READINESS GATES
nginx-deployment-7d9c8f6b5c-a1b2c  1/1     Running   0          2m    10.244.0.12   worker-1   <none>           <none>
```

The Pod IP and node are assigned at runtime and may change when a replacement Pod is created.

`kubectl get deployment nginx-deployment -o yaml` returns the full API object. This excerpt shows the desired template and reported replica status:

```yaml
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
status:
  replicas: 3
  updatedReplicas: 3
  readyReplicas: 3
  availableReplicas: 3
```

`kubectl describe deployment nginx-deployment` includes a readable summary like this:

```text
Name:                   nginx-deployment
Namespace:              <current-namespace>
Labels:                 app=my-app, env=dev, type=frontend
Selector:               app=my-app
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:               app=my-app, env=dev, type=frontend
  Containers:
   nginx:
    Image:              nginx
Conditions:
  Type          Status  Reason
  Available     True    MinimumReplicasAvailable
  Progressing   True    NewReplicaSetAvailable
```

The `Replicas` row compares the desired count with updated, total, available, and unavailable Pods. Conditions explain whether the Deployment is progressing and has enough available replicas. Events may be empty after the rollout has been stable for a while.

### Check the ownership chain

The ownership chain is Deployment → ReplicaSet → Pods. Find the ReplicaSet and inspect its owner:

```shell
kubectl get rs
kubectl describe rs <replicaset-name>
```

For a ReplicaSet created by this Deployment, the description includes a line similar to:

```text
Controlled By:  Deployment/nginx-deployment
```

Inspect one matching Pod to see that its direct controller is the ReplicaSet:

```shell
kubectl describe pod <pod-name>
```

Relevant output:

```text
Name:           nginx-deployment-7d9c8f6b5c-a1
Labels:         app=my-app,env=dev,type=frontend
Status:         Running
Controlled By:  ReplicaSet/nginx-deployment-7d9c8f6b5c
```

Pods refer to their direct owner, the ReplicaSet. The ReplicaSet in turn is owned by the Deployment. Use `kubectl get rs <replicaset-name> -o yaml` or `kubectl get pod <pod-name> -o yaml` to inspect `metadata.ownerReferences`.

### Watch rollout status and history

Watch the Deployment until the current rollout completes:

```shell
kubectl rollout status deployment/nginx-deployment
```

Successful output:

```text
deployment "nginx-deployment" successfully rolled out
```

List the revisions recorded for the Deployment:

```shell
kubectl rollout history deployment/nginx-deployment
```

Representative output:

```text
deployment.apps/nginx-deployment
REVISION  CHANGE-CAUSE
1         <none>
```

The history records Pod-template revisions. Scaling only the replica count does not create a new revision. `CHANGE-CAUSE` may be empty unless a cause is recorded; it is not a reliable audit log by itself.

## 3. Scale Replicas

Scale the Deployment to five desired replicas:

```shell
kubectl scale deployment/nginx-deployment --replicas=5
```

Example response:

```text
deployment.apps/nginx-deployment scaled
```

This updates the desired replica count. It does not wait for all five Pods to become ready. Watch the Deployment and Pods:

```shell
kubectl get deployment nginx-deployment
kubectl get pods -l app=my-app
```

The Deployment scales its active ReplicaSet; the ReplicaSet controller then creates or deletes Pods. Scale down with the same command and a smaller count, for example `--replicas=1`.

The checked-in manifest declares `replicas: 3`. If you scale it to five and later apply that unchanged manifest, the manifest can set the desired count back to three. For a declarative workflow, update `spec.replicas` in `deployment.yaml` and apply the file. If an HPA manages this Deployment, let the autoscaler control the replica count rather than repeatedly scaling it by hand.

## 4. Edit & Update

### Edit the live Deployment

Open the live Deployment in the configured editor:

```shell
kubectl edit deployment nginx-deployment
```

After a valid change is saved, kubectl typically prints:

```text
deployment.apps/nginx-deployment edited
```

Changes under `.spec.template` trigger a new rollout. Changes such as `.spec.replicas` change scale without creating a new Pod-template revision. The selector is immutable after creation. For a change that should persist in source control, edit `deployment.yaml` and use `kubectl apply -f deployment.yaml` rather than changing only the live object.

### Change the container image

The container in this manifest is named `nginx`. Update its image to a pinned version:

```shell
kubectl set image deployment/nginx-deployment nginx=nginx:1.27
```

Example response:

```text
deployment.apps/nginx-deployment image updated
```

This changes the live Pod template and starts a rollout. Monitor it:

```shell
kubectl rollout status deployment/nginx-deployment
```

If you only use `kubectl set image`, the file on disk still says `image: nginx`. Update the manifest too, or a later apply may restore the old image value.

The default `RollingUpdate` strategy gradually replaces Pods. The Deployment creates a new ReplicaSet for the updated template, then scales the new and old ReplicaSets according to `maxSurge` and `maxUnavailable`. When the rollout completes, the old ReplicaSet is usually scaled to zero and retained for revision history.

### Pause, resume, and roll back a rollout

Pause the Deployment controller while preparing a batch of template updates:

```shell
kubectl rollout pause deployment/nginx-deployment
```

Example response:

```text
deployment.apps/nginx-deployment paused
```

Edit `spec.template` in `deployment.yaml` (for example, change the container image), apply the change, then resume the Deployment:

```shell
# Apply after editing the Pod template in deployment.yaml:
kubectl apply -f deployment.yaml
kubectl rollout resume deployment/nginx-deployment
```

The apply command typically prints `deployment.apps/nginx-deployment configured`. The resume command prints:

```text
deployment.apps/nginx-deployment resumed
```

Pausing a Deployment does **not** pause or freeze its running containers. It pauses Deployment rollout progression; existing Pods continue to run. Check status after resuming with `kubectl rollout status deployment/nginx-deployment`.

Undo the latest rollout if the previous revision is still available:

```shell
kubectl rollout undo deployment/nginx-deployment
```

Example response:

```text
deployment.apps/nginx-deployment rolled back
```

You can target a particular revision with `kubectl rollout undo deployment/nginx-deployment --to-revision=<revision>`. Resume a paused Deployment before rolling it back. Rollback restores an earlier Pod template; it does not reverse unrelated changes to external systems or application data.

## 5. Expose via a Service

A Service provides a stable network endpoint and selects Pods by labels. It targets the Pods created from this Deployment, not the Deployment object itself. The Pods have `app=my-app`, `env=dev`, and `type=frontend`; `containerPort` is not declared, but nginx listens on port 80 in the container.

Create a cluster-internal Service for the Deployment:

```shell
kubectl expose deployment nginx-deployment --name=nginx-service --port=80 --target-port=80 --type=ClusterIP
```

Example response:

```text
service/nginx-service exposed
```

`port` is the port clients use to reach the Service; `target-port` is the destination port on each selected Pod. `ClusterIP` is reachable from inside the cluster. `kubectl expose` uses the Deployment's selector for the Service, so it selects Pods labeled `app=my-app`.

Inspect the Service and its endpoints:

```shell
kubectl get service nginx-service
kubectl describe service nginx-service
```

Typical `kubectl get service nginx-service` output:

```text
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-service   ClusterIP   10.96.120.35   <none>        80/TCP    10s
```

Relevant `kubectl describe service nginx-service` output:

```text
Name:       nginx-service
Type:       ClusterIP
IP:         10.96.120.35
Port:       <unset>  80/TCP
TargetPort: 80/TCP
Selector:   app=my-app
Endpoints:  10.244.0.12:80,10.244.0.13:80
```

The selector must match Pod labels. Endpoints should list ready matching Pods; if it shows `<none>`, check that the Pods exist, are ready, and have `app=my-app`. EndpointSlice controllers maintain the endpoint data; the Service data plane, commonly `kube-proxy` or a replacement, handles traffic forwarding.

## 6. Delete & Orphan

### Delete the Deployment and its workload

Delete the Deployment from its manifest:

```shell
kubectl delete -f deployment.yaml
```

Example response:

```text
deployment.apps "nginx-deployment" deleted
```

You can also delete it by name:

```shell
kubectl delete deployment nginx-deployment
```

The name-based command reports the same deletion, typically:

```text
deployment.apps "nginx-deployment" deleted
```

By default, deleting a Deployment cascades to the ReplicaSets and Pods it owns. Dependent objects may take a short time to disappear while Kubernetes performs cleanup. If another Deployment uses the same labels, check its ownership and selectors before removing matching resources.

### Delete the Deployment but keep its ReplicaSets

Orphan deletion removes the Deployment object but leaves its dependent ReplicaSets. Those ReplicaSets continue managing their Pods at their current desired count, but no Deployment remains to manage rollouts or rollbacks:

```shell
kubectl delete deployment nginx-deployment --cascade=orphan
```

Typical response:

```text
deployment.apps "nginx-deployment" deleted
```

Check whether the ReplicaSet and Pods remain:

```shell
kubectl get rs
kubectl get pods -l app=my-app
```

Use orphan deletion only when you intentionally want to hand off or preserve that lower-level workload. Do not use it as a normal Deployment update or rollback method; change the Deployment's Pod template instead. See the [ReplicaSet guide](../ReplicaSet/README.md) for ReplicaSet ownership and orphan behavior.

## References

- [Deployments concept](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Deployment API reference](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/deployment-v1/)
- [kubectl create deployment](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_deployment/)
- [kubectl explain](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_explain/)
- [kubectl rollout](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/)
- [kubectl scale](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_scale/)
- [kubectl expose](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_expose/)
- [kubectl delete](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_delete/)
- [Kubernetes workload management overview](../README.md)
- [ReplicaSet guide](../ReplicaSet/README.md)
- [Pod guide](../Pod/README.md)
