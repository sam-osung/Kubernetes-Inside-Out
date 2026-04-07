# TLS and SSL Termination in Kubernetes  
## Ingress, cert-manager, Self-Signed & Production Certificates (End-to-End)

This guide explains **TLS/SSL in Kubernetes**, how **TLS termination works**, and how to configure **cert-manager**, **ClusterIssuers**, **self-signed certificates**, and **production Let’s Encrypt certificates** using **Ingress**.

Everything is explained step-by-step with practical commands.

---

# What You Will Learn

By the end of this guide you will understand:

- What **TLS** is and why it matters
- How **TLS termination works in Kubernetes**
- Why TLS is usually terminated at **Ingress or Load Balancer**
- What **cert-manager** does
- What a **ClusterIssuer** is
- How to create **self-signed certificates**
- How to create **production certificates with Let’s Encrypt**
- How this works in **real cloud environments like EKS**

---

# What is TLS and Why It Matters

When a user accesses an application, traffic flows like this:

```
Browser → Ingress / Load Balancer → Service → Pod
```

Without TLS:

- Traffic is **plain text**
- Passwords, tokens, and cookies can be intercepted
- Attackers can modify data in transit

TLS provides three major protections:

| Feature | Explanation |
|------|-------------|
| Encryption | Data is encrypted between client and server |
| Identity | Client verifies the server is authentic |
| Integrity | Data cannot be modified during transit |

Note:

- **SSL is deprecated**
- **TLS is the modern protocol**
- People still say **“SSL certificate”**, but technically it is **TLS**

---

# TLS Termination Explained

TLS termination means **HTTPS ends at a specific component**, and the traffic inside the cluster becomes HTTP.

Typical flow:

```
Browser -- HTTPS --> Ingress Controller / Load Balancer
                          |
                     TLS ends here
                          |
                      HTTP inside cluster
                          |
                     Service → Pod
```

This is called **TLS termination**.

---

# Why TLS Is Not Terminated Inside Pods

You *could* terminate TLS inside pods, but it is not recommended.

Problems:

- Every pod must manage certificates
- Certificate rotation becomes difficult
- Applications must handle TLS logic
- Certificate renewal requires pod restarts

Instead, Kubernetes centralizes TLS at the **Ingress Controller**.

Advantages:

- Easier certificate management
- Centralized TLS configuration
- Automatic renewal with cert-manager
- Simpler application containers

---

# Kubernetes Architecture for TLS

Typical architecture:

```
Client
  |
  | HTTPS
  v
Ingress Controller (NGINX)
  |
  | HTTP
  v
Service
  |
  v
Pod
```

The **Ingress Controller handles HTTPS** and forwards **HTTP traffic internally**.

---

# Example Application Deployment

Apply the deployment:

```
kubectl apply -f deployment.yaml
```

Verify:

```
kubectl get deployments
kubectl get pods
```

---

# Create the Service

Apply the service:

```
kubectl apply -f service.yaml
```

Verify:

```
kubectl get svc
```

The service exposes the pods internally so the **Ingress can route traffic to them**.

---

# Installing cert-manager

cert-manager automates certificate management in Kubernetes.

Responsibilities of cert-manager:

- Request certificates
- Validate domain ownership
- Store certificates as **Kubernetes secrets**
- Automatically renew certificates
- Update Ingress when certificates change

Install cert-manager:

```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

Verify installation:

```
kubectl get pods -n cert-manager
```

You should see pods like:

- cert-manager
- cert-manager-cainjector
- cert-manager-webhook

---

# What is a ClusterIssuer?

A **ClusterIssuer** defines **where certificates come from**.

Think of it as the **certificate factory configuration**.

It tells cert-manager:

- which **certificate authority (CA)** to use
- how to **validate domain ownership**
- how certificates should be issued

ClusterIssuer is **cluster-wide**, meaning any namespace can use it.

Types of issuers:

| Type | Purpose |
|-----|--------|
| Self-Signed | Testing, labs, internal environments |
| ACME (Let’s Encrypt) | Production certificates |

---

# Creating a Self-Signed ClusterIssuer

Apply the self-signed issuer:

```
kubectl apply -f selfsigned-issuer.yaml
```

Verify:

```
kubectl get clusterissuer
```

This issuer tells cert-manager:

> generate certificates signed by itself.

This is useful for:

- development
- labs
- testing TLS setups

---

# Requesting a Certificate

Now request a certificate from the ClusterIssuer.

Apply:

```
kubectl apply -f certificate.yaml
```

Verify:

```
kubectl get certificates
kubectl describe certificate web-cert
```

cert-manager will automatically create a secret.

Check the secret:

```
kubectl get secrets
```

You should see:

```
web-tls
```

This secret contains:

- TLS certificate
- private key

---

# How Ingress Uses the Certificate

Ingress **does not talk to cert-manager directly**.

The flow is:

```
Ingress
   |
   v
