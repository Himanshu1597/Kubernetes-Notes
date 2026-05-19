# Kubernetes Architecture — Domain 6: Cluster Architecture, Installation and Configuration

---

## 1. What is Kubernetes Architecture?

> **Kubernetes Architecture** is the **set of components** that together make a Kubernetes cluster work. These components are split into two groups: the ones that **run the cluster** (the **Control Plane**), and the ones that **run your workloads** (the **Worker Nodes**).

In plain English: a Kubernetes cluster has a **"brain"** (the control plane) that makes all the decisions, and **"workers"** (the worker nodes) that actually run your pods and containers.

```
   ┌─────────────────────┐           ┌──────────────────────────────┐
   │                     │           │     Worker Nodes             │
   │   Control Plane     │ ────────► │   ┌────┐  ┌────┐  ┌────┐    │
   │     (the brain)     │           │   │N-1 │  │N-2 │  │N-3 │    │
   │                     │           │   └────┘  └────┘  └────┘    │
   └─────────────────────┘           │   (run pods + containers)    │
                                     └──────────────────────────────┘
```

---

## 2. Why Architecture — What Challenges Does It Solve?

### Challenge 1: Running Containers Manually Does Not Scale

Without Kubernetes, someone has to manually decide which server runs which container, restart failed containers, scale up under load, and patch nodes one by one. With hundreds of containers, this is impossible.

**The architecture solves this** by giving each responsibility to a dedicated component — scheduling, state storage, controllers, networking, container runtime — so the system as a whole can run thousands of pods automatically.

---

### Challenge 2: The Cluster Needs a Single Source of Truth

If pod state, configuration, secrets, and replica counts all live in different places, they will drift and conflict. There needs to be **one authoritative store** of what should exist in the cluster.

**The architecture solves this** with **etcd** — a single distributed key-value store that holds the entire cluster state. Every component reads from and writes to etcd through the API server.

---

### Challenge 3: Decisions Should Be Separate from Execution

Deciding *which* node a pod should run on is a different problem from *actually* running the pod. Mixing the two makes the system harder to scale and harder to secure.

**The architecture solves this** by separating decision-making (Control Plane: scheduler, controller-manager) from execution (Worker Nodes: kubelet, container runtime).

---

## 3. Plain-English Definition

> A Kubernetes cluster is built from **two groups of components**:
>
> - **Control Plane components** (run on the master node) — these make the cluster decisions: receive requests, store state, schedule pods, run controllers.
> - **Worker Node components** (run on every worker) — these execute the decisions: start containers, manage networking, report status back.

Think of it like a kitchen:
- The **Control Plane** is the **head chef** — takes orders, decides what to cook, assigns tasks.
- The **Worker Nodes** are the **line cooks** — actually cook the food based on the chef's instructions.

---

## 4. The Two Groups at a Glance

### Control Plane (Master Node) Components

| Component | Description |
|---|---|
| **kube-apiserver** | The front-end for the Kubernetes control plane. Exposes the Kubernetes API used by other components and users to interact with the cluster. |
| **etcd** | Key-value store used as the backing store for **all cluster data and configuration**. |
| **kube-scheduler** | Selects the best node for each Pod, based on resource requirements, constraints, and other factors. |
| **kube-controller-manager** | Runs the controllers that monitor cluster state and make changes to bring it to the desired state. |
| **cloud-controller-manager** *(optional)* | Integrates with cloud providers to manage cloud-specific resources like load balancers, storage, and VMs. |

### Worker Node Components

| Component | Description |
|---|---|
| **kubelet** | An agent that runs on each worker node. Ensures containers described in PodSpecs are actually running and healthy. |
| **kube-proxy** | Maintains the network rules on each node so traffic to Services reaches the right Pods. |
| **Container Runtime** | The software that actually runs containers — e.g., `containerd`, `CRI-O`. |

> All Kubernetes components are **just binaries** that you can download from the Kubernetes releases. When they are running, they form the cluster.

---

## 5. Control Plane — Components Explained

### 5.1 kube-apiserver — The Front Door

> The **API server** is the front-end for the Kubernetes control plane. It exposes the Kubernetes API. **Everything** that wants to talk to the cluster — `kubectl`, controllers, kubelet, scheduler, even etcd queries — goes through the API server.

```
   kubectl  ──►  ┌───────────────┐
   kubelet  ──►  │   API Server   │  ◄────► etcd
   scheduler──►  └───────────────┘
   controllers►
```

