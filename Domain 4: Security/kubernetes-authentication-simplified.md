# Kubernetes Authentication — Simplified Notes

---

## 1. What is Authentication?

> **Authentication** is the process of **verifying who you are** before Kubernetes lets you do anything.

Every request you send — whether it is `kubectl get pods`, creating a deployment, or reading a secret — first goes to the **API Server**.

Before the API Server processes your request, it asks:

> *"Who are you? Prove your identity."*

If you cannot prove it, the API Server immediately rejects you with:

```
401 Unauthorized
```

If you prove it successfully, the API Server notes down your **username** and moves to the next step: **Authorization** (checking what you are allowed to do).

**Authentication always comes first.**

---

## 2. Authentication vs Authorization

| Term | Question it Answers |
|------|---------------------|
| **Authentication** | "Who are you?" |
| **Authorization** | "What are you allowed to do?" |

You must pass authentication **before** authorization is even checked.

---

## 3. Why is Authentication Needed?

### Challenge 1: The API Server is Exposed Over the Network
The Kubernetes API Server listens on port **6443**. Anyone who can reach this port could send commands. Without authentication, they could delete your entire cluster or read all your secrets.

### Challenge 2: Many Different Clients Connect
- Human developers using `kubectl`
- CI/CD pipelines deploying automatically
- Applications running inside pods that need to talk to Kubernetes
- Monitoring and logging tools

Each needs a way to prove identity.

### Challenge 3: Kubernetes Does Not Store Human Users
There is **no** command like:
```bash
kubectl create user alice
```

Kubernetes does **not** have a built-in user database. It only looks at the credential you present (certificate, token, etc.) and extracts your name from it.

The only identity Kubernetes stores natively is the **ServiceAccount** (used by apps inside pods).

---

## 4. What is `kubeconfig`?

The `kubeconfig` file is a configuration file on your local machine that tells `kubectl`:
1. **Which cluster** to talk to (API Server address)
2. **Who you are** (your credentials — certificate, token, etc.)
3. **Which combination** is active right now (context)

### Where is it located?
| Operating System | Path |
|------------------|------|
| Linux / Mac | `~/.kube/config` |
| Windows | `%USERPROFILE%\.kube\config` |

### Why is it needed?
When you type `kubectl get pods`, `kubectl` does not know which cluster to contact or how to prove your identity. It reads this file to find out.

### How to inspect it
```bash
# View the full kubeconfig
kubectl config view

# View only the current active connection
kubectl config view --minify

# View raw credentials (including base64 certificates)
kubectl config view --minify --raw

# See which context is active
kubectl config current-context
```

---

## 5. Categories of Users in Kubernetes

| Category | Used By | Stored in Cluster? | Typical Auth Method |
|----------|---------|-------------------|---------------------|
| **Normal Users** | Human beings (developers, admins) | **No** — managed externally | X509 certificates, OIDC tokens, static token file |
| **Service Accounts** | Applications / processes inside pods | **Yes** — Kubernetes objects | Service account tokens (JWT) |

> **Why two categories?** Giving client certificates to every application inside every pod is difficult. ServiceAccounts give apps a simple, Kubernetes-native way to authenticate.

---

## 6. Authentication Methods

### Method 1: X509 Client Certificates (Default for kubeadm)

> Each user has their own **certificate** and **private key**. The **Common Name (CN)** inside the certificate becomes the **username**. The **Organization (O)** field becomes the **group**.

This is the default method for clusters created with `kubeadm`.

#### How it looks in kubeconfig
```yaml
users:
- name: kubernetes-admin
  user:
    client-certificate-data: LS0tLS1CRUdJTi...   # base64-encoded certificate
    client-key-data: LS0tLS1CRUdJTi...           # base64-encoded private key
```

#### How to practice

**Step 1:** See your current user and context
```bash
kubectl config current-context
```

**Step 2:** Extract your certificate from kubeconfig
```bash
kubectl config view --minify --raw -o jsonpath='{.users[0].user.client-certificate-data}'
```
Copy the long base64 string that is printed.

**Step 3:** Decode it to see your username and group
```bash
echo "<paste-the-base64-string-here>" | base64 -d | openssl x509 -text -noout | grep "Subject:"
```

