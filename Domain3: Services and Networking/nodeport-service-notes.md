# NodePort Service — Domain 3: Services and Networking

---

## 1. Recap: Types of Services

| Service Type | Key Features | Use Case |
|---|---|---|
| **ClusterIP** | Default type. Accessible only within the cluster. | Internal microservices communication |
| **NodePort** | Exposes service on a static port (30000–32767) on each Node. | Development testing, demo applications |
| **LoadBalancer** | Exposes service externally using the cloud provider's load balancer. | Production apps requiring external access |
| **ExternalName** | Maps service to an external DNS name. | External service integration |

This video focuses on **NodePort**.

---

## 2. What is NodePort?

> A **NodePort** is one of the service types used to **expose your application to the external world** — including the internet.

- When you create a NodePort service, Kubernetes opens a specific port on **every worker node** in the cluster.
- Any traffic sent to `<NodeIP>:<NodePort>` is automatically forwarded to the service, which then routes it to the backend pods.
- External users — including users on the internet — can reach your application through this port.

```
External User ──► Worker Node IP : NodePort ──► SERVICE ──► backend-1
                                                       \──► backend-2
```

---

## 3. Why NodePort — What Challenges Does It Solve?

### Challenge 1: ClusterIP Is Only Accessible Inside the Cluster

ClusterIP gives a stable internal IP, but that IP is unreachable from outside the cluster. If you have a web application that users on the internet need to access, ClusterIP alone cannot help.

```
External User  ✗ ──► ClusterIP Service   (blocked — internal IP only)
```

**NodePort solves this** by opening a port on the worker node itself — a node is a real machine with a public IP, so external traffic can now reach your pods through it.

---

### Challenge 2: No Simple Way to Expose Pods to the Internet Without a Cloud Load Balancer

In environments where you don't have a cloud provider (e.g., on-premise, bare-metal, local clusters), you can't use a LoadBalancer service. You need a straightforward way to let external traffic in.

**NodePort solves this** by using the node's own IP and a port — no cloud infrastructure required. This makes it ideal for development, testing, and demo environments where simplicity matters more than production-grade load balancing.

---

### Challenge 3: Developers Need to Test Applications From Outside the Cluster

During development you often need to hit your application from your local browser or terminal — not from inside the cluster. Without NodePort, you'd need workarounds like `kubectl port-forward` every time.

**NodePort solves this** by giving a persistent, stable port on the node that you can bookmark and reuse across sessions.

---

## 4. How NodePort Works — The Mechanism

> When you create a service of type NodePort, Kubernetes assigns a port from the **NodePort range (default: 30000–32767)** on **all nodes** in the cluster.

Any traffic sent to `<NodeIP>:<NodePort>` is forwarded to the corresponding service.

```
                           Worker Node
                    ┌────────────────────────┐
                    │                        │
External User ─────►│  Port: 30154           │
                    │       │                │
                    │       ▼                │
                    │    SERVICE             │
                    │    /      \            │
                    │  Pod-1   Pod-2         │
                    └────────────────────────┘
```

**Flow step by step:**
1. External user makes a request to `<Worker Node Public IP>:<NodePort>` (e.g., `203.0.113.10:30537`)
2. The request hits the worker node on that port
3. The worker node forwards it to the NodePort Service
4. The Service load balances it across the matching backend pods

**Reference — `kubectl get service` output:**

```
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
kubernetes      ClusterIP   10.109.0.1      <none>        443/TCP         22d
test-nodeport   NodePort    10.109.4.218    <none>        80:30537/TCP    10s
```

Reading the `PORT(S)` column for NodePort → `80:30537/TCP`:
- `80` — the port the **Service** listens on internally (port)
- `30537` — the **NodePort** opened on every worker node (accessible from outside)

---

## 5. Point to Note — Part 1: NodePort Service Also Has a ClusterIP

> A NodePort service is **not external-only**. It also gets a ClusterIP for internal communication.

This means a NodePort service serves **both** internal and external traffic:

| Access From | How to Connect |
|---|---|
| **External user / internet** | `<Worker Node Public IP>:<NodePort>` e.g., `203.0.113.10:30537` |
| **Internal pod (same cluster)** | `<ClusterIP>:<port>` e.g., `10.109.4.218:80` |

So pods inside the cluster do not need to use the node IP — they can still talk to the service via its ClusterIP, exactly like they would with a ClusterIP-only service.

```
External User ──► NodeIP:30537    ──► NodePort Service ──► backend-pod
Internal Pod  ──► ClusterIP:80    ──► NodePort Service ──► backend-pod
```

---

## 6. Point to Note — Part 2: The NodePort Range (30000–32767)

- Kubernetes automatically picks an available port from the range **30000 to 32767** when you don't specify one.
- This port is opened on **every worker node** — not just one.
- If you delete and recreate the service without specifying a port, Kubernetes assigns a **different port** from the range each time.

