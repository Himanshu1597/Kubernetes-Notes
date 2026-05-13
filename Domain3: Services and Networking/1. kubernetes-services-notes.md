# Kubernetes Services — Domain 3: Services and Networking

---

## 1. Foundation: How Pods Communicate

Before understanding Services, two things must be clear:

- Every Pod created in a Kubernetes cluster gets scheduled on a Worker Node and is assigned its own **private IP address**.
- Pods within the **same cluster** can communicate with each other using these private IPs.

```
+------------------------------+
|         Worker Node          |
|                              |
|  frontend-pod   10.244.0.4   |
|        |                     |
|        ▼                     |
|  backend-pod    10.244.0.5   |
+------------------------------+
```

```bash
# See pods along with their assigned private IPs
kubectl get pods -o wide
```

---

## 2. The Challenges — Why Direct Pod IPs Don't Work

### Challenge 1: Pods Are Ephemeral

```
frontend config
───────────────
backend: 10.244.0.5   ◄── hardcoded

  Worker Node 1              Worker Node 2
┌─────────────────┐        ┌─────────────────┐
│  frontend-pod   │        │  backend-pod    │
│                 │        │  10.244.0.6     │ ◄── new IP after rescheduling
│  backend-pod    │        └─────────────────┘
│  10.244.0.5 ✗  │
└─────────────────┘
```

If a backend pod crashes and Kubernetes reschedules it, it gets a **new IP address**. The frontend config still points to the old IP → **all requests fail**.

You would have to manually update every config file every time a pod restarts → not practical.

---

### Challenge 2: Deployments Run Many Pods — Hardcoding Is Impossible

```
frontend config
───────────────
backend:
  10.244.0.5   ◄── impossible to track
  10.244.0.6
  10.244.0.7
  ... (hundreds more)

        frontend-pod
       /      |      \
  backend-1  backend-2  backend-3
```

- In production, a deployment can have **hundreds of replicas**. You cannot hardcode all their IPs.
- Deployments dynamically create and destroy pods — IPs change constantly.

---

### Challenge 3: Load Balancing Must Be Handled

- You want traffic distributed equally across all backend pods.
- If the frontend sends 100% of requests to one pod, that pod gets overloaded.
- Without a Service, the frontend application itself would need to implement load balancing logic — that is not its job.

---

### Challenge 4: External Access

- Pods only have private IPs — unreachable from outside the cluster.
- When you have a web application that needs to be accessed by users on the internet, you need a way to expose it.
- There is no built-in way to do this with just pod IPs.

---

## 3. What is a Kubernetes Service?

> **A Service acts as a stable gateway that distributes incoming traffic between its endpoints.**

A Service sits between the caller (frontend) and the pods (backend). Instead of the frontend talking to pod IPs directly, it talks to the **Service**. The Service takes care of:

- Maintaining a **stable IP and DNS name** that never changes
- **Load balancing** requests across all healthy backend pods
- **Automatically tracking** which pods are currently running
- **Exposing** the application to external traffic when needed

```
frontend config
───────────────
backend: service.internal   ◄── stable, never changes

        frontend-pod
              │
              ▼
           SERVICE
          /   |   \
   backend-1  backend-2  backend-3
```

The frontend config only ever needs the Service name — never pod IPs.

### Traffic Distribution in Action

From inside the frontend pod, hitting the service repeatedly shows round-robin load balancing:

```
root@frontend-pod:/# curl service.internal
Backend Pod 2
root@frontend-pod:/# curl service.internal
Backend Pod 1
root@frontend-pod:/# curl service.internal
Backend Pod 2
root@frontend-pod:/# curl service.internal
Backend Pod 1
```

Requests are automatically distributed — no single pod gets overloaded.

### Service + Deployments

```
         SERVICE
        /   |    \
  backend-1 backend-2 backend-3
```

- When a Deployment removes a failing pod, the Service **stops routing** to it immediately.
- When a Deployment creates a new pod, the Service **picks it up** automatically.
- The frontend never needs to change its config — the Service always knows the current healthy pods.

### External Access via Service

```
Internet User  ◄──►  SERVICE  ──►  backend-1
                            \───►  backend-2
                             \──►  backend-3
```

A Service with an external IP allows users on the internet to reach pods that are otherwise private inside the cluster. The Service load balances across all pods just as it does internally.