**Expected output:**
```
Subject: O = system:masters, CN = kubernetes-admin
```

This means:
- **Username:** `kubernetes-admin`
- **Group:** `system:masters`

> The certificate's **CN** is your identity. Kubernetes does not store you anywhere — it just reads your certificate and trusts what is written inside.

---

### Method 2: Static Token File

> The API Server reads bearer tokens from a **CSV file** that you provide. Each line maps a token to a username.

#### Is the file present by default?
**No.** You must create it yourself.

This method only works on **self-managed clusters** where you control the API Server (e.g., `kubeadm`). It does **not** work on managed clusters like EKS, GKE, AKS, or DigitalOcean.

#### CSV File Format
The file needs at least **3 columns**, optionally 4:

```
token,username,uid,groups
```

| Column | Meaning | Required |
|--------|---------|----------|
| 1st | The secret token (password) | Yes |
| 2nd | The username Kubernetes will use | Yes |
| 3rd | A unique user ID (UID) | Yes |
| 4th | Group membership | Optional |

#### Example file (`/root/token.csv`)
```
Dem0Passw0rd#,bob,01,admins
MyS3cretT0k3n,alice,02,"developers,editors"
```

#### How to create it

**Step 1:** Log in to your **control plane node** (master node).

**Step 2:** Create the CSV file
```bash
cat > /root/token.csv <<EOF
Dem0Passw0rd#,bob,01,admins
AnotherT0ken#,alice,02,developers
EOF
```

#### How to tell the API Server about it

The API Server must be started with this flag:
```bash
--token-auth-file=/root/token.csv
```

On a kubeadm cluster, add this to the API Server manifest:
```bash
vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

Find the `command:` section and add the flag:
```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --token-auth-file=/root/token.csv    # ← ADD THIS LINE
    # ... other existing flags
```

**Save the file.** The kubelet automatically detects the change and restarts the API Server within seconds.

#### How to verify it is working

**Step 1:** Check if the API Server picked up the flag
```bash
ps aux | grep kube-apiserver | grep token-auth-file
```

**Step 2:** Test with the correct token
```bash
curl -k https://<api-server-ip>:6443/api/v1/pods \
     -H "Authorization: Bearer Dem0Passw0rd#"
```
> Should return the list of pods.

**Step 3:** Test with a wrong token (should fail)
```bash
curl -k https://<api-server-ip>:6443/api/v1/pods \
     -H "Authorization: Bearer WrongToken123"
```
> Should return `401 Unauthorized`.

#### Using the token with kubectl

You can add this token to your local kubeconfig:

```bash
# Add the user
kubectl config set-credentials bob-user --token=Dem0Passw0rd#

# Create a context using this user
kubectl config set-context bob-context \
    --cluster=<your-cluster-name> \
    --user=bob-user

# Switch to that context
kubectl config use-context bob-context

# Test
kubectl auth whoami
kubectl get pods
```

> ⚠️ **Warning:** Static token files are plain text passwords on disk. They are **not recommended for production**. Use only for learning, exams, or temporary testing.

---

### Method 3: Service Account Tokens

> When an **application inside a pod** needs to talk to the API Server, Kubernetes automatically provides a **JWT token**. This token is mounted inside the pod at a specific path.

ServiceAccounts are the **only** identity that Kubernetes stores natively as objects.

#### Where the token lives inside a pod
```
/var/run/secrets/kubernetes.io/serviceaccount/token
```

#### How to practice

**Step 1:** Create a ServiceAccount
```bash
kubectl create serviceaccount my-app-sa
```

**Step 2:** Create a Pod that uses this ServiceAccount
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  serviceAccountName: my-app-sa
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
```

Apply it:
```bash
kubectl apply -f pod.yaml
```

**Step 3:** Read the token from inside the pod
```bash
kubectl exec test-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
```
> This prints a long JWT string — the pod's proof of identity.

**Step 4:** Generate a temporary token manually
```bash
kubectl create token my-app-sa --duration=1h
```
> This creates a short-lived token valid for 1 hour.

**Step 5:** Test the token with curl
```bash
curl -k https://<api-server-ip>:6443/api/v1/pods \
     -H "Authorization: Bearer <paste-token-here>"
```

---

## 7. How to Know Which File is Present Where

### kubeconfig file
```bash
ls ~/.kube/config
```

