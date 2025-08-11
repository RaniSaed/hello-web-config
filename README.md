# hello-web-config

GitOps configuration for deploying **hello-web** to Kubernetes with **Argo CD**.
**Jenkins** updates the image tag in this repo, and **Argo CD** syncs the change to the cluster automatically.

---

## Repo Structure

```
.
├── argocd/
│   └── application.yaml
└── k8s/
    ├── namespace.yaml
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

---

## What’s Inside

* **`argocd/application.yaml`** — Argo CD `Application` definition pointing to this repo’s `k8s/` path.
* **`k8s/namespace.yaml`** — Creates the `dev` namespace for the application.
* **`k8s/deployment.yaml`** — `nginx` Deployment for the static site. **Image line** is updated by Jenkins.
* **`k8s/service.yaml`** — `ClusterIP` Service exposing the app on port 80.
* **`k8s/ingress.yaml`** — Ingress (`nginx` class) exposing the app at `hello.local`.

---

## Prerequisites

* Kubernetes cluster (**Minikube** or **Kind**)
* **Argo CD** installed in the `argocd` namespace
* For Minikube: **ingress addon** enabled
* Local DNS/hosts entry for `hello.local` pointing to the cluster ingress

---

## Cluster Setup (Minikube Example)

```bash
# Start cluster
minikube start

# Enable ingress controller
minikube addons enable ingress

# Install Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# (Optional) Argo CD CLI login
kubectl -n argocd port-forward svc/argocd-server 8080:80 &
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
argocd login localhost:8080 --username admin --password <PASTE> --insecure

# Create the Argo CD Application
kubectl apply -f argocd/application.yaml

# First-time sync (or wait for auto-sync)
argocd app sync hello-web --prune
```

---

## Hosts File Entry (Linux/macOS)

```bash
sudo nano /etc/hosts
# Add:
127.0.0.1 hello.local
```

Then open in browser:

```
http://hello.local
```

---

## Image & Tag Management

Initial image in `k8s/deployment.yaml`:

```yaml
image: rani19/hello-web:latest
```

**Jenkins** will update this line to a new tag (e.g., `:a1b2c3d4`) and commit it to this repo.
**Argo CD** detects the commit and automatically rolls out the update.

---

## Verify Deployment

```bash
kubectl -n dev get deploy,rs,po,svc,ing
kubectl -n dev rollout status deploy/hello-web
```

In the **Argo CD UI**, the `hello-web` app should be **Synced** and **Healthy**.

---

## Rollback

* **Argo CD UI**: select a previous Sync Revision → **Rollback**
* Or revert the last commit in this repo that changed the image tag → Argo CD reconciles automatically

---

## Common Issues

| Problem                   | Cause / Fix                                                                                 |
| ------------------------- | ------------------------------------------------------------------------------------------- |
| **App OutOfSync**         | Click **Sync** in Argo CD or check repo permissions/branch settings                         |
| **Ingress not reachable** | Verify ingress controller is running and `/etc/hosts` contains `127.0.0.1 hello.local`      |
| **CrashLoopBackOff**      | Inspect pod logs (`kubectl logs`), verify the image exists, and container port is set to 80 |

---
