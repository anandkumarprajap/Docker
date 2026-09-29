# Docker Containers, Network and Port Mapping — Revision Notes

## 1. Overview

In a multi-container application, containers can communicate with each other through a **Docker network**.

For the Devboard application:

```text
Frontend
   ↓
Backend
   ↓
PostgreSQL
```

We use one Docker network:

```text
devboard-net
```

Architecture:

```text
                         HOST / EC2
                            │
             ┌──────────────┼──────────────┐
             │              │              │
          :8080           :8081          :5432
             │              │              │
             ▼              ▼              ▼
        ┌─────────┐    ┌─────────┐    ┌──────────┐
        │Frontend │    │ Backend │    │Postgres  │
        │  :4173  │    │  :8080  │    │  :5432   │
        └────┬────┘    └────┬────┘    └──────────┘
             │              │
             └──────┐  ┌────┘
                    │  │
                    ▼  ▼
              devboard-net
```

---

# 2. Docker Network

Create a custom Docker network:

```bash
docker network create devboard-net
```

Check networks:

```bash
docker network ls
```

Example:

```text
NETWORK ID     NAME            DRIVER
xxxxxx         bridge          bridge
xxxxxx         devboard-net    bridge
```

Inspect the network:

```bash
docker network inspect devboard-net
```

This shows the containers connected to the network.

Example:

```text
devboard-net
│
├── postgres
├── backend
└── frontend
```

---

# 3. Why Use a Docker Network?

A Docker network allows containers to communicate with each other.

For example:

```text
backend → postgres:5432
```

The backend does not need to know the PostgreSQL container's IP address.

Docker provides internal DNS.

Therefore:

```text
postgres
```

can be used as the hostname for the PostgreSQL container.

---

# 4. Docker Image vs Container

## Docker Image

An image is a package/template used to create a container.

Example:

```bash
docker build -t devboard-backend .
```

creates:

```text
devboard-backend
```

image.

## Docker Container

A container is a running instance of an image.

Example:

```bash
docker run -d devboard-backend
```

creates and starts a container from the image.

### Remember

```text
Dockerfile
    ↓
docker build
    ↓
Docker Image
    ↓
docker run
    ↓
Container
```

---

# 5. Build Backend Image

Go to the backend directory:

```bash
cd backend
```

Build the image:

```bash
docker build -t devboard-backend .
```

Check images:

```bash
docker images
```

Example:

```text
REPOSITORY          TAG       IMAGE ID
devboard-backend    latest    xxxxx
```

---

# 6. Build Frontend Image

Go to the frontend directory:

```bash
cd frontend
```

Build:

```bash
docker build -t devboard-frontend .
```

Check:

```bash
docker images
```

You should have:

```text
devboard-backend
devboard-frontend
postgres
```

---

# 7. Run PostgreSQL Container

Command:

```bash
docker run -d \
  --name postgres \
  --network devboard-net \
  -e POSTGRES_USER=devboard \
  -e POSTGRES_PASSWORD=devboard \
  -e POSTGRES_DB=devboard \
  -v "$PWD/init/postgres":/docker-entrypoint-initdb.d:ro \
  -p 5432:5432 \
  postgres:16-alpine
```

## Explanation

### `docker run`

Creates and starts a container.

### `-d`

Runs the container in detached/background mode.

```text
-d = detached mode
```

### `--name postgres`

Gives the container the name:

```text
postgres
```

### `--network devboard-net`

Connects the container to:

```text
devboard-net
```

### `-e`

Sets environment variables.

```bash
-e POSTGRES_USER=devboard
-e POSTGRES_PASSWORD=devboard
-e POSTGRES_DB=devboard
```

### `-v`

Mounts the PostgreSQL initialization directory.

```bash
-v "$PWD/init/postgres":/docker-entrypoint-initdb.d:ro
```

`ro` means:

```text
read-only
```

### `-p 5432:5432`

Maps:

```text
HOST PORT       CONTAINER PORT
5432       →    5432
```

### `postgres:16-alpine`

