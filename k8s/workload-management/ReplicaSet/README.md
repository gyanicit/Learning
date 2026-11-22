# Kubernetes ReplicaSet

> **Use a Deployment for most applications.** A Deployment manages ReplicaSets and provides controlled rollouts and rollbacks. A ReplicaSet keeps a specified number of matching Pods running, but does not roll out changes to existing Pods by itself. ReplicaSet is the modern successor to the legacy ReplicationController and supports set-based selectors. This guide is useful for understanding ReplicaSets or working with one directly.

The example manifest is [replica-set.yaml](replica-set.yaml). It defines `nginx-rs` with three desired nginx Pod replicas and selects Pods labeled `app: my-app`.

> **About the outputs below:** These are representative examples. Names, ages, IP addresses, event counts, and status can differ by cluster and timing; output excerpts omit unrelated fields.

## 1. Create & Apply

### How a ReplicaSet works

A ReplicaSet controller continuously compares the number of Pods matching its selector with `.spec.replicas`:

- If there are too few matching Pods, it creates Pods from `.spec.template`.
- If there are too many, it deletes excess Pods.
- If a managed Pod is deleted or lost, it creates a replacement to move toward the desired count.

The ReplicaSet controller manages Pods, not containers directly. The kubelet and container runtime on each selected node start and monitor the containers inside each Pod.

A ReplicaSet is a lower-level workload controller. Deployments normally create and manage ReplicaSets for versioned rollouts. Avoid manually managing a ReplicaSet owned by a Deployment.

### Important fields in this manifest

| Field | Meaning |
| --- | --- |
| `apiVersion: apps/v1` and `kind: ReplicaSet` | Select the `apps/v1` ReplicaSet resource. |
| `metadata.name: nginx-rs` | Names the ReplicaSet. |
| `metadata.labels` | Labels the ReplicaSet object itself. These do not select the Pods it manages. |
| `spec.replicas: 3` | Desired number of matching Pods. If omitted, it defaults to 1. The observed count can temporarily differ while the controller reconciles. |
| `spec.minReadySeconds` | Optional time a newly ready Pod must remain ready before it is treated as available. It is not set in this example. |
| `spec.selector.matchLabels` | Selects Pods with `app: my-app`. `matchLabels` requirements are combined with logical AND. |
| `spec.selector.matchExpressions` | Optional set-based requirements. Supported operators are `In`, `NotIn`, `Exists`, and `DoesNotExist`; if used with `matchLabels`, all requirements must match. |
| `spec.template` | Pod template used when the ReplicaSet needs to create a Pod. It contains Pod metadata and a Pod spec. |
| `spec.template.metadata.name: nginx` | Name metadata in the template. The controller gives each created Pod a generated name based on the ReplicaSet, such as `nginx-rs-xxxxx`; do not rely on every Pod being named `nginx`. |
| `spec.template.metadata.labels` | Labels applied to created Pods. They must satisfy the ReplicaSet selector. This manifest adds `env: dev` and `type: frontend` in addition to `app: my-app`. |
| `spec.template.spec.containers` | Container configuration for each new Pod; this example runs one container named `nginx` with image `nginx`. |
| `spec.template.spec.restartPolicy` | This template omits it, so the Pod default is `Always`, the only restart policy accepted for ReplicaSet Pods. |

The selector is required for a ReplicaSet in `apps/v1`. It must match the Pod template labels. ReplicaSets support both exact-match labels and set-based expressions, unlike legacy ReplicationControllers, which only support equality-based selectors. Do not reuse a selector that overlaps with another controller's Pods: controllers with overlapping selectors can interfere with one another.

### Imperative creation

The current `kubectl` CLI does not provide a direct, manifest-free generator such as `kubectl create replicaset ...`. `kubectl run` creates a Pod; it does not create a ReplicaSet. A manifest is needed to define a ReplicaSet.

### Create once from the manifest

Use `kubectl create -f` for a one-time create from the manifest:

```shell
kubectl create -f replica-set.yaml
```

Example output:

```text
replicaset.apps/nginx-rs created
```

If the ReplicaSet already exists, `kubectl create` reports `AlreadyExists` and does not update it.

Example error:

```text
Error from server (AlreadyExists): error when creating "replica-set.yaml": replicasets.apps "nginx-rs" already exists
```

Use `kubectl apply -f` when you need to create or update from the manifest.

### Apply the manifest declaratively

