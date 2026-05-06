# Kubernetes Services - Domain 3: Services and Networking

---

## 1. Setting the Base

Before understanding Services, two foundational concepts must be clear:

- Every Pod created in a Kubernetes cluster gets scheduled on a Worker Node and is assigned its own **private IP address**.
- Pods within the **same cluster** can communicate with each other using these private IPs.

```
+----------------------------+
|       Worker Node          |
|                            |
|  frontend-pod  10.244.0.4  |
|       |                    |
|       v                    |
|  backend-pod   10.244.0.5  |
+----------------------------+
```

### Demo — View Pod IPs

```bash
# Create a backend pod (nginx server)
kubectl run backend-pod --image=nginx

# Create a frontend pod (ubuntu, sleeps so it stays alive)
kubectl run frontend-pod --image=ubuntu --command -- sleep 36000

# List pods with IP addresses
kubectl get pods -o wide
```

Sample output:
```
NAME            READY   STATUS    IP
backend-pod     1/1     Running   10.244.0.5
frontend-pod    1/1     Running   10.244.0.4
```

```bash
# Connect to frontend-pod and test connectivity to backend-pod
kubectl exec -it frontend-pod -- bash
apt-get update && apt-get -y install curl
curl 10.244.0.5    # hits backend-pod directly
```

---

## 2. The Problem: Why Hardcoding IPs Fails

### Problem 1 — Pods are Ephemeral

```
Config File
-----------
backend: 10.244.0.5   <--- hardcoded in frontend config

Worker Node 1              Worker Node 2
+-------------------+      +-------------------+
| frontend-pod      |      | backend-pod       |
|                   |      | 10.244.0.6        | <-- new IP after rescheduling
| backend-pod       |      +-------------------+
| 10.244.0.5 (DEAD) |
+-------------------+
```

If a backend pod crashes and Kubernetes reschedules it on another node, it gets a **new IP**. The frontend config still points to the old IP → **all requests fail**.

You would have to manually update the frontend config every time → not practical.

### Problem 2 — Deployments Run Multiple Pods

```
Config File
-----------
backend:
  10.244.0.5        <--- how do you list all replicas?
  10.244.0.6
  10.244.0.6

       frontend-pod
      /      |      \
backend-1  backend-2  backend-3
                       Worker Node 1
```

- With deployments you can have **hundreds of replicas**. Hardcoding all their IPs is impossible.
- Deployments create and destroy pods dynamically. IPs change constantly.
- The frontend also needs to **distribute traffic** across all backend pods — not send 100% to one.

---

## 3. Introducing Kubernetes Service

> **Service acts as a gateway that distributes incoming traffic between its endpoints.**

Instead of talking directly to backend pods, the frontend talks to the **Service**. The service handles:
- Discovering the current set of healthy pods
- Distributing (load balancing) requests across them

```
Config File
-----------
backend: service.internal   <-- stable name, never changes

       frontend-pod
            |
            v
         SERVICE
        /   |    \
  backend-1 backend-2 backend-3
```

The frontend config only needs the service name — **never pod IPs**.

### Reference: Traffic Distribution in Action

From inside frontend-pod, hitting the service repeatedly:

```bash
root@frontend-pod:/# curl service.internal
Backend Pod 2
root@frontend-pod:/# curl service.internal
Backend Pod 1
root@frontend-pod:/# curl service.internal
Backend Pod 2
root@frontend-pod:/# curl service.internal
Backend Pod 1
```

Requests are automatically **round-robin load balanced** across backend pods.

---

## 4. Service and Deployments

```
         SERVICE
        /   |    \
  backend-1 backend-2 backend-3
```

- When a Deployment creates or removes pods, the Service **automatically discovers** the updated set of running pods.
- If `backend-3` goes down, the Service stops routing to it and only routes to `backend-1` and `backend-2`.
- When the Deployment spins up a replacement pod, the Service picks it up automatically.

You don't need to touch the frontend config at all — the Service manages everything.

---

## 5. External Access via Service

```
Internet User  <--->  SERVICE  <--->  backend-1
                                 \--> backend-2
                                  \-> backend-3
```

Service also enables **users outside the cluster** (e.g., internet users) to reach pods that are internal to the cluster.

For example: a pod serving an HTML website needs to be accessible on the internet. You expose it via a Service with an external IP. Users hit that external IP and the service routes them to the appropriate pod.

---

## 6. Benefits of Service — Summary

| Benefit | Explanation |
|---|---|
| **Stable Endpoint** | Pods are ephemeral, IPs change. Services provide a fixed IP/DNS name. |
| **Load Balancing** | Distributes traffic across multiple pod replicas automatically. |
| **External Exposure** | Enables exposing pods to external traffic (internet). |

---

## 7. Types of Services

| Service Type | Key Features | Use Case |
|---|---|---|
| **ClusterIP** | Default type. Accessible only within the cluster. | Internal microservices communication |
| **NodePort** | Exposes service on a static port (30000–32767) on each Node. | Development testing, demo applications |
| **LoadBalancer** | Exposes service externally using the cloud provider's load balancer. | Production apps requiring external access |
| **ExternalName** | Maps service to an external DNS name. | External service integration |

