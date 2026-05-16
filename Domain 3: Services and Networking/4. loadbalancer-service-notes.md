# LoadBalancer Service — Domain 3: Services and Networking

---

## 1. Recap: Types of Services

| Service Type | Key Features | Use Case |
|---|---|---|
| **ClusterIP** | Default type. Accessible only within the cluster. | Internal microservices communication |
| **NodePort** | Exposes service on a static port (30000–32767) on each Node. | Development testing, demo applications |
| **LoadBalancer** | Exposes service externally using the cloud provider's load balancer. | Production apps requiring external access |
| **ExternalName** | Maps service to an external DNS name. | External service integration |

This video focuses on **LoadBalancer** — the service type extensively used in production environments for public-facing applications.

---

## 2. What is LoadBalancer?

> A **LoadBalancer** service type creates an **external load balancer in your cloud provider** and routes the traffic received by the load balancer to the underlying NodePort service, which then forwards it to the backend pods.

- The external load balancer gets a **dedicated external IP** (not the worker node IP).
- External users connect to your application via this IP or a DNS name — **no port numbers needed**.
- The load balancer handles forwarding traffic to the right NodePort on the worker nodes.

```
External User
     │
     │  kplabs.internal  (or just an IP like 146.190.10.2)
     ▼
  LOAD BALANCER  (created in cloud provider — AWS ELB, DigitalOcean LB, etc.)
     │
     │  forwards to NodePort (e.g., 31249)
     ▼
  Worker Node : NodePort
     │
     ▼
  SERVICE ──► Pod-1
         \──► Pod-2
```

---

## 3. Why LoadBalancer — What Challenges Does It Solve?

### Challenge 1: NodePort Requires an Ugly Port Number in the URL

With NodePort, external users had to connect using:

```
DNS/IP : NodePort
e.g.,  203.0.113.10:31514
```

```
External User ──► 31514 on Worker Node ──► Service ──► Pod-1
                                                   \──► Pod-2
```

Imagine if google.com was accessed as `google.com:31210` — nobody would use it. Asking real users to remember a high port number like `31514` is not acceptable in production.

**LoadBalancer solves this** by putting a proper cloud load balancer in front. Users connect to a clean IP or DNS name on standard ports like `80` or `443` — the load balancer silently handles the NodePort forwarding behind the scenes.

---

### Challenge 2: NodePort Exposes the Worker Node IP Directly

With NodePort, the public-facing IP is your **worker node's IP**. This is a security concern and an operational problem — if a node is replaced or its IP changes, your users lose access.

**LoadBalancer solves this** by providing a **dedicated, stable load balancer IP** that is completely independent of the worker nodes. Even if all worker nodes are replaced, the load balancer IP remains the same.

---

### Challenge 3: No Built-in Traffic Distribution Across Multiple Nodes

NodePort opens a port on each node but does not intelligently route traffic across them. The client picks one node's IP and hits it — if that node is busy or down, the client's request fails.

**LoadBalancer solves this** by distributing incoming traffic across all healthy nodes automatically, giving you true production-grade load balancing at the entry point.

---

## 4. How LoadBalancer Works — The Mechanism

> This service type creates an **External Load Balancer in a Cloud Provider** and routes the request received at the Load Balancer to the underlying NodePort.

**Step-by-step flow:**

```
1. User hits:     kplabs.internal  (DNS resolves to load balancer IP e.g. 146.190.10.2)
                        │
2. LB receives:   request on port 80
                        │
3. LB forwards:   traffic to Worker Node : NodePort  (e.g., :31249)
                        │
4. Node routes:   request to the Service
                        │
5. Service LBs:   request to a backend pod  (Pod-1 or Pod-2)
```

The cloud provider's load balancer forwarding rule looks like:

```
Incoming port 80  ──►  forward to NodePort 31249 on worker nodes
```

This is exactly what you see in the cloud provider's dashboard (e.g., DigitalOcean forwarding rules).

---

## 5. Point to Note — Part 1: LoadBalancer Automatically Creates a NodePort Behind the Scenes

> When you create a LoadBalancer service, Kubernetes **automatically creates a NodePort service** behind the scenes. The external load balancer then directs traffic to these NodePorts, and the traffic is subsequently forwarded to the appropriate pods.

```
kubectl get service output:
NAME                TYPE          CLUSTER-IP      EXTERNAL-IP     PORT(S)         AGE
kplabs-loadbalancer LoadBalancer  10.109.1.176    146.190.10.2    80:31249/TCP    8m42s
kubernetes          ClusterIP     10.109.0.1      <none>          443/TCP         22d
```

