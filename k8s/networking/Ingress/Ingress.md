## Ingress Management
### Get All Ingress
```
$ kubectl get ingress --all-namespaces
or
$ kubectl get ingress -A
```
### Get Ingress by name
```
$ kubectl get ingress hello-world
```
### Get Ingress in yaml format
```
$ kubectl get ingress hello-world -o yaml
```

### Create Ingress
```
$ kubectl apply -f -<<'EOF
apiVersion: networking.k8s.io/v1
metadata:
  name: hello
  namespace: default
spec:
  ingressClassName: nginx
  rules:
    - host: hello.local
      http:
        paths:
          - path:/hello
            pathType: Prefix
            backend:
              service:
                name: hello-world
                port:
                  number: 8080
EOF
```
### Edit Ingress
```
$ kubectl edit ingress hello-world
```
> After the above command press 'i' to enter in insert mode.
> 
> Post edit press `esc` button and then press colon `:` button and to save you need to press 'w' button for write/save and 'q' button for quit.   

### Enable Minikube addon
> Why Needed: The ingress manifest only creates routing rules. It does not create the component that receives HTTP trafic. It installs the NGINX Ingress Controller- the actual reverse proxy that reads and enforce your ingress rules
```
$ minikube addons enable ingress
```
> K8s supports other ingress controller support apart from NGINX, please refer the link for more details: https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/
>  For your minikube cluster, use the built-in controller choices first:
```
$ minikube addons list | grep -E 'ingress|traefik|kong|istio|ambassador'
```
> To switch from one ingress controller to anotger you can disable/enable them
```
$ minikube addons disable ingress # Removes the old ingress-nginx addon, you can keep multiple if needed
$ minikube addons enable traefik
$ kubectl get ingressclass
```
> Then you need to change the ingress class name in above ingress manifest file in below mentioned block
```
spec:
  ingressClassName: traefik
```

### Check Minikube addons
#### List all addons
```
$ minikube addons list
```
#### List particular addons
```
$ minikube addons list | grep ingress
```

### Make hosts entry for local dns
> Adds a local DNS override because hello.local is not public DNS. Without it, your shell/browser does not know which IP address hello.local should use.
> Note: Order does not matter much- you can apply the manifest before or after enabling addon. The rule will begin working only after both the controller is running and hello.local resolves to the minikube IP.
```
$ echo "$(minikube ip) hello.local" | sudo tee -a /etc/hosts
```
### Edit Ingress in yaml format
```
$ kubectl edit ingress hello-world 
```