The PostgreSQL image.

---

# 8. PostgreSQL Port Mapping

```bash
-p 5432:5432
```

means:

```text
EC2 / HOST
Port 5432
     │
     ▼
PostgreSQL Container
Port 5432
```

So the host can access PostgreSQL through:

```text
EC2-IP:5432
```

However, the backend does **not** need this port mapping to connect to PostgreSQL.

---

# 9. Run Backend Container

Command:

```bash
docker run -d \
  --name backend \
  --network devboard-net \
  -e PORT=8080 \
  -e POSTGRES_URL="postgres://devboard:devboard@postgres:5432/devboard?sslmode=disable" \
  -p 8081:8080 \
  devboard-backend
```

## Explanation

### Container name

```bash
--name backend
```

Container name:

```text
backend
```

### Network

```bash
--network devboard-net
```

The backend joins:

```text
devboard-net
```

### Backend port

```bash
-e PORT=8080
```

The application listens inside the container on:

```text
8080
```

### PostgreSQL URL

```text
postgres://devboard:devboard@postgres:5432/devboard?sslmode=disable
```

Important part:

```text
postgres:5432
```

Here:

```text
postgres
```

is the PostgreSQL container hostname.

```text
5432
```

is the PostgreSQL container port.

---

# 10. Backend Port Mapping

```bash
-p 8081:8080
```

means:

```text
HOST / EC2                  BACKEND CONTAINER

Port 8081  ──────────────→  Port 8080
```

Therefore:

```bash
curl http://EC2-IP:8081
```

can reach the backend application running on:

```text
backend:8080
```

---

# 11. Run Frontend Container

Command:

```bash
docker run -d \
  --name frontend \
  --network devboard-net \
  -p 8080:4173 \
  devboard-frontend
```

The frontend application runs inside the container on:

```text
4173
```

The host exposes it on:

```text
8080
```

Therefore:

```text
HOST / EC2
Port 8080
    │
    ▼
Frontend Container
Port 4173
```

Open:

```text
http://EC2-IP:8080
```

---

# 12. Complete Port Mapping

| Component  | Host Port | Container Port |
| ---------- | --------: | -------------: |
| Frontend   |    `8080` |         `4173` |
| Backend    |    `8081` |         `8080` |
| PostgreSQL |    `5432` |         `5432` |

Visual:

```text
EC2 :8080
    │
    ▼
Frontend :4173


EC2 :8081
    │
    ▼
Backend :8080


EC2 :5432
    │
    ▼
PostgreSQL :5432
```

---

# 13. Most Important Concept: `HOST:CONTAINER`

Whenever you see:

```bash
-p HOST_PORT:CONTAINER_PORT
```

remember:

```text
-p 8081:8080
   │    │
   │    └── Application port inside container
   └─────── Port exposed on host
```

Example:

```bash
-p 8080:4173
```

means:

```text
Host 8080 → Container 4173
```

It does **not** mean:

```text
Frontend → Backend
```

Port publishing is primarily for **host/outside-to-container** access.

---

# 14. Container-to-Container Communication

Containers connected to the same Docker network can communicate directly.

Our network:

```text
devboard-net
```

contains:

```text
frontend
backend
postgres
```

Therefore:

```text
backend → postgres:5432
```

works.

The backend does not need:

```text
EC2-IP:5432
```

and does not need:

```text
localhost:5432
```

---

# 15. Why Does `postgres:5432` Work?

Docker provides internal DNS for containers on the same network.

Suppose:

```text
devboard-net
│
├── backend
│
└── postgres
```

The backend requests:

```text
postgres:5432
```

Docker resolves:

```text
postgres
    ↓
PostgreSQL container IP
```

Then the connection becomes:

```text
Backend
   │
   │ postgres:5432
   ▼
PostgreSQL
```

---

# 16. Why Not `localhost:5432`?

Inside the backend container:

```text
localhost
```

means:

```text
backend container itself
```

It does **not** mean:

```text
PostgreSQL container
```

Therefore this is wrong for container-to-container communication:

