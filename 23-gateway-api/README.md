# Kubernetes Gateway API – Complete Beginner Guide (Traefik Edition)

Hello everyone! Welcome back.

In this guide we will learn **Kubernetes Gateway API from scratch** using **Traefik**.

By the end of this guide you will understand:

* What Gateway API is
* Why Gateway API was created
* How it differs from Kubernetes Ingress
* Popular Gateway API controllers
* Core Gateway API components
* How to deploy the **Traefik Gateway Controller**
* How to perform **path-based routing**
* How to perform **host-based routing**
* How to implement **canary deployments**
* How to verify traffic routing

This guide is written from **beginner → production understanding**.

---

# 1. What is Gateway API?

Gateway API is a **modern Kubernetes API** used to control **north-south traffic**.
North-south traffic means traffic that comes **from outside the cluster into services inside the cluster**.

Gateway API helps define:

* Where traffic enters the cluster
* How traffic is matched
* Where traffic is forwarded

Examples of routing rules:

* Path based routing (`/auth`, `/dashboard`)
* Host based routing (`auth.example.com`)
* Header based routing
* Method based routing
* Canary / weighted routing

---

# 2. Why Gateway API Exists

Before Gateway API, Kubernetes used **Ingress**.
Ingress works well but has several limitations.

| Ingress Limitation                     | Gateway API Improvement                         |
| -------------------------------------- | ----------------------------------------------- |
| One object controls everything         | Gateway API separates responsibilities          |
| Limited routing features               | Supports path, host, header, method             |
| Canary deployments require annotations | Native weighted routing                         |
| Difficult multi-team ownership         | Clear separation between platform and app teams |
| Limited cross-namespace routing        | HTTPRoute can reference external Gateways       |

**Important:** Gateway API does **NOT replace Ingress.** Ingress is still widely used and simpler.

---

# 3. Gateway API Controllers

Gateway API itself is **just a specification**. It requires a **controller to implement it**.

Popular controllers include:

| Controller               | Notes                                            |
| ------------------------ | ------------------------------------------------ |
| Traefik                  | Lightweight, simple setup, dynamic configuration |
| NGINX Gateway Controller | Beginner friendly, lightweight                   |
| Kong                     | Full API gateway with plugins                    |
| Contour                  | Uses Envoy proxy                                 |

For this tutorial we use **Traefik Gateway Controller**.

---

# 4. Gateway API Core Components

| Component            | Purpose                                   |
| -------------------- | ----------------------------------------- |
| GatewayClass         | Defines which controller manages gateways |
| Gateway              | Entry point for traffic                   |
| HTTPRoute            | Routing rules                             |
| Service / Deployment | Backend applications                      |

**Important Concept:**

* Platform teams usually manage: GatewayClass + Gateway
* Application teams manage: HTTPRoute

This separation allows **safe multi-team environments**.

---

# 5. Start a Kubernetes Cluster

Start Minikube or any Kubernetes cluster:

```bash
minikube start
```

This creates a **local Kubernetes cluster**.

---

# 6. Install Traefik Gateway Controller

### Step 1: Add Helm Repo

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

### Step 2: Inspect Default Values

You can check all configurable options for Traefik using or take a look at full-values.yaml:

```bash
helm show values traefik/traefik > values-full.yaml
```

> This allows you to **tweak listeners, service type, dashboard, TLS, and other settings** to suit your needs.

---

### Step 3: Create a `values.yaml`

Minimal example to enable GatewayAPI:

```yaml
providers:
  kubernetesIngress:
    enabled: false
  kubernetesGateway:
    enabled: true

gateway:
  namespacePolicy: All
```

> This enables the GatewayAPI provider and allows HTTPRoutes from all namespaces.

---

### Step 4: Install Traefik

```bash
kubectl create namespace traefik

helm upgrade --install traefik traefik/traefik \
  --namespace traefik \
  --version 39.0.7 \
  -f values.yaml
```

> This installation **automatically installs** the Kubernetes Gateway CRDs, creates a default `GatewayClass` named `traefik`, and a default `Gateway` named `traefik-gateway`.

Verify installation:

```bash
kubectl get pods -n traefik
kubectl get svc -n traefik
kubectl describe Gateway traefik-gateway -n traefik
kubectl describe GatewayClass traefik
```

---

# 7. Sample GatewayClass & Listener

