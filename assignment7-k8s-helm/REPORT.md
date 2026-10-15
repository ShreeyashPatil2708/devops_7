# Assignment 7: Kubernetes Architecture and Helm

## Aim

To understand the architecture of Kubernetes by exploring a real running
cluster, and to learn Helm (the package manager for Kubernetes) by creating,
customizing, installing, upgrading, rolling back, and uninstalling a sample
application chart.

## Objectives

1. Set up a real local Kubernetes cluster and identify its control-plane and
   worker components.
2. Understand how Kubernetes performs container orchestration: deployment,
   scaling, networking, service discovery, load balancing, and self-healing.
3. Create a Helm chart for a sample application (nginx) and customize it.
4. Perform the full Helm workflow: lint, template, install, verify, access,
   upgrade, view history, rollback, and uninstall.

## Theory

### Container orchestration

Running one container by hand is easy, but real applications need many
containers working together, restarted automatically if they crash, scaled up
and down with load, and reachable by other services without hardcoding IP
addresses. A **container orchestrator** like Kubernetes automates all of this:
it schedules containers onto machines, restarts failed ones, scales replicas,
and provides stable networking between them.

### Kubernetes architecture

A Kubernetes cluster has two kinds of nodes:

- **Control plane node(s)** — the "brain" of the cluster. It decides what
  should run where, and keeps track of the cluster's desired vs. actual state.
  - **kube-apiserver**: the single entry point for all requests (from
    `kubectl`, from other components, from users). Every change to the
    cluster goes through it.
  - **etcd**: a key-value database that stores the entire state of the
    cluster (every object definition).
  - **kube-scheduler**: decides which node a new pod should run on.
  - **kube-controller-manager**: runs background loops that notice when the
    actual state doesn't match the desired state (e.g. a pod died) and fixes
    it.
- **Worker node(s)** — where the actual application containers run.
  - **kubelet**: an agent on each node that talks to the API server and
    starts/stops containers as instructed.
  - **kube-proxy**: sets up networking rules so that Kubernetes Services can
    load-balance traffic to the right pods.
  - **Container runtime** (e.g. containerd): the software that actually pulls
    images and runs containers.

On top of this, Kubernetes gives us:
- **Deployments** to declare "I want N replicas of this pod" and scale them.
- **Services** for stable networking and load balancing across pods.
- **CoreDNS** for service discovery by name instead of IP.
- **Self-healing**: if a pod crashes or a node goes down, the controllers
  notice and create replacement pods automatically.

### Helm

Helm is often called "the package manager for Kubernetes." Writing raw YAML
files for every Deployment, Service, ConfigMap, etc. by hand gets repetitive
and error-prone, especially when the same app needs to be deployed with
different settings (dev vs. prod). Helm solves this with **charts**: a chart
is a folder of templated YAML files plus a `values.yaml` file of
configuration. Helm fills in the templates with the values and applies the
result to the cluster, and also tracks every install/upgrade as a numbered
**release** so it can be rolled back.

## Procedure

### Step 0: Environment setup

We checked `docker version`, `kubectl version --client`, and `helm version`.
Docker Desktop and `kubectl` were already installed; `helm`, `kind`, and
`minikube` were missing. We installed `kind` and `helm` using Homebrew
(`brew install kind helm`) — no `sudo` was needed for this. We then created a
real local cluster with:

```
kind create cluster --name assignment7
```

and confirmed it was healthy with `kubectl get nodes` and
`kubectl cluster-info`. Full output: `outputs/00_env_check.txt`.

### Step 1: Exploring the real cluster architecture (Part A)

We ran `kubectl get nodes -o wide`, `kubectl get pods -n kube-system -o wide`,
`kubectl cluster-info`, and `kubectl describe node <node>` on the live
cluster and mapped every pod we saw back to the Kubernetes component it
represents. The full diagram and component table are in
`docs/architecture.md`; the raw command output is in
`outputs/01_architecture_commands.txt`.

Note: our `kind` cluster is a single machine acting as both control plane and
worker (common for learning/local setups). All control-plane pods
(`kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager`) and
worker-side pieces (`kube-proxy`, `kindnet` as the CNI, `containerd` as the
runtime) run on that one node.

### Step 2: Creating and customizing the Helm chart (Part B)

We scaffolded a new chart:

```
helm create myapp
```

This generates a standard chart structure:
- **`Chart.yaml`** — metadata about the chart: name, version, description,
  type (`application` or `library`), and the app version it packages.
- **`values.yaml`** — the default configuration values that get substituted
  into the templates (replica count, image, service type, etc.).
- **`charts/`** — where any sub-charts (dependencies) would live. Empty here
  because our chart has no dependencies.
- **`templates/deployment.yaml`** — a templated Kubernetes Deployment
  manifest; Helm fills in `{{ .Values.* }}` placeholders from `values.yaml`.
- **`templates/service.yaml`** — a templated Service manifest that exposes
  the Deployment's pods.
- **`templates/_helpers.tpl`** — reusable template snippets (e.g. for
  generating consistent resource names/labels) that other templates call
  into, so we don't repeat the same logic everywhere.

We then edited:

**`myapp/Chart.yaml`**
```yaml
apiVersion: v2
name: myapp
description: Sample Kubernetes application
type: application
version: 0.1.0
appVersion: "1.0"
```

**`myapp/values.yaml`** (relevant parts)
```yaml
replicaCount: 2

image:
  repository: nginx
  pullPolicy: IfNotPresent
  # Pinned to 1.27 instead of "latest" so every install/upgrade uses the exact
  # same image. "latest" is unpredictable: it can silently change between
  # deployments and makes rollbacks and debugging unreliable.
  tag: "1.27"

service:
  type: NodePort
  port: 80
```

