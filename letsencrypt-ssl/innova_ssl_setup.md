# Innova-Path Free SSL Setup on GKE

This guide provides a complete, step-by-step setup for enabling free SSL using Let's Encrypt and Cert-Manager on your GKE cluster for `innova-path.com` while preserving existing SSL sites.

---

## Step 0 — Pre-check: Cert-Manager

Check if cert-manager is installed:

```bash
kubectl get pods -n cert-manager
```

- If pods exist → cert-manager already installed 
- If not → follow Step 1.

---

## Step 1 — Install Cert-Manager

**Purpose:** Handles certificate issuance, renewal, and webhook automation.

### Option 1: Official Jetstack YAML (simpler)

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.19.1/cert-manager.yaml
```

### Option 2: Custom GKE YAML (`cert-manager-fixed.yaml`) (Optional you don't have to do for IP)

- Adds resource limits, liveness probes, and GKE Autopilot annotations.

Apply:

```bash
kubectl apply -f cert-manager-fixed.yaml
```

Verify pods:

```bash
kubectl get pods -n cert-manager
```

Should see:
- `cert-manager`
- `cert-manager-cainjector`
- `cert-manager-webhook`

All **Running** 

---

## Step 2 — Apply RBAC for Leader Election (Optional)

**Purpose:** Ensures cert-manager replicas coordinate properly to avoid conflicts.

Apply the file:

```bash
kubectl apply -f cert-manager-leader-election-rbac.yaml
```

---

## Step 3 — Create ClusterIssuer for Let's Encrypt

**Purpose:** Defines how cert-manager requests certificates.

File: `cluster-issuer.yaml`

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: hkinnovapath@gmail.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-private-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

Apply:

```bash
kubectl apply -f cluster-issuer.yaml
```

Verify:

```bash
kubectl get clusterissuer
kubectl describe clusterissuer letsencrypt-prod
```

Status should show **True**.

---

## Step 4 — Create Certificate Resource

**Purpose:** Defines which domains to cover, the secret to store, and which ClusterIssuer to use.

File: `innova-path-cert.yaml`

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: innova-path-tls
  namespace: default
spec:
  secretName: innova-path-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - innova-path.com
    - www.innova-path.com
```

Apply:

```bash
kubectl apply -f innova-path-cert.yaml
```

Verify:

```bash
kubectl get certificate
kubectl describe certificate innova-path-tls
kubectl get secret innova-path-tls
```

---

## Step 5 — Update Ingress for innova-path.com

**Purpose:** Configure routing and enable SSL using cert-manager.

File: `ip-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ip-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - innova-path.com
        - www.innova-path.com
      secretName: innova-path-tls
  rules:
    - host: innova-path.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ip-frontend-service
                port:
                  number: 80
    - host: www.innova-path.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ip-frontend-service
                port:
                  number: 80
```

Apply:

```bash
kubectl apply -f ip-ingress.yaml
```

---

## Step 6 — Monitor SSL Issuance

```bash
kubectl describe certificate innova-path-tls
kubectl get certificate
kubectl get orders
kubectl get challenges
```

Check secret:

```bash
kubectl get secret innova-path-tls -n default
kubectl describe secret innova-path-tls -n default
```

Check cert-manager logs if needed:

```bash
kubectl logs -n cert-manager deploy/cert-manager -f
kubectl logs -n cert-manager deploy/cert-manager-webhook -f
```

---

## Step 7 — Verify HTTPS

- Open in browser:
  - `https://innova-path.com`
  - `https://www.innova-path.com`
- Both should show **Let’s Encrypt SSL**
- Existing GoDaddy SSL site remains unaffected.

---

## **Summary of Required Files**

| File | Purpose |
|------|---------|
| `cert-manager-fixed.yaml` | Deploy cert-manager with GKE-specific tweaks |
| `cert-manager-leader-election-rbac.yaml` | Ensures proper leader election (optional if >1 replica) |
| `cluster-issuer.yaml` | Tells cert-manager how to request certs from Let’s Encrypt |
| `innova-path-cert.yaml` | Defines the certificate, secret, and domains |
| `ip-ingress.yaml` | Ingress for routing + enabling SSL with cert-manager |

✅ All files are required for a scratch setup.

