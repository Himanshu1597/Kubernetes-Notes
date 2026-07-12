# FirstAlt — Architecture Walkthrough (Explained)

> Reconstructed from the architecture diagram (`architecture.png`) and the session
> transcript (`architecture.txt`). Speaker: **Venkateswarlu Vajrala (VV)** presenting;
> questions mostly from **Saptak Takalkar**, plus **Ayushi Soni**, **Parth Suva**,
> and **Hareesh Kumar Gaddam**.
>
> The auto-transcript garbled many technical terms. Section 0 lists the corrections
> so the rest of the document reads cleanly.

---

## 0. Transcription corrections (what was said vs. what was meant)

| Transcript captured | Actually means |
|---|---|
| "0.53", "Proud 53" | **Route 53** |
| "CloudFriend", "CloudFriend Edge" | **CloudFront** (CloudFront Edge) |
| "West", "US East 1" | **us-west-2**, **us-east-1** regions |
| "Argent group" / "agent group" switch | CloudFront **origin group** switch |
| "First Start", "First Alert", "festival", "Fast 2A", "pastalt", "First ALT" | **FirstAlt** (the platform being described) |
| "FirstView" | **FirstView** (sister product; built the new District View portal) |
| "Distinct View", "DISTV", "distinct KP" | **District View Portal** |
| "Data Guardian", "DMT", "DMP", "Dave" | **Data Guardian Portal** (internal app; referred to as DMT) |
| "Lambra" | **Lambda** |
| "Cognitive", "community tokens" | **Cognito**, Cognito tokens |
| "Azure Ready", "Azure AD" | **Azure AD** |
| "IODC" | **OIDC** |
| "SSO2 data guardian" | **SSO to** Data Guardian |
| "JOT token" | **JWT** token |
| "Aurora Postgres SQL", "Postgree SQL" | **Aurora PostgreSQL** |
| "LATAM values", "LATAM" | **lat/long** (latitude & longitude) |
| "toggle J library", "toggle the library", "Fisher Toggle Service" | **Togglz** library, **Feature Toggle Service** |
| "rollback RDAC API", "role-based access control" | **RBAC API** (Role-Based Access Control) |
| "DMS AP", "DMSA" | **DMS API** = **Data Management Service** (NOT AWS DMS) |
| "Future Toggle Service" | **Feature Toggle Service** |
| "eCase worker notes", "EK Service", "node ports in EKs" | **EKS** worker nodes / EKS services / NodePorts |
| "32700" | EKS **NodePort 32700** |
| "MSCA" | **MSK** (Managed Streaming for Apache Kafka) |
| "MirrorMaker", "mirror maker" | Kafka **MirrorMaker** |
| "Tripblueprints", "trip hyphen blueprints", "source.trip blueprints" | topic **trip-blueprints** with **source.** prefix on replication |
| "SSN Parameter store" | **SSM Parameter Store** |
| "FluentBit", "fluid method" | **Fluent Bit** |
| "AWS Distro for OpenTelemetry" (daemon set) | **ADOT** (AWS Distro for OpenTelemetry) |
| "GoCD", "Thoughtbox people" | **GoCD** (by **ThoughtWorks**) |
| "actions ARC controller" | GitHub **Actions Runner Controller (ARC)** |
| "Proud 53 Private Hosted Room" | **Route 53 Private Hosted Zone** |
| "division installed", "numer Open source worksheet" | **Debezium** (open-source CDC tool) |
| "glue schema registry" | **AWS Glue Schema Registry** |
| "slow flake" | **Snowflake** |
| "Power BI Gateway" | **Power BI Gateway** |
| "CADA ranges", "network shader ranges" | **CIDR ranges** |
| "code commit repo" | **AWS CodeCommit** repo |
| "VAPs", "web support" | **WAF** (Web Application Firewall) |
| "HTTP APAS", "IPA gateway", "APA Gateway" | **HTTP APIs** in **API Gateway** |
| "Intrinsic encryption" | **in-transit encryption** |
| "conditional forwarders" + "Route 53 inbound endpoints" | **Route 53 Resolver inbound endpoints** with DNS conditional forwarders |
| "FSIT", "FSITT" | **FS IT** team (First Student IT / central networking) |
| "the English" (Hareesh) | **the ingestion** |
| "Karchif", "Jeff" | people (data platform side) |

---

## 1. The big picture

FirstAlt is a school-transportation platform running on AWS as a **multi-region,
active–passive** deployment:

- **Primary region:** `us-east-1`
- **Secondary / DR region:** `us-west-2`

Failover between regions is **manual and self-initiated** (no automated health checks —
see §8). Each region is spread across two AZs (`us-east-1a`/`1b`, `us-west-2a`/`2b`).

There are **4 main client applications**:

1. **Driver App** — native **iOS and Android**, used by drivers running routes for students.
2. **Transportation Partner Portal (TSP / SP)** — used by transportation partners who
   manage the drivers they supply under contract.
3. **District View Portal** — a **legacy** portal still supported for business/infra
   reasons (a newer District View was built by **FirstView**, but the old one lives on).
4. **Data Guardian Portal (DMT)** — internal-facing app for FS employees. Handles
   trip management, blueprints, school schedules, reporting, etc.

---

## 2. Diagram legend (the numbered callouts 1–8)

| # | What it marks in the diagram |
|---|---|
| **1** | Client apps → **Route 53** (single entry point for all apps) |
| **2** | Web path: Route 53 → **CloudFront** → **S3** (with Primary Flow + DR Flow arrows to both regions) |
| **3** | API path: **API Gateway** → **VPC Link** → into the private subnet |
| **4** | **EKS** worker nodes (Node1 / Node2) running the services |
| **5** | **Aurora PostgreSQL Global Database** — Writer/Reader + Reader (primary), Reader (secondary) |
| **6** | **DynamoDB Global Tables** |
| **7** | **MSK (Kafka)** brokers + **Kafka MirrorMaker** + **VPC Peering Connection** between regions |
| **8** | **Bastion Host** (in the public subnet of the secondary region) |

Supporting services shown per region: CloudWatch, Cognito, EventBridge, AWS Batch,
QuickSight, Secrets Manager, KMS, Certificate Manager, Pinpoint, SNS, IGW, NAT GW.