Key observations:
- **TYPE** is `LoadBalancer`
- **EXTERNAL-IP** is `146.190.10.2` — this is the **load balancer's IP**, not the worker node IP
- **PORT(S)** shows `80:31249/TCP`:
  - `80` — the port the load balancer and service listen on
  - `31249` — the **NodePort** automatically created on every worker node

```bash
kubectl describe service <loadbalancer-service-name>
```

In the describe output you will see:
- `NodePort` field — the auto-assigned NodePort (e.g., 31249)
- `LoadBalancer Ingress` field — the external IP of the load balancer

So a LoadBalancer service is effectively a **superset**:

```
LoadBalancer service
  │
  ├── Has a ClusterIP      (for internal pod-to-pod communication)
  ├── Has a NodePort       (automatically created, used by the LB to reach nodes)
  └── Has an External IP   (the cloud load balancer's IP, for external access)
```

---

## 6. Point to Note — Part 2: Works Only with Supported Cloud Providers

> This service type works **only with supported cloud providers**. If you are running Kubernetes on-premise or in a local setup, additional configuration is required.

| Environment | Does LoadBalancer Work? |
|---|---|
| AWS (EKS) | Yes — creates an Elastic Load Balancer (ELB) automatically |
| DigitalOcean (DOKS) | Yes — creates a DigitalOcean Load Balancer automatically |
| GCP (GKE) | Yes — creates a Google Cloud Load Balancer automatically |
| Azure (AKS) | Yes — creates an Azure Load Balancer automatically |
| Minikube (local) | No — no cloud provider integration, EXTERNAL-IP stays `<pending>` |
| On-premise / bare-metal | No — requires additional tooling (e.g., MetalLB) |

**Why does it not work locally?**

Kubernetes needs to call the cloud provider's API to provision an actual load balancer. A local minikube setup has no such integration — so when you create a LoadBalancer service locally, the EXTERNAL-IP column stays stuck at `<pending>` forever.

**Managed Kubernetes clusters** (EKS, GKE, DOKS, AKS) already have this cloud integration built in — so the load balancer is created automatically as soon as the service is applied.

---

## 7. Point to Note — Part 3: Load Balancers Cost Money

> Cloud load balancers are **not free** — they are billed as a separate resource on top of your cluster cost.

| Provider | Approximate Cost |
|---|---|
| DigitalOcean Load Balancer | ~$12/month |
| AWS ELB / ALB | More expensive than DigitalOcean |
| GCP / Azure | Comparable to AWS pricing |

**Guideline:**
- **Development / Testing** → Use **NodePort** — free, no extra infrastructure
- **Production (public-facing)** → Use **LoadBalancer** — proper, production-grade, user-friendly URLs

Do not create LoadBalancer services in dev/test environments — the cost adds up quickly, especially if you have multiple services.

---

## 8. Hands-on Lab — Creating a LoadBalancer Service

> Note: This lab requires a managed Kubernetes cluster on a supported cloud provider (AWS, DigitalOcean, GCP, etc.). On minikube, the EXTERNAL-IP will remain `<pending>`.

### Step 1: Create the LoadBalancer Service Manifest

Save as `lb-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: simple-service
spec:
  selector:
    app: backend       # routes traffic to pods with this label
  type: LoadBalancer   # triggers cloud provider to create an external load balancer
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl create -f lb-service.yaml
kubectl get service
```

Initially the EXTERNAL-IP column will show `<pending>` — this means Kubernetes is provisioning the load balancer in the cloud provider. Wait a minute and run `kubectl get service` again.

Once ready, the output will look like:

```
NAME             TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
simple-service   LoadBalancer   10.109.1.176    146.190.10.2    80:31249/TCP   1m
```

### Step 2: Create a Backend Pod with the Matching Label

```bash
kubectl run backend-pod --image=nginx

# Add the label matching the service selector
kubectl label pod backend-pod app=backend
```

### Step 3: Verify Endpoints Are Populated

```bash
kubectl describe service simple-service
```

Check that the `Endpoints` field shows the backend pod's IP — the service is now wired to the pod.

Also note the fields:
- `NodePort` — the auto-created NodePort (e.g., 31249)
- `LoadBalancer Ingress` — the external IP of the cloud load balancer

### Step 4: Access From Outside the Cluster

```bash
# Using the EXTERNAL-IP from kubectl get service
curl <EXTERNAL-IP>
# e.g., curl 146.190.10.2
```

