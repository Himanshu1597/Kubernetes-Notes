# 📊 Kiali — Visualizing Your Istio Service Mesh

> **Audience:** Complete beginners to Kubernetes & Istio  
> **Source:** KodeKloud — Istio Service Mesh Course (3 pages)  
> **Pages Covered:**
> - Visualizing Service Mesh with Kiali
> - Demo Installing Kiali
> - Demo Create Traffic Into Your Mesh

---

## 📚 Table of Contents

1. [What is Kiali and Why Does It Exist?](#1-what-is-kiali-and-why-does-it-exist)
2. [Companion Tools — Prometheus, Grafana, Jaeger](#2-companion-tools--prometheus-grafana-jaeger)
3. [Installing Kiali — Step by Step](#3-installing-kiali--step-by-step)
4. [What Gets Created During Installation?](#4-what-gets-created-during-installation)
5. [Exploring the Kiali Dashboard](#5-exploring-the-kiali-dashboard)
6. [Creating Traffic Into Your Mesh](#6-creating-traffic-into-your-mesh)
7. [Simulating a Fault — Seeing Kiali Detect Problems](#7-simulating-a-fault--seeing-kiali-detect-problems)
8. [Key Terms Glossary](#8-key-terms-glossary)
9. [Full Command Cheatsheet](#9-full-command-cheatsheet)

---

## 1. What is Kiali and Why Does It Exist?

### 🤔 The Problem It Solves

Imagine you have 10 microservices all talking to each other inside Kubernetes. Istio manages all the traffic between them — but Istio itself gives you **no visual way to see what is happening**. If something goes wrong — say the Reviews service starts throwing errors — you'd have to dig through dozens of terminal logs across every service to find the problem.

> Think of it like flying a plane with all the instruments covered. The plane (Istio) is still flying, but you have no idea what's going on.

**Kiali solves this.** It is a **web-based dashboard** — a visual control panel — specifically built for Istio. It shows you a live, interactive map of how all your microservices are connected and how traffic is flowing between them, in real time.

---

### 📌 What Exactly is Kiali?

Kiali is a **powerful add-on for Istio** that brings a user-friendly graphical interface to your service mesh management.

> **Important:** Kiali is NOT part of Istio itself. It is an **add-on** — an optional but highly recommended companion. Istio works without Kiali, but you'd have no visibility into what's happening inside your mesh.

---

### ✅ Key Capabilities of Kiali

| Capability | What It Means in Simple Terms |
|---|---|
| **Microservices Connectivity** | See exactly which microservice talks to which — a live network map of your entire application |
| **Topology Visualization** | Displays the service mesh as an interactive graph with nodes (services) and arrows (traffic flow) |
| **Detailed Insights** | Shows request rates, error rates, latency, and circuit breaking status — per service |
| **Automated Wizards** | Helps you apply common traffic patterns and auto-generates Istio config — without writing YAML manually |

---

### 🔑 Why Did Kiali Come Into the Picture?

As microservices ecosystems expand, it becomes increasingly important to ensure that:

- Each service is **available** (not crashing)
- Each service is **performing optimally** (not slow)
- Traffic is **flowing correctly** between services
- Configuration mistakes are caught **early**

None of this was visible without a dedicated observability tool. Kiali was built specifically to solve this problem for Istio users.

---

## 2. Companion Tools — Prometheus, Grafana, Jaeger

When you install Kiali, it also installs **three companion tools** that feed it data. You cannot have a fully functional Kiali without these tools.

> **Note:** These add-ons are installed for **demonstration purposes only**. They are not tuned for production performance or security. For real production setups, you'd configure each tool separately with proper sizing and security.

---

### 🔵 Prometheus

**What it is:** A metrics collection and storage system.

**Why it exists:** Every Envoy sidecar proxy in every pod generates data — how many requests it handled, how many failed, how slow they were. Prometheus continuously scrapes (collects) this data from every sidecar and stores it.

**Why Kiali needs it:** Kiali reads from Prometheus to display numbers — request rates, error percentages, latency figures — on the dashboard. Without Prometheus, Kiali has no data to show.

```
Envoy Sidecar (in every pod)
        │
        │ metrics (requests, errors, latency)
        ▼
    Prometheus  ←── stores all the numbers
        │
        │ queries
        ▼
      Kiali  ←── displays the numbers as graphs and stats
```

---

### 🟡 Grafana

**What it is:** A charting and graphing tool.

**Why it exists:** While Kiali shows you the topology (the map), Grafana shows you **trends over time** — like a time-series graph of CPU usage, request latency over the past hour, error spikes, etc.

**Why Kiali needs it:** Grafana takes Prometheus data and renders it as beautiful, detailed dashboards. Kiali can link out to Grafana for deeper metric exploration.

**What `configmap/istio-grafana-dashboards` means:** This pre-installs Istio-specific dashboard templates into Grafana — ready-made charts for Istio metrics like request volume, success rate, and P99 latency.

---

### 🟣 Jaeger

**What it is:** A distributed tracing tool.

**Why it exists:** Prometheus tells you that the Reviews service had 50 errors per minute. But it doesn't tell you *which user requests* failed, *how long each step took*, or *which exact path* the request took through your microservices. That's what Jaeger does.

**Simple analogy:** Jaeger is like a package tracking system. Instead of just knowing a package was lost, you can see exactly which warehouse (service) it passed through, how long it sat at each stop, and where it got stuck.

**Why Kiali needs it:** Kiali can link directly to Jaeger traces so you can drill from "this service had errors" all the way down to "here is the exact trace of the failing request."

**What Zipkin is:** Zipkin is another distributed tracing format/tool. Jaeger supports the Zipkin format for compatibility — that's why you see `service/zipkin` created during installation. It's the same Jaeger, just speaking a different protocol.

---

### 📊 How All the Tools Work Together

```
Your App (Pods with Envoy Sidecars)
         │
         ├──── metrics ──────▶ Prometheus ──▶ Kiali (numbers & graphs)
         │                         │
         │                         └──────▶ Grafana (time-series charts)
         │
         └──── traces ──────▶ Jaeger ──────▶ Kiali (request tracing)
                                    │
                                    └──── (Zipkin format also supported)
```

---

## 3. Installing Kiali — Step by Step

### Prerequisites

Before installing Kiali, make sure:
- Minikube is running
- Istio is installed with the demo profile
- Bookinfo app is deployed and sidecars are injected (pods showing `2/2`)

---

### Step 1 — Install Kiali and All Add-ons in One Command

Istio ships with sample add-on YAML files. A single command installs everything — Kiali, Grafana, Jaeger, and Prometheus.

```bash
kubectl apply -f samples/addons
```

> **How this works:** The `-f samples/addons` points to a **folder** — Kubernetes applies every YAML file inside it at once. This is called "applying a directory." Each tool has its own YAML file inside that folder.

---

### Step 2 — Verify Services Are Running

Check that all add-on services are up and running in the `istio-system` namespace:

```bash
kubectl -n istio-system get services
```

You should see services for: `kiali`, `grafana`, `tracing`, `prometheus`, `zipkin`, `jaeger-collector`, plus the existing Istio gateway services.

---

### Step 3 — Launch the Kiali Dashboard

```bash
istioctl dashboard kiali
```

This automatically opens your browser to the Kiali dashboard.

> **Note:** The address bar shows **port 20001** — that is Kiali's default port. The URL will be something like `http://localhost:20001/kiali`.

---

## 4. What Gets Created During Installation?

Running `kubectl apply -f samples/addons` creates a large group of Kubernetes objects. Here is what each one means:

### Grafana Objects

| Object Created | What It Is | Why It's Needed |
|---|---|---|
| `serviceaccount/grafana` | An identity for Grafana | Lets Grafana authenticate itself within the cluster |
| `configmap/grafana` | Configuration settings | Stores Grafana's settings as key-value data so its pod can read them |
| `service/grafana` | A stable network address | So other pods and Kiali can reach Grafana at a fixed address |
| `deployment.apps/grafana` | Runs Grafana container | Tells Kubernetes to run Grafana and keep it alive |
| `configmap/istio-grafana-dashboards` | Pre-built dashboard templates | Installs Istio-specific Grafana dashboards automatically |

### Jaeger Objects

| Object Created | What It Is | Why It's Needed |
|---|---|---|
| `deployment.apps/jaeger` | Runs the Jaeger container | Keeps the tracing tool running |
| `service/tracing` | Network address for Jaeger UI | How you access the Jaeger web interface |
| `service/zipkin` | Zipkin-compatible endpoint | Jaeger supports the Zipkin tracing format for compatibility |
| `service/jaeger-collector` | Receives trace data | Envoy sidecars send their trace data here |

### Kiali Objects

| Object Created | What It Is | Why It's Needed |
|---|---|---|
| `serviceaccount/kiali` | Kiali's cluster identity | Lets Kiali authenticate itself |
| `configmap/kiali` | Kiali's configuration | Stores settings — which Prometheus URL to use, etc. |
| `clusterrole.rbac/kiali-viewer` | Read-only cluster-wide permission | Kiali needs to see all resources — pods, services, configs — to display them |
| `clusterrolebinding.rbac/kiali` | Connects the ClusterRole to Kiali's identity | Without this binding, the permission exists but Kiali can't use it |
| `role.rbac/kiali-controlplane` | Permission to interact with Istio's control plane | Kiali reads Istio configs (VirtualServices, Gateways, etc.) to display them |
| `rolebinding.rbac/kiali-controlplane` | Connects role to Kiali | Makes the controlplane role active for Kiali |
| `service/kiali` | Network address for Kiali UI | How `istioctl dashboard kiali` reaches the Kiali pod |
| `deployment.apps/kiali` | Runs Kiali container | Keeps Kiali running |

### MonitoringDashboard CRDs (Custom Kiali Resources)

These are pre-built dashboard templates that Kiali installs as **Custom Resources**. They define monitoring dashboards for different languages and frameworks:

| MonitoringDashboard | What It Monitors |
|---|---|
| `envoy` | Envoy proxy metrics — the sidecar inside every pod |
| `go` | Go language runtime metrics |
| `micrometer` | Java Micrometer metrics library |
| `micrometer-1.0.6-jvm` | JVM metrics via Micrometer 1.0.6 |
| `micrometer-1.1-jvm` | JVM metrics via Micrometer 1.1 |
| `springboot` | Spring Boot application metrics |
| `springboot-jvm` | Spring Boot JVM heap and GC metrics |
| `springboot-tomcat` | Embedded Tomcat server metrics |
| `vertx-jvm` | Vert.x JVM metrics |
| `vertx-server` | Vert.x HTTP server metrics |
| `vertx-client` | Vert.x HTTP client metrics |
| `vertx-eventbus` | Vert.x event bus metrics |
| `microprofile-1.1` | MicroProfile metrics spec |

### Prometheus Objects

| Object Created | What It Is | Why It's Needed |
|---|---|---|
| `serviceaccount/prometheus` | Prometheus's cluster identity | Authentication within cluster |
| `configmap/prometheus` | Scrape config | Tells Prometheus which targets (Envoy sidecars) to collect metrics from |
| `clusterrole.rbac/prometheus` | Cluster-wide read permission | Prometheus needs to discover all pods to scrape their metrics |
| `clusterrolebinding.rbac/prometheus` | Activates the permission | Binds the role to Prometheus's service account |
| `service/prometheus` | Network address | How Kiali queries Prometheus |
| `deployment.apps/prometheus` | Runs Prometheus container | Keeps Prometheus running and scraping |

> **Why so many objects?** Each tool (Kiali, Grafana, Jaeger, Prometheus) needs its own ServiceAccount (identity), ConfigMap (settings), Service (network address), and Deployment (running container). That is why one command creates 30+ objects.

---

## 5. Exploring the Kiali Dashboard

Once Kiali opens at `http://localhost:20001`, you will see several sections in the left menu.

---

### 🏠 Overview Page

The first screen you see. Shows all namespaces in your cluster, each displayed as a card showing how many applications are running inside it.

- Click the number next to "Applications" to drill into a namespace
- A **filter at the top left** lets you adjust what is displayed
- **Controls at the far right** let you filter by time intervals and adjust the data refresh rate

---

### 📱 Applications Section

Shows all applications Kiali has discovered in a namespace. For Bookinfo, you'll see four applications:

- `productpage`
- `details`
- `reviews`
- `ratings`

Each application shows:

| Field | What It Shows |
|---|---|
| **Health status** | Colored indicator — green = healthy, red = errors, yellow = warnings |
| **Labels** | Key-value Kubernetes labels attached to the app — app name, version, etc. |
| **Namespace switcher** | Switch between namespaces at the top of the page |

> **Note:** Initially services might not display. This is **normal** — the Service Mesh applications and objects may take some time to load. Wait a few seconds and refresh.

---

### ⚙️ Workloads Section

Shows all Kubernetes **Deployments** as "workloads." In Kubernetes, a Deployment is what actually runs your application containers.

Each workload shows:

| Field | What It Shows |
|---|---|
| **Health indicator** | Whether all pods in the deployment are running correctly |
| **App label** | Which application this workload belongs to |
| **Version label** | Which version — v1, v2, or v3 |

> **Workload vs Application — what's the difference?**  
> An Application can have multiple Workloads. For example, the "reviews" application has three workloads — `reviews-v1`, `reviews-v2`, and `reviews-v3` — all running simultaneously. Kiali tracks them separately so you can see which version is receiving what traffic.

All workloads show as **healthy** (green) when everything is running correctly, with labels indicating the application name and version.

---

### 🔗 Services Section

Shows all Kubernetes Services — the stable network addresses for groups of pods.

> **What is a Kubernetes Service?**  
> Pods come and go (they restart, get replaced, scale up/down) and their IP addresses change every time. A Kubernetes Service gives a group of pods a **fixed name and IP address** so other services can always reach them reliably.

Each service shows health, labels, and whether any Istio configuration is applied to it.

> **Important:** Initially, no services might be displayed. This is **completely normal** — services take a moment to load. Wait and refresh.

---

### 🔧 Istio Config Section

Shows all Istio-specific configurations applied in a namespace — things like VirtualServices, Gateways, DestinationRules, etc.

> **Initially this will be empty** — because we haven't applied any Istio traffic rules yet. Once you apply a Gateway and VirtualService (Step 6 below), they will appear here and you can inspect them directly in the Kiali UI.

---

### 📈 Graph Section — The Most Powerful Feature

This is the star of Kiali. It shows a **live, dynamic map** of how all your services are connected and how traffic is flowing between them.

> **Critical:** The graph is **empty when there is no traffic**. Kiali only draws connections it has actually seen traffic flow through. If nobody is hitting your app, the graph shows nothing. You must generate traffic first!

Once traffic flows, the graph shows:

| Element | What It Represents |
|---|---|
| **Nodes (circles/boxes)** | Each node = one service, workload, or application in your mesh |
| **Arrows** | Each arrow = traffic flowing from one service to another |
| **Green arrows** | Healthy traffic flowing without errors |
| **Red arrows/nodes** | Errors — something is failing |
| **Yellow** | Warnings — degraded performance |
| **Numbers on arrows** | Request rate in requests per second |

**Graph controls (top right):**
- Time interval filter — see data from last 1 min, 5 min, 10 min, 30 min, etc.
- Refresh rate — how often Kiali refreshes the graph data

---

## 6. Creating Traffic Into Your Mesh

The Kiali graph needs real traffic to visualize. Here is the complete process to get traffic flowing through your mesh and see it in Kiali.

---

### Step 1 — Apply the Gateway Configuration

Before external traffic can reach your app, you need a **Gateway** — think of it as a door into your service mesh from the outside world.

```bash
cd istio-1.10.3/
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

> **What is a Gateway?**  
> In Istio, a Gateway is a configuration object that tells the Istio Ingress Gateway which traffic to accept from outside the cluster and on which ports/hosts. Without a Gateway, no external traffic can reach your services — they are completely isolated inside the mesh.

> **What is in bookinfo-gateway.yaml?**  
> This file contains **two** Istio configurations:
> 1. A **Gateway** — opens the door for external traffic on HTTP port 80
> 2. A **VirtualService** — defines routing rules for traffic that comes through the Gateway
>
> Both are needed for external access to work. VirtualServices and Gateways will be covered in detail in later sections.

After applying, run `istioctl analyze` to confirm no validation issues.

---

### Step 2 — Configure Cluster Access

Determine the IP address and port where your local cluster is accessible, and export them as shell variables.

```bash
# Get Minikube's IP address
export INGRESS_HOST=$(minikube ip)

# Get the HTTP port from the Istio Ingress Gateway service
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway \
  -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')

# Verify they are set correctly
echo "Host: $INGRESS_HOST"
echo "Port: $INGRESS_PORT"
```

> **Why export as variables?**  
> So you can use `$INGRESS_HOST` and `$INGRESS_PORT` in subsequent commands instead of typing the full IP address and port number every time.

---

### Step 3 — Test the Product Page

Verify your setup is working by hitting the product page endpoint with cURL:

```bash
curl "http://$INGRESS_HOST:$INGRESS_PORT/productpage"
```

**Expected result:** The full HTML source of the Bookinfo product page is printed in your terminal.

You can also open the same URL in your browser:

```
http://<INGRESS_HOST>:<INGRESS_PORT>/productpage
```

> **Why do the stars change color on every refresh?**  
> Bookinfo has **three versions of the Reviews service** — v1 (no stars), v2 (black stars), v3 (red stars). Traffic is load-balanced between all three versions. Each refresh hits a different version, so the stars look different every time. This is intentional — it demonstrates Istio's traffic routing capabilities.

---

### Step 4 — Generate Continuous Traffic

The Kiali graph needs a steady stream of traffic to visualize. Run this loop to continuously send requests every 10 milliseconds:

```bash
while sleep 0.01; do curl -s "http://$INGRESS_HOST:$INGRESS_PORT/productpage" &> /dev/null; done
```

Breaking this command down:

| Part | What It Does |
|---|---|
| `while sleep 0.01` | Repeat every 0.01 seconds (100 requests per second) |
| `curl -s` | Silent mode — no progress output |
| `"http://$INGRESS_HOST:$INGRESS_PORT/productpage"` | The URL to hit — uses your exported variables |
| `&> /dev/null` | Throw away the HTML response so your terminal doesn't get flooded |

> **Important:** Copy this command carefully to avoid issues with backslashes or unexpected line-break formatting.

---

### Step 5 — Monitor Traffic in Kiali

Open the Kiali dashboard and navigate to the **Graph** section. Allow a few moments for data to be collected.

```bash
istioctl dashboard kiali
```

The graph will now show:

- Live traffic flowing: `productpage → reviews (v1, v2, v3) → ratings`
- Live traffic flowing: `productpage → details`
- All nodes and arrows **green** — healthy traffic
- **Request rates** displayed on each arrow (e.g., `2.4 rps`)
- **Healthy** status in Applications, Workloads, and Services sections

---

### Step 6 — Review Applied Istio Configurations in Kiali

Navigate to the **Istio Config** section in Kiali. You will now see the **Gateway** and **VirtualService** from the `bookinfo-gateway.yaml` file you applied.

These are the **only two Istio configurations** active at this point — no traffic routing rules yet, just the basic entry point into the mesh. You can click on each config in Kiali to inspect its YAML definition directly in the browser.

---

## 7. Simulating a Fault — Seeing Kiali Detect Problems

This is one of the most powerful demonstrations of why Kiali exists. We intentionally break something and watch Kiali immediately detect and display it.

---

### The Fault — Delete the ProductPage Deployment

```bash
kubectl delete deployments/productpage-v1
```

This simulates a real production failure — a service going completely down.

---

### What Happens in Kiali — Timeline

After executing the delete command, return to the Kiali dashboard and watch:

| Timing | What Kiali Shows |
|---|---|
| **Immediately** | The productpage node may appear "half-available" — Kiali retains data from the last minute, so it hasn't fully registered the failure yet |
| **After ~1 minute** | The productpage node turns **dark red** — all requests to it are returning `500` errors |
| **In Applications menu** | productpage disappears from the health column — no health data available because the pod is gone |
| **In Workloads menu** | `productpage-v1` vanishes from the list entirely |
| **In Services menu** | **Question marks** appear in the health column — health data is unavailable because no pods are backing the service |
| **In the Graph** | The cURL loop is still running but getting only errors — no traffic reaches reviews, ratings, or details because productpage (the entry point) is gone |

---

### Why This Matters

> In a real production system with dozens of microservices, you might not immediately know which service failed. Kiali shows you **exactly which node turned red** and **which downstream services stopped receiving traffic** — in real time, from a single screen.
>
> Without Kiali, you would have to grep through logs across every service individually to find the failing one.

---

### The Cascading Failure Concept

This demo also illustrates **cascading failure** — when productpage fails, none of the other services (reviews, ratings, details) receive any traffic either, even though they themselves are still perfectly healthy. Their pods are still running and ready — but nobody is calling them.

Kiali shows you this **entire picture at once** — which services are failing, which are healthy but idle, and how the failure radiates through the mesh.

---

## 8. Key Terms Glossary

| Term | Simple Explanation |
|---|---|
| **Kiali** | Web-based visual dashboard for Istio. Shows live topology, traffic flow, health status, and Istio configurations |
| **Prometheus** | Collects metrics from every Envoy sidecar (request rates, error rates, latency). Kiali reads from it to show numbers |
| **Grafana** | Charts and graphs metrics from Prometheus — shows trends over time like a monitoring dashboard |
| **Jaeger** | Distributed tracing — follows a single request through all microservices it touches, like a package tracking system |
| **Zipkin** | Another distributed tracing format. Jaeger supports the Zipkin format for compatibility — that's why you see `service/zipkin` created |
| **Add-on** | An optional tool that extends Istio's functionality. Kiali, Grafana, Jaeger, and Prometheus are all Istio add-ons |
| **Gateway (Istio)** | An Istio config object that opens a "door" for external traffic to enter the mesh on specific ports/hosts |
| **VirtualService** | An Istio config that defines routing rules — which traffic goes where, with what conditions |
| **Topology** | The map/structure of how services are connected — which service talks to which |
| **Workload** | In Kiali, a Workload = a Kubernetes Deployment. It's what actually runs your containers |
| **MonitoringDashboard CRD** | A custom Kiali resource type — pre-built dashboard templates for different languages (Go, Spring Boot, Vert.x, etc.) |
| **Port 20001** | The default port Kiali runs on. When you launch the dashboard, it opens at `http://localhost:20001/kiali` |
| **INGRESS_HOST** | Shell variable storing the cluster IP address — used to construct the URL to reach your app from outside |
| **INGRESS_PORT** | Shell variable storing the port of the Istio Ingress Gateway |
| **Cascading Failure** | When one service fails, other services that depend on it stop receiving traffic too — even if they themselves are healthy |
| **Request Rate** | How many HTTP requests per second are flowing through a service. Shown on graph arrows in Kiali |
| **Latency** | How long a request takes to get a response. Kiali shows this per service |
| **Distributed Tracing** | Following a single user request as it travels through multiple microservices — Jaeger does this |
| **Scraping** | How Prometheus collects metrics — it periodically calls (scrapes) each pod's metrics endpoint to pull data |
| **ServiceAccount** | A Kubernetes identity for a pod or tool to authenticate itself within the cluster |
| **ClusterRole** | Defines what actions are allowed across the entire cluster |
| **ClusterRoleBinding** | Connects a ClusterRole to a ServiceAccount — activates the permissions for a specific identity |
| **ConfigMap** | Stores configuration data as key-value pairs — like a settings file inside Kubernetes |

---

## 9. Full Command Cheatsheet

### Installation

```bash
# Install Kiali + Grafana + Jaeger + Prometheus all at once
kubectl apply -f samples/addons

# Confirm all add-on services are running in istio-system namespace
kubectl -n istio-system get services

# Launch Kiali dashboard in your browser (port 20001)
istioctl dashboard kiali
```

### Setting Up External Traffic Access

```bash
# Navigate into the Istio installation folder
cd istio-1.10.3/

# Apply the Gateway and VirtualService to allow external traffic
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml

# Verify no Istio configuration issues
istioctl analyze

# Export cluster IP and ingress port as shell variables
export INGRESS_HOST=$(minikube ip)
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway \
  -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')

# Confirm variables are set
echo "http://$INGRESS_HOST:$INGRESS_PORT/productpage"
```

### Generating Traffic

```bash
# One-time test — check if product page is reachable
curl "http://$INGRESS_HOST:$INGRESS_PORT/productpage"

# Continuous traffic loop — run this so Kiali graph fills up (100 req/sec)
while sleep 0.01; do curl -s "http://$INGRESS_HOST:$INGRESS_PORT/productpage" &> /dev/null; done
```

### Fault Simulation

```bash
# Simulate a failure — delete productpage to watch Kiali show errors in real time
kubectl delete deployments/productpage-v1

# Observe the effect — check remaining pods
kubectl get pods

# Restore the deployment (redeploy bookinfo)
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

### Useful Verification Commands

```bash
# Check that all pods are healthy (2/2 = app + Envoy sidecar)
kubectl get pods

# View all services in the default namespace
kubectl get services

# View all services in istio-system (Kiali, Grafana, etc.)
kubectl -n istio-system get services

# View all Istio configurations applied
kubectl get virtualservices,gateways

# Check if sidecar injection is enabled on the namespace
kubectl get namespace default --show-labels
```

---

## 📎 Important Notes and Reminders

- **The Kiali graph is empty without traffic.** Always run the continuous traffic loop before expecting to see anything in the graph.
- **Services may take time to load** in the Kiali Services section — this is normal. Wait and refresh.
- **The Istio Config section starts empty** — configs only appear after you apply Gateway and VirtualService YAML files.
- **Add-ons are for demo only** — Grafana, Jaeger, and Prometheus installed via `samples/addons` are not production-ready. They lack proper sizing, persistence, and security configuration.
- **Port 20001** is Kiali's default port. If it doesn't open automatically, navigate to `http://localhost:20001/kiali` manually.
- **Question marks in health columns** mean Kiali cannot get health data — usually because the pods for that service are gone or not ready yet.
- **Dark red nodes** in the Kiali graph mean 100% of requests to that service are failing with errors.

---

## 🔗 References

- [Istio Documentation](https://istio.io/latest/docs/)
- [Kiali Official Documentation](https://kiali.io/docs/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Jaeger Documentation](https://www.jaegertracing.io/docs/)
- [Kubernetes Basics](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/)

---

*Notes compiled from KodeKloud — Istio Service Mesh Course*  
*Pages: Visualizing Service Mesh with Kiali | Demo Installing Kiali | Demo Create Traffic Into Your Mesh*