Everything else (service account, probes, resources, etc.) was left at the
generated defaults, which already work correctly.

### Step 3: Helm operations

All commands below were run for real against the `kind` cluster; full output
for each is saved under `outputs/`.

1. `helm version` → `outputs/02_helm_version.txt`
2. `helm lint ./myapp` → `outputs/03_helm_lint.txt`
3. `helm template myapp ./myapp` → saved to `outputs/rendered.yaml`
   (full output also echoed in `outputs/04_helm_template.txt`)
4. `helm install myrelease ./myapp` → `outputs/05_helm_install.txt`
5. `helm list` and `helm status myrelease` → `outputs/06_helm_list_status.txt`
6. `kubectl get pods` / `get deployments` / `get services` →
   `outputs/07_kubectl_verify.txt`
7. App access via `kubectl port-forward` + `curl` → `outputs/08_app_access.txt`
8. `helm upgrade myrelease ./myapp --set replicaCount=3` + `helm history` →
   `outputs/09_helm_upgrade.txt`
9. `helm rollback myrelease 1` → `outputs/10_helm_rollback.txt`
10. `helm uninstall myrelease` + final cleanup check → `outputs/11_helm_uninstall.txt`

## Observations (real captured output)

**`helm lint ./myapp`**
```
==> Linting ./myapp
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```
Only an informational note about a missing icon — not an error. The chart is
valid.

**`helm template` (excerpt of rendered Service)**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  ...
spec:
  type: NodePort
  ports:
    - port: 80
```
This confirms our `values.yaml` change (NodePort, port 80) was correctly
substituted into the template.

**`helm install myrelease ./myapp`**
```
NAME: myrelease
LAST DEPLOYED: Wed Oct 14 18:17:38 2026
NAMESPACE: default
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
```

**Verification after install (`kubectl get pods` / `deployments` / `services`)**
```
NAME                              READY   STATUS    RESTARTS   AGE
myrelease-myapp-fdf85786d-5lcbx   1/1     Running   0          21s
myrelease-myapp-fdf85786d-kszxj   1/1     Running   0          21s

NAME              READY   UP-TO-DATE   AVAILABLE   AGE
myrelease-myapp   2/2     2            2           21s

NAME              TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
myrelease-myapp   NodePort   10.96.239.41   <none>        80:30540/TCP   21s
```
Two pods are running, matching `replicaCount: 2`.

**Accessing the app (`kubectl port-forward` + `curl`)**
```
HTTP/1.1 200 OK
Server: nginx/1.27.5
...
<title>Welcome to nginx!</title>
```
The nginx server responded successfully with the default welcome page,
confirming the pinned `1.27` image was pulled and is serving traffic.

**Upgrade to 3 replicas (`helm upgrade --set replicaCount=3`)**
```
Release "myrelease" has been upgraded. Happy Helming!
REVISION: 2
STATUS: deployed

$ kubectl get pods
myrelease-myapp-fdf85786d-5lcbx   1/1   Running   0   48s
myrelease-myapp-fdf85786d-fwndw   1/1   Running   0   9s
myrelease-myapp-fdf85786d-kszxj   1/1   Running   0   48s
```
A third pod was created automatically — this is the Deployment controller
noticing the desired replica count changed and scheduling a new pod to match
it (self-healing / scaling in action).

**`helm history myrelease`**
```
REVISION  UPDATED                   STATUS      CHART        APP VERSION  DESCRIPTION
1         Wed Oct 14 18:17:38 2026  superseded  myapp-0.1.0  1.0          Install complete
2         Wed Oct 14 18:18:17 2026  deployed    myapp-0.1.0  1.0          Upgrade complete
```

**Rollback to revision 1 (`helm rollback myrelease 1`)**
```
Rollback was a success! Happy Helming!
deployment "myrelease-myapp" successfully rolled out

$ kubectl get pods (after settling)
myrelease-myapp-fdf85786d-5lcbx   1/1   Running   0   70s
myrelease-myapp-fdf85786d-kszxj   1/1   Running   0   70s
```
The pod count went back down to 2, matching revision 1's `replicaCount: 2`.
`helm history` then showed a new revision 3 ("Rollback to 1") — Helm records
a rollback as a new revision rather than deleting history, so we can always
see what happened and when.

**Uninstall and cleanup (`helm uninstall myrelease`)**
```
release "myrelease" uninstalled

$ helm list
NAME  NAMESPACE  REVISION  UPDATED  STATUS  CHART  APP VERSION
(empty)

$ kubectl get all
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   3m17s
```
`helm list` is empty and `kubectl get all` only shows the cluster's built-in
`kubernetes` service — everything we created was cleanly removed.

## Conclusion

We successfully built a real local Kubernetes cluster, identified and
explained every control-plane and worker component by matching them to
actual running pods, and used Helm to package, deploy, verify, scale,
roll back, and clean up a sample nginx application. Every command produced
the expected result, confirming that Kubernetes correctly orchestrates
containers (scheduling, networking, scaling) and that Helm makes repeatable,
versioned application deployment on Kubernetes straightforward.

## Learning Outcome

- Understood the role of each Kubernetes control-plane and worker component
  by observing them directly on a running cluster, instead of only reading
  about them.
- Learned how Kubernetes Deployments achieve scaling and self-healing by
  watching pod counts change in real time during `helm upgrade` and
  `helm rollback`.
- Learned how a NodePort Service and `kube-proxy` make an application
  reachable, and verified it by actually receiving an HTTP response from
  nginx.
- Learned the practical Helm workflow — chart creation, linting, templating,
  install, upgrade, history, rollback, and uninstall — and understood why
  Helm tracks every change as a numbered release.
- Learned why pinning image tags (e.g. `nginx:1.27` instead of `latest`)
  matters for predictable, reproducible deployments.