You get the nginx welcome page — **no port number needed**. You can also open `http://<EXTERNAL-IP>` in a browser.

In production, you would point a DNS record (e.g., `app.yourdomain.com`) to this external IP, so users never see an IP address at all.

### Cleanup

```bash
kubectl delete pod backend-pod
kubectl delete -f lb-service.yaml
```

> Deleting the service also **removes the cloud load balancer** — stopping any further billing for it.

---

## 9. Exercises to Practice for the CKAD Exam

> **Note:** Most of these exercises require a managed Kubernetes cluster (EKS, GKE, AKS, DigitalOcean). On minikube or kind, the EXTERNAL-IP will stay `<pending>` — you can still verify the YAML is correct.

### Imperative (CLI-based) Exercises

**Exercise 1 — Create a LoadBalancer Service via CLI**
1. Create a LoadBalancer service named `lb-test` exposing port `80` to target `80`.
2. Run `kubectl get svc` and observe the EXTERNAL-IP field. If it shows `<pending>`, wait until the cloud provider provisions the LB.

**Exercise 2 — Expose a Pod as LoadBalancer**
1. Create a pod called `web-pod` using the `nginx` image.
2. Expose it as a LoadBalancer service called `web-lb` on port `80`.
3. Once the EXTERNAL-IP is assigned, `curl <EXTERNAL-IP>` and confirm you see the nginx welcome page **without any port number**.

**Exercise 3 — Generate the YAML via Dry Run**
1. Use a dry-run command to generate a LoadBalancer service YAML for `web-lb` (port `80`, target `80`).
2. Save the output to a file without creating the service.

**Exercise 4 — Inspect the Auto-Created NodePort**
1. After creating any LoadBalancer service, run `kubectl describe service <name>`.
2. Find the `NodePort` field and the `LoadBalancer Ingress` field.
3. Explain what each of them is used for.

---

### Declarative (YAML-based) Exercises

**Exercise 5 — LoadBalancer with Selector**
1. Write a Pod manifest for `backend-pod` running `nginx` with label `app: backend`.
2. Write a Service manifest of `type: LoadBalancer` with selector `app: backend`, `port: 80`, `targetPort: 80`.
3. Apply both, wait for the EXTERNAL-IP to be assigned, and `curl` it from outside.

**Exercise 6 — Confirm the Superset Behaviour**
1. Reuse the Service from Exercise 5.
2. Identify three different ways to reach the backend pod:
   - From outside the cluster: `<EXTERNAL-IP>`
   - From a worker node: `<node-ip>:<nodePort>`
   - From inside another pod: `<clusterIP>:80`
3. Confirm all three work.

**Exercise 7 — Verify Pending State on Local Cluster**
1. On a local cluster (minikube/kind), apply a LoadBalancer service manifest.
2. Observe the EXTERNAL-IP staying as `<pending>` indefinitely.
3. Explain *why* — and which environments would automatically provision a real LB.

**Exercise 8 — Cleanup and Cost Awareness**
1. Delete the LoadBalancer service from Exercise 5 using `kubectl delete -f`.
2. Confirm in your cloud provider's dashboard that the LB resource is also removed.
3. Explain why leaving an unused LoadBalancer service in dev/test environments is costly.

---

## 10. Summary

| Concept | Detail |
|---|---|
| **What it is** | A service type that provisions a cloud load balancer and routes external traffic to backend pods |
| **Why over NodePort** | NodePort requires ugly `IP:Port` URLs; LoadBalancer gives a clean IP/DNS on standard ports |
| **External IP** | The IP of the cloud load balancer — not the worker node IP |
| **NodePort behind the scenes** | Yes — LoadBalancer automatically creates a NodePort; the LB forwards to it |
| **Also has ClusterIP** | Yes — internal pods can still use ClusterIP for communication |
| **Cloud provider required** | Yes — only works with AWS, GCP, Azure, DigitalOcean, etc. Not on minikube/bare-metal |
| **Cost** | Not free — cloud LBs are billed separately (~$12+/month depending on provider) |
| **EXTERNAL-IP pending** | Means the cloud LB is still being provisioned — wait and re-run `kubectl get service` |
| **Best used for** | Production, public-facing applications |
| **Not suitable for** | Local dev/test — use NodePort instead to avoid unnecessary cost |
| **Created via manifest** | Set `type: LoadBalancer` in service spec |
| **Describe command** | `kubectl describe service <name>` — shows NodePort, LoadBalancer Ingress fields |
