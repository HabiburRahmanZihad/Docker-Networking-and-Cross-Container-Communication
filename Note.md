<div align="center">
  <p style="color: red; font-size: 1.4em; font-weight: bold; border: 2px solid red; padding: 14px; border-radius: 6px;">
    🚫 RESTRICTED — DO NOT COPY 🚫<br/>
    <span style="font-size: 0.85em; font-weight: normal;">
      This document is for personal study purposes only.<br/>
      Copying, reproducing, sharing, or distributing any content from this material is strictly prohibited.<br/><br/>
      ⚠️ <strong>Any unauthorized use of this content will result in immediate legal action.</strong><br/>
      <span style="font-size: 0.8em;">The author reserves all rights and will pursue copyright infringement claims to the fullest extent of the law.</span>
    </span>
  </p>
</div>

# Docker Study Notes — Container Networking & Communication

> Detailed study notes on Docker container networking: understanding why networking matters, the three communication types (container→internet, container→host, container→container), how Docker simulates real-world networks internally, hands-on implementation of each pattern, and a full capstone project in TypeScript + Express + MongoDB that demonstrates all three.

---

## Table of Contents

1. [Why Container Networking Matters](#1-why-container-networking-matters)
2. [Container Communication Types — Overview](#2-container-communication-types--overview)
3. [Container To WWW Communication](#3-container-to-www-communication)
4. [Container To Local Host Machine Communication](#4-container-to-local-host-machine-communication)
5. [Container To Container Communication](#5-container-to-container-communication)
6. [How Docker Networking Simulates Real-World Networks Internally](#6-how-docker-networking-simulates-real-world-networks-internally)
7. [Creating a Container and Communicating to the Web (Hands-on)](#7-creating-a-container-and-communicating-to-the-web-hands-on)
8. [Implementing Container to Host Communication (Hands-on)](#8-implementing-container-to-host-communication-hands-on)
9. [Container-to-Container Communication: IP vs Network Names](#9-container-to-container-communication-ip-vs-network-names)
10. [Docker Architecture Overview](#10-docker-architecture-overview)
11. [Recap](#11-recap)
12. [Quick Reference Cheat Sheet](#12-quick-reference-cheat-sheet)

---

## 1. Why Container Networking Matters

Containers are **isolated processes** on your machine. That isolation is a feature — but it creates a challenge: how do your services actually talk to each other, to the internet, or to databases on your host?

### Real-World Scenarios

Modern applications are rarely a single process. Consider:

| Scenario | What Needs to Communicate |
|---|---|
| **Web API + Database** | Your Express container → MongoDB container |
| **Microservices** | Service A container → Service B container |
| **Third-party APIs** | Your container → External REST API (the internet) |
| **Local development** | Your container → MongoDB Atlas or local DB on your machine |
| **Background workers** | Worker container → Message queue container |

### The Core Problem

```
Without understanding Docker networking:

  docker run my-app             ← container starts, app tries to reach MongoDB
  ... app crashes: "connection refused" ...

  Why? The container is isolated — it doesn't know where "localhost:27017" is.
  In a container, "localhost" means the container itself, not your host machine.
```

> Docker networking solves this by providing three distinct communication patterns. Understanding which pattern to use — and how — is the key skill covered in these notes.

---

## 2. Container Communication Types — Overview

Docker containers can communicate in three directions. Each has a different setup, use case, and set of rules.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         DOCKER NETWORKING MAP                       │
│                                                                     │
│   ┌──────────────┐                                                  │
│   │  Container   │──────────────────────────────────► 🌍 Internet  │
│   │  (your app)  │  Type 1: Container → WWW                        │
│   │              │                                                  │
│   │              │──────────────────────────────────► 🖥️ Host      │
│   │              │  Type 2: Container → Host Machine               │
│   │              │                                                  │
│   │              │──────────────────────────────────► 📦 Container │
│   └──────────────┘  Type 3: Container → Container                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### The Three Communication Types at a Glance

| Type | Direction | Requires Special Config? | Key Mechanism |
|---|---|---|---|
| **Type 1** | Container → Internet (WWW) | ❌ No — works by default | Docker NAT + default bridge |
| **Type 2** | Container → Host Machine | ✅ Yes — special hostname or flag | `host.docker.internal` or `--network host` |
| **Type 3** | Container → Container | ✅ Yes — shared Docker network | Custom network + container name as hostname |

---

## 3. Container To WWW Communication

### How It Works

By default, every Docker container is connected to Docker's **default bridge network** (`bridge`). Docker configures NAT (Network Address Translation) so that outbound traffic from the container is forwarded through the host's network interface and out to the internet.

```
Container (172.17.0.2)
       │
       │  outbound HTTP request to https://api.example.com
       ▼
Docker Bridge Network (docker0 — 172.17.0.1)
       │
       │  NAT: rewrites source IP to host IP
       ▼
Host Machine's Network Interface (e.g., 192.168.1.10)
       │
       │  request goes out to the internet
       ▼
🌍 Internet — https://api.example.com
```

### Key Point

No special flags or configuration needed. Any container can reach the internet by default. You do **not** need `--network` or any extra setup.

### Quick Test

```bash
# Start an Alpine container interactively
docker run -it alpine sh

# Inside the container — ping Google
ping google.com

# Or fetch a URL
wget -qO- https://jsonplaceholder.typicode.com/posts/1
```

Both commands work without any extra Docker flags because NAT handles outbound routing automatically.

### In Code (This Project — `src/index.ts`)

```typescript
import axios from 'axios';

app.get('/internet/:id', async (req, res) => {
    const { id } = req.params;
    const response = await axios.get(
        `https://jsonplaceholder.typicode.com/posts/${id}`
    );
    res.json({
        source: 'public internet',
        data: response.data,
    });
});
```

When this container calls `axios.get(...)`, the request travels through Docker's bridge → NAT → host network → internet. No `--network` flag needed on `docker run`.

---

### What Can Go Wrong

| Problem | Cause | Fix |
|---|---|---|
| DNS resolution fails inside container | Container uses Docker's internal DNS, which may not forward properly | Check `/etc/resolv.conf` inside the container; restart Docker |
| Firewall blocking outbound | Host or corporate firewall blocks egress | Check host firewall rules |
| `--network none` was set | Explicitly disabled networking | Remove `--network none` from `docker run` |

---

## 4. Container To Local Host Machine Communication

### The Problem with "localhost"

Inside a container, `localhost` (or `127.0.0.1`) refers to the **container itself**, not your host machine. This is a common source of confusion.

```
Your machine:
  MongoDB running on localhost:27017

Inside a container:
  mongoose.connect("mongodb://localhost:27017/demo")
  ← FAILS: "localhost" is the container, not your machine
  ← MongoDB is not running inside the container
```

### Solution 1 — `host.docker.internal` (Recommended, Cross-Platform)

Docker provides a special built-in hostname: **`host.docker.internal`**. It always resolves to the IP address of your host machine, from inside any container.

```
Container
    │
    │  mongoose.connect("mongodb://host.docker.internal:27017/demo")
    ▼
host.docker.internal  ←── Docker resolves this to your host machine's IP
    │
    ▼
Host Machine → MongoDB on port 27017
```

```bash
# Run the container and pass the host DB URI as an environment variable
docker run \
  -e MONGO_ATLAS_URI="mongodb://host.docker.internal:27017/demo" \
  -p 3000:3000 \
  my-app
```

**Availability:**

| Platform | `host.docker.internal` Available? |
|---|---|
| macOS (Docker Desktop) | ✅ Yes |
| Windows (Docker Desktop) | ✅ Yes |
| Linux | ✅ Yes (Docker 20.10+), may need `--add-host` on older versions |

### Solution 2 — `--network host` (Linux Only)

The `--network host` flag makes the container share the host's network stack entirely. The container has no isolated network — it uses the host's interfaces directly.

```bash
docker run -it --network host alpine sh

# Inside the container — "localhost" now means the host machine
ping localhost      ← reaches the host
# MongoDB on localhost:27017 is directly accessible
```

**Important:** `--network host` is **Linux only**. On macOS and Windows, Docker runs containers inside a lightweight Linux VM — so `--network host` connects to the VM's network, not your actual host machine.

| Method | Works On | Isolation | Recommended |
|---|---|---|---|
| `host.docker.internal` | Mac, Windows, Linux | Container stays isolated | ✅ Yes |
| `--network host` | Linux only | No network isolation | ⚠️ Dev/debug only |

### In Code (This Project — `src/index.ts`)

```typescript
app.get('/host-db', async (req, res) => {
    const uri = process.env.MONGO_ATLAS_URI;

    // Works for BOTH:
    // - MongoDB Atlas: "mongodb+srv://user:pass@cluster.mongodb.net/demo"
    // - Local host DB: "mongodb://host.docker.internal:27017/demo"

    const conn = await mongoose.createConnection(uri!).asPromise();
    const Item = conn.model('Item', itemSchema);

    await Item.create({ name: 'Hello from host DB' });
    const items = await Item.find();

    await conn.close();
    res.json({ source: 'host machine / Atlas', items });
});
```

The URI is passed in at runtime — the code itself doesn't hardcode `localhost` or any host-specific address.

---

## 5. Container To Container Communication

### The Default Behavior

By default, two containers started separately **cannot reach each other by name**. Each container is on Docker's default bridge network but Docker does **not** provide automatic DNS resolution between containers on the default bridge.

```
Container A (my-app):
  mongoose.connect("mongodb://mongo-container:27017/demo")
  ← FAILS on default bridge: "mongo-container" hostname not known

Container B (mongo-container):
  Running MongoDB on port 27017
```

### The Solution — Custom User-Defined Networks

When you create a **custom Docker network**, Docker automatically registers each container's name as a DNS hostname resolvable by every other container on that network.

```
Docker Custom Network (my-network)
┌────────────────────────────────────────────────────┐
│                                                    │
│   ┌────────────────┐       ┌──────────────────┐   │
│   │   my-app       │       │  mongo-container  │   │
│   │  (Express app) │──────►│  (MongoDB)        │   │
│   │  172.18.0.2    │       │  172.18.0.3       │   │
│   └────────────────┘       └──────────────────┘   │
│                                                    │
│  Docker DNS: "mongo-container" → 172.18.0.3  ✅   │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Step-by-Step Setup

```bash
# Step 1: Create a custom network
docker network create my-network

# Step 2: Start the MongoDB container ON that network
docker run -d \
  --name mongo-container \
  --network my-network \
  -e MONGO_INITDB_ROOT_USERNAME=root \
  -e MONGO_INITDB_ROOT_PASSWORD=secret \
  mongo:7

# Step 3: Start your app container ON the same network
docker run -d \
  --name my-app \
  --network my-network \
  -e MONGO_CONTAINER_URI="mongodb://root:secret@mongo-container:27017/demo?authSource=admin" \
  -p 3000:3000 \
  my-app-image
```

Now `my-app` can reach MongoDB using the hostname `mongo-container` — Docker resolves it automatically.

### In Code (This Project — `src/index.ts`)

```typescript
app.get('/container-db', async (req, res) => {
    const uri = process.env.MONGO_CONTAINER_URI;
    // URI example: "mongodb://root:secret@mongo-container:27017/demo?authSource=admin"
    //                                      ^^^^^^^^^^^^^^^^
    //                   Docker DNS resolves this container name on the shared network

    const conn = await mongoose.createConnection(uri!).asPromise();
    const Item = conn.model('Item', itemSchema);

    await Item.create({ name: 'Hello from container DB' });
    const items = await Item.find();

    await conn.close();
    res.json({ source: 'container-to-container', items });
});
```

### Default Bridge vs Custom Network

| Feature | Default Bridge | Custom Network |
|---|---|---|
| Container DNS (name resolution) | ❌ Not supported | ✅ Supported |
| Containers can reach each other | ✅ By IP only | ✅ By name or IP |
| Isolation between networks | ❌ All containers share it | ✅ Per-network isolation |
| Recommended for | Quick testing only | All real use cases |

---

## 6. How Docker Networking Simulates Real-World Networks Internally

Docker doesn't just open ports — it creates a complete virtual network inside your machine that mirrors how real-world networks behave.

### The Virtual Bridge (docker0)

When Docker is installed, it creates a virtual network interface called **`docker0`** on your host. This acts as a virtual switch/router that all containers connect to.

```
Host Machine
┌──────────────────────────────────────────────────────────┐
│                                                          │
│   Physical NIC (eth0 / en0)   ←── connects to internet  │
│         │                                                │
│         │  NAT (iptables / nftables)                     │
│         │                                                │
│   docker0 (172.17.0.1)  ←── Docker's virtual bridge     │
│    ┌─────┴──────┐                                        │
│    │            │                                        │
│  veth0        veth1  ←── virtual ethernet pairs          │
│    │            │                                        │
│ Container A  Container B  (172.17.0.2 / 172.17.0.3)      │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### How Each Component Works

| Component | Role | Real-World Equivalent |
|---|---|---|
| `docker0` bridge | Virtual switch connecting all containers | Office network switch |
| `veth` pair | Virtual cable: one end in container, one on host bridge | Network cable |
| Container IP (`172.17.x.x`) | Each container's private IP on the bridge | Device IP on a LAN |
| NAT (iptables) | Translates container IPs to host IP for outbound traffic | Home router NAT |
| Docker DNS (`127.0.0.11`) | Resolves container names to IPs inside custom networks | Corporate DNS server |

### How Outbound Traffic Works (Type 1 — WWW)

```
1. Container sends packet to 8.8.8.8 (Google DNS)
2. Packet hits the veth interface → reaches docker0 bridge
3. iptables NAT rule: rewrite source IP (172.17.0.2 → host IP)
4. Packet exits host's physical NIC to the internet
5. Response arrives at host → iptables DNAT: rewrite destination back to 172.17.0.2
6. Packet delivered to container via docker0 → veth
```

### How Docker DNS Works (Type 3 — Container-to-Container)

```
1. Container A: mongoose.connect("mongodb://mongo-container:27017")
2. DNS lookup: "mongo-container" → query sent to 127.0.0.11 (Docker's embedded DNS)
3. Docker DNS: resolves "mongo-container" → 172.18.0.3 (its IP on the custom network)
4. Container A connects directly to 172.18.0.3:27017
5. No NAT needed — both containers are on the same virtual network
```

### User-Defined Networks Are Isolated

```
Network A (my-network):
  Container 1 ←→ Container 2   ✅ can communicate

Network B (other-network):
  Container 3 ←→ Container 4   ✅ can communicate

Cross-network:
  Container 1 → Container 3    ❌ blocked — different networks
```

A container can be on multiple networks if needed — just run `docker network connect` to add it.

---

## 7. Creating a Container and Communicating to the Web (Hands-on)

### Goal

Call `GET /internet/:id` on the containerized Express app and verify it successfully reaches the public internet (JSONPlaceholder API).

### Step 1 — Build the Image

```bash
# From the project root
docker build -t docker-comm-demo .
```

**What the Dockerfile does:**

```dockerfile
FROM node:20-alpine          # 1. Minimal Node.js 20 base image

WORKDIR /app                 # 2. All commands run from /app

COPY package*.json tsconfig.json ./   # 3. Copy dependency manifests
RUN npm install              # 4. Install all dependencies

COPY src ./src               # 5. Copy TypeScript source files
RUN npm run build            # 6. Compile TypeScript → JavaScript (outputs to /app/dist/)

EXPOSE 3000                  # 7. Document that the app uses port 3000
CMD ["node", "dist/index.js"] # 8. Start the compiled app
```

### Step 2 — Run the Container

```bash
docker run -d \
  --name comm-demo \
  -p 3000:3000 \
  docker-comm-demo
```

No special `--network` needed for internet access.

### Step 3 — Test the Endpoint

```bash
# Fetch post #1 from JSONPlaceholder (a public test API)
curl http://localhost:3000/internet/1
```

**Expected response:**

```json
{
  "source": "public internet",
  "data": {
    "userId": 1,
    "id": 1,
    "title": "sunt aut facere repellat provident...",
    "body": "quia et suscipit..."
  }
}
```

The container reached `https://jsonplaceholder.typicode.com` directly — no extra config required.

### What Happened Inside Docker

```
curl → localhost:3000
       │
       │  port mapping: 3000 (host) → 3000 (container)
       ▼
comm-demo container (port 3000)
       │
       │  axios.get("https://jsonplaceholder.typicode.com/posts/1")
       ▼
Docker NAT → Host NIC → Internet → JSONPlaceholder API
       │
       │  response data returned
       ▼
JSON response back to curl
```

---

## 8. Implementing Container to Host Communication (Hands-on)

### Goal

Connect the containerized Express app to a MongoDB instance — either running locally on your host machine or on MongoDB Atlas (cloud).

### Option A — MongoDB Atlas (Cloud)

```bash
docker run -d \
  --name comm-demo \
  -p 3000:3000 \
  -e MONGO_ATLAS_URI="mongodb+srv://username:password@cluster0.abc123.mongodb.net/demo" \
  docker-comm-demo
```

Test:

```bash
curl http://localhost:3000/host-db
```

The container reaches Atlas over the internet (Type 1 + Type 2 combined: internet egress to a remote host).

---

### Option B — Local MongoDB on Your Host Machine

First, make sure MongoDB is running on your host:

```bash
# Check if MongoDB is running locally
mongosh --eval "db.adminCommand({ ping: 1 })"
```

Then run the container using `host.docker.internal`:

```bash
docker run -d \
  --name comm-demo \
  -p 3000:3000 \
  -e MONGO_ATLAS_URI="mongodb://host.docker.internal:27017/demo" \
  docker-comm-demo
```

Test:

```bash
curl http://localhost:3000/host-db
```

**Expected response:**

```json
{
  "source": "host machine / Atlas",
  "items": [
    { "_id": "...", "name": "Hello from host DB", "__v": 0 }
  ]
}
```

### Why `host.docker.internal` Works

```
Container (172.17.0.2)
       │
       │  DNS lookup: "host.docker.internal"
       ▼
Docker resolves → 192.168.65.2 (Docker Desktop's host-gateway IP)
       │
       │  TCP connection to 192.168.65.2:27017
       ▼
Host Machine → MongoDB on port 27017 ✅
```

Docker Desktop automatically adds `host.docker.internal` to the container's `/etc/hosts` file — no manual configuration needed.

---

## 9. Container-to-Container Communication: IP vs Network Names

There are two approaches to connect containers together. One is fragile; the other is the correct way.

### Approach 1 — By IP Address (Fragile, Avoid)

You can inspect a container's IP and hardcode it:

```bash
# Find the container's IP
docker inspect mongo-container | grep IPAddress
# "IPAddress": "172.17.0.3"

# Connect using the IP directly
docker run -e MONGO_CONTAINER_URI="mongodb://root:secret@172.17.0.3:27017/demo?authSource=admin" my-app
```

| Concern | Detail |
|---|---|
| IPs are dynamic | Docker reassigns IPs when containers restart — your hardcoded IP breaks |
| Not readable | `172.17.0.3` carries no semantic meaning |
| Default bridge limitation | On the default bridge, DNS doesn't work — IP is the only option |
| Verdict | ❌ Do not use in real workflows |

---

### Approach 2 — Custom Network + Container Name (Correct Way)

```bash
# 1. Create a named network
docker network create app-network

# 2. Run MongoDB on that network with a stable name
docker run -d \
  --name mongo-container \
  --network app-network \
  -e MONGO_INITDB_ROOT_USERNAME=root \
  -e MONGO_INITDB_ROOT_PASSWORD=secret \
  mongo:7

# 3. Run your app on the same network
docker run -d \
  --name my-app \
  --network app-network \
  -e MONGO_CONTAINER_URI="mongodb://root:secret@mongo-container:27017/demo?authSource=admin" \
  -p 3000:3000 \
  docker-comm-demo
```

```bash
# Test the container-to-container endpoint
curl http://localhost:3000/container-db
```

**Expected response:**

```json
{
  "source": "container-to-container",
  "items": [
    { "_id": "...", "name": "Hello from container DB", "__v": 0 }
  ]
}
```

### What Happens Under the Hood

```
my-app container:
  mongoose.connect("mongodb://root:secret@mongo-container:27017/demo?authSource=admin")
                                            ^^^^^^^^^^^^^^^^
  Docker DNS (127.0.0.11) resolves "mongo-container" → 172.18.0.3

  Direct TCP connection to 172.18.0.3:27017 (stays inside the virtual network)
  No NAT, no internet — pure internal Docker networking ✅
```

### IP vs Name: Side-by-Side Comparison

| | By IP | By Container Name |
|---|---|---|
| **Setup** | Inspect IP after start | Create custom network |
| **Stability** | ❌ Breaks on restart | ✅ Stable — name doesn't change |
| **Readability** | ❌ `172.17.0.3` | ✅ `mongo-container` |
| **Works on default bridge** | ✅ Yes | ❌ No — needs custom network |
| **Works on custom network** | ✅ Yes | ✅ Yes (preferred) |
| **Production-ready** | ❌ No | ✅ Yes |

---

### Managing Docker Networks

```bash
# Create a network
docker network create app-network

# List all networks
docker network ls

# Inspect a network (see connected containers and their IPs)
docker network inspect app-network

# Connect a running container to a network
docker network connect app-network my-container

# Disconnect a container from a network
docker network disconnect app-network my-container

# Remove a network (all containers must be disconnected first)
docker network rm app-network

# Remove all unused networks
docker network prune
```

---

## 10. Docker Architecture Overview

Understanding the full Docker architecture helps you reason about where networking fits into the bigger picture.

### The Big Picture

```
You (Developer)
       │
       │  docker build / docker run / docker pull
       ▼
┌──────────────────────────────────────────────────────┐
│                    Docker Client (CLI)                │
│           (docker, docker-compose, Docker Desktop)    │
└──────────────────────┬───────────────────────────────┘
                       │  REST API (Unix socket or TCP)
                       ▼
┌──────────────────────────────────────────────────────┐
│                  Docker Daemon (dockerd)              │
│                                                      │
│  ┌──────────┐  ┌────────────┐  ┌──────────────────┐ │
│  │  Images  │  │ Containers │  │    Networks       │ │
│  │ (layers) │  │ (running)  │  │ bridge/host/none/ │ │
│  └──────────┘  └────────────┘  │  custom           │ │
│                                └──────────────────┘ │
│  ┌──────────┐                                        │
│  │ Volumes  │                                        │
│  └──────────┘                                        │
└──────────────────────┬───────────────────────────────┘
                       │  pulls/pushes images
                       ▼
┌──────────────────────────────────────────────────────┐
│              Docker Registry (Docker Hub)             │
│    node:20-alpine  │  mongo:7  │  your-image:latest  │
└──────────────────────────────────────────────────────┘
```

### Core Components

| Component | Role |
|---|---|
| **Docker Client** | CLI tool you type commands into (`docker run`, `docker build`) |
| **Docker Daemon** | Background service that actually manages containers, images, networks, volumes |
| **Docker Image** | Read-only template — built from a Dockerfile, stored as layers |
| **Docker Container** | Running instance of an image — has its own filesystem, network, and process space |
| **Docker Network** | Virtual network connecting containers (`bridge`, `host`, `none`, custom) |
| **Docker Volume** | Persistent storage that outlives containers |
| **Docker Registry** | Remote store for images (Docker Hub, GitHub Container Registry, ECR) |

### Network Drivers

Docker supports multiple network drivers, each with different behavior:

| Driver | Default | Use Case |
|---|---|---|
| `bridge` | ✅ Yes | Single-host container-to-container communication |
| `host` | ❌ | Share host's network stack (Linux only, no isolation) |
| `none` | ❌ | Completely disable networking |
| `overlay` | ❌ | Multi-host networking (Docker Swarm / Kubernetes) |
| `macvlan` | ❌ | Assign a real MAC address — container appears as physical device on LAN |

For all local development and single-server deployments, `bridge` (custom user-defined) is the correct choice.

---

## 11. Recap

### Summary of the Three Communication Types

```
┌────────────────────────────────────────────────────────────────────────┐
│  Type 1 — Container → WWW                                              │
│  No config needed. Docker NAT handles it.                              │
│  Use: calling external APIs, downloading packages, web scraping        │
│  Example: axios.get("https://jsonplaceholder.typicode.com/posts/1")    │
├────────────────────────────────────────────────────────────────────────┤
│  Type 2 — Container → Host Machine                                     │
│  Use: host.docker.internal as the hostname                             │
│  Use: accessing local databases, local services during development     │
│  Example: mongodb://host.docker.internal:27017/demo                    │
├────────────────────────────────────────────────────────────────────────┤
│  Type 3 — Container → Container                                        │
│  Requires a custom Docker network                                      │
│  Use: microservices, app + database, any multi-container setup         │
│  Example: mongodb://root:secret@mongo-container:27017/demo             │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 12. Quick Reference Cheat Sheet

### Network Commands

| Command | Example | What It Does |
|---|---|---|
| `docker network create` | `docker network create app-net` | Create a custom bridge network |
| `docker network ls` | `docker network ls` | List all networks |
| `docker network inspect` | `docker network inspect app-net` | View network details and connected containers |
| `docker network connect` | `docker network connect app-net my-app` | Connect a running container to a network |
| `docker network disconnect` | `docker network disconnect app-net my-app` | Disconnect a container from a network |
| `docker network rm` | `docker network rm app-net` | Remove a network |
| `docker network prune` | `docker network prune` | Remove all unused networks |

---

### Communication Pattern Cheat Sheet

| Pattern | `docker run` Command | URI / Hostname |
|---|---|---|
| Container → Internet | `docker run my-app` (no flags needed) | `https://any-public-api.com` |
| Container → Host (any OS) | `docker run -e DB_URI="..." my-app` | `host.docker.internal:27017` |
| Container → Host (Linux) | `docker run --network host my-app` | `localhost:27017` |
| Container → Container | `docker run --network app-net my-app` | `other-container-name:port` |

---

### Three Communication Types Summary

| Type | Requires Special Config? | Key Mechanism | `docker run` Flag |
|---|---|---|---|
| Container → WWW | ❌ None | Docker NAT (default) | — |
| Container → Host | ✅ Special hostname | `host.docker.internal` | `-e URI="mongodb://host.docker.internal:..."` |
| Container → Container | ✅ Shared network | Custom network + container name DNS | `--network my-network` |

---

### Development Workflow for This Project

```bash
# Local dev (no Docker)
npm run dev

# Build Docker image
docker build -t docker-comm-demo .

# Run with internet only (Type 1)
docker run -p 3000:3000 docker-comm-demo

# Run with host DB (Type 1 + Type 2)
docker run -p 3000:3000 \
  -e MONGO_ATLAS_URI="mongodb://host.docker.internal:27017/demo" \
  docker-comm-demo

# Run with all three types (Type 1 + 2 + 3)
docker network create app-network
docker run -d --name mongo-container --network app-network \
  -e MONGO_INITDB_ROOT_USERNAME=root -e MONGO_INITDB_ROOT_PASSWORD=secret mongo:7
docker run -p 3000:3000 --network app-network \
  -e MONGO_ATLAS_URI="mongodb://host.docker.internal:27017/demo" \
  -e MONGO_CONTAINER_URI="mongodb://root:secret@mongo-container:27017/demo?authSource=admin" \
  docker-comm-demo
```

---

### Network Driver Summary

| Driver | Isolation | Multi-Host | When to Use |
|---|---|---|---|
| `bridge` (default) | Per-container | ❌ | Quick tests on default network |
| `bridge` (custom) | Per-network | ❌ | All real single-host use cases |
| `host` | None | ❌ | Debug only, Linux only |
| `none` | Full | ❌ | Security-sensitive containers with no networking |
| `overlay` | Per-network | ✅ | Docker Swarm, multi-host deployments |

---

<div align="center">
  <p style="font-size: 1.1em; font-weight: bold;">Habibur Rahman Zihad</p>
  <p style="color: gray; font-size: 0.95em;">Full-Stack Developer</p>
  <p style="color: gray; font-size: 0.85em;">© 2026 All Rights Reserved</p>
</div>
