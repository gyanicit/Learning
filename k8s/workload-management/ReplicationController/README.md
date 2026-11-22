# Kubernetes ReplicationController

## Table of Contents

- [1. Create & Apply](#1-create--apply)
  - [How a ReplicationController works](#how-a-replicationcontroller-works)
  - [Important fields in this manifest](#important-fields-in-this-manifest)
  - [Imperative creation](#imperative-creation)
  - [Apply the manifest](#apply-the-manifest)
- [2. Inspect & Monitor](#2-inspect--monitor)
  - [Explain the API fields](#explain-the-api-fields)
  - [Common inspection commands](#common-inspection-commands)
  - [Read the replica status](#read-the-replica-status)
  - [Check which ReplicationController owns a Pod](#check-which-replicationcontroller-owns-a-pod)
- [3. Scale Replicas](#3-scale-replicas)
- [4. Edit & Update](#4-edit--update)
- [5. Expose via a Service](#5-expose-via-a-service)
- [6. Delete & Orphan](#6-delete--orphan)
  - [Delete the ReplicationController and its Pods](#delete-the-replicationcontroller-and-its-pods)
  - [Delete the controller but keep its Pods](#delete-the-controller-but-keep-its-pods)
  - [Can an already-running Pod join a new ReplicationController?](#can-an-already-running-pod-join-a-new-replicationcontroller)
- [References](#references)

> **Legacy resource:** ReplicationControllers still work, but Kubernetes recommends Deployments for most replicated workloads. A Deployment manages ReplicaSets, which then manage Pods. Use this guide to understand or maintain ReplicationControllers already in use.

The example manifest is [replication-controller.yaml](replication-controller.yaml). It defines `nginx-rc` with three desired nginx Pod replicas.

> **About the outputs below:** These are representative examples. Names, ages, IP addresses, event counts, and status can differ by cluster and timing; output excerpts omit unrelated fields.

## 1. Create & Apply

### How a ReplicationController works

A ReplicationController continuously compares the number of Pods matching its selector with `.spec.replicas`:

- If there are too few matching Pods, it creates Pods from `.spec.template`.
- If there are too many, it deletes the excess Pods.
- If a managed Pod is deleted or lost, it creates a replacement to move back toward the desired count.

The controller manages Pods, not containers directly. The kubelet and container runtime on each selected node start and monitor the containers inside each Pod.

### Important fields in this manifest

| Field | Meaning |
| --- | --- |
| `apiVersion: v1` and `kind: ReplicationController` | Select the core API's ReplicationController resource. |
| `metadata.name: nginx-rc` | Names the controller. |
| `metadata.labels` | Labels the controller object itself. These do not determine which Pods it manages. |
| `spec.replicas: 3` | Desired number of matching Pods. The observed count can temporarily differ while the controller reconciles. |
| `spec.selector` | Identifies Pods the controller manages. This manifest explicitly selects Pods with `app: my-app`. ReplicationController selectors use equality-based matching. |
| `spec.template` | Pod template used when the controller needs to create a Pod. It includes the Pod labels and container specification. |
| `spec.template.metadata.labels` | Labels applied to each created Pod. They must match the controller selector. Avoid reusing the same selector for another controller. |
| `spec.template.spec.containers` | Container configuration for each newly created Pod; this example runs one `nginx` container. |

### Imperative creation

The current `kubectl` CLI does not provide a direct, manifest-free command to generate a ReplicationController (for example, there is no `kubectl create replicationcontroller ...` generator). `kubectl run` creates a Pod; it does not create a ReplicationController. A resource manifest is required to create this legacy resource.

### Apply the manifest

Use `kubectl apply -f` to create the ReplicationController if it does not exist, or apply the manifest to an existing object:

```shell
kubectl apply -f replication-controller.yaml
```

Example output when this is the first apply:

```text
replicationcontroller/nginx-rc created
```

On later applies, kubectl may print `configured` if it updates the object or `unchanged` if the manifest has no changes. `created`, `configured`, and `unchanged` describe the API object operation; they do not mean that all Pods are already ready.

Changes made to the Pod template and their effect on existing Pods are covered in **4. Edit & Update**.

## 2. Inspect & Monitor

### Explain the API fields

`kubectl explain` reads the API schema and field descriptions. The `rc` short name and the full resource name both identify this resource:

```shell
kubectl explain rc
kubectl explain replicationcontroller
kubectl explain rc.spec --recursive
kubectl explain rc.spec.template
```

Representative output for `kubectl explain rc` (the full resource name produces the same schema):

```text
KIND:       ReplicationController
VERSION:    v1

DESCRIPTION:
    ReplicationController represents the configuration of a replication controller.

FIELDS:
  apiVersion    <string>
  kind          <string>
  metadata      <Object>
  spec          <Object>
  status        <Object>
```

`kubectl explain rc.spec --recursive` includes the nested spec fields, for example:

```text
RESOURCE: spec <Object>

FIELDS:
  minReadySeconds <integer>
  replicas        <integer>
  selector        <map[string]string>
  template        <Object>
```

`kubectl explain rc.spec.template` prints the Pod template schema and its fields. The exact descriptions and field list depend on the Kubernetes version's API schema.

For example, its output identifies the template as an object and shows nested Pod metadata and spec fields:

```text
KIND:       ReplicationController
VERSION:    v1
RESOURCE:   template <Object>

DESCRIPTION:
    Template is the object that describes the pod that will be created.

FIELDS:
  metadata <Object>
  spec     <Object>
```

Use `--recursive` to see nested fields and their descriptions.

### Common inspection commands

`rc` is the short name for `replicationcontroller`. The commands below use the namespace selected by your current kubeconfig context.

| Task | Command | What it shows |
| --- | --- | --- |
| List ReplicationControllers | `kubectl get rc` | Desired, current, and ready replica counts. |
| Show one controller | `kubectl get rc nginx-rc -o wide` | The controller and additional columns. |
| Inspect the full controller manifest | `kubectl get rc nginx-rc -o yaml` | The live object specification, status, and metadata. |
| Describe one controller | `kubectl describe rc nginx-rc` | Selector, Pod template, counts, and related events. |
| List Pods selected by this manifest | `kubectl get pods -l app=my-app --show-labels` | Matching Pod names and their labels. |

Typical output for `kubectl get rc nginx-rc -o wide`:

```text
NAME      DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES   SELECTOR
nginx-rc  3         3         3       2m    nginx        nginx    app=my-app
```

`DESIRED`, `CURRENT`, and `READY` are replica counts. `SELECTOR` confirms which Pod labels the controller manages.

`kubectl get rc nginx-rc -o yaml` returns the full object. This excerpt highlights the fields to compare with the manifest; live output also contains generated metadata and status:

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-rc
spec:
  replicas: 3
  selector:
    app: my-app
  template:
    metadata:
      labels:
        app: my-app
status:
  replicas: 3
  readyReplicas: 3
```

`kubectl describe rc nginx-rc` includes a human-readable summary similar to:

```text
Name:               nginx-rc
Namespace:          <current-namespace>
Selector:           app=my-app
Replicas:           3 current / 3 desired
Pods Status:        3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:           app=my-app
  Containers:
   nginx:
    Image:          nginx
Events:
  Type    Reason             Message
  Normal  SuccessfulCreate   Created pod: nginx-rc-xxxxx
```

Events may be empty after the controller has been stable for a while. The namespace shown is the one selected by the current kubeconfig context.

Typical output for `kubectl get pods -l app=my-app --show-labels`:

```text
NAME                 READY   STATUS    RESTARTS   AGE   LABELS
nginx-rc-xxxxx       1/1     Running   0          2m    app=my-app
```

This lists only Pods matching `app=my-app`; `READY` and `STATUS` show whether each selected Pod is ready and running.

The controller may create Pod names with the `nginx-rc` prefix and a generated suffix. Use the actual name returned by `kubectl get pods` when inspecting a Pod.

### Read the replica status

Typical `kubectl get rc` columns are `NAME`, `DESIRED`, `CURRENT`, `READY`, and `AGE`:

```text
NAME      DESIRED   CURRENT   READY   AGE
nginx-rc  3         3         3       2m
```

- `DESIRED` is `.spec.replicas`.
- `CURRENT` is the number of matching Pods observed by the controller.
- `READY` is the number of matching Pods that are ready.

These counts can differ while Pods are being created, scheduled, or made ready.

### Check which ReplicationController owns a Pod

Describe a Pod to see its `Controlled By` field and recent events:

```shell
# Replace <pod-name> with a name returned by kubectl get pods.
kubectl describe pod <pod-name>
```

For a Pod managed by this example, the relevant output looks like this:

```text
Name:           nginx-rc-xxxxx
Namespace:      <current-namespace>
Labels:         app=my-app
Status:         Running
IP:             10.244.0.12
Controlled By:  ReplicationController/nginx-rc
```

`Controlled By` identifies the controller that owns the Pod. A Pod without a controlling owner is not currently managed by a workload controller. To inspect the full Pod object, including its labels and owner references:

```shell
kubectl get pod <pod-name> -o yaml
```

The `kubectl get pod <pod-name> -o yaml` output includes labels and owner references. A relevant excerpt is:

```yaml
metadata:
  name: nginx-rc-xxxxx
  labels:
    app: my-app
  ownerReferences:
    - apiVersion: v1
      kind: ReplicationController
      name: nginx-rc
      uid: 12345678-1234-1234-1234-123456789abc
      controller: true
```

The owner reference's `kind` and `name` identify the owning object; `controller: true` marks it as the controlling owner. Labels and selectors determine which Pods a ReplicationController targets.

## 3. Scale Replicas

Scale `nginx-rc` to five desired Pods:

```shell
kubectl scale rc nginx-rc --replicas=5
```

Example command response:

```text
replicationcontroller/nginx-rc scaled
```

This confirms the desired count was updated; it does not wait for all five Pods to become ready. Check `kubectl get rc` to monitor `DESIRED`, `CURRENT`, and `READY`.

To scale it down to one Pod:

```shell
kubectl scale rc nginx-rc --replicas=1
```

The response is also `replicationcontroller/nginx-rc scaled`. The controller will terminate excess Pods until it reaches the new desired count.

Scaling changes the controller's desired replica count. The ReplicationController then creates or removes matching Pods to move toward that count.

## 4. Edit & Update

Edit the live ReplicationController in your configured editor:

```shell
kubectl edit rc nginx-rc
```

This opens the live object in your configured editor. After saving a change, kubectl typically prints:

```text
replicationcontroller/nginx-rc edited
```

If you exit without saving changes, kubectl reports that the edit was cancelled or that no changes were made.

To update it from the manifest after changing `replication-controller.yaml`, apply the file again:

```shell
kubectl apply -f replication-controller.yaml
```

When the manifest changed, a typical response is:

```text
replicationcontroller/nginx-rc configured
```

`configured` means the API object was updated. It does not mean existing Pods were replaced or that new Pods are ready.

Applying or editing a ReplicationController updates the controller object, but it does **not** perform a controlled rolling update of existing Pods when the Pod template changes. Existing Pods keep their current image and settings; Pods created later as replacements use the updated template. For managed version updates and rollbacks, prefer a Deployment.

## 5. Expose via a Service

A Service gives clients a stable network endpoint and routes traffic to matching Pods. It selects Pods by their labels; it does not route traffic to the ReplicationController itself. This manifest's Pods use `app: my-app`.

Create an internal `ClusterIP` Service for the nginx Pods:

```shell
kubectl expose rc nginx-rc --name=nginx-service --port=80 --target-port=80 --type=ClusterIP
```

Example response:

```text
service/nginx-service exposed
```

This means the Service object was created. It does not by itself confirm that any ready Pods match its selector.

`kubectl expose` uses the ReplicationController's selector for the Service selector. Here, that selector matches Pods labeled `app=my-app`. The Service's `port` is the port clients use, and `target-port` is the port on the nginx containers. `ClusterIP` is reachable from within the cluster; use an appropriate Service type such as `NodePort` or `LoadBalancer` when external access is intended and supported by the cluster.

Inspect the Service and its selected endpoints:

```shell
kubectl get service nginx-service
kubectl describe service nginx-service
```

Typical `kubectl get service nginx-service` output:

```text
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-service   ClusterIP   10.96.120.35   <none>        80/TCP    10s
```

`CLUSTER-IP` is allocated by the cluster and is reachable only within the cluster for a `ClusterIP` Service. `PORT(S)` shows the Service port and protocol.

`kubectl describe service nginx-service` includes a summary like:

```text
Name:              nginx-service
Type:              ClusterIP
IP:                10.96.120.35
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Selector:          app=my-app
Endpoints:         10.244.0.12:80,10.244.0.13:80
```

The Service's endpoints should include addresses of ready Pods matching `app=my-app`. If `Endpoints` shows `<none>`, check that matching Pods exist, are ready, and carry the expected label.

## 6. Delete & Orphan

### Delete the ReplicationController and its Pods

By default, deleting a ReplicationController also deletes the Pods it manages:

```shell
kubectl delete rc nginx-rc
```

Example response:

```text
replicationcontroller "nginx-rc" deleted
```

With normal cascading deletion, the controller's dependent Pods are deleted too. Check the selected Pods with:

```shell
kubectl get pods -l app=my-app --show-labels
```

If none remain, kubectl typically prints:

```text
No resources found in <current-namespace> namespace.
```

A different controller could still create replacement Pods if it selects the same labels. The namespace in the message depends on the current kubeconfig context.

### Delete the controller but keep its Pods

Use orphan deletion when you need to remove or replace the controller while leaving its Pods running:

```shell
kubectl delete rc nginx-rc --cascade=orphan
```

Example response:

```text
replicationcontroller "nginx-rc" deleted
```

The message reports deletion of the RC object. Because `--cascade=orphan` was used, its Pods are left running rather than deleted; verify them with `kubectl get pods --show-labels`.

The Pods remain, but are no longer controlled by the deleted ReplicationController. A new RC can adopt matching orphan Pods.

### Can an already-running Pod join a new ReplicationController?

Yes. A ReplicationController can adopt an eligible orphan Pod when:

1. The Pod and ReplicationController are in the same namespace.
2. The Pod's labels match the new RC's `.spec.selector` (for this manifest, `app: my-app`).
3. The Pod does not have a controlling owner reference to another controller.

Check the running Pod's labels and controller before changing anything:

```shell
kubectl get pods --show-labels
```

Example output from `kubectl get pods --show-labels`:

```text
NAME                 READY   STATUS    RESTARTS   AGE   LABELS
standalone-nginx     1/1     Running   0          5m    app=my-app
```

This example Pod has the required `app=my-app` label. Check its owner separately:

```shell
kubectl describe pod <pod-name>
```

For an orphan Pod, the relevant output is:

```text
Name:           standalone-nginx
Labels:         app=my-app
Status:         Running
Controlled By:  <none>
```

`Controlled By: <none>` means there is no controller currently managing this Pod. To see the full Pod object and check `metadata.ownerReferences`:

```shell
kubectl get pod <pod-name> -o yaml
```

An unowned Pod's YAML excerpt may look like this; it has no `ownerReferences` entry:

```yaml
metadata:
  name: standalone-nginx
  labels:
    app: my-app
```

In the Pod YAML, inspect `metadata.labels` and `metadata.ownerReferences`. If the Pod has no owner reference with `controller: true` and is missing the `app` label, add it:

```shell
kubectl label pod <pod-name> app=my-app
```

Example response:

```text
pod/standalone-nginx labeled
```

This confirms the label was added. The new label makes the Pod match this manifest's selector; it does not change the Pod's container image or other spec fields.

If the Pod already has an `app` label with a different value, check which Services and controllers use that label before changing it. Do not make selectors overlap unintentionally.

Then create the RC from the manifest (or apply it if it is already defined):

```shell
kubectl apply -f replication-controller.yaml
```

If the RC does not exist yet, the response is typically:

```text
replicationcontroller/nginx-rc created
```

If it already exists, the response is typically `configured` or `unchanged`, as described in **1. Create & Apply**.

The controller counts the matching orphan Pod toward `spec.replicas` and creates only the additional Pods needed. For example, if one matching Pod exists and the desired replica count is three, the RC creates two more. Adoption does not rewrite the existing Pod to match the RC's template; the Pod keeps its current image and settings. If the number of matching Pods is higher than `spec.replicas`, the RC may remove excess Pods. Keep selectors unique so two controllers do not compete for the same Pods.

To hand off Pods from an old RC to a replacement, orphan the old Pods first, then apply a replacement whose selector matches them. Replace `old-rc` with the name of the current controller:

```shell
kubectl delete rc old-rc --cascade=orphan
kubectl apply -f replication-controller.yaml
```

Example responses:

```text
replicationcontroller "old-rc" deleted
replicationcontroller/nginx-rc created
```

The first response confirms the old controller was deleted; orphaning keeps its Pods. The second confirms the replacement controller was created. Check a retained Pod after the controller reconciles:

```shell
kubectl describe pod <pod-name>
```

The relevant output should now name the replacement RC:

```text
Controlled By:  ReplicationController/nginx-rc
```

Do not use orphan deletion if you intend to remove the workload and its Pods.

## References

- [ReplicationController concepts](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)
- [kubectl explain](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_explain/)
- [kubectl create](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/)
- [kubectl describe](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/)
- [kubectl scale](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_scale/)
- [kubectl expose](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_expose/)
- [kubectl delete](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_delete/)
- [Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Owners and dependents](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/)