Use `kubectl apply -f` to create the ReplicaSet if it does not exist, or apply later changes from the file:

```shell
kubectl apply -f replica-set.yaml
```

Example output on first apply:

```text
replicaset.apps/nginx-rs created
```

On later applies, kubectl may print `configured` when it updates the object or `unchanged` when the manifest has no changes. These messages describe the ReplicaSet object operation; they do not mean that all Pods are ready.

## 2. Inspect & Monitor

### Explain the API fields

`kubectl explain` reads the API schema and field descriptions. The `rs` short name and full resource name both identify this resource:

```shell
kubectl explain rs
kubectl explain replicaset
kubectl explain rs.spec --recursive
kubectl explain rs.spec.selector
kubectl explain rs.spec.template
```

Representative output for `kubectl explain rs` (the full resource name produces the same schema):

```text
KIND:       ReplicaSet
VERSION:    apps/v1

DESCRIPTION:
    ReplicaSet ensures that a specified number of pod replicas are running at any given time.

FIELDS:
  apiVersion    <string>
  kind          <string>
  metadata      <Object>
  spec          <Object>
  status        <Object>
```

`kubectl explain rs.spec --recursive` shows the nested specification fields:

```text
RESOURCE: spec <Object>

FIELDS:
  minReadySeconds <integer>
  replicas        <integer>
  selector        <Object>
  template        <Object>
```

`kubectl explain rs.spec.selector` describes the selector fields, including:

```text
RESOURCE: selector <Object>

FIELDS:
  matchExpressions <[]Object>
  matchLabels      <map[string]string>
```

`kubectl explain rs.spec.template` describes the nested Pod template, including its `metadata` and `spec`. Descriptions and some fields depend on the Kubernetes API version.

Representative output excerpt:

```text
RESOURCE: template <Object>

FIELDS:
  metadata <Object>
  spec     <Object>
```

### Common inspection commands

`rs` is the short name for `replicaset`. These commands use the namespace selected by your current kubeconfig context.

| Task | Command | What it shows |
| --- | --- | --- |
| List ReplicaSets | `kubectl get rs` | Desired, current, and ready replica counts. |
| Show one ReplicaSet | `kubectl get rs nginx-rs -o wide` | The ReplicaSet and additional columns. |
| Inspect the full ReplicaSet object | `kubectl get rs nginx-rs -o yaml` | Live specification, generated metadata, and status. |
| Describe one ReplicaSet | `kubectl describe rs nginx-rs` | Selector, Pod template, counts, and events. |
| List Pods selected by this manifest | `kubectl get pods -l app=my-app --show-labels` | Matching Pod names and labels. |

Typical `kubectl get rs` output:

```text
NAME      DESIRED   CURRENT   READY   AGE
nginx-rs  3         3         3       2m
```

- `DESIRED` is `.spec.replicas`.
- `CURRENT` is the number of Pods matching the selector.
- `READY` is the number of matching Pods that are ready.

The counts can differ while Pods are being created, scheduled, or made ready.

Typical output for `kubectl get rs nginx-rs -o wide`:

```text
NAME      DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES   SELECTOR
nginx-rs  3         3         3       2m    nginx        nginx    app=my-app
```

`SELECTOR` shows the label query used by this ReplicaSet.

`kubectl get rs nginx-rs -o yaml` returns the full object. This excerpt highlights the fields to compare with the manifest:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
        env: dev
        type: frontend
status:
  replicas: 3
  readyReplicas: 3
```

`kubectl describe rs nginx-rs` includes a human-readable summary similar to:

```text
Name:               nginx-rs
Namespace:          <current-namespace>
Selector:           app=my-app
Labels:             app=my-app, env=dev, type=frontend
Replicas:           3 current / 3 desired
Pods Status:        3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:           app=my-app, env=dev, type=frontend
  Containers:
   nginx:
    Image:          nginx
Events:
  Type    Reason             Message
  Normal  SuccessfulCreate   Created pod: nginx-rs-xxxxx
