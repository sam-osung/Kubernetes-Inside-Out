# Kubernetes Services – ClusterIP, NodePort, LoadBalancer

This guide explains how Kubernetes Services provide stable networking for Pods.

---

# Why Services Matter

Pods are ephemeral. If a Pod dies, its IP changes. Directly accessing Pods is unreliable.

**Services provide:**

- Stable IPs and DNS names
- Load balancing across multiple Pods
- Access within cluster or externally

**Key takeaway:** Pods are dynamic, Services are stable.

---

# Types of Services

| Type          | Description               | Example Use Case                       |
|---------------|---------------------------|---------------------------------------|
| ClusterIP     | Internal access only      | Pod-to-Pod communication inside cluster |
| NodePort      | Exposes Pod on node port  | Local access or testing externally    |
| LoadBalancer  | Exposes Pod via cloud LB  | Production cloud environment          |

⚠️ Local Development Notes:

- LoadBalancer only works fully in cloud clusters (EKS, GKE, AKS)
- Minikube or kind cannot create real cloud LBs
- Use NodePort or port-forwarding to simulate access

---

# Apply Deployment and Service

Apply the pre-created manifests:

```bash
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml
kubectl get svc
```

Check that `nginx-service` has a stable ClusterIP.

---

# Accessing Services

### ClusterIP

- Internal-only
- Access within cluster using DNS:

```
nginx-service.dev.svc.cluster.local:80
```

### NodePort

- Change Service type to NodePort in your manifest and apply:

```bash
kubectl apply -f nginx-service.yaml
```

- Kubernetes assigns port in range 30000–32767
- Access externally:

```
http://<node-ip>:<nodePort>
```

### LoadBalancer

- Change Service type to LoadBalancer in your manifest and apply:

```bash
kubectl apply -f nginx-service.yaml
```

- Cloud: provisions external LB with public IP
- Local (Minikube/kind): simulate with port-forward or `minikube service nginx-service`

---

# Port-forwarding for Local Development

Forward Service port to local machine:

```bash
kubectl port-forward svc/nginx-service 8080:80
```

- 8080 → local port
- 80 → Service port
- Access in browser: [http://localhost:8080](http://localhost:8080)

---

# Key Points on Selectors

- Services route traffic to all Pods matching their selector
- Scaling Deployment automatically distributes traffic across all replicas
- No Service updates are needed when Pods change