---

## 3. Web application flow (static hosting)

All four web apps are **static** — no dynamic server-side rendering, no app hosting.

**Serving path:** `Route 53 → CloudFront → S3`

**Deployment:**
- A new version = publish the bundle to CloudFront + perform a **cache invalidation**,
  which pushes the new version to all clients.

**DR for web (S3):**
- Deploy only to the **primary** S3 bucket; a separate copy exists in the secondary region.
- **Bi-directional replication** is set up between the per-app S3 buckets, so content is
  copied to `us-west-2`.
- Both buckets are wired into CloudFront as an **origin group**; traffic is steered by the
  origin-group switch.
- Normally `us-east-1` S3 is primary; during DR, the `us-west-2` bucket is promoted to primary.

### 📖 In plain English — Deployment & DR (with diagram)

**Mental model:**
- **S3 = the warehouse** — where the actual app files (the HTML/JS "bundle") physically live.
- **CloudFront = a chain of local corner shops** around the world, close to users. Each shop
  keeps a **copy** of the files so users get them fast. Users are served by the nearest **shop
  (CloudFront)**, not the warehouse.

**Part 1 — Deployment (shipping a new version):**
The problem: the corner shops hold onto their copies. If you only update the warehouse, the shops
keep handing out the **old** copy. So deployment is two steps:
1. **Put the new bundle in the warehouse (S3).**
2. **Cache invalidation** = tell every shop *"throw away your old copy."* Next time a user asks,
   the shop has nothing, grabs the fresh version from the warehouse, and hands that out.

That second step is what pushes the new version to **all** users at once.

> In short: upload new files → tell CloudFront to forget the old ones → everyone gets the new version.

**Part 2 — DR (surviving a region outage):**
- **Two warehouses:** Warehouse A in us-east-1 (primary), Warehouse B in us-west-2 (backup).
- **Deploy to A only** — devs don't want to upload to two places.
- **Auto-copy (bi-directional replication):** S3 automatically copies A → B, so B is always an
  up-to-date identical copy. "Bi-directional" = it can also flow the other way after a failover.
- **CloudFront knows both (origin group):** both warehouses are configured as sources, but only
  **one** is the active one to pull from at a time.
- **The switch:** normally A (east) is active. If east dies, they **flip the switch** so CloudFront
  pulls from **B (west)** instead. Since B is already up to date, users barely notice.

> In short: deploy to one bucket → AWS auto-copies it to a second region → CloudFront knows both →
> flip a switch to serve from the backup if the main region fails.

**The whole thing in one picture:**

```
Deploy once  →  [ S3 East ]  ──auto-copy──►  [ S3 West ]
                     │                            │
                     └──────► CloudFront ◄────────┘
                          (knows both; uses East
                           normally, flips to West
                           if East goes down)
                               │
                               ▼
                             Users
```

Recap:
- **Deployment** = new files + "forget the old cache."
- **DR** = a second auto-synced copy in another region, with a switch to fail over to it.

---

## 4. Backend / API flow (step by step)

Entry point for **all** mobile (driver) apps and web apps is again **Route 53**.

1. **Route 53 — weighted records** point to the **API Gateways** in both regions.
   - `us-east-1` weight = **100**, `us-west-2` weight = **0**.
   - DR flow flips these: west = 100, east = 0.

2. **API Gateway** (HTTP APIs, not REST — see §16) handles:
   - **Rate limiting** (how many requests can be sent).
   - **Authentication** via a **Lambda authorizer** attached per route.

3. **Lambda authorizer** (see §5 and §12):
   - Each route is configured with which **Cognito user pool** and/or **Azure AD** is allowed.
   - It reads the **JWT** (from Cognito or Azure), validates it, and checks the token belongs
     to a user pool / Azure AD configured for that route. If valid → proceed.

4. **VPC Link** → **Application Load Balancer** (internal, private — never public-facing).

5. **ALB → EKS** using **NodePort + path matching**:
   - Each service has its own **target group**; ALB **listeners** map path → NodePort.
   - Example: a `/districtview` request for the **DMS (Data Management Service)** maps to an
     ALB listener on NodePort **32700** → forwarded to the EKS **worker nodes** listening on
     that port → routed to the internal **EKS service** → to a **pod**.
   - Services registered this way include: **DMS API** (Data Management Service),
     **Trip Scheduler API**, **Blueprint API**, **RBAC API** (authorization),
     **Feature Toggle Service API**, **Document Management Service API**.

6. **Pod** processes the request and returns the response.

> **Note on the NodePort design (Q&A):** Saptak asked why NodePort instead of an
> **Ingress Gateway** with services as **ClusterIP**. VV: likely a decision from the early
> phase when there were only 4–5 services; even now (~3 more added) they haven't added that
> extra layer. Security is covered by SGs (only the ALB SG can reach the worker-node ports).

---

## 5. Authentication — why both Cognito *and* Azure AD

- **Authentication** happens at the **API Gateway (Lambda authorizer)** level.
- **Two mechanisms**: **Cognito** (primary) and **Azure AD**. Primary token type is **JWT**
  for both.

**History / rationale (Q&A "why both?"):**
- Originally **Cognito** was the single auth provider for *all* user bases — TSP/SP portal,
  Data Guardian (DMT), District View, driver app.
- They wanted **SSO** specifically for the **Data Guardian Portal** (internal, all client/FS
  users). Three approaches were evaluated:
  1. Cognito federated with **Azure AD via SAML**
  2. Cognito federated with **Azure AD via OIDC**
  3. **Direct integration with Azure AD**

**Why not stay all-Cognito:**
- **Cost:** federating a *non-social* (third-party) IdP into Cognito is priced **per user** — expensive.
- **DR limitation:** Cognito has **no multi-region DR**. You must **sync users** between user
  pools in each region, but **passwords are not synced**. On failover, every user must
  **reset their password**; on failback to east, they must reset again. (They only force reset
  for users who actually reset in the other region — since they can't know the other region's password.)
- AWS said they're working on integrating **DynamoDB with Cognito** for a global password sync
  (was targeted "by Q2" in a prior cycle) — needs follow-up.