- It is the **single entry point** into the cluster.
- It does **authentication** and **authorization** on every request.
- It writes to / reads from **etcd** on behalf of every component.
- If the API server is down, the cluster is unreachable.

> When you run `kubectl create pod`, the request goes to the API server first. The API server validates it, writes it to etcd, and other components then react.

---

### 5.2 etcd — The Source of Truth

> **etcd** is a distributed key-value store that holds **all** cluster data. It is essentially the **database for your Kubernetes cluster**.

```
   ┌────────────┐ ◄─────► ┌──────────────────┐
   │ API Server  │         │       etcd        │
   └────────────┘         │  (cluster state)   │
                          └──────────────────┘
```

Everything you create in Kubernetes — Pods, Deployments, Services, ConfigMaps, Secrets, Nodes, RBAC rules — is stored in etcd.

**Critical rule**: always **back up etcd**. If etcd is lost, the entire cluster's state is lost.

---

### 5.3 kube-scheduler — The Placement Decision-Maker

> The **scheduler** watches for newly created Pods that have **no node assigned**, and picks the best node for each one.

```
   New Pod (unassigned) ──► Scheduler ──► picks best node ──► assigns
                                                                  │
                                                                  ▼
                                                          kubelet on that node
                                                          starts the containers
```

To choose a node the scheduler looks at:
- CPU and memory availability
- Taints and tolerations
- Node affinity / anti-affinity
- Resource requests and limits
- Other constraints (e.g., topology)

> The scheduler only **decides**. The actual starting of the container is done by kubelet on the chosen node.

---

### 5.4 kube-controller-manager — The State Reconciler

> The **controller-manager** is a daemon that runs many small **controllers**. Each controller watches the cluster state and **takes action when the actual state does not match the desired state**.

```
   Desired state (etcd):   3 replicas
   Actual state:           2 replicas (one pod crashed)
        │
        ▼
   Replication Controller ──► create 1 new pod
        │
        ▼
   Now actual = desired = 3 replicas
```

Examples of controllers inside the controller-manager:

| Controller | Functionality |
|---|---|
| **Node Controller** | Monitors and manages node status. |
| **Replication Controller** | Ensures the correct number of pod replicas for a ReplicationController or ReplicaSet. |
| **Deployment Controller** | Manages Deployments — creates/updates ReplicaSets, handles rollouts and rollbacks. |
| **Job Controller** | Watches Jobs and creates pods to run them to completion. |
| **Namespace Controller** | Handles namespace deletion — removes resources inside a deleted namespace. |
| **Endpoint Controller** | Populates Endpoints objects (linking Services to Pods). |
| **Service Account Controller** | Creates default ServiceAccounts in new namespaces. |
| **Persistent Volume Controller** | Manages binding and reclaiming of persistent volumes. |

> Every controller follows the same loop: **observe → compare → act**.

---

### 5.5 cloud-controller-manager — Cloud Integration (Optional)

> The **cloud-controller-manager** lets Kubernetes talk to a cloud provider's APIs. It is only present when running on a supported cloud (AWS, GCP, Azure, DigitalOcean, etc.).

Examples of what it handles:
- Creating cloud **Load Balancers** when a Service of type `LoadBalancer` is created.
- Managing cloud **storage volumes** for PVCs.
- Telling the cluster when a **VM instance** has been removed by the cloud provider.

On-premises clusters typically do not need this component.

---

## 6. Worker Node — Components Explained

### 6.1 kubelet — The Node Agent

> **kubelet** is the **primary node agent**. It runs on every worker node and makes sure that the containers described in the PodSpecs are actually running and healthy.

```
   ┌─────────────────┐         ┌───────────── Worker Node ─────────┐
   │  Control Plane   │         │                                    │
   │                  │         │      ┌────────┐                    │
   │  API Server      │ ──────►│      │ kubelet │ ───► containerd ───►─ container
   │  (PodSpecs)      │         │      └────────┘                    │
   └─────────────────┘         │                                    │
                               └────────────────────────────────────┘
```

What kubelet does:
- Receives PodSpecs from the API server.
- Asks the **container runtime** (containerd, CRI-O) to actually start the containers.
- Monitors the health of pods and containers.
- Reports node and pod status back to the API server.
- Takes corrective action if something fails.

> kubelet does **not** create containers by itself — it tells the container runtime to do it via the **Container Runtime Interface (CRI)**.

---

### 6.2 kube-proxy — The Networking Component

> **kube-proxy** runs on every worker node and is responsible for **networking** — specifically, making **Service** traffic reach the right Pods.

