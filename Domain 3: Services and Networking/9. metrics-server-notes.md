# Kubernetes Metrics Server — Domain 3: Services and Networking

---

## 1. What is the Metrics Server?

> The **Metrics Server** is a cluster add-on that collects **CPU and memory usage data** from the **kubelet on every node** and makes that data available through the **Kubernetes Metrics API**.

By default, **Kubernetes does not expose resource usage metrics** — there is no built-in way to ask "how much CPU is this pod using?" or "how much memory is this node consuming?". The Metrics Server fills this gap.

```
         ┌─────────────────────────────────────────────────┐
         │                Kubernetes Cluster                │
         │                                                  │
         │   Node 1            Node 2            Node 3     │
         │   kubelet ──┐       kubelet ──┐       kubelet ──┐│
         │             │                 │                 ││
         │             ▼                 ▼                 ▼│
         │         ┌─────────────────────────────────────┐ │
         │         │         METRICS SERVER              │ │
         │         └─────────────────────────────────────┘ │
         │                       │                          │
         │                       ▼                          │
         │              Kubernetes Metrics API              │
         │                       │                          │
         │                       ▼                          │
         │      kubectl top pods    kubectl top nodes       │
         └─────────────────────────────────────────────────┘
```

---

## 2. Why Metrics Server — What Challenges Does It Solve?

### Challenge 1: No Built-in Resource Usage View

Kubernetes itself does not show CPU or memory usage of pods or nodes out of the box. If you try `kubectl top pods` on a cluster without Metrics Server, you get:

```
error: Metrics API not available
```

**Metrics Server solves this** by gathering CPU/memory data from each node's kubelet and serving it through the Metrics API — making `kubectl top pods` and `kubectl top nodes` work.

---

### Challenge 2: Hard to Troubleshoot Performance Issues

In production, when something is slow or crashing, you need to know **which pod is using too much CPU**, **which node is running out of memory**, or **which workload is causing pressure**. Without metrics, you are guessing.

**Metrics Server solves this** by giving operators a quick, real-time view of resource usage across the cluster — useful for spotting noisy neighbors and overloaded nodes.

---

### Challenge 3: Autoscalers Need Metrics

Features like the **Horizontal Pod Autoscaler (HPA)** rely on CPU/memory metrics to decide when to scale pods up or down. Without Metrics Server, HPA cannot work.

**Metrics Server solves this** by providing the metrics that autoscalers query.

---

## 3. Two Key Commands Provided by Metrics Server

| Command | What it shows |
|---|---|
| `kubectl top pods` | CPU and memory usage of all running pods (in current or specified namespace) |
| `kubectl top nodes` | CPU and memory usage of every node in the cluster (with absolute and percentage values) |

> Both commands work **only after** the Metrics Server is installed and its pod is `Running` and `Ready`.

---

## 4. Important Points to Remember

- Metrics Server is **not installed by default** — you must install it manually.
- After installation, the Metrics Server pod runs in the **`kube-system`** namespace.
- It can take a **minute or two** after installation before the pod is ready and metrics start showing up.
- Without Metrics Server, `kubectl top` commands return `Metrics API not available`.
- By default, `kubectl top pods` shows pods in the **default** namespace only — use `-A` (or `--all-namespaces`) to see all namespaces.

---

## 5. Syntax

### Install the Metrics Server (Official Manifest)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Verify the Metrics Server Pod is Running

```bash
kubectl get pods -n kube-system | grep metrics-server
```

### View Pod Resource Usage

```bash
# Pods in the default namespace
kubectl top pods

# Pods in a specific namespace
kubectl top pods -n <namespace-name>

# Pods across all namespaces
kubectl top pods -A
# or
kubectl top pods --all-namespaces
```

### View Node Resource Usage

```bash
kubectl top nodes
```

---

## 6. Examples

### Example 1: Before Metrics Server is Installed

```bash
kubectl top pods
# error: Metrics API not available

kubectl top nodes
# error: Metrics API not available
```

This confirms why you need Metrics Server.

---

### Example 2: Install the Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Wait about 1–2 minutes, then check the pod:

```bash
kubectl get pods -n kube-system
# NAME                              READY   STATUS    AGE
# metrics-server-74bb687b8-8mpbn    1/1     Running   1m
```

