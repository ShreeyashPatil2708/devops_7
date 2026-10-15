# Kubernetes Architecture (observed on our live cluster)

This cluster was created with `kind` (Kubernetes in Docker) and has **one node** that
plays both control-plane and worker roles. In a real production cluster these would
usually be separate machines, but the same components exist either way. All the
pod names and process behavior below are taken directly from our running cluster
(see `outputs/01_architecture_commands.txt` for the raw command output).

## ASCII diagram

```
                        ┌─────────────────────────────────────────┐
                        │              CONTROL PLANE               │
                        │     (node: assignment7-control-plane)    │
                        │                                           │
  kubectl  ───HTTPS───▶ │  ┌───────────────┐                       │
  (our CLI commands)    │  │  kube-apiserver│◀────────┐             │
                        │  └───────┬───────┘          │             │
                        │          │ reads/writes      │             │
                        │          ▼                   │             │
                        │  ┌───────────────┐           │             │
                        │  │     etcd       │           │  watches   │
                        │  │ (key-value DB) │           │  API       │
                        │  └───────────────┘           │             │
                        │          ▲                   │             │
                        │          │ watch              │             │
                        │  ┌───────┴───────┐   ┌────────┴────────┐   │
                        │  │ kube-scheduler│   │ kube-controller- │   │
                        │  │ (picks node   │   │ manager          │   │
                        │  │  for new pods)│   │ (Deployment,     │   │
                        │  └───────────────┘   │  ReplicaSet,     │   │
                        │                        │  Node controllers)│   │
                        │                        └──────────────────┘   │
                        └─────────────────────────────────────────┘
                                          │
                                          │ API calls over the network
                                          ▼
                        ┌─────────────────────────────────────────┐
                        │                WORKER NODE                │
                        │   (same physical node in our kind setup) │
                        │                                           │
                        │  ┌───────────────┐   ┌─────────────────┐ │
                        │  │    kubelet     │   │   kube-proxy     │ │
                        │  │ (talks to API, │   │ (sets up Service │ │
                        │  │ starts/stops   │   │  networking /    │ │
                        │  │ containers)    │   │  load-balancing) │ │
                        │  └───────┬───────┘   └─────────────────┘ │
                        │          │ CRI                            │
                        │          ▼                                │
                        │  ┌───────────────┐                        │
                        │  │  containerd    │  ◀── container runtime│
                        │  │ (runs actual   │                        │
                        │  │  containers)   │                        │
                        │  └───────┬───────┘                        │
                        │          ▼                                │
                        │  ┌───────────────┐  ┌───────────────┐     │
                        │  │  CoreDNS pod   │  │  App pods      │     │
                        │  │ (service       │  │ (e.g. our nginx│     │
                        │  │  discovery)    │  │  Helm release)  │     │
                        │  └───────────────┘  └───────────────┘     │
                        └─────────────────────────────────────────┘
```

## Component table (mapped to what we actually saw)

| Component | Role | Evidence on our cluster |
|---|---|---|
| **kube-apiserver** | Front door of Kubernetes. Every `kubectl` command and every other component talks to it over REST/HTTPS. Validates and stores requests. | Pod `kube-apiserver-assignment7-control-plane` in `kube-system`, `Running`. `kubectl cluster-info` shows the control plane endpoint `https://127.0.0.1:<port>`. |
| **etcd** | Distributed key-value store. Holds the entire cluster state (all objects: pods, deployments, services, secrets, etc.). Only the API server talks to it directly. | Pod `etcd-assignment7-control-plane` in `kube-system`, `Running`. |
| **kube-scheduler** | Watches for newly created pods with no node assigned and decides which node they should run on, based on resources and constraints. | Pod `kube-scheduler-assignment7-control-plane` in `kube-system`, `Running`. |
| **kube-controller-manager** | Runs the control loops that keep the cluster in the desired state: Deployment controller, ReplicaSet controller, Node controller, etc. (e.g. it is what notices a pod died and creates a replacement). | Pod `kube-controller-manager-assignment7-control-plane` in `kube-system`, `Running`. We'll see it act in Step 3 when Helm's Deployment keeps the right number of nginx pods running. |
| **kubelet** | Agent running on every node. Talks to the API server, receives pod specs, and tells the container runtime to start/stop containers accordingly. Also reports node/pod health back to the API server. | Not a separate pod (it's a host process on the node, visible via node `Conditions` in `kubectl describe node`: `KubeletReady`, `KubeletHasSufficientMemory`, etc.) |
| **kube-proxy** | Implements Kubernetes **Services**: sets up network rules (iptables/IPVS) on each node so traffic to a Service's virtual IP gets load-balanced across the right pods. | Pod `kube-proxy-w4zhj` in `kube-system`, `Running`. This is what makes our `NodePort` Service in Step 3 work. |
| **Container runtime (containerd)** | Actually pulls images and runs/stops containers, as instructed by the kubelet via the Container Runtime Interface (CRI). | `kubectl describe node` shows `Container Runtime Version: containerd://2.3.4`. |
| **CoreDNS** | Cluster add-on (not core control plane, but essential) providing internal DNS/service discovery, so pods can reach each other by Service name instead of IP. | Pods `coredns-559f6c778d-*` in `kube-system`, `Running`. |
| **kindnet** | The CNI (networking) plugin kind uses to give every pod its own IP and let pods reach each other across nodes. | Pod `kindnet-r6s9z` in `kube-system`, `Running`. |

## How this maps to "container orchestration" concepts

- **Deployment / scaling**: A Deployment object (which the controller-manager watches) declares how many replicas of a pod should run. Changing the replica count (as we do in Step 3 with `helm upgrade --set replicaCount=3`) causes the Deployment controller to create or delete pods until the actual count matches the desired count.
- **Networking**: Every pod gets its own IP from the CNI plugin (kindnet here). Pods on the same cluster can reach each other directly by IP; kube-proxy and CoreDNS build Service-level networking on top of that.
- **Service discovery**: CoreDNS resolves Service names (e.g. `myapp.default.svc.cluster.local`) to a stable virtual IP, so pods don't need to know the real IPs of other pods, which can change.
- **Load balancing**: A Kubernetes Service (ClusterIP/NodePort/LoadBalancer) spreads traffic across all healthy pods behind it. kube-proxy programs the actual packet-forwarding rules that do this.
- **Self-healing**: The kubelet continuously reports pod health to the API server. If a pod crashes, the kubelet restarts the container; if a whole pod disappears, the Deployment/ReplicaSet controller (in kube-controller-manager) notices the replica count is below desired and schedules a new pod via kube-scheduler.

Raw command output backing this document: `outputs/01_architecture_commands.txt`.