```
   ┌─── Node 1 ───┐                          ┌─── Node 2 ───┐
   │              │   Service: my-svc        │              │
   │   Pod A      │ ── ClusterIP ──►         │   Pod B      │
   │   (client)   │                          │   (server)   │
   │              │   kube-proxy intercepts  │              │
   │              │   and load-balances      │              │
   └──────────────┘                          └──────────────┘
```

What kube-proxy does:
- Watches Services and Endpoints from the API server.
- Sets up **network rules** (using iptables, IPVS, or eBPF) so that a request to a Service's ClusterIP is forwarded to one of its backing Pods.
- Performs **load balancing** across the matching pods.
- Enables Pod-to-Service communication anywhere in the cluster.

---

### 6.3 Container Runtime — Actually Runs the Containers

> The **container runtime** is the software that does the real work of starting and stopping containers. Examples: **containerd**, **CRI-O**.

kubelet talks to the container runtime through the **CRI (Container Runtime Interface)**. kubelet does **not** run the container itself — it asks the runtime to do it.

---

## 7. How a Pod Gets Created — End-to-End Flow

This ties all components together. When you run `kubectl run nginx --image=nginx`:

```
1. kubectl                 ──►  API Server     (request: create Pod)
2. API Server              ──►  etcd           (saves the Pod object)
3. API Server              ◄──  Scheduler      (sees an unscheduled Pod, picks a node)
4. Scheduler               ──►  API Server     (updates Pod with assigned node)
5. API Server              ──►  etcd           (saves the updated Pod)
6. kubelet on that node    ◄──  API Server     (sees a Pod assigned to it)
7. kubelet                 ──►  Container Runtime (starts the container)
8. kube-proxy              ◄──  API Server     (sees new Service endpoints if any)
9. kubelet                 ──►  API Server     (reports Pod status)
10. API Server             ──►  etcd           (saves status)
```

Every component talks to the API server. The API server talks to etcd. That is the core pattern.

---

## 8. Syntax — Inspecting the Architecture

These commands let you see and interact with the components in a real cluster.

### See Nodes (Control Plane + Worker)

```bash
kubectl get nodes
kubectl get nodes -o wide                        # more details
kubectl describe node <node-name>
```

### Look at Control Plane Pods (in kubeadm-style clusters)

Control plane components run as pods in the `kube-system` namespace:

```bash
kubectl get pods -n kube-system

# Typical output:
# kube-apiserver-master           1/1   Running
# etcd-master                     1/1   Running
# kube-scheduler-master           1/1   Running
# kube-controller-manager-master  1/1   Running
# kube-proxy-xxxxx                1/1   Running   (one per node)
# coredns-xxxxx                   1/1   Running
```

### See API Server Endpoint

```bash
kubectl cluster-info
```

### Look at kubelet on a Node (SSH into the node)

```bash
systemctl status kubelet
journalctl -u kubelet -f
```

### Look at the Container Runtime

```bash
# On the node
crictl ps                 # list containers via CRI
crictl info               # runtime info
```

### Static Pod Manifests for Control Plane (kubeadm)

Control plane components are typically defined as **static pod manifests**:

```bash
ls /etc/kubernetes/manifests/
# kube-apiserver.yaml
# kube-controller-manager.yaml
# kube-scheduler.yaml
# etcd.yaml
```

> Editing one of these files causes kubelet to recreate that component pod automatically.

---

## 9. Examples

### Example 1 — See All Cluster Components

```bash
kubectl get pods -n kube-system -o wide
```

You will see exactly one of each control plane pod on the control plane node, and `kube-proxy` as a DaemonSet on every node (one per node).

---

### Example 2 — Trace a Pod from Creation to Running

```bash
# Create a pod
kubectl run demo --image=nginx

# Watch it appear in etcd via the API
kubectl get pod demo -w

# See which node the scheduler assigned it to
kubectl get pod demo -o wide

# Describe to see the events from each component
kubectl describe pod demo
```

In the `Events` section you will see:
- `Scheduled` → scheduler picked a node.
- `Pulling` / `Pulled` → kubelet asked the container runtime to pull the image.
- `Created` → container runtime created the container.
- `Started` → container is running.

This is **all components working together**.

---

### Example 3 — Inspect etcd Data Indirectly

You cannot read etcd directly through `kubectl`, but everything in etcd is exposed through the API server:

```bash
kubectl get all --all-namespaces           # all objects in etcd
kubectl get pod demo -o yaml               # the exact object as stored
```

---

### Example 4 — Watch the Controller Manager React