TLS Secret
   |
   v
Certificate Resource
   |
   v
ClusterIssuer
   |
   v
Certificate Authority
```

So:

- cert-manager creates the **secret**
- Ingress simply **uses the secret**

Apply the ingress:

```
kubectl apply -f ingress.yaml
```

Verify:

```
kubectl get ingress
```

---

# Testing Self-Signed Certificate

Add the domain locally:

Edit:

```
/etc/hosts
```

Example:

```
<INGRESS-IP> demo.local
```

Now visit:

```
https://demo.local
```

Your browser will show a warning.

This is expected because the certificate is **self-signed**.

---

# Creating Production Certificates with Let’s Encrypt

For real applications we use **Let’s Encrypt**.

Let’s Encrypt provides **free trusted TLS certificates**.

Create the production ClusterIssuer:

```
kubectl apply -f letsencrypt-clusterissuer.yaml
```

Verify:

```
kubectl get clusterissuer
```

---

# Request Production Certificate

Apply:

```
kubectl apply -f prod-certificate.yaml
```

cert-manager will automatically:

1. Contact Let's Encrypt
2. Perform **HTTP-01 challenge**
3. Validate domain ownership
4. Issue the certificate
5. Store it in a Kubernetes secret

Verify certificate:

```
kubectl get certificate
kubectl describe certificate prod-cert
```

Verify secret:

```
kubectl get secret
```

---

# Ingress Using Production Certificate

Apply the ingress referencing the TLS secret:

```
kubectl apply -f prod-ingress.yaml
```

Check:

```
kubectl get ingress
```

Now visit:

```
https://app.example.com
```

Your browser should show a **trusted HTTPS connection**.

---

# Example Flow in AWS EKS

Assume the domain:

```
myapp.samueldevops.com
```

Full flow:

```
User Browser
      |
      | HTTPS
      v
AWS Load Balancer
      |
      v
NGINX Ingress Controller
      |
      v
Service
      |
      v
Pod
```

Meanwhile cert-manager handles certificates:

```
cert-manager
      |
      v
ClusterIssuer
      |
      v
Let's Encrypt
      |
      v
Certificate Secret
      |
      v
Ingress uses secret
```

Everything is automated.

Pods do **not manage certificates**.

---

# TLS Termination Recap

TLS ends at the **Ingress Controller**.

```
Browser --- HTTPS ---> Ingress
                           |
                      TLS terminates
                           |
                        HTTP
                           |
                        Service
                           |
                          Pod
```

Pods only receive **HTTP traffic**.

---

# Final Summary

Key takeaways:

- TLS encrypts communication between users and applications
- TLS termination usually happens at **Ingress or Load Balancer**
- **cert-manager automates certificate management**
- **ClusterIssuer defines where certificates come from**
- Self-signed certificates are good for testing
- Let’s Encrypt provides trusted production certificates
- Ingress consumes **TLS secrets created by cert-manager**
- Pods never handle certificates directly

---

# Quick Commands Reference

```
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# Verify cert-manager
kubectl get pods -n cert-manager

# Create self-signed issuer
kubectl apply -f selfsigned-issuer.yaml

# Request certificate
kubectl apply -f certificate.yaml

# Deploy application
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# Apply ingress
kubectl apply -f ingress.yaml

# Create production issuer
kubectl apply -f letsencrypt-clusterissuer.yaml

# Request production certificate
kubectl apply -f prod-certificate.yaml

# Apply production ingress
kubectl apply -f prod-ingress.yaml
```

---

**Important beginner concept:**

> Ingress never talks directly to the Certificate Authority.  
> cert-manager and ClusterIssuer handle certificate creation, renewal, and storage.  
> Ingress simply consumes the TLS secret.