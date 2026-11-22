# Kubernetes Ingress

## Table of Contents

- [1. Create & Apply](#1-create--apply)
  - [What an Ingress does](#what-an-ingress-does)
  - [Prerequisites: Minikube](#prerequisites-minikube)
  - [Prerequisites: Kind](#prerequisites-kind)
    - [Approach 1: Kind native Ingress with cloud-provider-kind](#approach-1-kind-native-ingress-with-cloud-provider-kind)
    - [Approach 2: Kind with Ingress NGINX and port mappings](#approach-2-kind-with-ingress-nginx-and-port-mappings)
      - [Create a Kind cluster with port mappings](#create-a-kind-cluster-with-port-mappings)
      - [Install and place the Ingress controller](#install-and-place-the-ingress-controller)
  - [Read the Ingress manifest](#read-the-ingress-manifest)
  - [Apply the Ingress](#apply-the-ingress)
- [2. Inspect & Monitor](#2-inspect--monitor)
  - [Explain the API fields](#explain-the-api-fields)
  - [Inspect the Ingress and class](#inspect-the-ingress-and-class)
  - [Check the backend Service and endpoints](#check-the-backend-service-and-endpoints)
- [3. Configure Host and Path Routing](#3-configure-host-and-path-routing)
  - [Browser and REST-client hostname access](#browser-and-rest-client-hostname-access)
  - [Test from Minikube on WSL](#test-from-minikube-on-wsl)
    - [Open Minikube Ingress from a browser or REST client](#open-minikube-ingress-from-a-browser-or-rest-client)
  - [Test from Kind using cloud-provider-kind](#test-from-kind-using-cloud-provider-kind)
    - [Open native Kind Ingress from a browser or REST client](#open-native-kind-ingress-from-a-browser-or-rest-client)
  - [Test from Kind using Ingress NGINX](#test-from-kind-using-ingress-nginx)
    - [Open Kind Ingress NGINX from a browser or REST client](#open-kind-ingress-nginx-from-a-browser-or-rest-client)
- [4. Edit & Update](#4-edit--update)
- [5. Backend Service & TLS](#5-backend-service--tls)
- [6. Delete & Troubleshoot](#6-delete--troubleshoot)
- [References](#references)

This guide explains the Ingress object in [ingress.yaml](ingress.yaml), how a request travels to a backend Pod, and how to prepare Minikube or Kind before applying an Ingress. Commands use the namespace selected by the current `kubectl` context; the sample manifest does not set `metadata.namespace`.

> **About command output:** Outputs are representative. Generated addresses, names, ages, event text, and controller status vary by cluster and timing.

## 1. Create & Apply

### What an Ingress does

An Ingress is an API object containing HTTP and HTTPS routing rules. It can match a request by host name and path, then direct it to a Service. The Ingress object stores the desired routing configuration; it does not itself receive traffic. A compatible Ingress implementation must be installed and running.

The two flows are related but distinct:

```text
Configuration flow
kubectl apply -f ingress.yaml
  -> kube-apiserver authenticates, validates, and stores the Ingress object
  -> etcd persists the API object
  -> the Ingress implementation watches the API and configures its data plane

Request flow
client resolves the host and connects to the Ingress entry point on port 80/443
  -> Ingress implementation matches the host and path
  -> it routes to the named Service and its ready backend endpoint(s)
  -> cluster networking (kube-proxy or an equivalent data plane) forwards traffic
     to a Pod IP and target port
```

`kube-apiserver` and etcd store and serve the desired configuration. The Ingress implementation (for example, Traefik or the legacy Minikube NGINX addon) watches Ingress, IngressClass, Service, and endpoint data and programs the traffic path. The Service's selector and EndpointSlices identify eligible Pods. `kube-proxy` does not evaluate Ingress host/path rules; it or an equivalent networking implementation handles Service-level forwarding where applicable.

### Prerequisites: Minikube

Start or select the Minikube cluster, then enable one Ingress implementation. Check that `kubectl` is pointed at Minikube:

```shell
minikube status
kubectl config current-context
```

#### Recommended current Minikube option: Traefik

Minikube's current handbook recommends the Traefik addon; it requires Minikube v1.39.0 or later. Enable it and wait for its Pod to become ready:

```shell
minikube addons enable traefik
kubectl get pods -n kube-system -l app.kubernetes.io/name=traefik
kubectl get ingressclass
```

The addon command reports that `traefik` is enabled. The Pod should reach `Running` and `READY` should show `1/1`. The Ingress class is normally named `traefik`; set `spec.ingressClassName: traefik` in the manifest. The addon is maintained by Traefik Labs, not by Minikube maintainers. See the [Minikube Traefik addon guide](https://minikube.sigs.k8s.io/docs/handbook/addons/traefik/).

#### Legacy Minikube option: NGINX

The existing sample uses the `nginx` class and an NGINX-specific annotation. To keep that sample unchanged, the legacy addon can be enabled with:

```shell
minikube addons enable ingress
minikube addons list
kubectl get pods -n ingress-nginx
kubectl get ingressclass
```

The addon list should show `ingress` as enabled and the controller Pod should become `Running`. Minikube marks its default NGINX addon as unmaintained and recommends Traefik for new Minikube setups. Upstream Ingress NGINX was retired and archived in March 2026, with no further security fixes. Do not enable both addons together: they both bind to ports 80 and 443. If switching to Traefik, use class `traefik` and remove NGINX-only annotations such as `nginx.ingress.kubernetes.io/rewrite-target`.

To switch an existing Minikube cluster from the legacy addon to Traefik, disable the first addon before enabling the second:

```shell
minikube addons disable ingress
minikube addons enable traefik
minikube addons list | grep ingress
```

The final command confirms the legacy `ingress` addon is disabled. `minikube addons list` without a filter shows every addon and its state.

### Prerequisites: Kind

Kind clusters have two documented ways to provide Ingress. Choose one approach for a cluster; the setup, controller, and access path differ.

### Approach 1: Kind native Ingress with cloud-provider-kind

This is the current Kind guide's path for `cloud-provider-kind` v0.9.0 or later. It does not require a separate third-party Ingress controller or special Kind node port mappings. It works with an existing Kind cluster.

In WSL, install the provider binary (Go is required):

```shell
go install sigs.k8s.io/cloud-provider-kind@latest
```

Start the provider in a separate WSL terminal and leave it running while using Ingress. On WSL2 the provider needs elevated permissions. Make the active kubeconfig visible to the process; the following uses the default kubeconfig location when `KUBECONFIG` is not set:

```shell
export KUBECONFIG="${KUBECONFIG:-$HOME/.kube/config}"
sudo env KUBECONFIG="$KUBECONFIG" "$(go env GOPATH)/bin/cloud-provider-kind"
```

The provider watches Kind clusters and supplies external connectivity for Ingress. Confirm the context is the intended Kind cluster before applying resources:

```shell
kind get clusters
kubectl config current-context
kubectl get nodes
```

You do not need to recreate an existing cluster to start the provider. See the [Kind Ingress guide](https://kind.sigs.k8s.io/docs/user/ingress/) and [cloud-provider-kind instructions](https://github.com/kubernetes-sigs/cloud-provider-kind).

### Approach 2: Kind with Ingress NGINX and port mappings

This is the traditional controller-based setup represented by the `ingress-ready=true` label and `extraPortMappings` configuration below. It requires creating a new Kind cluster with the mappings, then installing the Ingress NGINX controller. It is useful for learning the controller-based flow or maintaining an existing lab. **Ingress NGINX was archived in March 2026 and no longer receives releases or security fixes; do not choose it for a new production deployment.** See the [project retirement notice](https://github.com/kubernetes/ingress-nginx#ingress-nginx-retirement).

#### Create a Kind cluster with port mappings

Save this as `kind-ingress.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  - role: worker
  - role: worker
```

`extraPortMappings` forwards host ports 80 and 443 to the control-plane node container. The custom label is not a built-in Ingress setting; it is a marker that a controller Pod can select with a `nodeSelector`. These mappings are set when Kind creates node containers, so an existing cluster does not gain them by editing this file.

Create a new cluster (choose a name not already in use):

```shell
kind create cluster --name ingress-nginx-lab --config kind-ingress.yaml
kubectl config use-context kind-ingress-nginx-lab
```

Host ports 80 and 443 must be available. If they are occupied, map host ports such as 8080 and 8443 to container ports 80 and 443, then use those host ports when testing.

#### Install and place the Ingress controller

The Kind configuration only provides the node label and host-to-node port path. It does not install a controller. Apply the final Kind-specific manifest from the archived Ingress NGINX project:

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/kind/deploy.yaml
```

The current manifest binds controller ports 80 and 443 with `hostPort`, but its node selector does not use the custom `ingress-ready` label. Add that selector so the controller is placed on the control-plane node receiving the host port mappings:

```shell
kubectl patch deployment ingress-nginx-controller \
  --namespace ingress-nginx \
  --type=merge \
  --patch '{"spec":{"template":{"spec":{"nodeSelector":{"ingress-ready":"true"}}}}}'
```

Wait for the controller Pod and check that it runs on the mapped control-plane node:

```shell
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
kubectl get nodes -L ingress-ready
kubectl get pods --namespace ingress-nginx -o wide
kubectl get ingressclass
```

The node label column should show `ingress-ready=true` for `ingress-nginx-lab-control-plane`. The controller Pod's `NODE` column should show that same node, and the IngressClass should include `nginx`. Set `spec.ingressClassName: nginx` in Ingress resources for this approach. If the controller is scheduled on a worker instead, traffic sent through the mapped control-plane ports will not reach it.

### Read the Ingress manifest

The sample below uses the same backend name and Service port as the adjacent [ingress.yaml](ingress.yaml). Set `ingressClassName` to `traefik` for Minikube Traefik, `nginx` for a controller-based NGINX setup, or omit it for Kind's native `cloud-provider-kind` example.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  ingressClassName: traefik # Use nginx for NGINX; omit for Kind native Ingress.
  rules:
    - host: my-service.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-clusterip-service
                port:
                  number: 80
```

Important fields:

| Field | Responsibility |
| --- | --- |
| `apiVersion: networking.k8s.io/v1` and `kind: Ingress` | Select the stable Kubernetes Ingress API. |
| `metadata.name` | Names this Ingress in its namespace. |
| `spec.ingressClassName` | Selects the Ingress implementation. It must match an `IngressClass` installed by that implementation; Kind's native example omits it. |
| `rules[].host` | Matches the HTTP `Host` header. The name must resolve to the cluster's Ingress address, or be sent explicitly in the request header. |
| `paths[].path` and `pathType` | Match the URL path. `Prefix` matches the path prefix on path-segment boundaries. |
| `backend.service.name` | Names a Service in the same namespace as the Ingress. |
| `backend.service.port.number` | Refers to the Service's `spec.ports[].port`, not directly to a Pod's `containerPort`. |

The checked-in [ingress.yaml](ingress.yaml) is a starter file. Its current host value `://my-service.local` is not a valid host; change it to `my-service.local` (or another plain host name) before applying it. Also make sure its class matches the selected Minikube implementation. The NGINX rewrite annotation in that file is only interpreted by NGINX; remove it when using Traefik or Kind's native Ingress support unless you intentionally install a compatible controller.

### Apply the Ingress

After the matching controller/provider is running and the backend Service exists, apply the corrected manifest:

```shell
kubectl apply -f ingress.yaml
```

Representative first-apply output:

```text
ingress.networking.k8s.io/my-ingress created
```

Later applies can report `configured` or `unchanged`. This confirms the API object was accepted; it does not prove that the controller is ready, the Service has endpoints, or the request is reachable.

## 2. Inspect & Monitor

### Explain the API fields

`kubectl explain` reads the Ingress schema available from the API server:

```shell
kubectl explain ingress
kubectl explain ingress.spec --recursive
kubectl explain ingress.spec.rules.http.paths.backend
```

The recursive output includes `ingressClassName`, `rules`, `tls`, and `defaultBackend`. Exact descriptions can vary by Kubernetes version.

### Inspect the Ingress and class

Use these commands to see the desired rule and the implementation class:

| Task | Command | What it shows |
| --- | --- | --- |
| List Ingresses in the current namespace | `kubectl get ingress` | Name, class, hosts, address, ports, and age. |
| List Ingresses in every namespace | `kubectl get ingress --all-namespaces` or `kubectl get ingress -A` | The same summary across namespaces. |
| Read one Ingress | `kubectl get ingress my-ingress` | Summary for the named resource. |
| Read the full object | `kubectl get ingress my-ingress -o yaml` | Spec, metadata, and controller-reported status. |
| See rules and events | `kubectl describe ingress my-ingress` | Hosts, paths, backend Service ports, address, and events. |
| List available classes | `kubectl get ingressclass` | Class names and the controller that claims each class. |

Example output from `kubectl get ingress`:

```text
NAME         CLASS     HOSTS               ADDRESS       PORTS   AGE
my-ingress   traefik   my-service.local    192.168.49.2  80      2m
```

For Kind's native implementation the `CLASS` may be `<none>`. An empty `ADDRESS` can mean the provider/controller has not reconciled the Ingress, or that the selected implementation does not publish an address there; check its setup and logs before testing.

### Check the backend Service and endpoints

```shell
kubectl get service my-clusterip-service
kubectl describe service my-clusterip-service
kubectl get endpointslice -l kubernetes.io/service-name=my-clusterip-service
```

The Ingress and backend Service must be in the same namespace. The named Service port must exist, and its selector should match ready Pods. An EndpointSlice with no addresses usually means the Service selector has no matching ready Pods.

Example Service output:

```text
NAME                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
my-clusterip-service  ClusterIP   10.96.116.70   <none>        80/TCP    4m
```

The Ingress backend points to Service port `80`; the Service maps that port to its own `targetPort` on the Pod.

## 3. Configure Host and Path Routing

The `host` rule is evaluated from the HTTP `Host` header. For local learning, `curl -H` can send that header without setting up DNS. The URL path must also match the configured path. A `404` from an Ingress implementation often means the host/path did not match an Ingress rule; it is different from an application-level `404` returned by the backend.

### Browser and REST-client hostname access

A browser resolves the hostname in the URL before connecting, then sends that hostname as the HTTP `Host` header. If the Ingress rule has `host: my-service.local`, the client machine must resolve `my-service.local` to a reachable Ingress address. A REST client can use the same URL after that name mapping is set up. Alternatively, a REST client that allows a custom `Host` header can connect to a reachable IP or `localhost` port and send `Host: my-service.local` without a hosts-file entry.

Add the hosts entry on the machine running the browser or REST client:

| Client machine | Hosts file |
| --- | --- |
| Linux or a browser/client inside WSL | `/etc/hosts` |
| Windows browser or REST client | `C:\Windows\System32\drivers\etc\hosts` (edit as Administrator) |

The address to put in that entry depends on the cluster and access method, as shown below. A hosts-file entry only resolves the name; the machine still needs a network path to the IP and port.

### Test from Minikube on WSL

Get Minikube's node IP and send a request with the configured host from WSL. This needs no hosts-file entry because `curl` connects to the IP and supplies the host header:

```shell
MINIKUBE_IP="$(minikube ip)"
curl -i -H "Host: my-service.local" "http://${MINIKUBE_IP}/"
```

If the backend responds, `curl -i` shows an HTTP status line (for example `HTTP/1.1 200 OK`) followed by the response headers and body. The response body comes from the application behind the Service.

### Open Minikube Ingress from a browser or REST client

If the browser or REST client can reach the Minikube IP on port 80, add this mapping to the client machine's hosts file:

```shell
echo "$(minikube ip) my-service.local" | sudo tee -a /etc/hosts
```

For a Windows browser/client, use the IP from `minikube ip` in the Windows hosts file instead; editing WSL's `/etc/hosts` only affects programs inside WSL. Then open `http://my-service.local/` or use that URL in the REST client.

If a Windows client cannot reach the Minikube IP and you enabled the Traefik addon, forward the Traefik Service to a local WSL port:

```shell
kubectl port-forward --namespace kube-system service/traefik 8080:80
```

Add `127.0.0.1 my-service.local` to the Windows hosts file and open `http://my-service.local:8080/`. WSL2 normally forwards ports bound in WSL to Windows `localhost`; see [Microsoft's WSL networking guide](https://learn.microsoft.com/windows/wsl/networking). Keep the `kubectl port-forward` command running while using the browser or client.

### Test from Kind using cloud-provider-kind

After `cloud-provider-kind` is running, inspect the address assigned to the Ingress and send the matching host header from WSL. This test needs no hosts-file entry:

```shell
kubectl get ingress my-ingress -o wide
INGRESS_IP="$(kubectl get ingress my-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
curl -i -H "Host: my-service.local" "http://${INGRESS_IP}/"
```

The Kind guide's example shows an IP in the Ingress `ADDRESS` column and verifies the route with `curl`. On WSL2, the external IP can be accessed from WSL when the provider is running with the required permissions.

### Open native Kind Ingress from a browser or REST client

For a browser/client inside WSL, map the Ingress address to the rule's hostname in WSL `/etc/hosts`, then open `http://my-service.local/`:

```shell
INGRESS_IP="$(kubectl get ingress my-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
echo "${INGRESS_IP} my-service.local" | sudo tee -a /etc/hosts
```

For a Windows browser/client using Docker Desktop, `cloud-provider-kind` exposes the Ingress through a Docker-published localhost port. Find the published port for the `kindccm` container:

```shell
docker ps --filter name=kindccm --format 'table {{.Names}}\t{{.Ports}}'
```

In the `PORTS` column, find the host port mapped to container port `80` (for example, `0.0.0.0:42381->80/tcp`). Add `127.0.0.1 my-service.local` to the Windows hosts file, then open `http://my-service.local:42381/`, replacing `42381` with the displayed host port. Use the same URL in a REST client. On Windows, the provider relies on Docker's port mapping rather than direct access to the Kind node IP; see its [WSL2 and Windows notes](https://github.com/kubernetes-sigs/cloud-provider-kind#mac-windows-and-wsl2-support).

If no published host port appears, stop the provider with `Ctrl+C` and restart it with its port-mapping option:

```shell
sudo env KUBECONFIG="$KUBECONFIG" "$(go env GOPATH)/bin/cloud-provider-kind" --enable-lb-port-mapping
```

### Test from Kind using Ingress NGINX

For the port-mapping approach, the controller must be ready on the mapped control-plane node. If the Ingress rule has no `host`, send a request to localhost:

```shell
curl -i http://localhost/
```

If the Ingress rule specifies `my-service.local`, send the matching `Host` header:

```shell
curl -i -H "Host: my-service.local" http://localhost/
```

With custom host port mappings (for example, host port 8080 to node container port 80), use `http://localhost:8080/`. A hosts-file entry such as `127.0.0.1 my-service.local` lets a local browser resolve the name; a `Host` header is enough for a WSL `curl` test.

### Open Kind Ingress NGINX from a browser or REST client

With the default mappings in the cluster configuration above, add `127.0.0.1 my-service.local` to the hosts file on the browser/client machine. Open `http://my-service.local/` from a browser or use that URL in a REST client. If you mapped a different host port, include it in the URL, such as `http://my-service.local:8080/`.

For a REST client that supports custom headers, you can instead request `http://localhost/` (or `http://localhost:8080/` for a custom port) and add `Host: my-service.local`. A browser needs the hosts-file mapping because it does not provide a normal way to override the HTTP `Host` header.

## 4. Edit & Update

For a quick live edit, open the Ingress in the configured editor:

```shell
kubectl edit ingress my-ingress
```

Change the field, save, and exit. `kubectl edit` submits the changed API object; the Ingress implementation observes the update and reconciles its routing configuration. If the editor opens `vi`, press `i` to edit, then `Esc`, type `:wq`, and press Enter to save and quit.

For a repeatable change, edit `ingress.yaml` and apply it again:

```shell
kubectl apply -f ingress.yaml
kubectl describe ingress my-ingress
```

The `kubectl apply` response reports whether the API object was `configured` or `unchanged`. Re-test the host and path after reconciliation. Keep the Ingress class aligned with the installed implementation and update the host rule and test request together.

## 5. Backend Service & TLS

An Ingress sends traffic to a Service, not straight to a Deployment or container. The backend Service must:

- exist in the same namespace as the Ingress;
- expose the port named by `backend.service.port`;
- select the intended Pods; and
- have ready endpoints for successful traffic delivery.

The backend port is the Service `port`. Kubernetes then uses the Service's `targetPort` and the cluster's networking implementation to reach the selected Pod. A common mistake is putting a Pod's `containerPort` in the Ingress backend when the Service exposes a different port.

TLS can be added when a TLS Secret already exists in the Ingress namespace. A minimal spec addition is:

```yaml
spec:
  tls:
    - hosts:
        - my-service.local
      secretName: my-service-tls
  rules:
    - host: my-service.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-clusterip-service
                port:
                  number: 80
```

`my-service-tls` must be a TLS Secret containing a certificate and private key valid for `my-service.local`. The implementation must support TLS for this Ingress. Local controller defaults may use a self-signed certificate; browsers will warn unless the certificate is trusted.

## 6. Delete & Troubleshoot

Delete just the Ingress routing object:

```shell
kubectl delete ingress my-ingress
```

Representative output:

```text
ingress.networking.k8s.io "my-ingress" deleted
```

This removes the host/path rule. It does not delete the backend Service, Deployment, Pods, or the cluster's Ingress implementation. To delete the object from its manifest instead, use `kubectl delete -f ingress.yaml`.

| Symptom | Check |
| --- | --- |
| `no matches for kind "Ingress"` | Check the cluster API version and `apiVersion: networking.k8s.io/v1`. |
| Ingress exists, but no response | Verify the Minikube addon, Kind provider, or Ingress controller for the selected approach; check `kubectl describe ingress my-ingress`. |
| Default `404` from the Ingress endpoint | Confirm the request's `Host` header and path match `spec.rules`; verify the class is handled by the active implementation. |
| `503` or no healthy backend | Check Service port, selectors, Pod readiness, and EndpointSlice addresses. |
| Hostname does not resolve | Use the `curl -H "Host: ..."` form first, then add a hosts-file entry on the machine running the browser. |
| `ADDRESS` is empty | Wait for reconciliation and verify the selected implementation. Kind's native path requires `cloud-provider-kind` v0.9.0+; controller-based Ingress NGINX may publish `localhost` in this local manifest. |
| Traefik ignores an NGINX annotation | NGINX-specific annotations only apply to the NGINX controller; remove or replace them for Traefik. |

## References

- [Kubernetes Ingress concepts](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Minikube Traefik addon](https://minikube.sigs.k8s.io/docs/handbook/addons/traefik/)
- [Minikube addon commands](https://minikube.sigs.k8s.io/docs/commands/addons/)
- [Kind Ingress guide](https://kind.sigs.k8s.io/docs/user/ingress/)
- [Kind port mappings](https://kind.sigs.k8s.io/docs/user/configuration/#extra-port-mappings)
- [cloud-provider-kind setup and WSL2 notes](https://github.com/kubernetes-sigs/cloud-provider-kind)
- [Microsoft WSL networking](https://learn.microsoft.com/windows/wsl/networking)
- [Ingress NGINX Kind manifest (archived)](https://github.com/kubernetes/ingress-nginx/blob/controller-v1.15.1/deploy/static/provider/kind/deploy.yaml)
- [Ingress NGINX retirement notice](https://github.com/kubernetes/ingress-nginx#ingress-nginx-retirement)