```text
localhost:5432
```

Use:

```text
postgres:5432
```

instead.

---

# 17. Why Not `EC2-IP:5432`?

The backend and PostgreSQL are already connected to:

```text
devboard-net
```

Therefore the direct internal connection is:

```text
postgres:5432
```

Using:

```text
EC2-IP:5432
```

would unnecessarily go through the host's published port.

---

# 18. Full Architecture

```text
                         INTERNET
                            │
                            │
                     EC2 Public IP
                            │
              ┌─────────────┴─────────────┐
              │                           │
           :8080                        :8081
              │                           │
              ▼                           ▼
       ┌─────────────┐             ┌─────────────┐
       │  FRONTEND   │             │   BACKEND   │
       │             │             │             │
       │   :4173     │             │    :8080    │
       └──────┬──────┘             └──────┬──────┘
              │                           │
              │                           │
              │       devboard-net        │
              │                           │
              └───────────────────────────┤
                                          │
                                          │ postgres:5432
                                          ▼
                                   ┌─────────────┐
                                   │  POSTGRES   │
                                   │             │
                                   │    :5432    │
                                   └─────────────┘
```

---

# 19. Host-to-Container vs Container-to-Container

## Host → Container

Uses published ports:

```text
EC2:8080 → frontend:4173
EC2:8081 → backend:8080
EC2:5432 → postgres:5432
```

## Container → Container

Uses:

```text
container-name:container-port
```

Examples:

```text
backend → postgres:5432
```

Potentially:

```text
frontend → backend:8080
```

if the request is being made from another container.

---

# 20. Important Frontend Browser Note

There is an important difference between:

```text
Frontend container
```

and:

```text
User's browser
```

If JavaScript running in the user's browser needs to call the backend, the browser cannot normally resolve Docker's internal hostname:

```text
backend:8080
```

The browser is outside the Docker network.

Therefore the browser may need a host-accessible API URL such as:

```text
http://EC2-IP:8081
```

For production, a reverse proxy/domain is usually preferable.

---

# 21. Check Running Containers

```bash
docker ps
```

Example:

```text
CONTAINER ID   IMAGE                 PORTS
xxxx           postgres:16-alpine   0.0.0.0:5432->5432/tcp
xxxx           devboard-backend     0.0.0.0:8081->8080/tcp
xxxx           devboard-frontend    0.0.0.0:8080->4173/tcp
```

The PORTS column tells you the host-to-container mapping.

---

# 22. Check Network

```bash
docker network inspect devboard-net
```

Look for:

```text
Containers
```

You should find:

```text
postgres
backend
frontend
```

Visual:

```text
devboard-net
│
├── postgres
│
├── backend
│
└── frontend
```

---

# 23. Test Backend

If your backend has:

```text
GET /health
```

run from the host:

```bash
curl http://localhost:8081/health
```

Or:

```bash
curl http://EC2-IP:8081/health
```

Expected example:

```json
{
  "status": "ok"
}
```

---

# 24. Test Frontend

From a browser:

```text
http://EC2-IP:8080
```

Or:

```bash
curl http://localhost:8080
```

---

# 25. Check PostgreSQL

Check container:

```bash
docker ps
```

Check logs:

```bash
docker logs postgres
```

Check PostgreSQL readiness:

```bash
docker exec postgres pg_isready
```

Expected:

```text
accepting connections
```

---

# 26. Enter Backend Container

```bash
docker exec -it backend sh
```

Inside the container:

```bash
env
```

Check PostgreSQL variable:

```bash
echo $POSTGRES_URL
```

Expected:

```text
postgres://devboard:devboard@postgres:5432/devboard?sslmode=disable
```

Exit:

```bash
exit
```

---

# 27. Container Logs

Backend:

```bash
docker logs backend
```

Follow logs:

```bash
docker logs -f backend
```

Frontend:

```bash
docker logs frontend
```

PostgreSQL:

```bash
docker logs postgres
```

---

# 28. Stop Containers

```bash
docker stop frontend backend postgres
```

Remove:

