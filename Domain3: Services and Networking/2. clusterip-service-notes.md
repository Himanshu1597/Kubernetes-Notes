# ClusterIP Service — Domain 3: Services and Networking

---

## 1. Recap: Types of Services

Before diving in, recall the four service types available in Kubernetes:

| Service Type | Key Features | Use Case |
|---|---|---|
| **ClusterIP** | Default type. Accessible only within the cluster. | Internal microservices communication |
| **NodePort** | Exposes service on a static port (30000–32767) on each Node. | Development testing, demo applications |
| **LoadBalancer** | Exposes service externally using the cloud provider's load balancer. | Production apps requiring external access |
| **ExternalName** | Maps service to an external DNS name. | External service integration |

This video focuses on the first type — **ClusterIP**.

---

## 2. What is ClusterIP?

> A Kubernetes Service of type **ClusterIP** provides an internal, stable IP address to expose your application **only within the Kubernetes cluster**.

- The IP address assigned to a ClusterIP service is an **internal IP** — it is only reachable from inside the cluster.
- Any pod within the same cluster can connect to the service using this internal IP.
- **External users cannot access a ClusterIP service** — it is strictly cluster-internal.

```
                        ┌─────────────────────────────────┐
                        │       Kubernetes Cluster         │
  External User  ✗ ──►  │                                 │
  (no access)           │          pod-1                  │
                        │            │                    │
                        │            ▼                    │
                        │         SERVICE                 │
                        │        (ClusterIP)              │
                        │        /         \              │
                        │   backend-1    backend-2        │
                        └─────────────────────────────────┘
```

This makes ClusterIP ideal for **internal microservices communication** — for example, a frontend pod talking to a backend pod, where neither needs to be exposed to the internet.

---

## 3. Why ClusterIP — What Challenges Does It Solve?

### Challenge 1: Pod IPs Are Ephemeral and Unreliable

Pods get a new IP every time they restart or get rescheduled. If one internal service (e.g., a frontend pod) hardcodes the IP of another internal service (e.g., a backend pod), it breaks as soon as that backend pod restarts.

```
frontend-pod config
───────────────────
backend: 10.244.0.5   ◄── hardcoded pod IP

Backend pod crashes → restarts with IP 10.244.0.9
→ frontend requests now fail immediately
```

**ClusterIP solves this** by giving the backend service a **stable virtual IP** that never changes. The frontend always points to that stable IP, regardless of how many times the backend pods restart or get rescheduled.

---

### Challenge 2: Multiple Replicas — Which Pod IP Do You Call?

When a backend runs as a Deployment with multiple replicas, the frontend would need to know the IP of every single replica and implement its own logic to decide which one to call.

```
frontend-pod config
───────────────────
backend:
  10.244.0.5   ◄── replica 1
  10.244.0.6   ◄── replica 2
  10.244.0.7   ◄── replica 3
  ... (could be hundreds)
```

This is both impractical and fragile — replica IPs also change whenever pods are created or destroyed.

**ClusterIP solves this** by acting as a single stable entry point. The frontend calls one IP (the ClusterIP), and Kubernetes automatically **load balances** the request across all healthy replicas behind it.

```
frontend-pod
     │
     ▼
ClusterIP (stable: 10.109.12.34)
     │
     ├──► backend-replica-1
     ├──► backend-replica-2
     └──► backend-replica-3
```

---

### Challenge 3: Internal Services Should Not Be Exposed to the Outside World

Not every service in a cluster needs to be internet-facing. For example, a database, a cache layer, or an internal API should only be reachable by other pods — never by external users. Exposing them publicly is a security risk.

**ClusterIP solves this** by being accessible **only from within the cluster**. There is no way for an external user to reach it, making it the right choice for any internal-only service.

```
External User  ✗ ──► ClusterIP Service   (blocked — no external IP)
Internal Pod   ✓ ──► ClusterIP Service   (allowed — same cluster)
```

---

### Challenge 4: Tight Coupling Between Internal Services

Without a stable abstraction, every internal service must know the exact pod IPs of every other service it depends on. This creates tight coupling — change one service's pods and everything that calls it breaks.

**ClusterIP solves this** by decoupling the caller from the actual pods. The caller only needs to know the service name (or ClusterIP). The Kubernetes DNS system resolves service names automatically, so internal services can simply call each other by name:

```bash
# Instead of:  curl 10.244.0.5
# You can use: curl simple-service   (Kubernetes DNS resolves this to the ClusterIP)
```

---

## 4. Point to Note — Part 1: ClusterIP is the Default Service Type

> If you create a Service **without specifying a type**, Kubernetes automatically sets the type to **ClusterIP**.

### Example: Service Manifest Without a Type Field

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: simple-service
spec:
  ports:
  - port: 80
    targetPort: 80