**Where each is used today (Q&A):**
- Only **DMT (Data Guardian)** currently uses **Azure AD** → users see **"Sign in with Microsoft"**.
  All FS employees are in Azure.
- The **other apps** (District View, etc.) still use **Cognito** (username/password → request
  goes to Cognito), with the password-reset DR limitation.
- **No**, a user does **not** get two JWTs (one Cognito + one Azure); it's one or the other per app.

---

## 6. Application-level authorization

- **Authentication** = "is the token valid / is the user allowed on this route?" → done at API Gateway.
- **Authorization** = "is *this* user allowed to access *this* route/feature?" → done **inside the app**.
- Handled by a dedicated **RBAC (Role-Based Access Control) service** — one of FirstAlt's own
  **Java** services (clarified in Q&A: these are *our services*, not AWS services). It handles
  both Cognito and Azure tokens.

---

## 7. Feature toggles

- Each service also checks whether a feature/route is **enabled** via the **Feature Toggle Service**
  (another pod in the EKS cluster).
- Uses the **Togglz** Java library; toggle state (feature name + active flag) is kept in the database.
- Toggle values are changed through a **console endpoint** with **password-based auth** (log in, flip the toggle).

---

## 8. Data layer

**Aurora PostgreSQL — Global Database (primary store for all data):**
- Deployed as a **global database** with replicas in the other region.
- **Primary region:** 1 **Writer/Reader** + 1 **Reader**.
- **Secondary region:** 1 **Reader**.

**DynamoDB:**
- Primarily stores **trip latitude/longitude**.
- As the driver drives, the app batches lat/long (every ~15 seconds or a configured interval)
  and sends them to the **Trip Scheduler Service**, which stores them in the **DynamoDB** table
  (partition key = **trip ID**).
- DynamoDB uses **Global Tables** with **built-in bi-directional replication** (AWS-managed).

### 📖 In plain English — the data layer (with FirstAlt example)

**Mental model:** FirstAlt keeps data in **two very different stores**, each for a different job:
- **Aurora PostgreSQL = the main filing cabinet** (the system of record for everything important).
- **DynamoDB = a fast live-tracking whiteboard** (only for the constant stream of bus GPS pings).

**Aurora PostgreSQL (the core data):**
- Holds all the important stuff — students, drivers, trips, blueprints, schedules.
- **"Global database"** = one place does the **writing**, and read-only **copies (replicas)** are kept
  in the other region. In the **primary** region there's 1 **Writer/Reader** (can write *and* read) plus
  1 **Reader** (read-only, to share the query load); in the **secondary** region 1 **Reader** kept in
  sync and ready for DR.
- Why: writes funnel to one place (keeps data consistent), reads spread out (faster), and the west
  copy means the data is already there if east dies.

**DynamoDB (the live GPS):**
- A separate, ultra-fast database used **only** for the driver's live location (lat/long).
- Why not just use Aurora? The bus sends its position **every ~15 seconds** — a firehose of tiny
  writes. DynamoDB is built exactly for that high-volume, high-speed pattern. All pings for one trip
  are grouped under **partition key = trip ID**.
- **Global Tables** = AWS automatically copies the data to the other region **both ways**, no manual work.

**FirstAlt example:**
- An admin creates a blueprint or edits a student's details → saved in **Aurora**.
- Ravi's bus pings its GPS mid-route → saved in **DynamoDB** under **trip #5501**.

```
Core data (students, trips, blueprints)  → Aurora PostgreSQL (1 Writer + Readers)
Live GPS pings every ~15s (trip #5501)   → DynamoDB (partition key = trip ID)
        both auto-replicated to us-west-2 for DR
```

---

## 9. Async / eventing (Kafka)

Internal service-to-service communication is a mix of **direct calls** and **Kafka** as an intermediary.

**Blueprint → trips example (async):**
- At the start of a **new school year**, a **blueprint** is created (which students, which
  drivers, which dates, leaves, etc.).
- **Publishing the blueprint** emits an event to **Kafka** → the **Trip Scheduler Service (TSS)**
  consumes it → **generates trips for the whole year** and updates the database.

### 📖 In plain English — async & Kafka (with FirstAlt example)

**Mental model:** **Kafka is a conveyor belt** between services. Instead of one worker doing a giant
job while everyone waits, a service **drops a message on the belt and moves on**; a different worker
picks it up and does the heavy work in the background.

**Two ways FirstAlt's services talk:**
- **Direct call** = "I'll ask you and wait for the answer" (synchronous — fine for quick things).
- **Kafka** = "I'll drop a message and carry on; someone processes it later" (asynchronous — good for
  big/slow jobs, and it keeps services independent).

**FirstAlt example — publishing a blueprint:**
- At the start of the school year an admin builds a **blueprint** (the master plan: which driver takes
  which students, on which dates, holidays, etc.).
- Publishing it can mean generating **tens of thousands of trips** for the whole year — far too slow to
  make the admin sit and wait.
- So "publish" just drops **one event** onto Kafka and instantly returns.
- The **Trip Scheduler Service** is listening, picks up that event, and **generates all the trips in
  the background**, writing them to Aurora.

```
Admin clicks "Publish blueprint"
        │ drops ONE event
        ▼
     [ Kafka ] ───► Trip Scheduler Service (background worker)
                         │ generates the whole year's trips
                         ▼
                    Aurora (trips saved)
Admin doesn't wait — gets an instant "done, generating…" response
```

---

## 10. Disaster Recovery (DR) design