```

Events can be empty after the ReplicaSet has been stable for a while. The namespace shown is the one selected by the current kubeconfig context.

Typical output for `kubectl get pods -l app=my-app --show-labels`:

```text
NAME                 READY   STATUS    RESTARTS   AGE   LABELS
nginx-rs-xxxxx       1/1     Running   0          2m    app=my-app,env=dev,type=frontend
```

This command lists Pods matching `app=my-app`. The ReplicaSet-generated Pod names usually begin with `nginx-rs-` followed by a generated suffix.

### Check which ReplicaSet owns a Pod

Describe a Pod to see its `Controlled By` field and recent events:

```shell
# Replace <pod-name> with a name returned by kubectl get pods.
kubectl describe pod <pod-name>
```

For a Pod managed by this example, the relevant output looks like:

```text
Name:           nginx-rs-xxxxx
Namespace:      <current-namespace>
Labels:         app=my-app,env=dev,type=frontend
Status:         Running
IP:             10.244.0.12
Controlled By:  ReplicaSet/nginx-rs
```

`Controlled By` identifies the owner managing this Pod. To inspect the full Pod object and its owner references:

```shell
kubectl get pod <pod-name> -o yaml
```

A relevant excerpt is:

```yaml
metadata:
  name: nginx-rs-xxxxx
  labels:
    app: my-app
    env: dev
    type: frontend
  ownerReferences:
    - apiVersion: apps/v1
      kind: ReplicaSet
      name: nginx-rs
      uid: 12345678-1234-1234-1234-123456789abc
      controller: true
```

The `kind` and `name` identify the owner; `controller: true` marks it as the controlling owner.

## 3. Scale Replicas

Scale `nginx-rs` to five desired Pods:

```shell
kubectl scale rs nginx-rs --replicas=5
```

Example response:

```text
replicaset.apps/nginx-rs scaled
```

This confirms the desired count changed. It does not wait for all five Pods to become ready. Use `kubectl get rs` to monitor `DESIRED`, `CURRENT`, and `READY`.

To scale it down to one Pod:

```shell
kubectl scale rs nginx-rs --replicas=1
```

The response is also `replicaset.apps/nginx-rs scaled`. The controller terminates excess Pods until it moves toward the new desired count.

## 4. Edit & Update

Edit the live ReplicaSet in your configured editor:

```shell
kubectl edit rs nginx-rs
```

After you save a change, kubectl typically prints:

```text
replicaset.apps/nginx-rs edited
```

If you exit without saving, kubectl reports that the edit was cancelled or that no changes were made.

To update the ReplicaSet from its manifest after changing `replica-set.yaml`, apply the file again:

```shell
kubectl apply -f replica-set.yaml
```

When the manifest changed, a typical response is:

```text
replicaset.apps/nginx-rs configured
```

`configured` means the ReplicaSet object changed; it does not mean the existing Pods were replaced or that new Pods are ready. A ReplicaSet does not perform a rolling update when its Pod template changes. Existing Pods remain as they are; Pods created later use the updated template. Use a Deployment when you need controlled updates and rollbacks. Do not manually edit or scale a ReplicaSet managed by a Deployment.

## 5. Expose via a Service

A Service provides a stable network endpoint and routes traffic to matching Pods. It selects Pods by their labels; it does not route to the ReplicaSet object itself. This manifest's Pods use the label `app: my-app`.

Create an internal `ClusterIP` Service for the nginx Pods:

```shell
kubectl expose rs nginx-rs --name=nginx-service --port=80 --target-port=80 --type=ClusterIP
```

Example response:

```text
service/nginx-service exposed
```

This means the Service object was created. It does not by itself confirm that matching ready Pods exist.

`kubectl expose` uses the ReplicaSet's selector for the Service selector. The Service's `port` is the port clients use; `target-port` is the port on the nginx containers. `ClusterIP` is reachable from inside the cluster. Use an appropriate type such as `NodePort` or `LoadBalancer` when external access is intended and supported by the cluster.

Inspect the Service and its selected endpoints:

```shell
kubectl get service nginx-service
kubectl describe service nginx-service
```

Typical output for `kubectl get service nginx-service`:

```text
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-service   ClusterIP   10.96.120.35   <none>        80/TCP    10s
```

`CLUSTER-IP` is allocated by the cluster. For a `ClusterIP` Service it is reachable only from within the cluster.

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

The `Selector` should match the Pod labels, and `Endpoints` should contain addresses of ready matching Pods. If `Endpoints` shows `<none>`, check that Pods exist, are ready, and carry `app=my-app`.

## 6. Delete & Orphan

### Delete the ReplicaSet and its Pods

By default, deleting a ReplicaSet also deletes the Pods it owns:

```shell
kubectl delete rs nginx-rs
```

Example response:

```text
replicaset.apps "nginx-rs" deleted
```

Check for any remaining Pods selected by this manifest:

```shell
kubectl get pods -l app=my-app --show-labels
```

If none remain, kubectl typically prints:

```text
No resources found in <current-namespace> namespace.
```

The ReplicaSet and its dependent Pods are deleted. Another controller with a matching selector could still create or manage matching Pods.

### Delete the ReplicaSet but keep its Pods

Use orphan deletion when you need to remove or replace a standalone ReplicaSet while leaving its Pods running:

```shell
kubectl delete rs nginx-rs --cascade=orphan
```

Example response:

```text
replicaset.apps "nginx-rs" deleted
```

The response reports deletion of the ReplicaSet object. With `--cascade=orphan`, its Pods are left running and their controlling owner reference is removed.

> **Deployment-managed ReplicaSet:** Do not delete or orphan a ReplicaSet owned by a Deployment to perform an update. The Deployment manages that ReplicaSet and may recreate or change it. Update the Deployment instead.

### Can an already-running Pod join a new ReplicaSet?

Yes. A ReplicaSet can acquire an eligible orphan Pod when:

1. The Pod and ReplicaSet are in the same namespace.
2. The Pod's labels match the ReplicaSet's `.spec.selector` (for this manifest, `app: my-app`).
3. The Pod does not have a controlling owner reference to another controller.

A Pod with no owner reference, or with an owner reference that is not marked `controller: true`, can be acquired if its labels match. A Pod controlled by another ReplicaSet or Deployment-managed ReplicaSet will not be acquired by this standalone ReplicaSet.

Check a running Pod's labels and owner before changing anything:

```shell
kubectl get pods --show-labels
```

Example output:

```text
NAME                 READY   STATUS    RESTARTS   AGE   LABELS
standalone-nginx     1/1     Running   0          5m    app=my-app
```

This Pod has the selector label. Inspect its controlling owner:

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

`Controlled By: <none>` means there is no controller currently managing this Pod. To inspect the Pod's labels and `metadata.ownerReferences` directly:

```shell
kubectl get pod <pod-name> -o yaml
```

A standalone Pod may have YAML like this, with no controlling owner reference:

```yaml
metadata:
  name: standalone-nginx
  labels:
    app: my-app