```

Notice there is **no `type` field** in the `spec`. When you apply this:

```bash
kubectl create -f service.yaml
kubectl get service
```

Sample output:

```
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
simple-service   ClusterIP   10.109.12.34    <none>        80/TCP    5s
kubernetes       ClusterIP   10.109.0.1      <none>        443/TCP   21d
```

Key observations:
- The type is automatically set to **ClusterIP**.
- The **CLUSTER-IP** column has an internal IP assigned (`10.109.12.34`).
- The **EXTERNAL-IP** column shows `<none>` — confirming no external access.

---

## 5. Point to Note — Part 2: The ClusterIP is a Stable Virtual IP

> The ClusterIP service gets a **virtual IP address** (the cluster IP) that **remains stable as long as the service exists**.

- This is not a dynamic IP — it does not change after a few days or weeks.
- As long as you do not delete and recreate the service, the IP stays the same.
- All pods inside the cluster use this stable IP to reliably reach the service.

```
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kplabs-service   ClusterIP   10.109.12.34    <none>        8080/TCP   6m47s
kubernetes       ClusterIP   10.109.0.1      <none>        443/TCP    21d
```

- `10.109.12.34` is the stable virtual ClusterIP — any pod in the cluster can hit this IP.
- `<none>` under EXTERNAL-IP confirms it is not reachable from outside the cluster.

---

## 6. Hands-on Lab — Creating a ClusterIP Service

There are two ways to create a ClusterIP service: using a **manifest file** or using a **CLI command**.

---

### Method 1: Using a Manifest File

Save as `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: simple-service
spec:
  ports:
  - port: 80
    targetPort: 80
```

> No `type` field is set — Kubernetes will default to `ClusterIP`.

```bash
kubectl create -f service.yaml
kubectl get service
```

Verify the TYPE column shows `ClusterIP` and EXTERNAL-IP shows `<none>`.

---

### Method 2: Using CLI Commands (Imperative)

You can create a ClusterIP service directly without writing a manifest file.

**Step 1: Explore available service subcommands**

```bash
kubectl create service --help
```

This shows all available service types you can create via CLI:
- `clusterip`
- `externalname`
- `loadbalancer`
- `nodeport`

**Step 2: Get help specific to ClusterIP creation**

```bash
kubectl create service clusterip --help
```

This shows the syntax and examples for creating a ClusterIP service.

**Step 3: Create the ClusterIP service**

```bash
# Format: kubectl create service clusterip <name> --tcp=<port>:<targetPort>
kubectl create service clusterip test-service --tcp=80:80
```

```bash
kubectl get service
```

Sample output:

```
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
test-service   ClusterIP   10.105.67.22    <none>        80/TCP    3s
```

**Step 4: Generate the equivalent manifest file (dry-run)**

If you created via CLI but want the YAML manifest for reference or version control:

```bash
kubectl create service clusterip test-service --tcp=80:80 --dry-run=client -o yaml
```

This prints the manifest to your terminal without actually creating the service — useful for generating a base YAML you can save and modify.

---

## 7. Exercises to Practice for the CKAD Exam

### Imperative (CLI-based) Exercises

**Exercise 1 — Create a ClusterIP Service from CLI**
1. Create a ClusterIP service called `test-service` exposing port `80` to target port `80`.
2. Verify the type, ClusterIP, and EXTERNAL-IP fields using `kubectl get service`.

**Exercise 2 — Generate a ClusterIP Manifest with Dry Run**
1. Use a dry-run command to generate the YAML for a ClusterIP service named `web-svc` (port 80, target 8080).
2. Save the output to a file `web-svc.yaml` without creating the service.

**Exercise 3 — Expose an Existing Pod**
1. Create a pod called `web-pod` using the `nginx` image.
2. Use `kubectl expose` to create a ClusterIP service called `web-svc` on port `80`.
3. From a temporary `busybox` pod, run `wget -O- web-svc` and confirm the nginx page comes back.

**Exercise 4 — Inspect a ClusterIP Service**
1. List the services in the default namespace.
2. Find a service whose EXTERNAL-IP is `<none>` and confirm it is a ClusterIP type.
3. Use `kubectl describe service <name>` and find the endpoints.

---

### Declarative (YAML-based) Exercises

**Exercise 5 — ClusterIP by Default (No Type Field)**
1. Write a Service manifest named `simple-service` with only `port: 80` and `targetPort: 80` — **do not specify the `type` field**.
2. Apply it and confirm the TYPE column shows `ClusterIP`.
3. Confirm EXTERNAL-IP is `<none>`.

**Exercise 6 — Explicit ClusterIP Service with Selector**
1. Write a Pod manifest for `backend-pod` running `nginx` with label `app: backend`.
2. Write a Service manifest of `type: ClusterIP` with selector `app: backend`, `port: 80`, `targetPort: 80`.
3. Apply both manifests and verify the Endpoints object is auto-populated.

**Exercise 7 — Internal-Only Access**
1. Using the Service from Exercise 6, launch a temporary pod (`kubectl run tmp --rm -it --image=busybox -- sh`).
2. From inside, run `wget -O- <service-clusterip>:80` and confirm it works.
3. Confirm that the ClusterIP is **not reachable** from your host machine outside the cluster.

**Exercise 8 — DNS-Based Access**
1. Reuse the Service from Exercise 6.
2. From a temporary pod, run `wget -O- simple-service` — using **the service name only**, not the IP.
3. Confirm the request is resolved by Kubernetes DNS and reaches the backend pod.

---

## 8. Summary

| Concept | Detail |
|---|---|
| **What it is** | An internal service type that exposes pods only within the cluster |
| **Default type** | Yes — if no `type` is specified in the manifest, Kubernetes defaults to ClusterIP |
| **External access** | Not possible — EXTERNAL-IP is always `<none>` |
| **IP stability** | The ClusterIP is a virtual IP that stays stable as long as the service exists |
| **Who can access it** | Only pods inside the same Kubernetes cluster |
| **Created via manifest** | `kubectl create -f service.yaml` |
| **Created via CLI** | `kubectl create service clusterip <name> --tcp=<port>:<targetPort>` |
| **Generate YAML from CLI** | Add `--dry-run=client -o yaml` to the CLI create command |
