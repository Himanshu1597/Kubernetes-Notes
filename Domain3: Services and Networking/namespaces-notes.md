# Kubernetes Namespaces — Domain 3: Services and Networking

---

## 1. What is a Namespace?

> A **Namespace** in Kubernetes is a way to **isolate a group of resources** (like Pods, ConfigMaps, Secrets) inside a **single Kubernetes cluster**.

Think of a namespace as a "folder" inside the cluster. Each folder keeps its own resources separate from other folders, even though all folders live in the same cluster.

```
┌──────────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster                       │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  Development │  │   Staging    │  │      QA      │         │
│  │              │  │              │  │              │         │
│  │  Pods        │  │  Pods        │  │  Pods        │         │
│  │  ConfigMaps  │  │  ConfigMaps  │  │  ConfigMaps  │         │
│  │  Secrets     │  │  Secrets     │  │  Secrets     │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. Why Namespaces — What Challenges Do They Solve?

### Challenge 1: Multiple Teams Sharing One Cluster

In real production, a single cluster is rarely used by just one team. There can be many teams or projects sharing the same cluster, and each team may run **hundreds of pods**. Without separation, everything mixes together and becomes hard to manage.

**Namespaces solve this** by giving each team or project its own isolated space. The development team works only in the `development` namespace, the QA team works only in the `qa` namespace — they do not interfere with each other.

---

### Challenge 2: Separating Environments (Dev / QA / Staging)

You want different environments like development, QA, and staging to be kept apart, but you also do not want to spend money on a separate cluster for each one (managed clusters are expensive).

**Namespaces solve this** by letting you run development, QA, and staging inside the **same cluster** while keeping their pods, configmaps, and secrets fully separated.

> Note: **Production is usually kept on its own separate cluster** — not mixed with dev/QA namespaces. This is done to reduce the attack surface and avoid accidental permission leaks.

---

### Challenge 3: Resources Getting Mixed Up

If everything lives in one default place, the list of pods, configmaps, and secrets becomes one giant mess. It is hard to know which resource belongs to which team or environment.

**Namespaces solve this** by grouping resources logically — making it much easier to view, manage, and clean up resources per team or per environment.

---

## 3. Default Namespaces in Kubernetes

Kubernetes comes with **four built-in namespaces** out of the box:

| Namespace | Description |
|---|---|
| **default** | Used for resources created without specifying any namespace |
| **kube-system** | Holds resources created by the Kubernetes system itself — **do not touch this** |
| **kube-public** | Visible to all users; usually used for publicly accessible cluster data |
| **kube-node-lease** | Used internally for node heartbeat tracking |

> The `kube-system` namespace contains the pods and configmaps that keep Kubernetes itself running. Messing with them can break your cluster.

---

## 4. Important Points to Remember

- If you do **not** specify a namespace when creating a resource, it goes into the **default** namespace.
- `kubectl get pods` (without any flag) only shows pods from the **default** namespace — even if many pods exist in other namespaces.
- To see resources across **all** namespaces, you must use `--all-namespaces` (or `-A`).
- To work inside a specific namespace, use the `-n <namespace>` flag.
- A namespace must exist **before** you can create resources inside it.

---

## 5. Syntax

### Listing Namespaces

```bash
kubectl get namespace
# or short form:
kubectl get ns
```

### Creating a Namespace

```bash
kubectl create namespace <namespace-name>
```

### Viewing Resources Across All Namespaces

```bash
kubectl get pods --all-namespaces
kubectl get configmaps --all-namespaces
```

### Viewing Resources in a Specific Namespace

```bash
kubectl get pods -n <namespace-name>
```

### Creating a Resource in a Specific Namespace

```bash
kubectl run <pod-name> --image=<image> -n <namespace-name>
```

### Specifying Namespace in a Manifest File

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: qa        # resource will be created in the qa namespace
spec:
  containers:
  - name: nginx
    image: nginx
```

### Generating a Manifest with Namespace (Dry Run)

```bash
kubectl run test-pod --image=nginx --namespace=qa --dry-run=client -o yaml
```

---

## 6. Examples

### Example 1: See That Default Namespace is Empty, but Cluster is Not

```bash
kubectl get pods
# No resources found in default namespace.

kubectl get pods --all-namespaces
# Shows many pods running in the kube-system namespace
```

This proves that "no pods in default" does not mean "no pods in the cluster".

---

### Example 2: List All ConfigMaps Across the Cluster

```bash
kubectl get configmaps
# Shows only configmaps in the default namespace

kubectl get configmaps --all-namespaces
# Shows configmaps in kube-system, kube-public, and others
```