```

If the Pod is an orphan and does not yet have the `app` label, add the label so it matches this ReplicaSet:

```shell
kubectl label pod <pod-name> app=my-app
```

Example response:

```text
pod/standalone-nginx labeled
```

This adds a label; it does not change the Pod's image or other spec fields. If the Pod already has `app` set to a different value, check which Services or controllers use that label before changing it. Do not create overlapping selectors accidentally.

Apply the ReplicaSet manifest:

```shell
kubectl apply -f replica-set.yaml
```

If the ReplicaSet does not exist, the response is typically:

```text
replicaset.apps/nginx-rs created
```

If it already exists, the response is typically `configured` or `unchanged`. The controller counts matching orphan Pods toward `.spec.replicas` and creates only the additional Pods needed. For example, one matching Pod with `replicas: 3` means the ReplicaSet creates two more.

Adoption does not rewrite an existing Pod to match the template. The Pod keeps its current image and settings. If there are more matching Pods than the desired replica count, the ReplicaSet may remove excess Pods. Keep selectors unique so controllers do not compete for the same Pods.

To hand off Pods from one standalone ReplicaSet to a replacement, orphan the old Pods first, then create a replacement whose selector matches them. Replace `old-rs` with the current ReplicaSet name:

```shell
kubectl delete rs old-rs --cascade=orphan
kubectl apply -f replica-set.yaml
```

Example responses:

```text
replicaset.apps "old-rs" deleted
replicaset.apps/nginx-rs created
```

The old object is removed while its Pods remain. After the replacement controller reconciles, verify a retained Pod:

```shell
kubectl describe pod <pod-name>
```

The relevant output should show the replacement as owner:

```text
Controlled By:  ReplicaSet/nginx-rs
```

Do not orphan a ReplicaSet's Pods if you intend to remove the workload and its Pods.

## References

- [ReplicaSet concepts](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [kubectl explain](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_explain/)
- [kubectl create](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/)
- [kubectl apply](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
- [kubectl get](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
- [kubectl describe](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/)
- [kubectl label](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_label/)
- [kubectl scale](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_scale/)
- [kubectl expose](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_expose/)
- [kubectl delete](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_delete/)
- [Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Owners and dependents](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/)