> Each type will be covered in detail in separate videos.

---

## 8. Hands-on Lab 1: Service with Manual Endpoints

This lab shows how a Service works **without selectors** — you manually define which pod IPs the service routes to.

### Step 1: Create Backend and Frontend Pods

```bash
kubectl run backend-pod --image=nginx
kubectl run frontend-pod --image=ubuntu --command -- sleep 36000
```

### Step 2: Verify Pod IPs

```bash
kubectl get pods -o wide
```

Note the IP of `backend-pod` (e.g., `10.244.0.23`).

### Step 3: Test Direct Pod-to-Pod Communication

```bash
kubectl exec -it frontend-pod -- bash
apt-get update && apt-get -y install curl
curl <BACKEND-POD-IP>    # replace with actual IP from above
```

You should see the nginx welcome page — confirming pod-to-pod communication works.

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

At this point, `kubectl describe` will show **no endpoints** — because there is no selector and no endpoint object yet.

### Step 5: Manually Create the Endpoint

> The Endpoint object **must have the same name** as the Service.

Save as `endpoint.yaml` (replace `10.244.0.23` with your backend-pod IP):

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: simple-service     # must match Service name exactly
subsets:
  - addresses:
      - ip: 10.244.0.23    # backend-pod IP
    ports:
      - port: 80
```

```bash
kubectl create -f endpoint.yaml
kubectl describe service simple-service
```

Now `kubectl describe` will show the endpoint IP listed — the service is wired up.

### Step 6: Test via Service IP

```bash
kubectl exec -it frontend-pod -- bash
curl <SERVICE-IP>      # use the CLUSTER-IP shown by kubectl get service
```

The response comes from `backend-pod` — traffic now routes through the service.

---

### Step 7: Understanding `port` vs `targetPort`

- **`port`** — the port the Service listens on (what clients call)
- **`targetPort`** — the port on the Pod that actually handles the traffic

Modify `service.yaml` to change the service port to `8080` while the pod still listens on `80`:

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

### Step 8: Test with New Port

```bash
kubectl exec -it frontend-pod -- bash
curl <SERVICE-IP>:8080     # note: now port 8080, not 80
```

Still reaches `backend-pod` on port 80 — the Service transparently translates 8080 → 80.

### Step 9: Cleanup

```bash
kubectl delete -f service.yaml
kubectl delete pod backend-pod
kubectl delete pod frontend-pod
```

---

## 9. Hands-on Lab 2: Service with Selectors (Automatic Endpoint Management)

Instead of manually creating Endpoint objects, you can add a **selector** to the Service. Kubernetes will then **automatically manage** the endpoints by watching for pods that match the label.

### Base Service with Selector

Save as `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: simple-service
spec:
  selector:
    app: backend         # automatically includes pods with this label
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl create -f service.yaml
```

At this point, no pods match `app=backend` yet — so endpoints will be empty.

### Create Pods and Add Labels

```bash
kubectl run backend-pod-1 --image=nginx
kubectl run backend-pod-2 --image=nginx

# Add the label that the Service selector watches for
kubectl label pod backend-pod-1 app=backend
kubectl label pod backend-pod-2 app=backend
```

### Verify Endpoints Auto-Populated

```bash
kubectl describe service simple-service
```

You will see **both pod IPs** listed under Endpoints — Kubernetes added them automatically when the label matched.

### Test Dynamic Endpoint Removal

Remove the label from `backend-pod-1`:

```bash
kubectl label pod backend-pod-1 app-      # the trailing dash removes the label
```

```bash
kubectl describe service simple-service
```

Now only `backend-pod-2` appears in the endpoints — the Service dynamically removed `backend-pod-1` as soon as it no longer matched the selector.

Add the label back to re-include it:

```bash
kubectl label pod backend-pod-1 app=backend
kubectl describe service simple-service   # both pods back
```

### Cleanup

```bash
kubectl delete -f service.yaml
kubectl delete pod backend-pod-1
kubectl delete pod backend-pod-2
```

---

## 10. Key Concepts Summary

```
Without Service:
  frontend-pod  ----hardcoded IP---->  backend-pod
  Problem: pod IPs change, breaks when pod restarts

With Service (Manual Endpoints):
  frontend-pod  -->  Service  -->  Endpoint (manually defined IP)  -->  backend-pod
  Advantage: stable Service IP/DNS, but you manage endpoints yourself

With Service (Selector):
  frontend-pod  -->  Service  -->  auto-discovered pods (by label match)  -->  backend-pods
  Advantage: fully automatic, Kubernetes manages endpoints as pods come and go
```

### Important Points

- Service gets a stable **ClusterIP** that never changes (unlike pod IPs).
- Service also gets a stable **DNS name** inside the cluster (e.g., `simple-service.default.svc.cluster.local`).
- The `name` of an Endpoint object **must match** the Service name for manual binding.
- When a pod's label is removed, it is **immediately** removed from the Service endpoints.
- `port` on the Service and `targetPort` on the pod can be different — Service translates between them.
- Services with selectors continuously watch pods — no restart needed when pods change.
