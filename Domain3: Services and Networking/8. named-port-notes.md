# Kubernetes Named Ports — Domain 3: Services and Networking

---

## 1. What is a Named Port?

> A **Named Port** is a way to give a **human-readable name** (like `http`) to a container port inside a Pod, instead of always referring to it by its number (like `80`).

Normally, when you expose a port in a Pod spec, you only specify the port number:

```yaml
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
```

With a **named port**, you also attach a name to that port:

```yaml
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
      name: http        # ◄── this is the named port
```

> The name **must be unique within the Pod** — two ports in the same pod cannot share the same name.

---

## 2. Why Named Ports — What Challenges Do They Solve?

### Challenge 1: Hardcoded Port Numbers Are Fragile

Normally, a Service references a Pod's port using its number (for example, `targetPort: 80`). If the container's port number ever changes (say from `80` to `8080`), you have to remember to update **every Service** that points to it. Forget one, and that Service stops working.

**Named ports solve this** by letting the Service reference the port by **name** (like `http`) instead of by number. As long as the Pod keeps a port named `http`, the Service keeps working — even if the actual port number behind that name changes.

---

### Challenge 2: Hard to Read and Understand

Looking at `targetPort: 80` does not tell you what the port is for. Is it HTTP? HTTPS? Metrics? A database?

**Named ports solve this** by making the spec self-documenting. `targetPort: http` clearly tells the reader what kind of traffic flows through it.

---

## 3. How Named Ports Work With Services

Inside a Service manifest, you can put the **named port** in the `targetPort` field instead of a number. Kubernetes will look up the matching port name on the pod and route traffic to that port's actual number.

```
Service spec:               Pod spec:
  ports:                      ports:
  - port: 80                  - containerPort: 80
    targetPort: http   ─────►   name: http
```

So a request that hits the Service on port `80` is forwarded to whichever container port has the name `http`.

---

## 4. Syntax

### Defining a Named Port in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
      name: http        # named port
```

### Referencing a Named Port in a Service (YAML)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kplabs-svc
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: http    # references the named port from the Pod
  selector:
    run: nginx
  type: NodePort
```

### Referencing a Named Port via CLI (`kubectl expose`)

```bash
kubectl expose pod <pod-name> --port=<service-port> --target-port=<named-port> --name=<service-name>
```

Example:

```bash
kubectl expose pod nginx --port=80 --target-port=http --name="kplabs-svc"
```

---

## 5. Examples

### Example 1: Pod **Without** a Named Port — Service Fails

**Step 1:** Create a pod with only a port number (no name):

```bash
kubectl run nginx --image=nginx --port=80 --dry-run=client -o yaml
```

Resulting Pod spec:

```yaml
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80    # no name field
```

**Step 2:** Expose this pod using a Service that references `http` as the target port:

```bash
kubectl expose pod nginx --port=80 --target-port=http --name=first-svc --type=NodePort
```

**Step 3:** Try to access it via the NodePort:

```bash
curl <worker-node-ip>:32227
# Connection refused
```

It fails because the Pod has **no port named `http`** — the Service cannot map traffic to anything.

---

### Example 2: Pod **With** a Named Port — Service Works

**Step 1:** Create a pod manifest with a named port:

```bash
kubectl run nginx2 --image=nginx --port=80 --dry-run=client -o yaml > port_name.yaml
```

Edit `port_name.yaml` and add the `name` field:

```yaml
spec:
  containers:
  - image: nginx
    name: nginx2
    ports:
    - containerPort: 80
      name: custom-http      # named port
```

**Step 2:** Apply the pod:

```bash
kubectl apply -f port_name.yaml
```

**Step 3:** Expose the pod, referencing the named port:

```bash
kubectl expose pod nginx2 --port=80 --target-port=custom-http --name=named-svc --type=NodePort
```

**Step 4:** Confirm the Service is running:

```bash
kubectl get svc
# named-svc   NodePort   ...   80:31566/TCP
```