---

### Example 3: Create Two New Namespaces

```bash
kubectl create namespace development
kubectl create namespace qa

kubectl get namespace
# NAME              STATUS   AGE
# default           Active   24d
# development       Active   5s
# kube-node-lease   Active   24d
# kube-public       Active   24d
# kube-system       Active   24d
# qa                Active   3s
```

---

### Example 4: Create Pods in Specific Namespaces

```bash
# Pod in the development namespace
kubectl run development-pod --image=nginx -n development

# Pod in the qa namespace
kubectl run qa-pod --image=nginx -n qa
```

---

### Example 5: View Pods in a Specific Namespace

```bash
kubectl get pods                  # default namespace only
kubectl get pods -n development   # only development pods
kubectl get pods -n qa            # only qa pods
kubectl get pods --all-namespaces # everything in the cluster
```

---

### Example 6: Pod Manifest with Namespace Defined

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: qa
spec:
  containers:
  - name: nginx
    image: nginx
```

```bash
kubectl apply -f pod.yaml
# Pod gets created inside the qa namespace, not the default one
```

---

## 7. Exercises to Practice for the CKAD Exam

### Imperative (CLI-based) Exercises

**Exercise 1 — List and Inspect Namespaces**
1. List all namespaces in the cluster using `kubectl get ns`.
2. Identify the four default namespaces and write down what each one is used for.

**Exercise 2 — Create New Namespaces**
1. Create a namespace called `frontend`.
2. Create a namespace called `backend`.
3. Verify both namespaces exist.

**Exercise 3 — Create Pods in Specific Namespaces**
1. Create a pod named `web` using the `nginx` image inside the `frontend` namespace.
2. Create a pod named `api` using the `httpd` image inside the `backend` namespace.
3. Run `kubectl get pods` — confirm you see **nothing** (because default is empty).
4. Run `kubectl get pods -n frontend` and `kubectl get pods -n backend` to see the pods.

**Exercise 4 — View Resources Across All Namespaces**
1. Run `kubectl get pods --all-namespaces` and identify in which namespace `kube-apiserver`, `kube-scheduler`, and `coredns` are running.
2. List **all configmaps** across all namespaces.

**Exercise 5 — Dry Run to Generate Namespace YAML**
1. Use `kubectl create namespace dev --dry-run=client -o yaml` to generate a Namespace manifest.
2. Save it to a file.

**Exercise 6 — Cleanup via CLI**
1. Delete the `frontend` and `backend` namespaces using `kubectl delete ns`.
2. Confirm that **all pods inside them are also deleted** (deleting a namespace deletes everything in it).

---

### Declarative (YAML-based) Exercises

**Exercise 7 — Namespace Manifest**
1. Write a YAML manifest for a namespace called `qa`.
2. Apply it and confirm it appears in `kubectl get ns`.

**Exercise 8 — Pod in a Specific Namespace via Manifest**
1. Write a Pod manifest for `db-pod` using the `redis` image.
2. Set `metadata.namespace: backend` so it lands in the `backend` namespace.
3. Apply and verify with `kubectl get pods -n backend`.

**Exercise 9 — Dry Run to Generate Pod YAML with Namespace**
1. Run `kubectl run test-pod --image=nginx -n qa --dry-run=client -o yaml` to generate a manifest.
2. Save it, apply it, and confirm the pod exists only in the `qa` namespace.

**Exercise 10 — Bulk Resource Manifest**
1. Write a single YAML file containing:
   - A `Namespace` called `app1`
   - A `Pod` (image: `nginx`) in the `app1` namespace
   - A `ConfigMap` in the `app1` namespace
2. Apply the file and verify all three exist using `-n app1`.

---

## 8. Summary

| Concept | Detail |
|---|---|
| **What it is** | A logical way to isolate groups of resources inside a single cluster |
| **Why** | Lets multiple teams / environments (dev, QA, staging) share one cluster without interfering |
| **Default namespace** | Where resources go if no namespace is specified |
| **kube-system** | System pods and configmaps — never modify |
| **kube-public / kube-node-lease** | Built-in namespaces for public data and node heartbeats |
| **List namespaces** | `kubectl get namespace` |
| **Create namespace** | `kubectl create namespace <name>` |
| **Create resource in namespace** | `kubectl run <pod> --image=<img> -n <namespace>` |
| **View pods in a namespace** | `kubectl get pods -n <namespace>` |
| **View across all namespaces** | `kubectl get pods --all-namespaces` |
| **In manifest** | Add `namespace: <name>` under `metadata` |
| **Production rule** | Keep production in a **separate cluster**, not just a separate namespace |
