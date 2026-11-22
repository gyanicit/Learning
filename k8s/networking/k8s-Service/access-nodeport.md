# Access a Pod through a NodePort Service

## Table of Contents

- [Purpose and request path](#purpose-and-request-path)
- [Understand the ports](#understand-the-ports)
- [Check the nodes, Service, and Pods](#check-the-nodes-service-and-pods)
- [Send a request to the NodePort](#send-a-request-to-the-nodeport)
- [Use kubectl port-forward for local access](#use-kubectl-port-forward-for-local-access)
- [Troubleshoot access](#troubleshoot-access)
- [References](#references)

This guide uses [nodeport-service.yaml](nodeport-service.yaml). It creates the `my-nodeport-service` Service, which selects Pods labeled `app=my-app` and sends traffic to port `80` on those Pods. The Service and Pods are in the namespace selected by the current kubectl context; the example cluster output below shows `default`.

> **Important:** A NodePort exposes the Service on node IP addresses. It does not open that port on a Pod IP. You connect to `NodeIP:NodePort`; Kubernetes then forwards the connection to a matching Pod's `targetPort`.

## Purpose and request path

The same NodePort is available on each node. Each node provides an entry point to the same Service and its ready endpoints:

```text
WSL client
    |
    +--> 172.22.0.2:30080  (worker2)
    +--> 172.22.0.3:30080  (worker)
    +--> 172.22.0.4:30080  (control-plane)
                 |
                 v
       NodePort Service data plane
       my-nodeport-service:80
                 |
                 | selects one ready endpoint for the connection
       +---------+----------+
       |                    |
       v                    v
  PodIP:80              PodIP:80 ...
  nginx container      nginx container
```

In this cluster, the Service reports `External Traffic Policy: Cluster`, so a request arriving at any node can be sent to a ready endpoint on any node. The node IPs are separate entry points, not one shared external IP. The Service data plane (commonly `kube-proxy`, or a cluster networking replacement) selects an endpoint and forwards the connection. A NodePort alone does not create one unified address or a public load balancer.

## Understand the ports

| Address or field | Value in this example | What it means |
| --- | --- | --- |
| Node IP | `172.22.0.2`, `172.22.0.3`, or `172.22.0.4` | Address of a Kubernetes node that is reachable from the client. |
| `nodePort` | `30080` | Port used by a client outside the cluster to enter through a node. |
| Service `port` | `80` | Port on the Service, used by clients inside the cluster and by `kubectl port-forward`. |
| Service `targetPort` | `80` | Port on a selected Pod where the application listens. |
| Pod IP and port | For example, `10.244.2.25:80` | One backend endpoint. Pod IPs are assigned to individual Pods and can change when Pods are replaced. |
| Service `clusterIP` | `10.96.116.70` | Stable virtual IP for this Service inside the cluster. It is not the NodePort address. |

The port mapping is:

```text
Client outside cluster:  NodeIP:30080
Client inside cluster:   my-nodeport-service:80
Service to Pod:          PodIP:80
```

The manifest's `containerPort` is not what makes the application reachable. Nginx must listen on port `80`, and the Service's `targetPort: 80` must point to that listening port.

The address `10.244.2.25` from the example is a Pod IP. The Service lists it as an endpoint at port `80`, so `http://10.244.2.25:30080/` uses the wrong port and bypasses the NodePort entry point. Use a node IP with port `30080`, such as `http://172.22.0.2:30080/`.

## Check the nodes, Service, and Pods

Run these commands from the WSL shell configured for this cluster.

### Find reachable node IPs

```shell
kubectl get nodes -o wide
```

Representative output from the cluster used for this example:

```text
NAME                             STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP
mcu-multicluster-control-plane   Ready    control-plane   8d    v1.37.0   172.22.0.4    <none>
mcu-multicluster-worker          Ready    <none>          8d    v1.37.0   172.22.0.3    <none>
mcu-multicluster-worker2         Ready    <none>          8d    v1.37.0   172.22.0.2    <none>
```

Use an `INTERNAL-IP` only if it is reachable from the client. These addresses can change if the local cluster is recreated.

### Verify the Service type and NodePort

```shell
kubectl get service my-nodeport-service
```

Output:

```text
NAME                  TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
my-nodeport-service   NodePort   10.96.116.70   <none>        80:30080/TCP   12m
```

`80:30080/TCP` means clients use Service port `80` inside the cluster and NodePort `30080` on a node. `<none>` under `EXTERNAL-IP` is expected for a NodePort Service; NodePort does not allocate an external load-balancer IP.

### Verify selector and endpoints

```shell
kubectl describe service my-nodeport-service
```

Relevant output from this example:

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

The selector must match the Pod labels. The listed endpoints show that the Service currently has three backends listening on port `80`. If `Endpoints` is `<none>`, check the selector and whether matching Pods are ready.

Check the matching Pods and their node placement:

```shell
kubectl get pods -l app=my-app -o wide --show-labels
```

Representative output:

```text
NAME                                READY   STATUS    RESTARTS   AGE   IP            NODE                       LABELS
nginx-deployment-7bf97c9564-bws6q   1/1     Running   0          92m   10.244.2.25   mcu-multicluster-worker    app=my-app,env=dev,type=frontend
nginx-deployment-7bf97c9564-ssqr8   1/1     Running   0          92m   10.244.1.23   mcu-multicluster-worker2   app=my-app,env=dev,type=frontend
nginx-deployment-7bf97c9564-zzwfr   1/1     Running   0          92m   10.244.2.26   mcu-multicluster-worker    app=my-app,env=dev,type=frontend
```

`READY 1/1` and `STATUS Running` indicate that each example Pod is running and ready. The Pod labels include `app=my-app`, so they match the Service selector.

## Send a request to the NodePort

From WSL, send an HTTP request to any reachable node IP on port `30080`:

```shell
curl -i http://172.22.0.2:30080/
curl -i http://172.22.0.3:30080/
curl -i http://172.22.0.4:30080/
```

Each request enters through a different node, but all three node ports lead to the same Service. The Service data plane forwards each new connection to a ready Pod endpoint. It does not promise an even round-robin distribution, and repeated HTTP requests on a reused connection can continue to the same Pod. `Session Affinity: None` means the Service is not configured to pin a client IP to one Pod.

With the default nginx image, a successful request returns an HTTP response similar to:

```text
HTTP/1.1 200 OK
Server: nginx/<image-version>
Content-Type: text/html
...
<title>Welcome to nginx!</title>
```

The exact headers and image version can differ. The key check is an HTTP `200 OK` response with the nginx welcome page.

To observe which Pods receive requests, follow their container logs in one terminal, then make requests from another:

```shell
kubectl logs -f -l app=my-app -c nginx --prefix=true
```

The prefix identifies the Pod and container for each log line. The nginx image writes access logs to the container's standard output when requests arrive.

## Use kubectl port-forward for local access

Port forwarding provides a convenient local URL when you do not want to route directly to a node IP:

```shell
kubectl port-forward service/my-nodeport-service 8080:80
```

Typical output:

```text
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

Open `http://localhost:8080/` or test it from WSL:

```shell
curl -i http://localhost:8080/
```

Here, `8080` is the local port and `80` is the Service port. `kubectl` opens a local listener and tunnels traffic to one Pod selected for this port-forward session. This is for local development and debugging; it is not a load balancer and does not distribute each request across all three Pods. Keep the port-forward command running while using the local URL; press `Ctrl+C` to stop it.

To choose a specific Pod instead of letting kubectl select one, first list the Pods and then forward directly to one of them:

```shell
kubectl get pods -l app=my-app
kubectl port-forward pod/<pod-name> 8080:80
```

With the Pod name specified, that Pod receives traffic for the forwarding session. If it terminates, stop and restart the port-forward using a current Pod name.

## Troubleshoot access

| Symptom | Check |
| --- | --- |
| Timeout using `10.244.x.x:30080` | `10.244.x.x` is a Pod IP, and NodePort `30080` is exposed on node IPs. Use a reachable node IP with port `30080`. Pod IPs are temporary and are mainly useful for in-cluster diagnosis. |
| NodePort works from WSL but not a Windows browser | The node IPs may be reachable only on the WSL/container network. Use `kubectl port-forward` and open `http://localhost:8080/`, or configure a host port mapping for the local cluster. |
| Connection refused or timeout on all node IPs | Confirm the node IP is reachable, the node firewall permits TCP `30080`, and the Service still reports `80:30080/TCP`. |
| Service shows `Endpoints: <none>` | Compare the Service selector with Pod labels and check that matching Pods are ready. |
| Request reaches nginx but returns an error | Check the application and Pod logs; confirm nginx listens on port `80` and the Service `targetPort` is `80`. |

For this cluster, the successful tests were `curl -i http://172.22.0.2:30080/` and `curl -i http://localhost:8080/` while the `kubectl port-forward` process was running. The first used the NodePort path; the second used a local tunnel.

## References

- [Kubernetes Service concepts and NodePort](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Virtual IPs and Service proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/)
- [kubectl port-forward reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_port-forward/)
- [Kind cluster configuration and port mappings](https://kind.sigs.k8s.io/docs/user/configuration/)
