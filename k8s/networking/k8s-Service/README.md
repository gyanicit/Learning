# Kubernetes Services

## Table of Contents

- [1. Service Basics](#1-service-basics)
  - [What a Service does](#what-a-service-does)
  - [Service routing components](#service-routing-components)
  - [Understand Service ports](#understand-service-ports)
  - [Service types](#service-types)
  - [Headless Services](#headless-services)
- [2. Create & Apply](#2-create--apply)
  - [Read the NodePort manifest](#read-the-nodeport-manifest)
  - [Create a Service imperatively](#create-a-service-imperatively)
  - [Create once from the manifest](#create-once-from-the-manifest)
  - [Apply the manifest declaratively](#apply-the-manifest-declaratively)
- [3. Inspect & Monitor](#3-inspect--monitor)
  - [Explain the Service API](#explain-the-service-api)
  - [Inspect the Service and its endpoints](#inspect-the-service-and-its-endpoints)
- [4. Connect & Access](#4-connect--access)
  - [Connect using Service DNS](#connect-using-service-dns)
  - [Access the NodePort](#access-the-nodeport)
  - [Forward a local port for development](#forward-a-local-port-for-development)
  - [Access a LoadBalancer Service](#access-a-loadbalancer-service)
- [5. Edit & Update](#5-edit--update)
- [6. Troubleshoot & Delete](#6-troubleshoot--delete)
  - [Troubleshoot Service traffic](#troubleshoot-service-traffic)
  - [Delete a Service](#delete-a-service)
- [References](#references)

A Kubernetes Service provides a stable network endpoint for a logical set of Pods. Pod IPs can change when Pods are replaced; clients continue using the Service's DNS name or virtual IP while Kubernetes updates the Service endpoints.

> **Service selector:** A selector-based Service targets Pods by their labels. It does not target containers or nodes directly.

## 1. Service Basics

### What a Service does

A Service gives clients a stable way to reach matching Pods. The Service selector chooses Pods, and Kubernetes maintains EndpointSlices containing the matching endpoint addresses and ports. The Service data plane forwards connections to ready endpoints.

```text
Client
  |
  v
Service DNS name or virtual IP : port
  |
  v
Service data plane
  |
  v
ready Pod IP : targetPort
```

The endpoint list can change as Pods become ready, stop being ready, or are replaced. Clients do not need to track each Pod IP themselves.

### Service routing components

| Component | Responsibility |
| --- | --- |
| Service API object | Stores the type, selector, ports, and desired network behavior for the Service. |
| EndpointSlice controller | Watches Services and Pods, then publishes matching backend addresses and ports in EndpointSlices. |
| CoreDNS | Resolves Service DNS names for clients in the cluster. |
| `kube-proxy` or replacement data plane | Programs or provides node networking rules that forward Service traffic to endpoints. Some CNI implementations provide this function instead of `kube-proxy`. |

`kubectl` creates and inspects the API objects; it does not proxy application traffic itself. Service networking components observe the API and update their data plane accordingly.

### Understand Service ports

| Field | Meaning |
| --- | --- |
| `port` | Port clients use on the Service. |
| `targetPort` | Port on each selected Pod where the application listens. It may be a number or a named container port. If omitted, it defaults to `port`. |
| `nodePort` | Port opened on eligible node IPs for a `NodePort` Service. Kubernetes can assign it when the field is omitted. |
| `protocol` | Protocol for the Service port, commonly `TCP`. |

For example, `port: 80` and `targetPort: 8080` means clients connect to Service port 80 and Kubernetes forwards the connection to port 8080 on a selected Pod.

Declaring `containerPort` in a Pod specification does not expose that port. The application must listen on the port named or numbered by `targetPort`.

### Service types

| Type | What it provides | Typical reachability |
| --- | --- | --- |
| `ClusterIP` | A stable virtual IP and DNS name inside the cluster. This is the default type. | Cluster-internal clients. |
| `NodePort` | A Service port on node IPs. It also has a ClusterIP. | `NodeIP:NodePort`, if the client can reach the node and network rules allow the traffic. |
| `LoadBalancer` | Requests an external load balancer from a cloud provider or installed implementation. It also has a ClusterIP and usually a NodePort. | External address when the cluster has a load-balancer implementation. |
| `ExternalName` | Maps the Service DNS name to an external DNS name using a CNAME record. | DNS alias only; it does not select Pods or proxy traffic. |

Use `ClusterIP` for most in-cluster communication. Use `NodePort` for access through node addresses, often for learning or local setups. `LoadBalancer` needs a provider or controller that can provision an external address; a local cluster may leave `EXTERNAL-IP` pending.

### Headless Services

Set `clusterIP: None` on a selector-based Service to make it headless. Kubernetes does not allocate a virtual IP; DNS returns the endpoint addresses instead. This is useful when clients need to discover individual Pods, for example, members of a StatefulSet. A headless Service does not provide the normal virtual-IP forwarding behavior.

## 2. Create & Apply

The repository's [NodePort manifest](nodeport-service.yaml) creates `my-nodeport-service`. It selects Pods labeled `app: my-app` and forwards to port 80:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
```

The selector must match the labels on the Pods that should receive traffic. This manifest creates only a Service; it does not create Pods.

### Read the NodePort manifest

- `type: NodePort` exposes the Service through a node port and also assigns a ClusterIP.
- `selector: app: my-app` identifies the target Pods by label.
- `port: 80` is the client-facing Service port.
- `targetPort: 80` is the destination port on each selected Pod.
- `nodePort: 30080` fixes the external node port. The default Kubernetes NodePort range is `30000-32767`; clusters can configure a different range.

The same Service may be declared as `ClusterIP` by changing its type and omitting `nodePort`. A Service with multiple ports should give each port a unique name.

### Create a Service imperatively

If a Pod named `nginx` already exists and listens on port 80, create a ClusterIP Service for it with `kubectl expose`:

```shell
kubectl expose pod nginx --name=nginx-service --port=80 --target-port=80 --type=ClusterIP
```

Example output:

```text
service/nginx-service exposed
```

The command creates a Service using labels from the selected Pod. It does not create or change the Pod. Use the NodePort manifest below when you specifically need a node port.

### Create once from the manifest

Run from the repository root to create the NodePort Service once:

```shell
kubectl create -f k8s/networking/K8s-Service/NodePort/nodeport-service.yaml
```

Example output:

```text
service/my-nodeport-service created
```

If the Service already exists, `kubectl create` returns an `AlreadyExists` error and does not update it. Use `kubectl apply` for subsequent declarative changes.

### Apply the manifest declaratively

Create or update the Service from the manifest:

```shell
kubectl apply -f k8s/networking/K8s-Service/NodePort/nodeport-service.yaml
```

On first apply, kubectl typically prints:

```text
service/my-nodeport-service created
```

Later applies typically print `configured` if the object changed or `unchanged` if it did not. These messages confirm the Service API object operation; check its endpoints and test a request to verify traffic.

## 3. Inspect & Monitor

### Explain the Service API

`svc` is the short name for `service`. Use `kubectl explain` to inspect the API schema:

```shell
kubectl explain service
kubectl explain service.spec --recursive
kubectl explain service.spec.ports
kubectl explain service.spec.selector
```

Representative output for `kubectl explain service`:

```text
KIND:       Service
VERSION:    v1

DESCRIPTION:
    Service is a named abstraction of software service consisting of local ports that proxy to pods.

FIELDS:
  apiVersion    <string>
  kind          <string>
  metadata      <Object>
  spec          <Object>
  status        <Object>
```

The recursive spec output includes fields such as `clusterIP`, `ports`, `selector`, `type`, and `externalTrafficPolicy`. Exact descriptions can vary by Kubernetes version.

### Inspect the Service and its endpoints

Use these commands to check the Service object and the Pods that can receive its traffic:

| Task | Command | What it shows |
| --- | --- | --- |
| List Services in the current namespace | `kubectl get services` | Service names, types, virtual IPs, ports, and age. |
| Show one Service in detail | `kubectl get service my-nodeport-service -o wide` | Additional Service columns. |
| Read the live object | `kubectl get service my-nodeport-service -o yaml` | Full Service spec and status. |
| Describe the Service | `kubectl describe service my-nodeport-service` | Selector, ports, endpoints, traffic policy, and events. |
| List matching Pods | `kubectl get pods -l app=my-app -o wide --show-labels` | Pod readiness, IP, node, and labels. |
| List EndpointSlices | `kubectl get endpointslices -l kubernetes.io/service-name=my-nodeport-service` | Backend addresses and ports published for this Service. |

Typical output for `kubectl get service my-nodeport-service`:

```text
NAME                  TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
my-nodeport-service   NodePort   10.96.116.70   <none>        80:30080/TCP   12m
```

`80:30080/TCP` means Service port 80 is exposed as NodePort 30080. `<none>` is expected in `EXTERNAL-IP` because this is not a `LoadBalancer` Service.

Relevant `kubectl describe service my-nodeport-service` output:

```text
Name:                     my-nodeport-service
Namespace:                default
Selector:                 app=my-app
Type:                     NodePort
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  30080/TCP
Endpoints:                10.244.1.23:80,10.244.2.25:80,10.244.2.26:80
Session Affinity:         None
External Traffic Policy:  Cluster
```

The `Selector` must match Pod labels. The `Endpoints` row should list ready matching Pods with their target ports. If it shows `<none>`, the Service currently has no ready backends; check the selector and Pod readiness.

EndpointSlices are the API objects that hold the current endpoint addresses. To inspect their contents:

```shell
kubectl get endpointslices -l kubernetes.io/service-name=my-nodeport-service -o yaml
```

## 4. Connect & Access

### Connect using Service DNS

Clients in the same namespace can use the Service name and port:

```text
http://my-nodeport-service:80
```

Clients in a different namespace can qualify the Service name:

```text
http://my-nodeport-service.default:80
```

The fully qualified DNS name is `my-nodeport-service.default.svc.cluster.local` when the cluster uses the default DNS suffix. The suffix can be configured differently. This DNS name and the Service ClusterIP are for cluster networking; neither is automatically reachable from an external workstation.

### Access the NodePort

Use a reachable node IP and the Service's NodePort, for example:

```text
http://172.22.0.2:30080/
```

NodePort access depends on the node IP being reachable from the client and on cluster, firewall, and host networking rules. NodePort exposes the Service, not one fixed Pod; the Service data plane forwards each connection to a ready matching endpoint.

For the WSL and Kind example, node addresses, request commands, outputs, traffic flow, and troubleshooting are documented in [Access a Pod through a NodePort Service](access-nodeport.md).

### Forward a local port for development

For local testing, forward a port on the workstation to the Service:

```shell
kubectl port-forward service/my-nodeport-service 8080:80
```

Typical output:

```text
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

Then open `http://localhost:8080/`. The local port is 8080; the remote Service port is 80. `kubectl` selects one matching Pod for the forwarding session, so this is useful for debugging but does not load-balance requests across all Service endpoints. Keep the command running while using the local address; press `Ctrl+C` to stop it.

### Access a LoadBalancer Service

A `LoadBalancer` Service can receive an external address when the cluster has a cloud-provider integration or a local load-balancer implementation. Check its status with:

```shell
kubectl get service <service-name> -o wide
```

If `EXTERNAL-IP` remains `<pending>`, the cluster may not have a controller that can provision load balancers. In that case, use a supported local access method, such as NodePort or port-forwarding.

## 5. Edit & Update

Edit the live Service in your configured editor:

```shell
kubectl edit service my-nodeport-service
```

After saving a valid change, kubectl typically prints:

```text
service/my-nodeport-service edited
```

For a repeatable update, edit `nodeport-service.yaml` and apply it from the repository root:

```shell
kubectl apply -f k8s/networking/K8s-Service/NodePort/nodeport-service.yaml
```

The Service selector controls which Pods receive traffic. Changing the selector does not change Pod labels; verify that matching Pods exist before applying a selector change. Some allocated fields, such as `clusterIP`, are assigned by Kubernetes and cannot be changed like ordinary labels or ports.

## 6. Troubleshoot & Delete

### Troubleshoot Service traffic

Start with these checks:

```shell
kubectl get service my-nodeport-service -o wide
kubectl describe service my-nodeport-service
kubectl get pods -l app=my-app -o wide --show-labels
kubectl get endpointslices -l kubernetes.io/service-name=my-nodeport-service
```

Check the following:

1. **The Service is in the expected namespace.** A name-only Service lookup uses the current context namespace.
2. **The selector matches Pod labels.** Compare the Service's `.spec.selector` with `metadata.labels` on the Pods.
3. **The Pods are ready.** Check `kubectl get pods` and `kubectl describe pod <pod-name>`; unready Pods are generally not used as ready Service endpoints.
4. **The target port is correct.** Confirm the application listens on `targetPort` and the Service's `port` is the port used by clients.
5. **The access path matches the Service type.** Use Service DNS or ClusterIP inside the cluster; use `NodeIP:NodePort` for NodePort access; use an external address only when a load balancer is provisioned.
6. **The network permits the connection.** Review NetworkPolicies, node firewalls, and local-cluster host networking when applicable.

If the EndpointSlice has no endpoints, first check the selector, Pod labels, and Pod readiness. If a request works through `kubectl port-forward` but not through NodePort, check node IP reachability and firewall or local-cluster port mappings. See the [NodePort access guide](access-nodeport.md) for the WSL/Kind checks.

### Delete a Service

Delete only the Service object by name:

```shell
kubectl delete service my-nodeport-service
```

Example output:

```text
service "my-nodeport-service" deleted
```

Deleting a Service removes its virtual IP and Service routing configuration. It does **not** delete the Pods selected by that Service.

To delete the Service from its manifest, run from the repository root:

```shell
kubectl delete -f k8s/networking/K8s-Service/NodePort/nodeport-service.yaml
```

## References

- [Kubernetes Service concepts and types](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Virtual IPs and Service proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/)
- [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [kubectl expose](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_expose/)
- [kubectl port-forward](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_port-forward/)
- [kubectl create](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/)
- [kubectl apply](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
- [kubectl edit](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_edit/)
- [kubectl delete](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_delete/)
- [NodePort access guide](access-nodeport.md)