---

## 4. Benefits of Service — Summary

| Benefit | Explanation |
|---|---|
| **Stable Endpoint** | Pods are ephemeral; their IPs change. Services provide a fixed ClusterIP and DNS name. |
| **Load Balancing** | Automatically distributes traffic across all matching pod replicas. |
| **External Exposure** | Enables exposing pods to traffic from outside the cluster (internet). |

---

## 5. Types of Services

| Service Type | Key Features | Use Case |
|---|---|---|
| **ClusterIP** | Default type. Accessible only within the cluster. | Internal microservices communication |
| **NodePort** | Exposes service on a static port (30000–32767) on each Node. | Development testing, demo applications |
| **LoadBalancer** | Exposes service externally using the cloud provider's load balancer. | Production apps requiring external access |
| **ExternalName** | Maps service to an external DNS name. | External service integration |

> Each type is covered in detail in separate sections.

---

## 6. Core Concept: The Endpoint Object — How a Service Actually Routes Traffic

> This is the most important concept to understand before running the labs.

A Service does **not** talk directly to pods. The internal chain is:

```
Client  ──►  Service  ──►  Endpoints Object  ──►  Pod IPs
```

### What is an Endpoint Object?

An **Endpoint** (`kind: Endpoints`) is a **separate Kubernetes resource** that holds the list of actual `IP:port` pairs that the Service should forward traffic to — i.e., the real pod addresses.

- Every Service has a corresponding Endpoints object with the **exact same name**.
- When a request hits the Service, Kubernetes looks up the Endpoints object to find where to send the traffic.

```bash
# Inspect the endpoints for all services
kubectl get endpoints

# Inspect endpoints for a specific service
kubectl describe endpoints <service-name>
```

### The Two Ways Endpoints Get Populated

| Method | How | When |
|---|---|---|
| **Manual** | You create the `Endpoints` YAML yourself with pod IPs | Service has **no selector** |
| **Automatic** | Kubernetes watches pods matching a label and auto-manages the `Endpoints` object | Service has a **selector** |

### Visual Comparison

```
Service WITHOUT selector (manual endpoints):

  Service "simple-service"
         │
         ▼
  Endpoints "simple-service"   ◄── YOU create and update this
         │
         └── ip: 10.244.0.23, port: 80   (backend-pod)


Service WITH selector (automatic endpoints):

  Service "simple-service"  selector: app=backend
         │
         ▼
  Endpoints "simple-service"   ◄── KUBERNETES creates and updates this
         │
         ├── ip: 10.244.0.4, port: 80   (backend-pod-1, has label app=backend)
         └── ip: 10.244.0.5, port: 80   (backend-pod-2, has label app=backend)
```

### How Kubernetes Keeps Endpoints in Sync (with selector)

| Event | What Kubernetes Does |
|---|---|
| Pod gets label `app=backend` | Adds pod IP to the Endpoints object |
| Pod's label `app=backend` is removed | Removes pod IP from Endpoints object |
| Pod crashes and restarts with a new IP | Updates Endpoints with the new IP |
| Pod is deleted | Removes its IP from Endpoints |

The **Service ClusterIP never changes**. Only the Endpoints list behind it changes as pods come and go.
This is the entire mechanism that makes Services powerful.

---

## 7. Understanding `port` vs `targetPort`

These are two different port values on a Service:

| Field | Meaning |
|---|---|
| `port` | The port the **Service** listens on — what clients use to reach the Service |
| `targetPort` | The port on the **Pod** that actually handles the traffic |

```
Client  ──►  Service:8080  ──►  Pod:80
              (port)             (targetPort)
```

They can be the same or different. The Service transparently translates between them.
This is useful when you want to expose a standard port (e.g., `80`) through the Service while the container internally listens on a different port.

---

## 8. Hands-on Lab 1 — Service with Manual Endpoints (No Selector)

**Goal:** Understand the Endpoint object by creating it manually and wiring it to a Service.

### Step 1: Create Backend and Frontend Pods

```bash
kubectl run backend-pod --image=nginx
kubectl run frontend-pod --image=ubuntu --command -- sleep 36000
```

### Step 2: Check Pod IPs

```bash
kubectl get pods -o wide
```

