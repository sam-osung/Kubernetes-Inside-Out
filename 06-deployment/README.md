# Kubernetes Deployments – Replicas, Rolling Updates & Rollbacks

This guide explains how Kubernetes Deployments manage Pods declaratively and provide self-healing, rolling updates, and rollback capabilities.

---

# Why Deployments Matter

Pods are ephemeral. If a Pod dies, it’s gone. Deployments ensure:

- Pods are always running
- Multiple replicas for load balancing
- Rolling updates without downtime
- Rollback if updates fail

**Key takeaway:** Declarative Deployment YAML is the single source of truth.

---

# Apply Deployment

Apply the pre-created manifest:

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get pods -o wide
```

Check that 3 Pods are automatically created.

---

# Self-healing Demonstration

Delete a Pod manually:

```bash
kubectl delete pod <pod-name>
kubectl get pods -o wide
```

Kubernetes automatically recreates the Pod to maintain 3 replicas.

---

# Rolling Updates

Update the Deployment image in `nginx-deployment.yaml`:

```yaml
containers:
  - name: nginx
    image: nginx:1.23
```

Apply the update:

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get pods -o wide
kubectl rollout status deployment/nginx-deployment
```

Pods are updated gradually, one at a time, without downtime.

---

# Rollback

If something goes wrong:

```bash
kubectl rollout undo deployment/nginx-deployment
kubectl get pods -o wide
```

Deployment rolls back to the previous version safely.

---

# Summary

- Pods are ephemeral; Deployments ensure desired state.
- Deployments run multiple replicas and self-heal.
- Rolling updates allow zero-downtime updates.
- Rollbacks allow safe recovery.
- Declarative YAML is the source of truth for cluster state.