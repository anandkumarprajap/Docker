# Docker Network Modes

Docker provides different network modes that control how containers communicate with the host, other containers, and the internet.

The three important built-in Docker network modes are:

1. `bridge`
2. `host`
3. `none`

---

## 1. Bridge Network

### Definition

`bridge` is the **default Docker network mode**.

When a container is created without specifying a network, Docker normally connects it to the default `bridge` network.

### Example

```bash
docker run -d --name web nginx
```

Check networks:

```bash
docker network ls
```

Example output:

```text
NETWORK ID     NAME      DRIVER
xxxxxx         bridge    bridge
xxxxxx         host      host
xxxxxx         none      null
```

### Visualization

```text
HOST / EC2
┌─────────────────────────────────────┐
│                                     │
│       Docker Bridge Network         │
│                                     │
│       ┌───────────────┐             │
│       │ nginx         │             │
│       │ Container     │             │
│       │ 172.17.0.2    │             │
│       └───────────────┘             │
│                                     │
└─────────────────────────────────────┘
```

The container gets its **own Docker IP address**.

### Port Mapping Example

```bash
docker run -d \
  --name web \
  -p 8080:80 \
  nginx
```

Meaning:

```text
Host Port        Container Port

   8080     --->      80
```

Request flow:

```text
Browser
   |
   ↓
EC2 / Host :8080
   |
   ↓
Docker Bridge
   |
   ↓
Nginx Container :80
```

### Important

Bridge networking is commonly used for application containers.

For example:

```text
Frontend
   |
   ↓
Backend
   |
   ↓
PostgreSQL
```

For real applications, a **custom bridge network** is generally preferred over relying on the default bridge.

---

# 2. Host Network

### Definition

With `host` networking, the container uses the **host's network namespace** instead of getting a separate Docker network interface/IP.

### Example

```bash
docker run -d \
  --network host \
  --name web \
  nginx
```

### Visualization

```text
HOST / EC2
┌──────────────────────────────┐
│                              │
│       Host Network           │
│                              │
│   ┌──────────────────────┐   │
│   │ Nginx Container      │   │
│   │                      │   │
│   │ Uses Host Network    │   │
│   └──────────────────────┘   │
│                              │
└──────────────────────────────┘
```

The container does **not** get an isolated Docker IP in the normal bridge-network sense.

### Port Mapping

With host networking:

```bash
docker run --network host nginx
```

You normally do **not** use:

```bash
-p 8080:80
```

because the container is using the host's network directly.

If nginx listens on port `80`:

```text
Host :80
   |
   ↓
Nginx Container :80
```

### Important

`host` networking gives less network isolation between the container and host.

Use it only when sharing the host network is actually required.

---

# 3. None Network

### Definition

`none` disables normal container networking.

The container has only the loopback interface for local communication inside itself.

### Example

```bash
docker run -it \
  --network none \
  alpine sh
```

Inside the container:

```bash
ip addr
```

You will normally see:

```text
lo
127.0.0.1
```

### Visualization

```text
HOST
┌──────────────────────────────┐
│                              │
│   ┌──────────────────────┐   │
│   │ Container            │   │
│   │                      │   │
│   │ 127.0.0.1 (lo)       │   │
│   │                      │   │
│   │ ❌ Internet           │   │
│   │ ❌ Docker network     │   │
│   └──────────────────────┘   │
│                              │
└──────────────────────────────┘
```

Example:

```bash
docker run --rm \
  --network none \
  alpine ping 8.8.8.8
```

Normal external network connectivity will not be available.

### Use Case

`none` can be useful when a container should have **no normal network access**.

---

# Bridge vs Host vs None

| Network  | Separate Container Network | Uses Host Network | Internet | Port Mapping     |
| -------- | -------------------------- | ----------------- | -------- | ---------------- |
| `bridge` | ✅ Yes                      | ❌ No              | ✅ Yes    | Usually required |
| `host`   | ❌ No                       | ✅ Yes             | ✅ Yes    | ❌ Not normally   |
| `none`   | ❌ No normal network        | ❌ No              | ❌ No     | ❌ No             |

---

# Quick Memory Trick

```text
bridge
HOST
  |
  ↓
Docker Network
  |
  ↓
CONTAINER
Separate network
```

```text
host
HOST
  |
  └──── CONTAINER
       Same host network
```

```text
none
HOST

     ❌ Network

CONTAINER
127.0.0.1 only
```

---

# Most Important Commands

### List Docker networks

```bash
docker network ls
```

### Inspect a network

```bash
docker network inspect bridge
```

### Run container using bridge

```bash
docker run -d --name web nginx
```

### Run container using host

```bash
docker run -d --network host --name web nginx
```

### Run container with no network

```bash
docker run -it --network none alpine sh
```

---

# Devboard Example

For a 3-tier application like Devboard, a typical architecture is:

```text
                    Internet
                       |
                       ↓
                  Frontend
                       |
                       ↓
                  Backend API
                       |
                       ↓
                  PostgreSQL
```

Docker networking:

```text
┌────────────────────────────────────────────┐
│          Custom Docker Bridge              │
│                                            │
│  ┌────────────┐    ┌────────────┐          │
│  │ Frontend   │───▶│ Backend    │          │
│  └────────────┘    └─────┬──────┘          │
│                           │                 │
│                           ↓                 │
│                    ┌────────────┐           │
│                    │ PostgreSQL │           │
│                    └────────────┘           │
│                                            │
└────────────────────────────────────────────┘
```

The containers can communicate through the Docker network.

For example:

```text
backend → postgres:5432
```

The backend does not need to know PostgreSQL's changing container IP when using Docker's network DNS/service-name mechanism.

---

# Interview Revision

### Q1. What is Docker bridge network?

**Answer:**

> Bridge is Docker's default network mode. Containers connected to it have their own network interfaces and can communicate through Docker networking.

### Q2. What is host networking?

**Answer:**

> Host networking makes the container use the host's network namespace directly, so the container does not have the usual isolated Docker network.

### Q3. What is none networking?

**Answer:**

> None disables normal container networking and leaves the container with only its loopback interface.

### One-Line Revision

```text
bridge = Separate Docker network
host   = Use host network
none   = No normal network
```
