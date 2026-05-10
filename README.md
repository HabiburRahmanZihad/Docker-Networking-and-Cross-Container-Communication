# Docker Container Communication Demo

> A hands-on TypeScript + Express project demonstrating all three Docker networking patterns: container-to-internet, container-to-host, and container-to-container — each exposed as a dedicated REST endpoint.

![Node.js](https://img.shields.io/badge/Node.js-20--alpine-339933?logo=node.js&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-6.x-3178C6?logo=typescript&logoColor=white)
![Express](https://img.shields.io/badge/Express-5.x-000000?logo=express&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-Mongoose%209.x-47A248?logo=mongodb&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-ready-2496ED?logo=docker&logoColor=white)

---

## Table of Contents

- [Overview](#overview)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Build the Image](#build-the-image)
- [Demo 1 — Container → Internet](#demo-1--container--internet)
- [Demo 2 — Container → Host Machine / Atlas](#demo-2--container--host-machine--atlas)
- [Demo 3 — Container → Container](#demo-3--container--container)
- [Environment Variables](#environment-variables)
- [Cleanup](#cleanup)

---

## Overview

```
┌────────────────────────────────────────────────────────────┐
│                                                            │
│   Container (Express app)                                  │
│                                                            │
│   GET /internet/:id  ──────────────────────► 🌍 Internet  │
│   GET /host-db       ──────────────────────► 🖥️  Host     │
│   GET /container-db  ──────────────────────► 📦 Container │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

| Endpoint | Pattern | Key Mechanism |
|---|---|---|
| `GET /internet/:id` | Container → Internet | Docker NAT — no config needed |
| `GET /host-db` | Container → Host Machine | `host.docker.internal` hostname |
| `GET /container-db` | Container → Container | Custom Docker network + DNS |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js 20 (Alpine) |
| Language | TypeScript 6 |
| Framework | Express 5 |
| Database | MongoDB via Mongoose 9 |
| HTTP Client | Axios |
| Container | Docker |

---

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) installed and running
- (Optional) MongoDB running locally on port `27017` for Demo 2 host option

---

## Build the Image

```bash
docker build -t docker-demo .
```

The Dockerfile compiles TypeScript to JavaScript (`dist/`) inside the image — no local `npm run build` required.

---

## Demo 1 — Container → Internet

Containers can reach the public internet by default. Docker NAT handles outbound routing automatically — no `--network` flag needed.

```bash
# Run the container
docker run --rm -p 3000:3000 docker-demo

# Call the endpoint (use any post ID 1–100)
curl http://localhost:3000/internet/1
```

**Expected response:**

```json
{
  "demo": "Container → Internet ✅",
  "data": {
    "userId": 1,
    "id": 1,
    "title": "sunt aut facere repellat provident...",
    "body": "quia et suscipit..."
  }
}
```

The container fetched this from `https://jsonplaceholder.typicode.com` — a public test API — entirely from inside the container.

---

## Demo 2 — Container → Host Machine / Atlas

Inside a container, `localhost` refers to the container itself, not your machine. Docker provides `host.docker.internal` — a special hostname that always resolves to your host machine's IP.

### Option A — Local MongoDB on your machine

```bash
docker run --rm -d --name demo-app -p 3000:3000 \
  -e MONGO_ATLAS_URI="mongodb://host.docker.internal:27017/demo" \
  docker-demo

curl http://localhost:3000/host-db
```

### Option B — MongoDB Atlas (cloud)

```bash
docker run --rm -d --name demo-app -p 3000:3000 \
  -e MONGO_ATLAS_URI="mongodb+srv://<user>:<password>@<cluster>.mongodb.net/demo" \
  docker-demo

curl http://localhost:3000/host-db
```

**Expected response:**

```json
{
  "demo": "Container → Host / Atlas ✅",
  "items": [
    { "_id": "...", "name": "Hello from host DB" }
  ]
}
```

---

## Demo 3 — Container → Container

On a custom Docker network, each container's name acts as a resolvable DNS hostname. Any container on the same network can reach another by name — no IP addresses needed.

```bash
# 1. Create a shared network
docker network create demo-net

# 2. Start a MongoDB container on that network
docker run -d --name mongo-container --network demo-net \
  -e MONGO_INITDB_ROOT_USERNAME=root \
  -e MONGO_INITDB_ROOT_PASSWORD=secret \
  mongo:7

# 3. Start the app on the same network
docker run -d --name demo-app --network demo-net -p 3000:3000 \
  -e MONGO_CONTAINER_URI="mongodb://root:secret@mongo-container:27017/demo?authSource=admin" \
  docker-demo

# 4. Test the endpoint
curl http://localhost:3000/container-db
```

**Expected response:**

```json
{
  "demo": "Container → Container (MongoDB) ✅",
  "items": [
    { "_id": "...", "name": "Hello from container DB" }
  ]
}
```

Docker's internal DNS resolves `mongo-container` to its IP on `demo-net` — the connection never leaves the virtual network.

---

## Environment Variables

| Variable | Required By | Example Value | Description |
|---|---|---|---|
| `MONGO_ATLAS_URI` | `GET /host-db` | `mongodb://host.docker.internal:27017/demo` | MongoDB URI for host or Atlas connection |
| `MONGO_CONTAINER_URI` | `GET /container-db` | `mongodb://root:secret@mongo-container:27017/demo?authSource=admin` | MongoDB URI for container-to-container connection |

---

## Cleanup

```bash
# Stop and remove containers
docker stop demo-app mongo-container
docker rm demo-app mongo-container

# Remove the custom network
docker network rm demo-net

# Remove the image (optional)
docker rmi docker-demo
```