```bash
# Create a Deployment with 3 replicas
kubectl create deployment web --image=nginx --replicas=3

# Delete one pod
kubectl delete pod -l app=web --field-selector status.phase=Running --no-headers \
  | head -1 | awk '{print $1}' | xargs kubectl delete pod

# Watch the controller-manager bring it back automatically
kubectl get pods -l app=web -w
```

You will see a new pod created automatically — that is the **Deployment Controller** inside the controller-manager reconciling state.

---

### Example 5 — Watch kube-proxy at Work

```bash
# Create a service
kubectl expose deployment web --port=80

# From inside any pod, hit the ClusterIP
kubectl run tmp --rm -it --image=busybox -- sh
wget -O- web                # kube-proxy routes this to a backing pod
```

---

## 10. Exercises to Practice for the CKAD Exam

### Imperative (CLI-based) Exercises

**Exercise 1 — List Cluster Nodes**
1. Run `kubectl get nodes -o wide`.
2. Identify which node is the control plane (look at the `ROLES` column).
3. Note the Kubernetes version on each node.

**Exercise 2 — Find Control Plane Pods**
1. Run `kubectl get pods -n kube-system`.
2. Identify the pods for each control plane component: `kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager`.
3. Run `kubectl describe pod <name> -n kube-system` on the API server pod and identify the container image used.

**Exercise 3 — Find kube-proxy on Every Node**
1. Run `kubectl get pods -n kube-system -o wide -l k8s-app=kube-proxy`.
2. Confirm there is one kube-proxy pod per node.

**Exercise 4 — Trace a Pod Through the Lifecycle**
1. Run `kubectl run nginx --image=nginx`.
2. Immediately run `kubectl get events --sort-by=.lastTimestamp` and identify events caused by each component (scheduler, kubelet, container runtime).

**Exercise 5 — Inspect the API Server Endpoint**
1. Run `kubectl cluster-info`.
2. Identify the URL of the control plane (Kubernetes API server endpoint).

**Exercise 6 — Watch Controller Reconciliation**
1. Create a Deployment with 3 replicas.
2. Delete one of its pods.
3. Watch `kubectl get pods -w` and confirm a new pod is created automatically.

---

### Declarative (YAML-based) Exercises

**Exercise 7 — Read the kube-apiserver Static Pod Manifest**
1. On a kubeadm cluster, look at `/etc/kubernetes/manifests/kube-apiserver.yaml`.
2. Identify the `--etcd-servers` flag and the `--authorization-mode` flag.
3. Identify the certificate file paths used.

**Exercise 8 — Read the etcd Static Pod Manifest**
1. Look at `/etc/kubernetes/manifests/etcd.yaml`.
2. Identify the `--data-dir` flag — this is where etcd stores cluster data.

**Exercise 9 — Read the kube-scheduler Manifest**
1. Look at `/etc/kubernetes/manifests/kube-scheduler.yaml`.
2. Confirm it is configured to use the API server's secure port.

**Exercise 10 — Identify Which Component Does What**
For each scenario, predict which component is responsible:
- A new Pod is sitting unscheduled. → **scheduler**
- A pod's status is being saved to disk so it survives restart. → **etcd** (via the API server)
- A request comes in from `kubectl`. → **API server**
- A failed Pod gets replaced automatically. → **controller-manager**
- A request to a Service's ClusterIP reaches a Pod. → **kube-proxy**
- A container is actually launched on a worker node. → **container runtime** (instructed by kubelet)
- `kubelet` sends a status update for a Pod. → **kubelet → API server → etcd**

---

## 11. Summary

| Concept | Detail |
|---|---|
| **Two groups** | Control Plane (the brain) and Worker Nodes (the workers) |
| **kube-apiserver** | The single entry point; all components talk to it |
| **etcd** | The cluster's database — single source of truth |
| **kube-scheduler** | Picks the best node for unscheduled Pods |
| **kube-controller-manager** | Runs many controllers that reconcile actual state with desired state |
| **cloud-controller-manager** | Optional; integrates with cloud providers (LB, storage, instances) |
| **kubelet** | Per-node agent; talks to container runtime via CRI; starts and monitors containers |
| **kube-proxy** | Per-node networking; routes Service traffic to backing Pods |
| **Container Runtime** | The software that actually runs containers (containerd, CRI-O) |
| **Communication pattern** | Every component talks **through the API server**, never directly to etcd |
| **etcd backup** | Critical — losing etcd loses the entire cluster state |
| **Form** | Each component is a binary; on kubeadm clusters they run as static Pods in `kube-system` |
