# 📘 Istio Service Mesh — Installation & First App Deployment

> **Audience:** Complete beginners to Kubernetes & Istio  
> **Source:** KodeKloud — Istio Service Mesh Course (5 pages)  
> **Topics Covered:** Installing Istio, Deploying Bookinfo, Sidecar Injection

---

## 📚 Table of Contents

1. [What is Istio & How Can It Be Installed?](#1-what-is-istio--how-can-it-be-installed)
2. [Installing `istioctl` — The Command-Line Tool](#2-installing-istioctl--the-command-line-tool)
3. [Installing Istio on Your Kubernetes Cluster](#3-installing-istio-on-your-kubernetes-cluster)
4. [Deploying Your First App on Istio — Theory](#4-deploying-your-first-app-on-istio--theory)
5. [Deploying Your First App on Istio — Live Demo](#5-deploying-your-first-app-on-istio--live-demo)
6. [Quick Command Reference (Cheatsheet)](#6-quick-command-reference-cheatsheet)
7. [Common Errors & Fixes](#7-common-errors--fixes)

---

## 1. What is Istio & How Can It Be Installed?

### 🤔 What is Istio?

Istio is a **Service Mesh** — think of it as an invisible layer that sits between all your running apps (microservices) inside Kubernetes. It handles **traffic management, security, and monitoring** for you — without you having to write that logic yourself in every application.

### 🛠️ Three Ways to Install Istio

| Method | Description | When to Use |
|---|---|---|
| **`istioctl`** (CLI tool) | Simplest command-line approach | Learning, demos, getting started |
| **Istio Operator** | A Kubernetes controller that manages Istio's lifecycle | Production environments |
| **Helm** | Kubernetes package manager | Teams already using Helm for all installs |

> ✅ **This guide uses `istioctl`** — it is the simplest approach for learning and is what the `demo` profile is designed around.

---

### 📦 What Gets Installed When You Run Istio?

When Istio installs, it creates a brand-new Kubernetes **Namespace** called `istio-system`. Everything Istio needs lives there.

```
istio-system/
├── Istiod              ← Istio's brain (control plane)
├── Istio Ingress Gateway  ← Handles traffic coming INTO the cluster
├── Istio Egress Gateway   ← Handles traffic going OUT of the cluster
└── 13 Custom Resource Definitions (CRDs)
```

> 📌 **Namespace** — A way to logically group resources inside Kubernetes, like separate folders. `kube-system` has Kubernetes internals, `default` is where your apps go by default, and `istio-system` is where all Istio components live.

---

### 🧠 What is Istiod?

Istiod is Istio's brain — it is made of **three sub-components** bundled together:

| Component | Role |
|---|---|
| **Pilot** | Tells all the Envoy proxies how to route traffic |
| **Citadel** | Handles security — certificates, encryption (mTLS) |
| **Galley** | Validates and distributes configuration to the rest of Istio |

---

### 📋 What are CRDs (Custom Resource Definitions)?

Kubernetes normally understands objects like `Pod`, `Service`, `Deployment`.  
**CRDs teach Kubernetes brand-new object types.**

Istio adds **13 new object types**, including:

- `VirtualService` — Controls how traffic is routed between services
- `Gateway` — Manages traffic entering/leaving the mesh
- `DestinationRule` — Defines policies for traffic going to a service
- `Sidecar` — Configures the Envoy sidecar proxy behaviour
- `AuthorizationPolicy` — Controls who can talk to whom
- `PeerAuthentication` — Enforces mTLS between services

---

### 🎚️ What is the Demo Profile?

Istio has different "**profiles**" — sets of features pre-tuned for different use cases.

| Profile | What It Includes | Best For |
|---|---|---|
| `demo` | Everything — Istiod, ingress gateway, egress gateway | Learning & demos |
| `default` | Istiod + ingress gateway only | Light production use |
| `minimal` | Only the control plane (Istiod) | Custom production setups |

> We use **`demo`** throughout this guide.

---

## 2. Installing `istioctl` — The Command-Line Tool

### 🖥️ What is Minikube and Why Do We Need It?

**Minikube** lets you run a full Kubernetes cluster on your own laptop. Think of it as a mini test environment. Before installing Istio, you need a running cluster — Minikube provides that locally.

---

### Step 1 — Start Minikube

```bash
minikube start
```

Minikube automatically picks a **driver** (like Docker or a VM). The driver determines how the cluster runs on your machine.

> ⚠️ **macOS + Docker Driver Problem:**  
> If Docker is running, Minikube may select the Docker driver. On macOS, the Docker driver has a networking limitation — the ingress addon **does not work** with it.
>
> You'll see this error when trying to enable ingress:
> ```
> Exiting due to MK_USAGE: Due to networking limitations of driver docker on darwin,
> ingress addon is not supported.
> ```
>
> **Why?** The cluster runs inside a Docker container, and the ingress addon can't properly receive external traffic in that environment.
>
> **Fix:** Delete the cluster and restart with a VM-based driver:
> ```bash
> minikube delete
> minikube start --vm=true   # Uses hyperkit on macOS automatically
> ```

---

### Step 2 — Enable the Ingress Addon

```bash
minikube addons enable ingress
```

This enables Kubernetes to receive traffic from outside the cluster — required for Istio's gateway to work properly.

---

### Step 3 — Download the Istio Release

```bash
curl -L https://istio.io/downloadIstio | sh -
```

This one command downloads everything — the `istioctl` binary, sample apps, manifests, and tools.  
It creates a folder like `istio-1.10.3/` in your current directory.

> ⚠️ Run this from a location where you want the folder to be created, since it downloads into your **current directory**.

**Inside the downloaded folder:**

```
istio-1.10.3/
├── bin/
│   └── istioctl          ← The CLI tool you'll use to install & manage Istio
├── samples/
│   ├── bookinfo/         ← The demo app we'll use in this guide
│   ├── helloworld/
│   ├── httpbin/
│   └── ...
├── manifests/            ← YAML files describing all Istio components
└── tools/                ← Shell completions and certificate utilities
```

---

### Step 4 — Add `istioctl` to Your PATH

```bash
export PATH=$PWD/bin:$PATH
```

> 📌 **PATH** is a list of folders your terminal searches when you type a command.  
> By adding the `bin/` folder to PATH, you can now type `istioctl` from anywhere in your terminal.

Run this from **inside the downloaded Istio folder** (e.g., inside `istio-1.10.3/`).

---

### Step 5 — Verify `istioctl` is Working

```bash
istioctl version
```

Expected output:

```
no running Istio pods in "istio-system"
1.10.3
```

This is **expected** — Istio isn't installed in the cluster yet, so no pods are running.

---

### Step 6 — Run Pre-Installation Check

```bash
istioctl x precheck
```

This checks if your cluster meets Istio's requirements **before** you install anything. Always run this first.

> ❌ **Common Mistake:** Running `istioctl verify-install` before Istio is installed gives:
> ```
> error while fetching revision: the server could not find the requested resource
> ```
> **Why?** `verify-install` checks if Istio is already installed. Nothing is installed yet, so it fails.  
> **Fix:** Use `istioctl x precheck` for pre-install checks, then `istioctl verify-install` after installing.

---

## 3. Installing Istio on Your Kubernetes Cluster

### Step 1 — Run the Install Command

```bash
istioctl install --set profile=demo -y
```

Breaking this down:

| Flag | Meaning |
|---|---|
| `istioctl install` | Tells istioctl to install Istio into the connected cluster |
| `--set profile=demo` | Use the demo preset (includes ingress + egress gateways) |
| `-y` | Auto-confirm without prompting "Are you sure?" |

**Expected output:**

```
√ Istio core installed
Processing resources for Istiod. Waiting for Deployment/istio-system/istiod
∘ Istiod installed
∘ Ingress gateways installed
∘ Egress gateways installed
Installation complete
```

---

### What Kubernetes Objects Does Istio Create?

| Object Type | What It Is | Why Istio Needs It |
|---|---|---|
| `ClusterRole` / `ClusterRoleBinding` | Permissions | Tells Kubernetes what Istio is allowed to do cluster-wide |
| `ServiceAccount` | An identity | Lets Istio's components authenticate themselves inside the cluster |
| `ValidatingWebhookConfiguration` | A config validator | Asks Istiod to validate Istio YAML before it's applied — like a spell-checker |
| `CustomResourceDefinition` (×13) | New resource types | Adds VirtualService, Gateway, DestinationRule, etc. |
| `ConfigMap` | Key-value config store | Stores Istiod's configuration so pods can read it |
| `Deployment` | Runs containers | Keeps Istiod and gateway containers running |

> 📌 **Deployment** — A Kubernetes object that says "run X copies of this container and keep them running". When Istiod is deployed, Kubernetes creates a Deployment for it in the `istio-system` namespace.

---

### Step 2 — Verify the Installation

```bash
istioctl verify-install
```

**Expected output:**

```
1 Istio control planes detected, checking --revision "default" only
✓ ClusterRole: istiod-istio-system.istio-system checked successfully
✓ ClusterRoleBinding: istiod-istio-system.istio-system checked successfully
✓ ServiceAccount: istiod-service-account.istio-system checked successfully
✓ ValidatingWebhookConfiguration: istiod-istio-system.istio-system checked successfully
✓ CustomResourceDefinition: gateways.networking.istio.io checked successfully
✓ CustomResourceDefinition: virtualservices.networking.istio.io checked successfully
✓ Deployment: istiod-istio-system checked successfully
...
✓ Istio is installed and verified successfully
```

> ✅ Istio is now running in your cluster inside the `istio-system` namespace. Your apps are not connected to it yet — that comes next.

---

## 4. Deploying Your First App on Istio — Theory

### 🗂️ What is the Bookinfo App?

Bookinfo is a **sample application** bundled with Istio (in the `samples/` folder). It simulates a simple book review website made of multiple microservices — perfect for learning how Istio manages traffic between services.

```
User Request
    │
    ▼
┌──────────────┐
│  ProductPage │  ← Front page — calls Details and Reviews
└──────┬───────┘
       │
   ┌───┴──────────────────────┐
   │                          │
   ▼                          ▼
┌─────────┐           ┌──────────────────────────┐
│ Details │           │  Reviews (v1 / v2 / v3)  │
└─────────┘           └────────────┬─────────────┘
                                   │
                                   ▼
                             ┌─────────┐
                             │ Ratings │
                             └─────────┘
```

| Microservice | What It Does |
|---|---|
| **ProductPage** | The main front page — calls Details and Reviews |
| **Details** | Returns book metadata (author, pages, ISBN, etc.) |
| **Ratings** | Returns star ratings for the book |
| **Reviews v1** | Shows reviews — no stars |
| **Reviews v2** | Shows reviews — with black stars (calls Ratings) |
| **Reviews v3** | Shows reviews — with red stars (calls Ratings) |

> The **3 versions of Reviews** are intentional — they let you demonstrate Istio's traffic routing (e.g., send 80% of traffic to v1, 20% to v3).

---

### Step 1 — Deploy the App

```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

> 📌 `kubectl apply -f` reads a YAML file and creates all Kubernetes objects defined inside it — Pods, Services, ServiceAccounts, Deployments, etc.

This creates deployments and services for all 6 microservices in the `default` namespace.

---

### Step 2 — Check Pod Status (Problem Found!)

```bash
kubectl get pods
```

Output:

```
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-79f774bd9-5gqjb        1/1     Running   0          22s
productpage-v1-6b746f74dc-k486v   1/1     Running   0          21s
ratings-v1-b6994bb9-ds6gk         1/1     Running   0          22s
reviews-v1-546db7b795-spvl6       1/1     Running   0          21s
reviews-v2-7bf8c964f-h9php        1/1     Running   0          22s
reviews-v3-84779c7bbc-df51c       1/1     Running   0          22s
```

⚠️ **Notice `1/1` — this is a problem!**

Each pod shows only **1 container** running. But Istio should have injected an **Envoy sidecar proxy** into every pod, giving us `2/2`.

---

### Step 3 — Diagnose Using `istioctl analyze`

```bash
istioctl analyze
```

Output:

```
Info [IST0102] (Namespace default) The namespace is not enabled for Istio injection.
Run 'kubectl label namespace default istio-injection=enabled' to enable it, or
'kubectl label namespace default istio-injection=disabled' to explicitly mark it as
not needing injection.
```

> ❓ **Why doesn't Istio inject sidecars automatically?**  
> Kubernetes has many namespaces — `kube-system` runs Kubernetes core components that should **NOT** have Istio sidecars injected into them. So Istio is conservative and only injects where you **explicitly allow it** by labeling the namespace.

---

### Step 4 — Enable Sidecar Injection for the Namespace

```bash
kubectl label namespace default istio-injection=enabled
```

Output:

```
namespace/default labeled
```

> 📌 A **Label** in Kubernetes is a `key=value` tag you attach to any object. Istio watches for the label `istio-injection=enabled` on namespaces. When it sees this label, it automatically injects the Envoy proxy into every **new** pod created in that namespace.

> ⚠️ **Important:** Labeling the namespace only affects **new** pods. Existing pods don't get the sidecar injected — you must recreate them.

---

### Step 5 — Delete Old Pods and Redeploy

```bash
kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

---

### Step 6 — Confirm Sidecars Are Injected (`2/2`)

```bash
kubectl get pods
```

Output:

```
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-5f449d8bb9-vgzrh       2/2     Running   0          26s
productpage-v1-6f9df695b7-cx259   2/2     Running   0          26s
ratings-v1-857bb87c57-hmpfd       2/2     Running   0          26s
reviews-v1-68f9c47f69-cbj7c       2/2     Running   0          26s
reviews-v2-5d56c4885f-wb4v7       2/2     Running   0          26s
reviews-v3-869ff44845-h5pfp       2/2     Running   0          26s
```

✅ **`2/2` = 1 app container + 1 Envoy sidecar proxy**

Istio is now managing all traffic between these pods!

---

## 5. Deploying Your First App on Istio — Live Demo

This is the same process as Chapter 4 but walked through as a live demo. Here are additional insights and observations.

### Using `istioctl apply` vs `kubectl apply`

In the demo, `istioctl apply` is used instead of `kubectl apply`:

```bash
istioctl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

| Command | Difference |
|---|---|
| `kubectl apply` | Standard Kubernetes apply — no Istio awareness |
| `istioctl apply` | Istio-aware — validates config against Istio's rules before applying |

Both work, but `istioctl apply` is safer as it catches Istio-specific configuration errors early.

---

### Understanding the `Terminating` Pod Status

After redeploying, you may briefly see:

```
NAME                              READY   STATUS        RESTARTS   AGE
ratings-v1-b6994bb9-ds6gk         1/1     Terminating   0          97s
ratings-v1-b6994bb9-w2bk          2/2     Running       0          14s
```

> **Why does this happen?**  
> When you delete a pod, Kubernetes sends a graceful shutdown signal and waits for the pod to finish any in-progress work before fully removing it. This is completely **normal** behaviour called "graceful termination".  
>
> **Fix:** Just wait 30–60 seconds and run `kubectl get pods` again — the Terminating pods will disappear on their own.

---

### Final Verification

```bash
istioctl analyze
```

Expected output:

```
✓ No validation issues found when analyzing namespace: default.
```

✅ Your service mesh is live and correctly configured!

---

### What Does `2/2` Actually Mean?

Every pod now has **2 containers** running inside it:

```
┌─────────────────────────────────────┐
│              Pod                    │
│                                     │
│  ┌────────────────┐  ┌───────────┐  │
│  │   Your App     │  │  Envoy    │  │
│  │  (e.g. reviews)│  │  Sidecar  │  │
│  └────────────────┘  └─────┬─────┘  │
│                            │        │
└────────────────────────────┼────────┘
                             │
                    All traffic passes through here
                    Istio controls this proxy
```

| Container | What It Is |
|---|---|
| **Container 1 — Your App** | The actual microservice (e.g., productpage, reviews, ratings) |
| **Container 2 — Envoy Sidecar** | A proxy injected by Istio — all traffic in/out of the pod flows through it |

> 💡 **The Envoy sidecar is the magic of Istio.**  
> Since every pod has one, Istio can intercept, observe, secure, and control 100% of the traffic flowing between your services — **without changing a single line of your application code.**

---

## 6. Quick Command Reference (Cheatsheet)

### Setup & Cluster

```bash
# Start Minikube (use VM driver on macOS for ingress support)
minikube start --vm=true

# Enable ingress addon
minikube addons enable ingress

# Delete and restart with different driver (if needed)
minikube delete
minikube start --vm=true
```

### Download & Configure `istioctl`

```bash
# Download latest Istio release (creates istio-<version>/ folder)
curl -L https://istio.io/downloadIstio | sh -

# Go into the downloaded folder
cd istio-1.10.3/

# Add istioctl to your terminal PATH
export PATH=$PWD/bin:$PATH

# Confirm istioctl works
istioctl version
```

### Install Istio

```bash
# Pre-installation check (run before installing)
istioctl x precheck

# Install Istio with demo profile
istioctl install --set profile=demo -y

# Verify installation was successful (run after installing)
istioctl verify-install
```

### Deploy & Manage Apps

```bash
# Deploy the Bookinfo sample application
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

# Check pod status (look for 2/2 to confirm sidecar injection)
kubectl get pods

# Diagnose Istio configuration issues
istioctl analyze

# Enable sidecar injection for the default namespace
kubectl label namespace default istio-injection=enabled

# Delete and redeploy to trigger sidecar injection
kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

# (Alternative) Redeploy using Istio-aware apply
istioctl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

---

## 7. Common Errors & Fixes

### ❌ Error 1 — Ingress addon fails on macOS (Docker driver)

```
Exiting due to MK_USAGE: Due to networking limitations of driver docker on darwin,
ingress addon is not supported.
```

**Why:** The Docker driver on macOS runs the cluster inside a Docker container, which blocks the ingress networking.

**Fix:**
```bash
minikube delete
minikube start --vm=true
minikube addons enable ingress
```

---

### ❌ Error 2 — `verify-install` fails before Istio is installed

```
error while fetching revision: the server could not find the requested resource
0 Istio injectors detected
Error: could not load IstioOperator from cluster
```

**Why:** `verify-install` checks for an existing installation. Nothing is installed yet.

**Fix:** Run in the correct order:
```bash
istioctl x precheck       # 1. Pre-install check
istioctl install ...      # 2. Install
istioctl verify-install   # 3. Post-install verify
```

---

### ❌ Error 3 — Pods show `1/1` instead of `2/2` (no sidecar)

```
NAME                   READY   STATUS    RESTARTS   AGE
details-v1-xxx         1/1     Running   0          22s
```

**Why:** The namespace wasn't labeled for Istio injection before the pods were created.

**Fix:**
```bash
# 1. Label the namespace
kubectl label namespace default istio-injection=enabled

# 2. Delete existing pods so they get recreated with sidecars
kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml

# 3. Redeploy
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

# 4. Verify — should now show 2/2
kubectl get pods
```

---

### ❌ Error 4 — Pods stuck in `Terminating` state

```
ratings-v1-b6994bb9-ds6gk    1/1   Terminating   0   97s
```

**Why:** Kubernetes gracefully shuts down pods — it waits for in-progress requests to finish before fully deleting the pod. This is normal.

**Fix:** Wait 30–60 seconds and re-check:
```bash
kubectl get pods
```

The Terminating pods will disappear on their own.

---

### ❌ Error 5 — `istioctl analyze` reports namespace not enabled

```
Info [IST0102] (Namespace default) The namespace is not enabled for Istio injection.
```

**Why:** The `default` namespace doesn't have the Istio injection label.

**Fix:**
```bash
kubectl label namespace default istio-injection=enabled
```

---

## 📎 Key Kubernetes Terms Explained

| Term | Simple Explanation |
|---|---|
| **Namespace** | A folder inside Kubernetes to group resources. `default` for apps, `kube-system` for Kubernetes internals, `istio-system` for Istio. |
| **Pod** | The smallest unit in Kubernetes — one or more containers running together. |
| **Deployment** | Tells Kubernetes to keep a certain number of Pods running. If one crashes, it restarts it. |
| **Service** | A stable address for a group of Pods. Pods have changing IPs; a Service gives them a fixed name. |
| **ServiceAccount** | An identity for a Pod to authenticate itself within the cluster. |
| **Label** | A `key=value` tag on any Kubernetes object. Used to select and filter objects. |
| **CRD (Custom Resource Definition)** | Teaches Kubernetes a new object type. Istio adds 13 new types like `VirtualService`, `Gateway`, etc. |
| **ConfigMap** | Stores config data as key-value pairs — like a settings file inside Kubernetes. |
| **ClusterRole** | Defines what actions are allowed across the entire cluster. |
| **ClusterRoleBinding** | Connects a ClusterRole to a ServiceAccount — "this identity can do these things". |
| **Sidecar** | A second container injected into every Pod by Istio. The Envoy proxy — all traffic flows through it. |
| **Envoy** | A high-performance proxy. Istio uses it as the sidecar to intercept and control network traffic. |
| **Ingress Gateway** | Manages traffic entering the cluster from outside. |
| **Egress Gateway** | Manages traffic leaving the cluster to external services. |

---

## 🔗 Resources

- [Istio Official Documentation](https://istio.io/latest/docs/)
- [Istio Installation Guide](https://istio.io/latest/docs/setup/install/)
- [Minikube Documentation](https://minikube.sigs.k8s.io/docs/)
- [Kubernetes Basics](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/)
- [Helm Charts Repository](https://artifacthub.io/)
- [Minikube Ingress Issue Tracker (macOS)](https://github.com/kubernetes/minikube/issues/7332)

---

*Notes compiled from KodeKloud — Istio Service Mesh Course*
