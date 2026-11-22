
# Kubernetes Services: Create and Manage

A Kubernetes Service provides a stable address for reaching a set of Pods. Pods can be replaced and their IP addresses can change; clients connect to the Service name or IP while Kubernetes updates the backend endpoints.

In the common case, a Service selects Pods by label. Kubernetes tracks matching Pods in EndpointSlices, and the cluster's Service networking implementation routes connections to those endpoints.

## How Service ports work

| Field | Meaning |
| --- | --- |
| `port` | Port clients use on the Service. |
| `targetPort` | Port on the selected Pod that receives the traffic. It can be a number or a named container port. If omitted, it defaults to `port`. |
| `nodePort` | Port opened on each eligible node for a `NodePort` Service. Usually assigned by Kubernetes when omitted. |

For example, a Service can accept traffic on port `80` and send it to Pod port `8080` by setting `port: 80` and `targetPort: 8080`.

`containerPort` in a Pod specification documents a port and lets you refer to it by name. It does not open the port by itself. The application must listen on the target port, and the Service must target that port.

## Service types

| Type | Use | Reachability |
| --- | --- | --- |
| `ClusterIP` | Default type for traffic within the cluster. | Internal Service IP and DNS name. |
| `NodePort` | Expose a Service through a port on cluster nodes. | `NodeIP:NodePort`, subject to cluster networking and firewall rules. Also has a ClusterIP. |
| `LoadBalancer` | Request a load balancer from a cloud provider or installed load-balancer implementation. | External address when the environment supports provisioning one. Also has a ClusterIP. |
| `ExternalName` | Give an external DNS name a name in cluster DNS. | Returns a DNS CNAME; it does not select Pods or proxy traffic. |

Use `ClusterIP` for most app-to-app communication. Choose `NodePort` or `LoadBalancer` when traffic must enter from outside the cluster. A `LoadBalancer` Service needs a provider or controller that can provision a load balancer; on a local cluster, its external address may remain pending.

### Headless Services

Set `clusterIP: None` on a selector-based Service to make it headless. Kubernetes does not allocate a virtual IP; DNS returns the endpoint addresses instead. This is useful when clients, such as a StatefulSet peer, need to discover individual Pods.

## Create a Deployment and Service with YAML

Save the following as `web.yaml`. The Deployment puts the label `app: web` on its Pods, and the Service selects Pods with that same label.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: http
```

Apply the file and check that the objects and endpoints exist:

```bash
kubectl apply -f web.yaml
kubectl get deployment,pods,service
kubectl get endpointslices -l kubernetes.io/service-name=web
```

Here, `port: 80` is the Service port and `targetPort: http` resolves to the Pod's named port `http` (container port 80). Using a named target port can make the Service easier to maintain if the numeric container port changes later.

For a Service with multiple ports, give each port a unique `name`.

## Create a Service imperatively

`kubectl expose` creates a Service using a workload's Pod selector:

```bash
kubectl expose deployment web \
  --name=web \
  --type=ClusterIP \
  --port=80 \
  --target-port=80
```

To expose a Deployment with a NodePort instead:

```bash
kubectl expose deployment web \
  --name=web-nodeport \
  --type=NodePort \
  --port=80 \
  --target-port=80
```

Use declarative YAML for repeatable environments and version control. Avoid applying both examples with the same Service name if you already created that Service in `web.yaml`.

## Connect to a Service

Pods in the same namespace can usually connect using the Service name:

```text
http://web:80
```

From another namespace, use the namespace-qualified name:

```text
http://web.default:80
```

The fully qualified DNS name is `web.default.svc.cluster.local` when the cluster uses the default DNS suffix. The cluster DNS suffix can be configured differently.

For local testing from your workstation, forward a local port to the Service:

```bash
kubectl port-forward service/web 8080:80
```

Then open `http://localhost:8080`. Port forwarding is useful for development and debugging; it does not make the Service publicly reachable.

## Manage Services

```bash
kubectl get services
kubectl get svc -A
kubectl get svc web -o wide
kubectl describe svc web
```

These commands list Services in the current namespace, list them across namespaces, show additional details, and describe a specific Service.

Update a Service by changing its manifest and applying it again:

```bash
kubectl apply -f web.yaml
```

For a one-off change, edit the live object:

```bash
kubectl edit service web
```

Change the Service type with a patch:

```bash
kubectl patch service web -p '{"spec":{"type":"NodePort"}}'
```

Delete the Service without deleting the Deployment or its Pods:

```bash
kubectl delete service web
```

To remove both objects created by the example manifest, delete the manifest's resources:

```bash
kubectl delete -f web.yaml
```

## Troubleshoot a Service

Start with these checks:

```bash
kubectl get svc web -o wide
kubectl describe svc web
kubectl get pods -l app=web -o wide
kubectl get endpointslices -l kubernetes.io/service-name=web
```

Check the following:

1. **The Service exists in the namespace your client is using.** A Service name without a namespace resolves within the client's namespace.
2. **The Service selector matches the Pod labels.** Compare `spec.selector` on the Service with `metadata.labels` on the Pods. A typo or label mismatch leaves the Service without backends.
3. **The Pods are ready.** Check `kubectl get pods` and `kubectl describe pod <pod-name>`. Unready Pods are generally not used as ready Service endpoints.
4. **The target port is correct.** Make sure the app listens on the port named or numbered by `targetPort`; the Service's `port` is the client-facing port.
5. **The application responds directly.** If possible, test the Pod or its IP and port from inside the cluster to separate an application issue from a Service issue.
6. **The network allows the traffic.** Review NetworkPolicies, node firewalls, and cloud load-balancer rules when applicable.

If the EndpointSlice has no endpoints, first check the selector, Pod labels, and Pod readiness. To inspect only the endpoint addresses and ports, run:

```bash
kubectl get endpointslices \
  -l kubernetes.io/service-name=web \
  -o yaml
```

## Further reading

- [Kubernetes Services, Load Balancing, and Networking](https://kubernetes.io/docs/concepts/services-networking/)
- [Service concepts and types](https://kubernetes.io/docs/concepts/services-networking/service/)
- [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [Expose an application with a Service](https://kubernetes.io/docs/tutorials/kubernetes-basics/expose/expose-intro/)
- [Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
