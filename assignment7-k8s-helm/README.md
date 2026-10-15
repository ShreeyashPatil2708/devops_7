# Assignment 7 — Kubernetes Architecture and Helm

This folder contains the full work for Assignment 7: exploring Kubernetes
architecture on a real local cluster, and packaging/deploying a sample app
(nginx) using a Helm chart.

## Folder structure

```
assignment7-k8s-helm/
├── README.md                 <- this file
├── REPORT.md                 <- full lab write-up (Aim, Theory, Procedure, Observations, Conclusion)
├── docs/
│   └── architecture.md       <- Kubernetes architecture diagram + component table (Part A)
├── myapp/                    <- the Helm chart (Part B)
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
└── outputs/                  <- real terminal output captured for every command we ran
    ├── 00_env_check.txt
    ├── 01_architecture_commands.txt
    ├── 02_helm_version.txt
    ├── 03_helm_lint.txt
    ├── 04_helm_template.txt
    ├── rendered.yaml          <- full `helm template` output
    ├── 05_helm_install.txt
    ├── 06_helm_list_status.txt
    ├── 07_kubectl_verify.txt
    ├── 08_app_access.txt
    ├── 09_helm_upgrade.txt
    ├── 10_helm_rollback.txt
    └── 11_helm_uninstall.txt
```

## Tools used

- Docker Desktop (container runtime for the cluster)
- `kind` (Kubernetes IN Docker) — used to create a real local Kubernetes cluster
- `kubectl` — command line tool to talk to the cluster
- `helm` — package manager for Kubernetes

## Setting up the cluster

The `kind` cluster is ephemeral and gets torn down between sessions, so spin
it back up before running anything else.

```bash
cd assignment7-k8s-helm

# 1. Make sure Docker Desktop is running (open the app if it isn't)

# 2. Recreate the cluster (takes ~30-60 sec)
kind create cluster --name assignment7

# 3. Confirm it's healthy
kubectl get nodes
kubectl cluster-info
```

The `myapp` chart folder under this directory is static — nothing needs to
be recreated for Part B, it's ready to `helm install` as soon as the cluster
is up.

## Full walkthrough

Run these in order from the `assignment7-k8s-helm` directory.

```bash
# --- Part A: architecture ---
kubectl get nodes -o wide
kubectl get pods -n kube-system -o wide
kubectl cluster-info
kubectl describe node $(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')

# --- Part B: Helm chart ---
cat myapp/Chart.yaml
cat myapp/values.yaml
helm lint ./myapp
helm template myapp ./myapp | less      # or just | head -40

# --- Install ---
helm install myrelease ./myapp
helm list
helm status myrelease
kubectl get pods
kubectl get deployments
kubectl get services

# --- Access the app (needs a second terminal tab) ---
kubectl port-forward svc/myrelease-myapp 18080:80
# in the SECOND tab:
curl -i http://localhost:18080/

# --- Upgrade ---
helm upgrade myrelease ./myapp --set replicaCount=3
kubectl get pods
helm history myrelease

# --- Rollback ---
helm rollback myrelease 1
kubectl get pods

# --- Cleanup ---
helm uninstall myrelease
helm list
kubectl get all
```

### Winding down

```bash
kind delete cluster --name assignment7
```

### Notes

- If `kubectl get nodes` ever hangs or errors, Docker Desktop probably isn't
  running yet — start it first.
- Stop the `kubectl port-forward` command with `Ctrl+C` before moving to the
  next command in the same tab.
- All of this matches exactly what's documented in `REPORT.md` and
  `outputs/`, so if anything looks different live, the saved reference
  output there is a backup.

## How to reproduce this from scratch

```bash
# 1. Install tools (macOS, Homebrew)
brew install kind helm

# 2. Create a local cluster
kind create cluster --name assignment7
kubectl get nodes
kubectl cluster-info

# 3. Create and customize the Helm chart
helm create myapp
# then edit myapp/Chart.yaml and myapp/values.yaml (see Step 2 in REPORT.md)

# 4. Helm lifecycle
helm lint ./myapp
helm template myapp ./myapp > outputs/rendered.yaml
helm install myrelease ./myapp
helm list
helm status myrelease
kubectl get pods
kubectl get deployments
kubectl get services

# 5. Access the app
kubectl port-forward svc/myrelease-myapp 18080:80
curl http://localhost:18080/

# 6. Upgrade / rollback
helm upgrade myrelease ./myapp --set replicaCount=3
helm history myrelease
helm rollback myrelease 1

# 7. Clean up
helm uninstall myrelease
kind delete cluster --name assignment7
```

See `REPORT.md` for the full lab write-up with explanations and real captured output.