**Step 5:** Test the connection:

```bash
curl <worker-node-ip>:31566
# Returns the nginx welcome page — works perfectly
```

The Service used the name `custom-http` to find the matching port on the Pod, mapped it to port `80`, and traffic flowed end-to-end.

---

### Example 3: Inspecting the Service YAML

```bash
kubectl get svc named-svc -o yaml
```

Output snippet:

```yaml
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: custom-http    # name, not a number
  selector:
    run: nginx2
  type: NodePort
```

Notice the `targetPort` is the **name** `custom-http`, not the number `80`.

---

## 6. Exercises to Practice for the CKAD Exam

### Imperative (CLI-based) Exercises

**Exercise 1 — Expose with a Named Port Target**
1. Create a Pod called `web` using `nginx` (any way) — but make sure it has a named port `http` on `80` (you may need declarative for this; see Exercise 5).
2. Use `kubectl expose pod web --port=80 --target-port=http --name=web-svc --type=NodePort`.
3. Confirm the service works via `curl <node-ip>:<nodeport>`.

**Exercise 2 — Generate Pod YAML for Editing**
1. Run `kubectl run nginx --image=nginx --port=80 --dry-run=client -o yaml` to generate a base Pod manifest.
2. Save the output to a file `pod.yaml`.
3. Note that the generated YAML has only `containerPort: 80` — it does **not** include a name.

**Exercise 3 — Negative Test (Imperative)**
1. Create a Pod named `nginx` with `nginx` image (do **not** add a named port).
2. Expose it with `kubectl expose pod nginx --port=80 --target-port=http --name=first-svc --type=NodePort`.
3. `curl <node-ip>:<nodeport>` — observe `connection refused` and explain *why* it fails.

---

### Declarative (YAML-based) Exercises

**Exercise 4 — Pod with a Single Named Port**
1. Write a Pod manifest for `web` using the `nginx` image.
2. Under `containers[0].ports`, set `containerPort: 80` and `name: http`.
3. Apply and verify the named port appears in `kubectl get pod web -o yaml`.

**Exercise 5 — Service Referencing the Named Port**
1. Write a NodePort Service manifest with selector matching the pod from Exercise 4.
2. Set `port: 80` and `targetPort: http` (the name, not the number).
3. Apply and confirm `kubectl describe svc <name>` shows the correct endpoint.

**Exercise 6 — Multiple Named Ports in One Pod**
1. Write a Pod manifest with two named ports: `http` on `80` and `https` on `443`.
2. Ensure both names are unique within the pod.
3. Create two Services — one with `targetPort: http`, the other with `targetPort: https`.
4. Verify both Services route correctly to the same pod on different ports.

**Exercise 7 — Negative Test (Declarative)**
1. Write a Pod manifest **without any named port** (just `containerPort: 80`).
2. Write a Service manifest with `targetPort: custom-http`.
3. Apply both and try to `curl` through the service. Confirm the request fails and explain why.

**Exercise 8 — Refactor: Number → Name**
1. Take any working Pod + Service pair using `targetPort: 80`.
2. Edit the Pod to add `name: http` to its port.
3. Edit the Service to change `targetPort` from `80` to `http`.
4. Confirm the application still works — proving that name-based routing is equivalent.

---

## 7. Summary

| Concept | Detail |
|---|---|
| **What it is** | A name (like `http`) attached to a container port in a Pod |
| **Where it goes** | Under `spec.containers[].ports[].name` |
| **Uniqueness rule** | The name must be unique within the Pod |
| **Where it is used** | In a Service's `targetPort` field — instead of a port number |
| **Why use it** | Easier to read; Pod port numbers can change without breaking Services |
| **CLI usage** | `kubectl expose pod <pod> --port=80 --target-port=<named-port> --name=<svc>` |
| **Mismatch behaviour** | If the named port does not exist in the Pod, the Service routes nowhere → `connection refused` |