Note the IP of `backend-pod` — you will need it in Step 5.

Sample output:
```
NAME            READY   STATUS    IP
backend-pod     1/1     Running   10.244.0.23
frontend-pod    1/1     Running   10.244.0.4
```

### Step 3: Confirm Direct Pod-to-Pod Communication Works

```bash
kubectl exec -it frontend-pod -- bash
apt-get update && apt-get -y install curl
curl <BACKEND-POD-IP>    # e.g., curl 10.244.0.23
```

You should get the nginx welcome page — this confirms pod-to-pod communication works using private IPs.

### Step 4: Create a Service Without a Selector

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

```bash
kubectl create -f service.yaml
kubectl get service
kubectl describe service simple-service
```

At this point, `kubectl describe` shows **no endpoints** — because no selector is defined and no Endpoint object has been created yet. The Service exists but has nowhere to route traffic.

### Step 5: Manually Create the Endpoint Object

> The Endpoint object **must have the exact same name** as the Service. This is how Kubernetes binds them.

Save as `endpoint.yaml` (replace the IP with your actual `backend-pod` IP from Step 2):

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: simple-service     # must match the Service name exactly
subsets:
  - addresses:
      - ip: 10.244.0.23    # backend-pod IP
    ports:
      - port: 80
```

```bash
kubectl create -f endpoint.yaml

# Now verify the service has endpoints
kubectl describe service simple-service
kubectl get endpoints simple-service
```

The `kubectl describe` output will now show the backend pod IP listed under **Endpoints** — the Service is wired up.

### Step 6: Test Connectivity Through the Service

```bash
kubectl exec -it frontend-pod -- bash
curl <SERVICE-CLUSTER-IP>   # use CLUSTER-IP from: kubectl get service
```

Traffic goes: `frontend-pod → Service → Endpoints object → backend-pod`. You get the nginx page.

---

### Step 7: Demonstrate `port` vs `targetPort`

Modify `service.yaml` — the Service will now listen on port `8080`, but forward to the pod on port `80`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: simple-service
spec:
  ports:
  - port: 8080        # clients connect on 8080
    targetPort: 80    # traffic forwarded to pod port 80
```

```bash
kubectl delete -f service.yaml
kubectl create -f service.yaml
kubectl create -f endpoint.yaml
kubectl describe service simple-service
```

### Step 8: Test with the New Port

```bash
kubectl exec -it frontend-pod -- bash
curl <SERVICE-CLUSTER-IP>:8080    # must use port 8080 now
```

The response still comes from `backend-pod` on port 80 — the Service translated `8080 → 80` transparently.

### Step 9: Cleanup

```bash
kubectl delete -f service.yaml
kubectl delete pod backend-pod
kubectl delete pod frontend-pod
```

---

## 9. Hands-on Lab 2 — Service with Selectors (Automatic Endpoint Management)

**Goal:** Use a `selector` on the Service so Kubernetes automatically manages the Endpoint object as pods are labelled and unlabelled.

### Step 1: Create the Service with a Selector

Save as `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: simple-service
spec:
  selector:
    app: backend         # Kubernetes will auto-manage Endpoints for pods with this label
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl create -f service.yaml

# Check endpoints — empty right now, no pods have the label yet
kubectl describe service simple-service
kubectl get endpoints simple-service
```

### Step 2: Create Pods (Without Labels Yet)

```bash
kubectl run backend-pod-1 --image=nginx
kubectl run backend-pod-2 --image=nginx

# Endpoints are still empty — no label yet
kubectl get endpoints simple-service
```

### Step 3: Add Labels to Pods — Watch Endpoints Auto-Populate

```bash
kubectl label pod backend-pod-1 app=backend
kubectl label pod backend-pod-2 app=backend

# Now check endpoints — both pod IPs should appear automatically
kubectl describe service simple-service
kubectl get endpoints simple-service
```

Kubernetes detected the matching label and added both pod IPs to the Endpoints object without any manual action.

### Step 4: Remove a Label — Watch Endpoint Disappear Instantly

```bash
# The trailing dash (-) removes the label from the pod
kubectl label pod backend-pod-1 app-

# Immediately check — backend-pod-1 is gone from endpoints
kubectl describe service simple-service
```

Only `backend-pod-2` appears now. Kubernetes removed `backend-pod-1` the moment its label no longer matched the selector.

