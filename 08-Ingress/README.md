# Kubernetes Ingress – Beginner-Friendly Guide

Kubernetes Ingress provides a single entry point to route HTTP/HTTPS traffic to multiple services in a cluster, simplifying access management and reducing the number of LoadBalancers required.

---

# Why Ingress Matters

Without Ingress:

- NodePort → exposes Pods on high ports (30000–32767), hard to manage
- LoadBalancer → one LB per Service, expensive
- ClusterIP → internal only, cannot be accessed externally

Ingress solves this by:

- Providing a single entry point
- Routing traffic intelligently to multiple services
- Acting as a smart router/gateway

---

# Ingress Basics

Key concepts:

- **Ingress Resource**: Defines routing rules (path-based or host-based)
- **Ingress Controller**: Implements routing rules; required for Ingress to work

Popular Ingress Controllers:

| Controller | Notes |
|------------|------|
| NGINX      | Most common, open source, works locally & on cloud |
| HAProxy    | High performance, supports advanced routing |
| Cloud LBs  | AWS ALB Ingress Controller, GKE, AKS etc. |

- Runs as a Deployment/Pod
- Handles routing, TLS termination, and load balancing
- Local development: EXTERNAL-IP may be empty; use port-forwarding to test

---

# Installing NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.14.1/deploy/static/provider/cloud/deploy.yaml
```

Verify Pods:

```bash
kubectl get pods -n ingress-nginx
```

Verify Service:

```bash
kubectl get svc -n ingress-nginx
```

Local workaround for EXTERNAL-IP:

```bash
kubectl port-forward svc/ingress-nginx-controller 8080:80 -n ingress-nginx
```
If on minikube, do 

```bash
minikube addons enable ingress
```

```bash
minikube tunnel
```
```bash
kubectl get svc -n ingress-nginx
```

---

# Creating Services for Ingress

## Step 1 – Deploy Services

### NGINX Deployment + Service

```bash
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml
```

### API Deployment + Service

```bash
kubectl apply -f api-deployment.yaml
kubectl apply -f api-service.yaml
```

*(YAML files already available in GitHub repo)*

---

# Step 2 – Create Ingress Resource

```bash
kubectl apply -f path-based-ingress.yaml
```
```bash
kubectl apply -f host-based-ingress.yaml
```

Check:

```bash
kubectl get ingress
```

Ingress routes:---

- `/web` → nginx-service
- `/api` → api-service
- Host-based routing example:

```
web.myapp.local → nginx-service
api.myapp.local → api-service
```

Add entries to `/etc/hosts` on your local machine:

```
127.0.0.1 web.myapp.local
127.0.0.1 api.myapp.local
```

Test on your browser or:

```bash
curl http://web.myapp.local:8080
curl http://api.myapp.local:8080
```

---

# Step 3 – IngressClass

Defines which controller handles the Ingress resource.

Example:

```bash
kubectl get ingressclass
```

- `ingressClassName: nginx` links Ingress to this controller

---


# Step 4 – Local Testing

- EXTERNAL-IP may be empty locally
- Use port-forwarding or do minikube tunnel (if on minikube):

```bash
kubectl port-forward svc/ingress-nginx-controller 8080:80 -n ingress-nginx
```

- Test host-based routing via `/etc/hosts` and curl or browser

---

# Summary & Key Points

- Ingress provides a single entry point for multiple services
- Requires an Ingress Controller
- Supports path-based and host-based routing
- IngressClass links resources to controllers
- Handles can also handle TLS termination for HTTPS
- For local clusters, use port-forwarding when EXTERNAL-IP is unavailable