---

### Example 3: View CPU and Memory for Pods

```bash
kubectl top pods
# NAME    CPU(cores)   MEMORY(bytes)
# curl    0m           0Mi
# nginx   0m           2Mi
```

For every namespace:

```bash
kubectl top pods -A
# NAMESPACE     NAME                              CPU(cores)   MEMORY(bytes)
# default       curl                              0m           0Mi
# default       nginx                             0m           2Mi
# kube-system   coredns-86bbcb7bcf-88kqj          2m           16Mi
# kube-system   metrics-server-74bb687b8-8mpbn    3m           18Mi
```

---

### Example 4: View Node Resource Usage

```bash
kubectl top nodes
# NAME                 CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
# kind-control-plane   123m         12%      604Mi           30%
# kind-worker          45m          4%       268Mi           13%
# kind-worker2         41m          4%       211Mi           10%
```

This shows both absolute usage and percentage of capacity, for every node.

---

## 7. Exercises to Practice for the CKAD Exam

### Imperative (CLI-based) Exercises

**Exercise 1 — Confirm Metrics Are Missing by Default**
1. On a fresh cluster, run `kubectl top pods` and `kubectl top nodes`.
2. Verify both return the `Metrics API not available` error.

**Exercise 2 — Install the Metrics Server**
1. Install the Metrics Server using the official `components.yaml`.
2. Watch the Metrics Server pod until it becomes `Running` and `Ready` in the `kube-system` namespace.

**Exercise 3 — Pod-Level Metrics**
1. Run `kubectl top pods` for the default namespace.
2. Run `kubectl top pods -A` and identify the **top 3 pods** consuming the most memory.

**Exercise 4 — Namespace-Specific Metrics**
1. Run `kubectl top pods -n kube-system`.
2. Identify which kube-system component is consuming the most CPU.

**Exercise 5 — Node-Level Metrics**
1. Run `kubectl top nodes`.
2. Identify which node has the **highest memory utilization percentage**.
3. Save the output to a file using `> nodes.txt`.

---

### Declarative (YAML-based) Exercises

**Exercise 6 — Verify the Metrics Server Pod via Manifest**
1. After installing Metrics Server, run `kubectl get deploy metrics-server -n kube-system -o yaml`.
2. Read the manifest and identify:
   - The container image used.
   - The arguments passed (e.g., `--kubelet-insecure-tls`, `--metric-resolution`).

**Exercise 7 — Generate a Test Workload and Observe Metrics**
1. Write a Pod manifest for a pod called `cpu-burner` running the `busybox` image with the command:
   ```yaml
   command: ["sh", "-c", "while true; do :; done"]
   ```
2. Apply it.
3. Run `kubectl top pods` and observe the CPU usage spike for that pod.

**Exercise 8 — Multi-Pod Manifest**
1. Write a single YAML file with three pods (`nginx-1`, `nginx-2`, `nginx-3`), all running `nginx`.
2. Apply the file.
3. Run `kubectl top pods` and confirm all three appear with their CPU and memory usage.

**Exercise 9 — Cleanup the Metrics Server**
1. Delete the Metrics Server using the same `components.yaml`:
   ```bash
   kubectl delete -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
   ```
2. Confirm `kubectl top pods` again returns `Metrics API not available`.

---

## 8. Summary

| Concept | Detail |
|---|---|
| **What it is** | A cluster add-on that collects CPU and memory usage from kubelets and exposes it via the Metrics API |
| **Why it's needed** | Kubernetes does not provide resource usage data out of the box |
| **Where it runs** | As a pod in the **`kube-system`** namespace |
| **Data source** | The **kubelet** on each node |
| **Exposes** | The Kubernetes **Metrics API** |
| **Enables** | `kubectl top pods`, `kubectl top nodes`, and the **Horizontal Pod Autoscaler (HPA)** |
| **Pod metrics command** | `kubectl top pods` (use `-A` for all namespaces) |
| **Node metrics command** | `kubectl top nodes` |
| **Default behaviour** | Without Metrics Server, `kubectl top` returns `Metrics API not available` |
| **Install method** | `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml` |
| **Readiness wait** | 1–2 minutes for the pod to become `Ready` |