**GatewayClass (`traefik`)**:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: traefik
spec:
  controllerName: traefik.io/gateway-controller
```

**Gateway with a listener (`web`)**:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: traefik-gateway
  namespace: traefik
spec:
  gatewayClassName: traefik
  listeners:
    - name: web
      protocol: HTTP
      port: 8000
      allowedRoutes:
        namespaces:
          from: All
```

> Traefik automatically creates this by default. You can tweak the listener ports in your `values.yaml`.

---

# 8. Deploy Backend Applications

Deploy your services:

```bash
kubectl apply -f auth-deploy.yaml
kubectl apply -f dashboard-deploy.yaml
```

> No need to modify these files if you already have them.

---

# 9. Path-Based Routing

Example `/auth` → Auth Service, `/dashboard` → Dashboard Service:

```bash
kubectl apply -f httproute-path.yaml
```

Test:

```bash
curl http://<gateway-ip>/auth
curl http://<gateway-ip>/dashboard
```

---

# 10. Host-Based Routing

Example:

```bash
kubectl apply -f httproute-host.yaml
```

Test with host header:

```bash
curl -H "Host: auth.example.com" http://<gateway-ip>
```

---

# 11. Canary Deployment (Weighted Routing)

Canary deployments allow you to **gradually roll out a new version of your service** without impacting all users.

**Example scenario:**

* You have **Auth v1** running (stable).
* You want to introduce **Auth v2** (new version) to 10% of traffic for testing.

With Traefik and Gateway API, you can define **weighted routing** in your HTTPRoute. Traefik handles traffic distribution automatically.

**Canary Deployment:**

```bash
kubectl apply -f auth-v2-deploy.yaml
kubectl apply -f httproute-auth-canaray.yaml
```

**Testing the canary:**

```bash
for i in {1..30}; do curl http://<gateway-ip>/auth; done
```

> You should see roughly 9/10 requests hitting **Auth v1** and 1/10 hitting **Auth v2**.

**Why use Canary Deployments:**

* Reduce risk by limiting exposure of new versions
* Test performance and stability before full rollout
* Traefik supports this **natively**, no annotations required

**Tip:** You can adjust `weight` values dynamically to gradually increase traffic to the new version as confidence grows.

---

# 12. Key Concepts Recap

| Component    | Responsibility          |
| ------------ | ----------------------- |
| GatewayClass | Defines controller      |
| Gateway      | Entry point for traffic |
| HTTPRoute    | Routing logic           |
| Service      | Backend destination     |
| Deployment   | Application pods        |

Routing types supported:

* Path routing
* Host routing
* Header routing
* Method routing
* Weighted/canary routing

---

# 13. Architecture Flow

```text
Client Request
      ↓
Gateway
      ↓
HTTPRoute
      ↓
Service
      ↓
Pods
```

---

# 14. Tips

---

## NamespacePolicy: Same

* HTTPRoute must live in the same namespace as the Gateway listener.
* Service (backend) must also live in the same namespace as the Gateway listener.
* Cross-namespace references are not allowed.

**Example:**

* Namespace: traefik
* Gateway: traefik-gateway
* HTTPRoute: host-routing
* Service: auth

All resources live in the `traefik` namespace.

---

## NamespacePolicy: All

* HTTPRoute can live in any namespace.
* Service must live in the same namespace as the HTTPRoute, or a ReferenceGrant must be created for cross-namespace access.

**Example:**

* Gateway: traefik-gateway (Namespace: traefik)
* HTTPRoute: host-routing (Namespace: default)
* Service: auth (Namespace: default)

✅ Works because HTTPRoute and Service are in the same namespace.

---

## ReferenceGrant

* Required to allow HTTPRoutes to reference Services in a different namespace.
* Needed when `NamespacePolicy: All` and Service is not in the same namespace as HTTPRoute.

---

## Rule of Thumb

| Policy | HTTPRoute Namespace | Service Namespace                       |
| ------ | ------------------- | --------------------------------------- |
| Same   | Gateway namespace   | Gateway namespace                       |
| All    | Any namespace       | Same as HTTPRoute OR use ReferenceGrant |

---

## Notes

* Traefik Gateway listeners default to port **8000** (HTTP).
* Gateway listener port must match Service exposure.
* `Allowed Routes: Namespaces` controls which HTTPRoutes the Gateway can attach to.