### Current active context
```bash
kubectl config current-context
```

### Certificates on the control plane node
```bash
ls /etc/kubernetes/pki/
```

### Admin kubeconfig on the server
```bash
ls /etc/kubernetes/admin.conf
```

### Static token file location
Check the API Server configuration:
```bash
# Method 1: Check running process
ps aux | grep kube-apiserver | grep token-auth-file

# Method 2: Check the manifest file
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep token-auth-file
```

### ServiceAccounts and their secrets
```bash
# List all service accounts
kubectl get serviceaccounts

# View details of a specific service account
kubectl get serviceaccount default -o yaml

# List all secrets (including tokens)
kubectl get secrets
```

---

## 8. Practice Exercises

### Exercise 1: Inspect Your kubeconfig
```bash
kubectl config view
kubectl config current-context
kubectl config view --minify --raw
```

### Exercise 2: Decode Your Client Certificate
```bash
# Extract the certificate
kubectl config view --minify --raw -o jsonpath='{.users[0].user.client-certificate-data}'

# Decode and inspect
echo "<base64-string>" | base64 -d | openssl x509 -text -noout | grep "Subject:"
```

### Exercise 3: Test Authentication with curl + Token
```bash
# Get a service account token
kubectl create token <service-account-name> --duration=1h

# Use it in a curl request
curl -k https://<api-server>:6443/api/v1/pods \
     -H "Authorization: Bearer <token>"
```

### Exercise 4: Force an Authentication Failure
```bash
curl -k https://<api-server>:6443/api/v1/pods \
     -H "Authorization: Bearer ThisIsAFakeToken"
```
> Confirm the response is `401 Unauthorized`.

### Exercise 5: Create and Use a ServiceAccount
```bash
# Create the service account
kubectl create sa my-sa

# Create a pod using it (see YAML in Method 3)
kubectl apply -f pod.yaml

# Verify the token is mounted
kubectl exec test-pod -- ls /var/run/secrets/kubernetes.io/serviceaccount/
```

### Exercise 6: Set Up Static Token Authentication
```bash
# On the control plane node:
# 1. Create /root/token.csv
# 2. Add --token-auth-file=/root/token.csv to /etc/kubernetes/manifests/kube-apiserver.yaml
# 3. Verify with: ps aux | grep token-auth-file
# 4. Test with curl using the correct token
# 5. Test with curl using a wrong token
```

### Exercise 7: Identify Auth Method on Different Clusters
```bash
# Look at your local kubeconfig for multiple clusters
kubectl config view

# For each cluster, identify:
# - Does it use client-certificate-data? → X509 Certificates
# - Does it use token? → Token-based
# - Does it use exec? → External plugin (like AWS IAM, GKE auth)
```

---

## 9. Summary Table

| Concept | Detail |
|---------|--------|
| **What it is** | Verifying who the requester is |
| **Comes before** | Authorization (checking permissions) |
| **Native user management?** | **No** — Kubernetes does not store normal users |
| **Two user categories** | Normal Users (humans, external) and Service Accounts (apps, native) |
| **Common methods** | X509 certificates, static token file, service account tokens, OIDC, webhook |
| **kubeadm default** | X509 client certificate authentication |
| **Managed cluster default** | Often token-based (varies by provider) |
| **Static token file** | CSV format `token,username,uid,groups`; pointed at by `--token-auth-file` |
| **X509 auth identity** | CN = username; O = group |
| **ServiceAccount token** | Mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token` |
| **Failure response** | `401 Unauthorized` |
| **Check identity** | `kubectl auth whoami`, `kubectl config view` |

---

## 10. Key Takeaways

1. **Kubernetes does not store human users.** It only reads your credential and extracts your name from it.

2. **Authentication always happens first.** No identity proof = immediate rejection.

3. **kubeconfig** is just a local file holding your cluster addresses and credentials.

4. **X509 Certificates** are the default for self-managed clusters. Your certificate's CN is your username.

5. **Static Token Files** must be created manually and configured via the API Server manifest. They are plain text and not for production.

6. **ServiceAccounts** are for pods. Kubernetes automatically mounts a token inside the pod.

7. **If your credential is wrong or missing → `401 Unauthorized`.**

---

*Happy Learning!* 🚀
