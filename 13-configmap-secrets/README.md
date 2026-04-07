# Kubernetes ConfigMaps and Secrets

ConfigMaps and Secrets in Kubernetes allow you to manage application configuration and sensitive data separately from container images. This guide explains how to create, use, and inject ConfigMaps and Secrets into Pods.

---

# Why Configs Matter

When building applications:

- Deployments and StatefulSets manage Pods  
- Services expose Pods and route traffic  
- HPA scales Pods automatically  

But hardcoding values like database URLs, passwords, or environment variables in your container is **bad practice**.  

Kubernetes provides **ConfigMaps** and **Secrets** to separate configuration from application code, making your deployments more flexible and secure.

---

# Part 1 – What are ConfigMaps and Secrets

## ConfigMaps

- Store **non-sensitive configuration data**  
- Examples: application properties, environment variables, config files  
- Not encrypted → not suitable for passwords  

## Secrets

- Store **sensitive data**: passwords, tokens, API keys  
- Base64 encoded (and optionally encrypted at rest depending on cluster setup)  
- Protects secrets from being visible in plain YAML  

| Feature       | ConfigMap               | Secret                      |
|---------------|-----------------------|----------------------------|
| Purpose       | Non-sensitive config   | Sensitive data             |
| Encoding      | Plain text             | Base64 encoded             |
| Usage         | Env vars, config files | Env vars, mounted files    |
| Security      | Not encrypted          | Encoded / can be encrypted |

Think of **ConfigMap** as your settings.json and **Secret** as your password vault.

---

# Part 2 – Creating a ConfigMap

Apply the ConfigMap:

```bash
kubectl apply -f app-config.yaml
```

Verify:

```bash
kubectl get configmap app-config
kubectl describe configmap app-config
kubectl get configmap app-config -o yaml
```

**Observation**:

- Contains key-value pairs like `DATABASE_URL` and `APP_MODE`  
- Can be injected into Pods as environment variables or config files  

---

# Part 3 – Using ConfigMap in a Deployment

Apply the Deployment referencing the ConfigMap:

```bash
kubectl apply -f nginx-configmap.yaml
```

Exec into the Pod and check environment variables:

```bash
kubectl exec -it <pod-name> -- /bin/sh
echo $DATABASE_URL
echo $APP_MODE
```

✅ Values from the ConfigMap are available in the Pod.

---

# Part 4 – Creating a Secret

Apply the Secret:

```bash
kubectl apply -f app-secret.yaml
```

Verify:

```bash
kubectl get secret app-secret 
kubectl describe secret app-secret 
kubectl get secret app-secret -o jsonpath="{.data.DB_PASSWORD}" | base64 --decode
```

**Observation**:

- Secret data is **base64 encoded**  
- Example: `password123` → `cGFzc3dvcmQxMjM=`  

Other secret types:

- `kubernetes.io/tls` → TLS certificate and key  
- `kubernetes.io/dockerconfigjson` → Docker registry credentials  
- `Opaque` → generic base64-encoded secret (most common)  

---

# Part 5 – Using Secret in a Deployment

## As Environment Variable

Reference the secret:

```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: app-secret
      key: DB_PASSWORD
```

## As Mounted Volume

Mount the secret as files:

```yaml
volumeMounts:
- name: secret-volume
  mountPath: /etc/secrets
volumes:
- name: secret-volume
  secret:
    secretName: app-secret
```

Exec into the Pod to check secret content:

```bash
kubectl exec -it <pod-name> -n dev -- cat /etc/secrets/DB_PASSWORD
```

✅ Secret content is available **without hardcoding** into container images.

---

# Part 6 – Key Points

- **ConfigMaps** → non-sensitive config  
- **Secrets** → sensitive data, base64 encoded  
- Can be referenced as environment variables or mounted files  
- Secrets are safer than ConfigMaps for passwords  
- Opaque type → generic secret; others exist for TLS, Docker credentials  
- Keep configs and secrets **out of container images** → easier to manage and secure  

---
