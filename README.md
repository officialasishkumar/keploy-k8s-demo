# Keploy K8s Demo

Reference implementation for deploying [Keploy](https://keploy.io) k8s-proxy on a local Kind cluster using **ArgoCD** or **Flux CD** with **Contour** as the ingress controller.

## Repository Structure

```
.
├── kind-cluster.yaml             # Kind config with NodePort 30080
├── sample-app/                   # Sample Go HTTP service
│   ├── main.go
│   ├── go.mod
│   └── Dockerfile
├── k8s/                          # Raw K8s manifests for the sample app
│   ├── namespace.yaml
│   ├── deployment.yaml
│   └── service.yaml
├── argocd/                       # ArgoCD Applications
│   ├── keploy-k8s-proxy.yaml    # ArgoCD App for k8s-proxy Helm chart
│   ├── sample-app.yaml          # ArgoCD App for sample-order-service
│   └── k8s-proxy-httpproxy.yaml # Contour HTTPProxy (TLS passthrough)
└── flux/                         # Flux CD manifests
    ├── keploy-source.yaml        # OCI HelmRepository for Keploy
    ├── keploy-k8s-proxy.yaml    # HelmRelease for k8s-proxy
    └── k8s-proxy-httpproxy.yaml # Contour HTTPProxy (TLS passthrough)
```

## Prerequisites

- [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installing-from-release-binaries)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/)
- [Docker](https://docs.docker.com/get-docker/)
- A [Keploy Enterprise](https://app.keploy.io) account with an access key

## Quick Start (Common Steps)

These steps are the same regardless of whether you use ArgoCD or Flux CD.

### 1. Create the Kind cluster

```bash
kind create cluster --name keploy-demo --config kind-cluster.yaml
```

### 2. Build and load the sample app

```bash
docker build -t sample-order-service:latest sample-app/
kind load docker-image sample-order-service:latest --name keploy-demo
```

### 3. Deploy the sample app

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

Verify it's running:

```bash
kubectl -n staging get pods
kubectl -n staging port-forward svc/sample-order-service 9090:80 &
curl http://localhost:9090/healthz
curl http://localhost:9090/api/orders
```

### 4. Install Contour

```bash
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
kubectl -n projectcontour rollout status deployment/contour
kubectl -n projectcontour rollout status daemonset/envoy
```

Patch Envoy so the HTTPS listener uses NodePort 30080 (mapped to the host):

```bash
kubectl patch svc envoy -n projectcontour --type='json' -p='[
  {"op": "replace", "path": "/spec/type", "value": "NodePort"},
  {"op": "replace", "path": "/spec/ports/0/nodePort", "value": 30081},
  {"op": "replace", "path": "/spec/ports/1/nodePort", "value": 30080}
]'
```

Verify:

```bash
kubectl get svc -n projectcontour
# envoy  NodePort  ...  80:30081/TCP,443:30080/TCP
```

### 5. Add /etc/hosts entry

The k8s-proxy uses TLS passthrough with SNI routing, so you need a hostname (not an IP). Add an entry to `/etc/hosts`:

```bash
echo '127.0.0.1  keploy-demo.local' | sudo tee -a /etc/hosts
```

### 6. Create the Keploy access key secret

Go to [app.keploy.io/clusters](https://app.keploy.io/clusters) → **Connect New Cluster**:
- **Cluster Name**: `keploy-demo`
- **Ingress URL**: `https://keploy-demo.local:30080`

Copy the access key and create the secret:

```bash
kubectl create namespace keploy
kubectl -n keploy create secret generic keploy-credentials \
  --from-literal=access-key="<YOUR_ACCESS_KEY>"
```

> **Never commit your access key to Git.**

---

Now choose **one** of the GitOps tools below.

---

## Option A: Deploy with ArgoCD

### A1. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deployment/argocd-server
```

Get the admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

### A2. Update manifests with your values

Edit `argocd/keploy-k8s-proxy.yaml`:
- Replace `<YOUR_CLUSTER_NAME>` with the cluster name from the Keploy UI (e.g. `keploy-demo`)
- Replace `<YOUR_INGRESS_HOST>` with `keploy-demo.local`

Edit `argocd/k8s-proxy-httpproxy.yaml`:
- Replace `<YOUR_INGRESS_HOST>` with `keploy-demo.local`

Edit `argocd/sample-app.yaml`:
- Replace `<YOUR_GITHUB_USERNAME>` with your GitHub username

### A3. Apply the HTTPProxy and ArgoCD Applications

```bash
kubectl apply -f argocd/k8s-proxy-httpproxy.yaml
kubectl apply -f argocd/keploy-k8s-proxy.yaml
kubectl apply -f argocd/sample-app.yaml
```

### A4. Verify

```bash
# Check ArgoCD apps
kubectl get applications -n argocd

# Check k8s-proxy pod
kubectl get pods -n keploy

# Check HTTPProxy status (should show "valid")
kubectl get httpproxy -A

# Test TLS passthrough
curl -sk https://keploy-demo.local:30080/healthz
```

### A5. Access ArgoCD UI

```bash
kubectl -n argocd port-forward svc/argocd-server 8443:443 &
```

Open `https://localhost:8443` — login with `admin` and the password from step A1.

### A6. Test GitOps self-healing

ArgoCD auto-syncs from Git. Try scaling the sample app manually:

```bash
kubectl -n staging scale deployment sample-order-service --replicas=5
```

Watch ArgoCD revert it back to 2 replicas (the value in `k8s/deployment.yaml`):

```bash
kubectl -n staging get pods -w
```

### A7. Test GitOps sync from Git

Change the replica count in `k8s/deployment.yaml`, commit, and push:

```bash
# Edit k8s/deployment.yaml — change replicas: 2 to replicas: 3
git add k8s/deployment.yaml
git commit -m "Scale sample app to 3 replicas"
git push
```

ArgoCD detects the change and scales the deployment:

```bash
kubectl -n staging get pods -w
# You should see a third pod come up
```

---

## Option B: Deploy with Flux CD

### B1. Install Flux CLI

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

### B2. Bootstrap Flux

```bash
flux bootstrap github \
  --owner=<YOUR_GITHUB_USERNAME> \
  --repository=keploy-k8s-demo \
  --branch=main \
  --path=flux \
  --personal
```

This installs Flux on the cluster and configures it to watch the `flux/` directory.

### B3. Update manifests with your values

Edit `flux/keploy-k8s-proxy.yaml`:
- Replace `<YOUR_CLUSTER_NAME>` with the cluster name from the Keploy UI (e.g. `keploy-demo`)
- Replace `<YOUR_INGRESS_HOST>` with `keploy-demo.local`

Edit `flux/k8s-proxy-httpproxy.yaml`:
- Replace `<YOUR_INGRESS_HOST>` with `keploy-demo.local`

Commit and push:

```bash
git add flux/
git commit -m "Configure Keploy k8s-proxy for local cluster"
git push
```

### B4. Wait for Flux to reconcile

```bash
# Force immediate reconciliation (optional)
flux reconcile source git flux-system

# Check HelmRelease status
flux get helmreleases -n keploy

# Check pods
kubectl get pods -n keploy

# Check HTTPProxy (should show "valid")
kubectl get httpproxy -A
```

### B5. Verify

```bash
# Test TLS passthrough
curl -sk https://keploy-demo.local:30080/healthz
```

### B6. Test GitOps sync from Git

Change a Helm value in `flux/keploy-k8s-proxy.yaml` (e.g. `replicaCount: 2`), commit, and push:

```bash
git add flux/keploy-k8s-proxy.yaml
git commit -m "Scale k8s-proxy to 2 replicas"
git push

# Watch Flux apply the change
flux get helmreleases -n keploy -w
kubectl get pods -n keploy -w
```

---

## Testing with Keploy

Once either ArgoCD or Flux has deployed the k8s-proxy and the sample app is running:

### Record traffic

1. Go to [app.keploy.io/clusters](https://app.keploy.io/clusters)
2. Open your cluster → you should see `sample-order-service` under **Deployments**
3. Click **Record** on the deployment

### Send test traffic

```bash
# Port-forward the sample app
kubectl -n staging port-forward svc/sample-order-service 9090:80 &

# List orders
curl http://localhost:9090/api/orders

# Create an order
curl -X POST http://localhost:9090/api/orders \
  -H "Content-Type: application/json" \
  -d '{"product": "Headphones", "quantity": 3, "price": 79.99}'

# Get specific order
curl http://localhost:9090/api/orders/1
```

### Replay traffic

1. Stop recording in the Keploy UI
2. Click **Replay** to replay the captured requests
3. Keploy compares responses to detect regressions

## Cleanup

```bash
kind delete cluster --name keploy-demo
```