```bash
# First creation — assigned 31850
kubectl get service
# NAME               PORT(S)
# nodeport-service   80:31850/TCP

kubectl delete service nodeport-service
kubectl create -f service.yaml

# Second creation — assigned a different port automatically
kubectl get service
# NAME               PORT(S)
# nodeport-service   80:30558/TCP
```

---

## 7. Point to Note — Part 3: You Can Manually Specify the NodePort

If you need a **fixed, predictable port** (e.g., you've already configured a firewall rule or shared the URL with someone), you can hardcode the `nodePort` in the manifest.

```yaml
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30556    # manually specified — must be within 30000–32767
```

With this, regardless of how many times you delete and recreate the service, it will always use port `30556`.

---

## 8. Hands-on Lab — Creating a NodePort Service

### Step 1: Create the NodePort Service Manifest

Save as `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodeport-service
spec:
  selector:
    app: backend       # routes traffic to pods with this label
  type: NodePort       # this is what makes it a NodePort service
  ports:
  - port: 80           # service port (internal)
    targetPort: 80     # pod port
```

> Unlike ClusterIP, you must explicitly set `type: NodePort`.

```bash
kubectl create -f service.yaml
kubectl get service
```

You will see the NodePort automatically assigned in the `PORT(S)` column (e.g., `80:30558/TCP`).

### Step 2: Verify NodePort Is Assigned

```bash
kubectl describe service nodeport-service
```

Look for the `NodePort` field — it shows the port that has been opened on all worker nodes. Also note that `Endpoints` will be empty until a matching pod exists.

### Step 3: Create a Backend Pod with the Matching Label

```bash
kubectl run backend-pod --image=nginx

# Add the label that matches the service selector
kubectl label pod backend-pod app=backend

# Verify the endpoint is now populated
kubectl describe service nodeport-service
```

The `Endpoints` field should now show the backend pod's IP.

### Step 4: Get the Worker Node's External IP

```bash
kubectl get nodes -o wide
```

Look for the **EXTERNAL-IP** column — this is the public IP of the worker node. Do **not** use the INTERNAL-IP; that won't be reachable from outside.

### Step 5: Access the Application From Outside the Cluster

```bash
# From your local terminal (outside the cluster):
curl <EXTERNAL-IP>:<NodePort>

# Example:
curl 203.0.113.10:30558
```

You should get the nginx welcome page — confirming external access is working.

You can also open `http://<EXTERNAL-IP>:<NodePort>` in a browser and see the same page.

### Step 6: Verify Internal Access Still Works (via ClusterIP)

```bash
# From inside any pod within the cluster:
kubectl exec -it <any-pod> -- bash
curl <CLUSTER-IP>:80
```

Same nginx page — the NodePort service also handles internal traffic via its ClusterIP.

---

### Method 2: Create NodePort via CLI (Imperative)

You can create a NodePort service without writing a manifest:

```bash
# See help for nodeport subcommand
kubectl create service nodeport --help

# Create the service
kubectl create service nodeport test-nodeport --tcp=80:80

kubectl get service
```

**Generate the manifest from CLI (dry-run):**

```bash
kubectl create service nodeport test-nodeport --tcp=80:80 --dry-run=client -o yaml
```

Sample generated YAML:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: test-nodeport
spec:
  selector:
    app: test-nodeport
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
```

Use `--dry-run=client -o yaml` to get a base manifest you can save and modify — including adding a manual `nodePort` value.

---

### Manually Specifying the NodePort

If you want a fixed port instead of a randomly assigned one:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodeport-service
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30556    # fixed port — always this, every time
```

```bash
kubectl delete service nodeport-service
kubectl create -f service.yaml
kubectl get service
# PORT(S) will always show 80:30556/TCP
```

### Cleanup

```bash
kubectl delete pod backend-pod
kubectl delete -f service.yaml
```

---

## 9. Summary

| Concept | Detail |
|---|---|
| **What it is** | A service type that exposes your application to external traffic including the internet |
| **How it works** | Opens a port on every worker node; traffic to `NodeIP:NodePort` is forwarded to the service |
| **NodePort range** | 30000–32767 (automatically assigned unless manually specified) |
| **Also has ClusterIP** | Yes — NodePort service works for both external (NodeIP:NodePort) and internal (ClusterIP:port) traffic |
| **EXTERNAL-IP column** | Shows `<none>` — the external access is via the Node IP, not a dedicated external IP |
| **Port column format** | `80:30537/TCP` → `80` is service port, `30537` is the NodePort on the node |
| **Manual nodePort** | Add `nodePort: <value>` under `ports` in the manifest to fix the port |
| **Get Node IP** | `kubectl get nodes -o wide` → use EXTERNAL-IP (not INTERNAL-IP) |
| **Created via manifest** | Set `type: NodePort` in service spec |
| **Created via CLI** | `kubectl create service nodeport <name> --tcp=<port>:<targetPort>` |
| **Generate YAML from CLI** | Add `--dry-run=client -o yaml` to the CLI create command |
| **Best used for** | Development, testing, demo environments — not recommended for production |