```bash
docker rm frontend backend postgres
```

Or force remove:

```bash
docker rm -f frontend backend postgres
```

---

# 29. Remove Docker Network

After removing containers:

```bash
docker network rm devboard-net
```

Check:

```bash
docker network ls
```

---

# 30. Complete Manual Deployment Flow

```bash
# 1. Create network
docker network create devboard-net

# 2. Build backend image
cd backend
docker build -t devboard-backend .

# 3. Build frontend image
cd ../frontend
docker build -t devboard-frontend .

# 4. Run PostgreSQL
docker run -d \
  --name postgres \
  --network devboard-net \
  -e POSTGRES_USER=devboard \
  -e POSTGRES_PASSWORD=devboard \
  -e POSTGRES_DB=devboard \
  -v "$PWD/../init/postgres":/docker-entrypoint-initdb.d:ro \
  -p 5432:5432 \
  postgres:16-alpine

# 5. Run backend
docker run -d \
  --name backend \
  --network devboard-net \
  -e PORT=8080 \
  -e POSTGRES_URL="postgres://devboard:devboard@postgres:5432/devboard?sslmode=disable" \
  -p 8081:8080 \
  devboard-backend

# 6. Run frontend
docker run -d \
  --name frontend \
  --network devboard-net \
  -p 8080:4173 \
  devboard-frontend

# 7. Check containers
docker ps

# 8. Check network
docker network inspect devboard-net

# 9. Test backend
curl http://localhost:8081/health

# 10. Test frontend
curl http://localhost:8080
```

> **Note:** In step 4, the `$PWD` path depends on your current directory. Make sure it points to the project's `init/postgres` directory.

---

# 31. Docker Compose Equivalent

Instead of manually creating:

```text
network
images
containers
port mappings
environment variables
```

Docker Compose can manage them from:

```text
docker-compose.yml
```

Then simply:

```bash
docker compose up -d --build
```

Compose automatically creates a default network.

For example:

```text
devboard_default
│
├── frontend
├── backend
└── postgres
```

The backend can still connect using:

```text
postgres:5432
```

---

# 32. Manual Docker vs Docker Compose

| Manual Docker                           | Docker Compose                  |
| --------------------------------------- | ------------------------------- |
| `docker network create`                 | Network automatically created   |
| `docker build`                          | `docker compose build`          |
| `docker run`                            | `docker compose up`             |
| Many commands                           | One command                     |
| Environment variables manually supplied | `.env` can be used              |
| Port mapping manually supplied          | Defined in YAML                 |
| Volume manually supplied                | Defined in YAML                 |
| Harder to manage multiple containers    | Easier for multi-container apps |

---

# 33. Quick Revision

### Image

```bash
docker build -t devboard-backend .
```

Creates an image.

### Network

```bash
docker network create devboard-net
```

Creates a network.

### Container

```bash
docker run ...
```

Creates and starts a container.

### Host → Container

```text
-p HOST:CONTAINER
```

Example:

```text
8081:8080
```

means:

```text
Host 8081 → Backend 8080
```

### Container → Container

```text
container-name:port
```

Example:

```text
postgres:5432
```

means:

```text
Backend → PostgreSQL container port 5432
```

### Main Rule

```text
-p 8081:8080
```

is for:

```text
HOST → CONTAINER
```

while:

```text
postgres:5432
```

is for:

```text
CONTAINER → CONTAINER
```

### Final Mental Model

```text
                  Docker Host
                       │
         ┌─────────────┼─────────────┐
         │             │             │
      :8080         :8081          :5432
         │             │             │
         ▼             ▼             ▼
    Frontend       Backend       PostgreSQL
      :4173         :8080           :5432
         │             │
         └──────┐ ┌────┘
                │ │
                ▼ ▼
            devboard-net

Backend connection:
postgres:5432

NOT:
localhost:5432
NOT:
EC2-IP:5432
```

**Remember:**

> **Published ports (`-p`) are for reaching containers through the host. Docker-network communication uses the container/service name and the container's internal port.**