**Failover model:** manual, self-initiated. **No active Route 53 health checks** — they want to
be the ones who trigger DR (Q&A confirmed: it's a manual Route 53 change to switch to `us-west-2`).

Per-layer DR posture:

| Layer | DR setup |
|---|---|
| **Route 53** | Weighted records; flip API Gateway weights east↔west (100/0 → 0/100) |
| **API Gateway** | Present in both regions |
| **ALB** | Present in the second region |
| **EKS** | **Control plane only** in secondary; **worker nodes = 0** (scaled up on failover) |
| **Aurora** | Global DB; **1 Reader** replica in secondary |
| **DynamoDB** | **Global Tables**, self-managed bi-directional replication |
| **Kafka (MSK)** | Cluster in **both** regions + **MirrorMaker** |
| **S3 (web)** | Bi-directional bucket replication + CloudFront origin-group switch (see §3) |

**Kafka DR — MirrorMaker with topic prefixing:**
- MSK has **no global clusters**; each region has its own cluster (linked via **VPC Peering**).
- A **MirrorMaker** instance (running in `us-west-2`) replicates events **bi-directionally**
  with a **source-cluster prefix**.
- Example: an event published to `trip-blueprints` in east is replicated to
  **`source.trip-blueprints`** in the west (destination) cluster.
- **Consumers use a regex pattern** (e.g. `*.trip-blueprints`) so they read **both** the main
  topic and the prefixed/replicated topic.
  - In primary region: main topic has the live data; the replicated topic is empty (nothing is
    being actively produced there), so events come from the main topic.
  - After failover to west: consumers read the replicated (`source.`) topic for anything not yet
    consumed, then continue on the local topic. **Publishing always goes to the local
    `trip-blueprints`**, but **consumption happens from both.**
  - On failback, west→east replication (`target.` prefix) lets east resume where it left off.

**DR drills (Q&A):** Yes — performed multiple times; they've **switched to prod DR and switched back**.

### 📖 In plain English — disaster recovery (with FirstAlt example)

**Mental model:** west (`us-west-2`) is a **spare tire / backup generator** — fully built and kept
ready, but not carrying traffic. If east fails, you switch over to the spare.

**The key ideas:**
- **Active–passive:** east does all the work; west is a **warm standby**.
- **Manual failover:** they flip the switch **themselves** (they want to decide *when* — not have it
  auto-trigger on a false alarm). "Flip the switch" = change the **Route 53 weights** (east 100→0,
  west 0→100).
- **Pre-built but cheap in west** until needed:
  - API Gateway + ALB already there.
  - **EKS: the control plane exists but 0 worker nodes** — no servers running means no cost; they
    **scale the nodes up** on failover.
  - **Aurora:** a read-only replica already in sync (promoted on failover).
  - **DynamoDB:** global tables auto-syncing both ways.
  - **Kafka:** a cluster in both regions + **MirrorMaker** copying events across.

**The tricky part — Kafka MirrorMaker:** MSK can't be one cluster spanning regions, so there are **two
separate clusters**. MirrorMaker copies events across but **renames the copy with a `source.` prefix**
(so `trip-blueprints` becomes `source.trip-blueprints` in the other cluster) to avoid loops. Consumers
subscribe with a **regex** (`*.trip-blueprints`) so they read **both** the real topic and the copied
one. Normally east's real topic has the data and the copy is empty; after failover, west reads the
copied (`source.`) topic to **catch up on anything not yet consumed**, then continues locally — so **no
events are lost**. On failback, the reverse (`target.`) copy lets east resume where it left off.

**FirstAlt example:** `us-east-1` has an outage. An engineer flips Route 53 (east→0, west→100), scales
up west's EKS worker nodes, and promotes west's Aurora reader — traffic now runs from west. Because the
DB replica, DynamoDB global tables, and Kafka MirrorMaker were keeping west in sync all along, little to
no data is lost. Later they switch back the same way. (They've actually **drilled this in production**.)

```
NORMAL:    Route 53 → east (100%)        west = warm standby (0 EKS workers)
FAILOVER:  flip weights → west (100%) → scale up west EKS + promote west DB
           Aurora replica + DynamoDB global tables + Kafka MirrorMaker
           had kept west in sync → little/no data lost
```

---

## 11. Configuration & secrets

- **AWS Secrets Manager** — all database secrets and most secrets.
- **SSM Parameter Store** — static, non-secret config.
- Even a secret that doesn't need Secrets Manager features (e.g. **rotation**) may be stored
  with **encryption enabled**.

**Certificate management (currently manual):**
- On cert expiry, they generate and send a **CSR** to the client; the client purchases the cert
  and **emails it back**; they download and **upload it into AWS Certificate Manager**.
- Renewal cadence dropped from **1 year** to roughly **every 3–4 months** — so they've requested
  **APIs to automate** this.

### 📖 In plain English — config & secrets (with FirstAlt example)

**Mental model:** two places to keep settings:
- **Secrets Manager = a locked safe** — for passwords and keys.
- **SSM Parameter Store = a labeled notice board** — for non-secret settings anyone on the team can read.

**The rule of thumb:**
- Sensitive things (DB passwords, API keys) → **AWS Secrets Manager** (encrypted, and it can
  **auto-rotate** them).
- Plain, non-secret config values → **SSM Parameter Store**.
- Even a secret that doesn't need rotation may still go in **Secrets Manager with encryption on** — better safe.

**Certificates — the manual pain point:** an SSL/TLS **certificate** is what makes the padlock/`https`
work. Certs expire and must be renewed. Today it's **manual**: FirstAlt generates a **CSR** (a signing
request), emails it to the client → the client **buys** the cert and emails it back → FirstAlt uploads
it into **AWS Certificate Manager**. Renewals used to be **yearly** but are now **every 3–4 months**, so
doing this by hand is painful — hence the request for **APIs to automate** it.

**FirstAlt example:** the Aurora DB password lives in the **safe (Secrets Manager)**; a feature's
non-secret setting lives on the **notice board (Parameter Store)**; and the `firstalt.com` certificate
gets renewed every few months through the **CSR-by-email dance**.

---

## 12. Push notifications

- **SNS → Firebase (FCM)** to push to the driver mobile app.
- **Why Firebase (Q&A):** SNS can't send directly to the app. The mobile app is built with the
  **Firebase SDK** and **AWS Amplify**; Amplify sends notifications using the **SNS + Firebase**
  configuration, and it must go through an endpoint — that endpoint is **Firebase**. (There is a
  Firebase cost; the exact rationale should be confirmed with the mobile developer.)

### 📖 In plain English — push notifications (with FirstAlt example)

**Mental model:** it's like texting a driver's phone. **AWS SNS is the sender**, but phones (Apple/
Google) only accept push messages through **their own delivery service** — so the message must hand off
to **Firebase (Google's messenger, FCM)** to actually land on the device.

**Why the extra hop:**
- SNS **can't push to the phone directly**.
- The Driver App is built with the **Firebase SDK** + **AWS Amplify**; Amplify wires SNS to Firebase,
  and **Firebase Cloud Messaging (FCM)** delivers the notification to the device.

**FirstAlt example:** a schedule change assigns Ravi a **new trip** → the backend fires **SNS** → routed
through **Amplify + Firebase (FCM)** → Ravi's phone **buzzes** 🔔 with the alert. *(There's a Firebase
cost; the exact reason for choosing it should be confirmed with the mobile developer.)*

```
FirstAlt backend → SNS → (Amplify + Firebase/FCM) → Driver's phone 🔔
                         (SNS alone can't reach the phone)
```

---

## 13. Scheduled jobs

- Previously used **AWS Batch** + **EventBridge Scheduler** to send scheduled/future-trip
  notifications to drivers (e.g. next month's trip events).
- **Migrated away from AWS Batch** to **their own services** — mainly because of **DR limitations**:
  keeping schedules in sync east↔west was costly; you can't have active schedules in both regions
  (would send **duplicate** notifications), so they'd have to keep one side disabled and, on
  failover, manually decide which of **4,000–5,000 schedules** to enable.
- **EventBridge is still used** (Q&A by Parth) — but for **keeping Lambda warm** (avoiding
  **cold starts**): the **Lambda authorizer** is pinged every **~10 minutes** so token validation
  against Azure/Cognito stays fast.

### 📖 In plain English — scheduled jobs (with FirstAlt example)

**Mental model:** a **calendar of reminders** — "at 6am tomorrow, notify this driver about his trip."

**What changed and why:**
- **Old way:** **AWS Batch + EventBridge Scheduler** ran scheduled/future notifications (e.g. remind a
  driver about next month's trips).
- **The problem was DR:** you can't safely run the **same schedules in both regions** — they'd fire
  twice and send **duplicate notifications**. So you'd keep one region's schedules disabled and, on
  failover, **manually re-enable the right ones** — but there are **4,000–5,000 schedules** to sift
  through. Too painful and error-prone.
- **New way:** they moved scheduling into **FirstAlt's own services**, handled in a **DR-friendly** way.

**EventBridge didn't disappear — it just does a different job now:** a Lambda that hasn't run in a while
is **slow to wake up** ("cold start"). The **Lambda authorizer** checks *every* token, so it can't be
slow. EventBridge **pings it every ~10 minutes** to keep it **warm** (already running) → logins stay fast.

**FirstAlt example:** "Remind Ravi about tomorrow's route at 6am" now runs on **FirstAlt's own scheduler**
(clean DR). Meanwhile EventBridge quietly pokes the **auth Lambda** every 10 min so token checks never lag.

---

## 14. Lambda authorizer modernization

- Previously: separate Lambda authorizers per service (e.g. a DMS authorizer, per-API authorizers).
- Now: **one central Lambda authorizer** handles authentication for **all** services.
- It holds a **mapping of route → allowed user pool(s)** (e.g. which route uses DMT / a backend /
  Azure). These configs are injected as **Lambda environment variables** (per app: DMT, District
  View, etc.). *(Q&A note: the "audience" value is config, not a secret.)*

### 📖 In plain English — one central authorizer (with FirstAlt example)

**Mental model:** they replaced **many separate doormen** (one per door) with **a single smart doorman
who carries a rulebook for every door**.

**What changed:**
- **Before:** each service had its **own** Lambda authorizer (a DMS one, a Blueprint one, …) — lots of
  duplicated bouncers to maintain.
- **Now:** **one central Lambda authorizer** does auth for **all** services.
- It carries a **rulebook**: which route allows which identity/user pool (e.g. this route accepts
  Azure/DMT, that one accepts a driver Cognito pool). The rulebook is injected as **environment
  variables** per app. *(The "audience" value in there is just config, not a secret.)*

**FirstAlt example:** requests to `/blueprints` (internal admins) and `/driver/trips` (drivers) both hit
the **same one Lambda**. It checks its rulebook — *"`/blueprints` → Azure; `/driver/trips` → driver
Cognito pool"* — validates the token against the **right** one, and allows or denies. One bouncer, one
rulebook, all the doors.

```
Before:  /dms → DMS authorizer
         /blueprints → Blueprint authorizer     (many bouncers)
After:   ALL routes → ONE central Lambda authorizer
                        └ rulebook (env vars): route → allowed pool / Azure
```

---

## 15. Observability / monitoring

- **Monitoring tool: CloudWatch** (not **Datadog** — the client found Datadog **too expensive**).
- **Metrics:** AWS service-level metrics feed **CloudWatch alarms**; dashboards, alerts, alarms
  all in CloudWatch.
  - **Gap:** no **application-level metrics** yet (heap/non-heap usage). Currently they take
    **thread dumps manually** and analyze them — flagged as something to prioritize.
- **Logs:** all backend services → **CloudWatch** via **Fluent Bit** (deployed as a service in
  EKS; scrapes configured log paths per service and routes to the right **log groups**).
- **Traces:** **AWS X-Ray** via **OpenTelemetry** — specifically **ADOT (AWS Distro for
  OpenTelemetry)** deployed as a **daemon set** in EKS; apps send traces to it → X-Ray.
  - **Automatic (agent-based) instrumentation**, no manual instrumentation.
  - **Cost is minimal (<$50/month)** because of **sampling rules**.

### 📖 In plain English — monitoring (with FirstAlt example)

**Mental model:** the system's **dashboard + warning lights + flight recorder**. Three kinds of signals:
- **Metrics = the gauges** (numbers over time: CPU, request counts, error rates).
- **Logs = the written record** (what each service actually did, line by line).
- **Traces = the breadcrumb trail** of one single request as it hops across services.

**How FirstAlt does each:**
- **Metrics → CloudWatch.** AWS service metrics feed **alarms + dashboards**. They chose **CloudWatch
  over Datadog** because Datadog was **too expensive**. *Gap:* no **app-level** metrics yet (e.g. Java
  heap memory) — when there's a memory issue they still take **thread dumps by hand** (flagged to improve).
- **Logs → Fluent Bit → CloudWatch.** Each service's logs are shipped by **Fluent Bit** (running in
  EKS) into the right **log groups**.
- **Traces → ADOT/OpenTelemetry → X-Ray.** A trace follows a request end to end. They only **sample** a
  fraction of requests, which is why tracing costs **<$50/month**.

**FirstAlt example:** the Trip Scheduler starts throwing errors → a **CloudWatch alarm** fires →
engineers read its **Fluent Bit logs** → and use an **X-Ray trace** to see exactly where a slow request
spent its time (API Gateway → service → DB).

```
Metrics → CloudWatch alarms/dashboards     (gauges + warning lights)
Logs    → Fluent Bit → CloudWatch log groups   (the written record)
Traces  → ADOT/OpenTelemetry → X-Ray (sampled) (breadcrumbs of one request)
```

---

## 16. CI/CD

- **Hosted on the same EKS cluster** as the apps.
- Previously **GoCD** (built by **ThoughtWorks**); dropped it due to **no PR checks** and high
  **maintenance overhead**.
- Now **GitHub Actions** using the **Actions Runner Controller (ARC)** deployed in EKS. Runners
  run as pods; **~10 pods always running**, new pods created on demand.
- **Cost control:** app EKS instances stay running (devs debug at night / run automation), but
  **CI/CD instances scale down to 1** during non-working hours (nights, weekends) and back up to
  **3** during working hours.
- **Same cluster** for GoCD app + runners: GoCD app on a **small** instance, runners on **big** instances.

### 📖 In plain English — CI/CD (with FirstAlt example)

**Mental model:** CI/CD is the **factory assembly line** that takes code → **builds** it → **tests** it
→ **ships** it, automatically.

**What changed:**
- **Old tool: GoCD** (by ThoughtWorks). Dropped because it had **no PR checks** and was **high-maintenance**.
- **New tool: GitHub Actions**, with the **Actions Runner Controller (ARC)** running the build "runners"
  as **pods inside the same EKS cluster**. About **10 runner pods are always ready**, and more spin up
  on demand.
- **Cost control:** the **build machines scale down to 1** at night/weekends and back to **3** during
  work hours. (The **app** servers stay up because devs sometimes debug at night.)

**FirstAlt example:** a dev opens a **PR** → a GitHub Actions **runner pod** in EKS builds and tests it →
on **merge**, it deploys. At 9pm the runner fleet shrinks to **1** to save money; at 9am it's back to **3**.

---

## 17. Security & firewalls

- **API Gateway:** Layer 7, but **no WAF** — they use **HTTP APIs** (cheaper/faster) rather than
  **REST APIs**; moving to REST would roughly **double/triple cost** for the same load and they
  don't need REST-only features. Drawback: no WAF support on HTTP APIs.
- **ALB:** internal/private; only accepts traffic from the **VPC Link security group**.
- **VPC Link → EKS worker nodes:** worker nodes accept traffic only from other worker nodes and
  the **internal ALB security group**. Confirmed proper **Security Groups + NACLs** on the
  NodePorts (Q&A). Parth: it's a **private endpoint**, outside traffic disabled.
- **Not used:** AWS Network Firewall, network traffic filtering, DNS firewall.

**TLS termination points:**
- **Backend APIs:** HTTPS **terminated at API Gateway**; internal hops are HTTP.
- **CloudFront:** terminated at the **CloudFront edge**.
- **GoCD:** terminated at the **load balancer**.
- **Encryption in transit:** yes (in-transit encryption in place).

### 📖 In plain English — security & firewalls (with FirstAlt example)

**Mental model:** **layers of locked doors and ID checks**, plus a note on where the **"sealed envelope"
(HTTPS encryption) gets opened**.

**The boundaries:**
- **API Gateway** uses **HTTP APIs** (cheaper/faster) instead of REST APIs. Trade-off: **no WAF** (web
  application firewall) on HTTP APIs. Switching to REST would **2–3× the cost** for the same load and
  they don't need REST-only features.
- **Everything after that is private:** the internal **ALB only accepts traffic from the VPC Link's
  security group**, and the **EKS worker nodes only accept traffic from other nodes + the ALB's security
  group**. So **nothing from the internet can reach the services directly.**
- They **don't** use AWS Network Firewall or DNS firewall.

**Where the envelope is opened (TLS termination):**
- **Backend APIs:** HTTPS is **decrypted at API Gateway**; from there it's plain HTTP — but only **inside
  the locked private network**.
- **CloudFront:** decrypted at the **edge**. **GoCD:** at its **load balancer**. Encryption in transit is in place.

**FirstAlt example:** a driver's request arrives over **HTTPS**; **API Gateway** checks the token and
terminates TLS; the request then travels as HTTP but **only** through the private network, where security
groups ensure the **only** path is ALB → worker nodes. An outsider can't reach a service directly.

```
User --HTTPS--> API Gateway (TLS terminated + token checked)
                    │  --HTTP-- (private network only)
                    ▼
              VPC Link SG → internal ALB → EKS nodes (SG-locked)
   No WAF (HTTP APIs) · no internet path to the services
```

---

## 18. External data flow (integration with the data platform)

FirstAlt publishes data to a **central Kafka cluster in a separate AWS *data* account**.
There **used to be 3 streams**; it's moving to **2** (stream 1 is being sunset).

1. **Debezium CDC (being sunset):** an EC2 instance runs **Debezium** (open-source CDC). It
   watches the database via a **logical replication slot**, captures row-level changes, and
   sends them as events to the **central Kafka cluster** in the data account. Downstream
   **FirstView / data-platform systems** consume them.
   - **Why sunsetting:** all events downstream systems need are now emitted by the application
     itself (stream 2), so this stream is redundant.

2. **Application domain events (current):** because fields needed **transformation/customization**,
   they agreed contracts per table/domain and built **custom producers/consumers** (e.g. **student**
   domain events, **trip** events). Services push **live update** events to the **same central
   Kafka cluster** → consumed by downstream systems.

3. **DynamoDB Streams (current):** trip lat/long in DynamoDB (partition key = trip ID) →
   a **Lambda** reads the **DynamoDB Stream** → sends events to the **central Kafka cluster** →
   consumed by downstream systems.

- Streams 2 & 3 have a **schema** registered in the **AWS Glue Schema Registry**.
- The **FirstView / "Fast" downstream systems** also **publish** events they want back into the
  central Kafka cluster, which FirstAlt then consumes.

**Connectivity:**
- Set up via an **MSK VPC connection** to the central Kafka cluster. The data account **allow-lists
  FirstAlt's account ID** and attaches an **IAM role** to the VPC connection; FirstAlt configures its
  own IAM role/actions on that connection.
- **Topic creation:** previously **auto-topic-creation** was on; now, if a topic doesn't exist, they
  raise a **PR in an AWS CodeCommit repo** (where topic config is maintained); once merged, the topic
  is created in the central cluster.

**Power BI / Snowflake ingestion (future — Q&A):**
- Use case: ingest FirstAlt data directly into the analytics/data platform for **Power BI**.
- Discussed with **Jeff**. Two candidate paths:
  1. A **Power BI Gateway** installed in FirstAlt's account, taking DB events and sending to Power BI.
  2. Ingest into **Snowflake**, do **silver/gold** staging/transformation there, then to **Power BI**
     (Snowflake↔Power BI integration already exists on the platform side).
- **Open:** how data actually flows from FirstAlt → Snowflake is not yet defined. Network **CIDR
  ranges** were shared to check overlap; **3 environments had overlaps**, so **new CIDR ranges** were
  introduced. Awaiting a **working session** (with "Karchif"/data team) to configure it.

**DR for this external flow (pending):**
- The central Kafka clusters are owned by the **data team**. FirstAlt requested a **replicated
  Kafka cluster**.
- Done for **UAT** (FirstAlt produces from **pre-prod** → their UAT, since data team has no pre-prod
  equivalent). **Prod** replicated cluster is planned "this cycle" — **needs follow-up**.

### 📖 In plain English — external data flow (with FirstAlt example)

**Mental model:** FirstAlt is a **factory**; the **data platform (FirstView / analytics)** is the
**warehouse next door**. They ship "shipments" (data events) to a **shared loading dock** — a central
Kafka cluster living in a **separate data account**.

**Three streams feed that shared dock (moving from 3 → 2):**
1. **Debezium CDC (being retired):** a tool that **watches every change** in the Aurora DB and ships it
   as events. Being sunset because the app now emits those events itself (stream 2).
2. **Application domain events (current):** the **services themselves** publish clean, agreed-format
   events (student events, trip events). Better, because they can **shape/transform** the data first.
3. **DynamoDB Streams (current):** a **Lambda** watches the GPS table's change stream and ships those
   **location** events too.

- Streams 2 & 3 register their event **shape/format** in the **Glue Schema Registry** so consumers know
  how to read them.
- The **data platform also publishes events back**, which FirstAlt consumes.
- **Connectivity:** an **MSK VPC connection** links the two accounts; **IAM** controls who can do what;
  new topics are created by raising a **PR to a CodeCommit repo**.

**Future (Power BI):** they want FirstAlt data in **Power BI** — either a **Power BI Gateway** in
FirstAlt's account, or into **Snowflake** (silver/gold layers) → Power BI. The **FirstAlt → Snowflake**
path isn't built yet; they sorted out **CIDR** overlaps and are awaiting a working session.
**DR for this flow:** a replicated Kafka cluster exists for **UAT**; **prod is pending**.

**FirstAlt example:** a student's record changes → the app emits a **"student updated" event** → it lands
in the **central Kafka** → the analytics platform **ingests** it for reporting / Power BI.

```
FirstAlt sources ──► Central Kafka (data account) ──► FirstView / analytics
  1) Debezium CDC (retiring)
  2) App domain events (student, trip)     Glue Schema Registry = event formats
  3) DynamoDB Streams (GPS) via Lambda     MSK VPC connection + IAM
                                           → (future) Snowflake → Power BI
```

---

## 19. Data Guardian Portal — privatization (recent change)

Data Guardian (DMT) was made **private / VPN-only** (internal employees).

**Serving path (now):**
`Route 53 Private Hosted Zone → internal ALB → S3 interface (VPC) endpoints → S3`

- Created a **Route 53 Private Hosted Zone**; traffic forwarded to the **internal ALB**, which
  forwards to **S3 interface endpoints** created for the buckets.
- **Requirement:** the **bucket name must equal the domain name** (e.g.
  `dataguardian-dev.nonprod.firstalt.com`). The interface endpoint is added as a **target** on the ALB.

**On-prem / network connectivity:**
- Asked the **FS IT team** to add **DNS conditional forwarders**: requests for the Data Guardian
  domain are forwarded to FirstAlt's **Route 53**. For that, FirstAlt created **Route 53 Resolver
  inbound endpoints** (configured in FS IT's on-prem DNS).
- Traffic path: a **central AWS networking account** (managed by FS IT) shares a **Transit Gateway**
  with the FirstAlt account. TGW routes let traffic flow FirstAlt account → central networking account
  → **on-prem** (via the **VPN solution** in the central networking account).

### 📖 In plain English — making Data Guardian private (with FirstAlt example)

**Mental model:** they took a **public shop** and turned it into a **members-only office** you can only
reach **from inside the company network (VPN)** — there's no public street entrance anymore.

**What changed and why:**
- Data Guardian is the **internal FS-employee** app, so they made it **private / VPN-only** — no public
  internet access.
- **New serving path:** **Route 53 Private Hosted Zone** (a private phone book only visible inside the
  network) → **internal ALB** → **S3 interface endpoints** → **S3**.
- **Quirk:** the **S3 bucket name must exactly match the domain name** (e.g.
  `dataguardian-dev.nonprod.firstalt.com`).

**Connecting the office network to AWS:**
- FS IT adds **DNS conditional forwarders**: *"any request for the Data Guardian domain → send it to
  FirstAlt's Route 53."* That reaches FirstAlt via **Route 53 Resolver inbound endpoints**.
- The actual traffic flows through a **central AWS networking account** (run by FS IT) over a shared
  **Transit Gateway**, then out to **on-prem** through their **VPN**.

**FirstAlt example:** an FS employee **on the corporate VPN** opens Data Guardian → their DNS (via the
conditional forwarder) resolves it through FirstAlt's **private Route 53** → hits the **internal ALB** →
**S3**. An **outsider on the public internet simply can't reach it** — the address won't resolve or route
for them.

```
FS employee (on VPN)
   → on-prem DNS (conditional forwarder) → Route 53 Resolver inbound → private Route 53
   → internal ALB → S3 interface endpoint → S3 (bucket name = domain name)
Traffic rides: FirstAlt acct → Transit Gateway → central networking acct → on-prem VPN
No public-internet path.
```

---

## 20. Open items / follow-ups (action items)

- [ ] **Cognito global password sync** — follow up with AWS on the DynamoDB-backed solution
      (was promised "by Q2"). (§5)
- [ ] **App-level metrics** (heap/non-heap) — prioritize; currently manual thread dumps. (§15)
- [ ] **Certificate automation** — get client APIs to automate CSR/cert upload (renewals every
      3–4 months). (§11)
- [ ] **External-flow DR** — confirm data team's **prod** replicated Kafka cluster status. (§18)
- [ ] **Power BI / Snowflake ingestion** — schedule the working session; confirm Jeff's role. (§18)
- [ ] **Data account ownership** — Hareesh to explore connectivity/ingestion (MSK topics as source);
      this data account will come under the team's purview. (§18)
- [ ] **DR deep-dive** — separate session to cover failover/failback flow and action items. (§10)

---

## 21. Appendix — every question asked (with answers)

| Time | Asker | Question | Answer (VV / Parth) |
|---|---|---|---|
| 03:27 | Saptak | Do we have failover health checks at Route 53? Talk through failover use cases. | No active health checks configured — they want to **initiate** DR themselves. |
| 03:49 | Saptak | So it's a **manual** Route 53 change to switch to us-west-2? | **Yes.** |
| 06:29 | Saptak | Any specific reason for using **both Cognito and Azure AD**? | Cognito was original for all; wanted **SSO** for Data Guardian. Evaluated Cognito+Azure SAML, Cognito+Azure OIDC, direct Azure. Cognito is **per-user pricey** for 3rd-party IdP + **no multi-region DR** (password-reset problem). |
| 08:42 | Saptak | Do we generate the JWT **twice** (Cognito + Azure)? | **No.** Only DMT (Data Guardian) uses Azure now. |
| 09:04 | Saptak | DMP? | **Data Guardian Portal.** |
| 09:17 | Saptak | This is an **internal app** for FS employees? | **Yes** — all in Azure; "Sign in with Microsoft." |
| 12:51 | Saptak | Why **NodePort** instead of **Ingress Gateway + ClusterIP**? | Likely early-phase decision with only 4–5 services; never added the extra layer. |
| 13:35 | Saptak | With services on NodePort, do we have proper **SGs and NACLs**? | **Yes** — only the ALB can reach the private EC2 instances (SGs configured). |
| 14:09 | Parth | *(clarifies)* It's a **private endpoint**; outside traffic disabled, internal SGs in place. | — |
| 15:43 | Ayushi | Is the **RBAC** thing an AWS service or an EKS service? | Our **own Java** service. |
| 22:01 | Ayushi | What about the **consume topic** (Kafka)? | Regex/prefix design — consume from **both** main and `source.` topics; publish local, consume both. |
| 26:12 | Saptak | Monitoring is primarily **CloudWatch**, not Datadog? | **Yes** — Datadog too expensive. |
| 26:32 | Saptak | Have we set up **alerting** on critical failures? | **Yes** — dashboards/alarms in CloudWatch (AWS metrics). No app-level metrics yet (manual thread dumps). |
| 28:44 | Saptak | Cost of the entire **tracing**? | **<$50/month**, thanks to sampling. |
| 29:06 | Saptak | Do we **auto shut down compute** off-hours? | EKS: no (devs debug at night). **CI/CD scales to 1** off-hours, **3** during work hours. |
| 30:23 | Saptak | Where is **CI/CD** hosted? | **EKS.** Was GoCD (ThoughtWorks) → now **GitHub Actions + ARC** on the same cluster. |
| 31:17 | Saptak | Are **app + CI/CD runners** in the same cluster? | For GoCD, yes — GoCD app on a small instance, runners on big instances. |
| 33:56 | Saptak | Any **firewalls** in front of the app? | API Gateway L7 but **no WAF** (HTTP APIs). SG-based isolation. No AWS Network/DNS firewall. |
| 35:39 | Saptak | Is the request flow **entirely HTTPS** end to end? | HTTPS **terminated at API Gateway** (then HTTP internal); CloudFront at edge; GoCD at LB. |
| 36:32 | Saptak | Is it **encrypted in transit**? | **Yes.** |
| 37:09 | Saptak | **Why Firebase**? | SNS can't send directly; app uses **Firebase SDK + Amplify**; must route through Firebase endpoint. (Confirm exact reason with mobile dev.) |
| 38:14 | Saptak | There's a **Firebase cost**, right? | Yeah. |
| 40:02 | Parth | *(adds)* EventBridge still used — to **keep Lambda warm** (cold start), pinging every ~10 min. | — |
| 42:28 | Saptak | Hope that **audience** value isn't a secret key. | It's config (env var), not a secret. |
| 42:34 | Saptak | Use case for **AWS DMS** at the data management service? | It's **not AWS DMS** — **DMS = our Data Management Service**. |
| 48:07 | Saptak | Use case to **ingest FirstAlt data directly into the data platform**? | **Power BI.** Via Power BI Gateway, or Snowflake (silver/gold) → Power BI. Flow to Snowflake still TBD. |
| 49:16 | Saptak | We already have **Snowflake↔Power BI** integration. | Yes on platform side; FirstAlt→Snowflake path not yet built. |
| 49:32 | Saptak | What's happening for that integration? | Discussion open; shared CIDR ranges, fixed 3 overlaps with new ranges; awaiting working session. |
| 50:32 | Saptak | Which role is **Jeff** playing? | Unknown. |
| ~52:04 | Hareesh | *(notes)* MSK topics appear as a **source** for ingestion / FirstAlt. | Auto-topic-creation or PR to **CodeCommit** to create topics; ingestion via **MSK VPC connection**. |
| 53:59 | Hareesh | Are we performing **DR drills**? | **Yes**, multiple — switched prod to DR and back. |
| 54:16 | Saptak | How long to cover **DR**? | Needs a **separate session** (failover/failback flow + action items). |