### Step 5: Add the Label Back — Watch Endpoint Reappear

```bash
kubectl label pod backend-pod-1 app=backend

# Both pods are back in endpoints
kubectl describe service simple-service
```

### Step 6: Cleanup

```bash
kubectl delete -f service.yaml
kubectl delete pod backend-pod-1
kubectl delete pod backend-pod-2
```

---

## 10. Exercises to Practice for the CKAD Exam

### Imperative (CLI-based) Exercises

**Exercise 1 — Inspect Pod IPs**
1. Create two pods: `frontend` (image: `nginx`) and `backend` (image: `nginx`).
2. List both pods along with their assigned private IPs.
3. From inside the `frontend` pod, `curl` the `backend` pod's IP and confirm you get the nginx welcome page.

**Exercise 2 — Generate a Service YAML from CLI**
1. Generate the YAML for a ClusterIP service named `web-svc` on port 80 targeting port 8080, **without creating it**.
2. Save it to a file called `web-svc.yaml`.

**Exercise 3 — Expose a Pod Quickly**
1. Create a pod called `web` using the `nginx` image.
2. Use `kubectl expose` to create a service named `web-svc` on port 80.
3. List all endpoints — confirm the backend pod IP appears.

**Exercise 4 — Inspect Endpoints**
1. Run `kubectl get endpoints` and identify the endpoints for any one service.
2. Use `kubectl describe service <name>` and find the `Endpoints` field.

---

### Declarative (YAML-based) Exercises

**Exercise 5 — Service Without Selector + Manual Endpoints**
1. Write a Pod manifest for `backend-pod` running `nginx`.
2. Write a Service manifest called `simple-service` with `port: 80` and `targetPort: 80` — **no selector**.
3. Write an Endpoints manifest with the exact name `simple-service` that points to the `backend-pod`'s IP on port 80.
4. Apply all three manifests in order and verify connectivity from another pod using `curl <service-ClusterIP>`.

**Exercise 6 — Service With Selector (Automatic Endpoints)**
1. Write a Pod manifest for `backend-pod` with the label `app: backend`.
2. Write a Service manifest called `simple-service` with selector `app: backend`, `port: 80`, `targetPort: 80`.
3. Apply both and verify the Endpoints object got populated automatically.

**Exercise 7 — Test `port` vs `targetPort`**
1. Modify the Service from Exercise 6 so that `port: 8080` but `targetPort: 80`.
2. From another pod, `curl <service-ClusterIP>:8080` and confirm you reach the nginx page on the backend's port 80.

**Exercise 8 — Label-Driven Endpoint Changes**
1. Create two pods (`pod-1`, `pod-2`) running `nginx` with the label `app: backend`.
2. Apply the Service from Exercise 6.
3. Remove the label from `pod-1` using `kubectl label pod pod-1 app-` and verify it disappears from the Endpoints.
4. Add the label back and confirm it reappears.

---

## 11. Summary

### The Full Picture

```
Without Service:
  frontend-pod ──hardcoded IP──► backend-pod
  ✗ Breaks when pod restarts (new IP), breaks with many pods, no load balancing

With Service + Manual Endpoint:
  frontend-pod ──► Service ──► Endpoints (you manage) ──► backend-pod
  ✓ Stable IP, but you manually keep Endpoints in sync with pod IPs

With Service + Selector:
  frontend-pod ──► Service ──► Endpoints (Kubernetes manages) ──► backend-pods
  ✓ Fully automatic — Kubernetes updates Endpoints as pods come and go
```

### Key Points to Remember

- A Service does not route directly to pods — it routes to an **Endpoints object**, which holds the pod IPs.
- The Endpoints object **must have the same name** as its Service (for manual binding).
- Service gets a stable **ClusterIP** — this never changes, unlike pod IPs.
- Service also gets a stable **DNS name** inside the cluster (e.g., `simple-service.default.svc.cluster.local`).
- `port` = what the Service listens on; `targetPort` = what the pod listens on. They can differ.
- When a pod's label is removed, it is **immediately** dropped from the Endpoints — no delay.
- Services with selectors continuously watch pods — no restart required when pods change.
- `kubectl get endpoints` and `kubectl describe service <name>` are the key commands to inspect routing.
