# Docker Compose – Devboard 3-Tier Application

## 1. What is Docker Compose?

Docker Compose is used to define and run **multiple Docker containers together** using one YAML file, normally:

```text
docker-compose.yml
```

For the Devboard application, we have three services:

```text
                    Docker Compose
                         │
        ┌────────────────┼────────────────┐
        │                │                │
     frontend          backend         postgres
     :4173              :8080            :5432
        │                │                │
        └───────────────►└──────────────►
                         database
```

### Services

| Service    | Purpose              | Port   |
| ---------- | -------------------- | ------ |
| `frontend` | Frontend application | `4173` |
| `backend`  | Backend/API          | `8080` |
| `postgres` | PostgreSQL database  | `5432` |

---

# 2. Install Docker Compose V2

Modern Docker installations use **Docker Compose V2**.

Check:

```bash
docker compose version
```

Example:

```text
Docker Compose version v2.x.x
```

> Important: Modern syntax is `docker compose`, not the old `docker-compose`.

For Ubuntu, if Docker is already installed:

```bash
sudo apt-get update
sudo apt-get install docker-compose-plugin
```

Verify:

```bash
docker compose version
```

You can also check Docker:

```bash
docker --version
```

---

# 3. Project Structure

Example project:

```text
devboard/
├── docker-compose.yml
├── .env
├── backend/
│   └── Dockerfile
├── frontend/
│   └── Dockerfile
└── init/
    └── postgres/
        ├── 01_schema.sql
        └── 02_command.sql
```

---

# 4. `.env` File

The `.env` file stores environment variables used by Docker Compose.

Example:

```env
POSTGRES_USER=devboard
POSTGRES_PASSWORD=devboard123
POSTGRES_DB=devboard

BACKEND_PORT=8080
```

Your Compose file reads these values using:

```yaml
${POSTGRES_USER}
${POSTGRES_PASSWORD}
${POSTGRES_DB}
${BACKEND_PORT}
```

### Example

If `.env` contains:

```env
POSTGRES_USER=devboard
POSTGRES_PASSWORD=devboard123
POSTGRES_DB=devboard
```

Then:

```yaml
POSTGRES_USER: ${POSTGRES_USER}
```

becomes effectively:

```yaml
POSTGRES_USER: devboard
```

### Check Compose configuration

You can see the resolved Compose configuration with:

```bash
docker compose config
```

This is useful for checking whether `.env` variables are being loaded correctly.

> **Security:** Do not commit `.env` containing real passwords to Git.

Add it to `.gitignore`:

```gitignore
.env
```

---

# 5. PostgreSQL Service

```yaml
postgres:
  container_name: "devboard-db"
  image: postgres:16-alpine

  environment:
    POSTGRES_USER: ${POSTGRES_USER}
    POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    POSTGRES_DB: ${POSTGRES_DB}
```

### Explanation

```yaml
image: postgres:16-alpine
```

Uses PostgreSQL 16 with the Alpine Linux base image.

```yaml
container_name: "devboard-db"
```

Gives the PostgreSQL container a fixed name.

```yaml
environment:
```

Passes PostgreSQL configuration into the container.

---

# 6. PostgreSQL Volume

```yaml
volumes:
  - pgdata:/var/lib/postgresql/data
```

This stores PostgreSQL data in a Docker named volume.

Without the volume:

```text
Container deleted
      ↓
Database data can be lost
```

With the volume:

```text
Container deleted
      ↓
pgdata volume remains
      ↓
Database data remains
```

Check volumes:

```bash
docker volume ls
```

Inspect:

```bash
docker volume inspect devboard_pgdata
```

The exact volume name can vary depending on the Compose project name.

---

# 7. PostgreSQL Initialization Scripts

```yaml
- ./init/postgres:/docker-entrypoint-initdb.d:ro
```

This mounts:

```text
./init/postgres
```

inside the PostgreSQL container as:

```text
/docker-entrypoint-initdb.d
```

For example:

```text
init/postgres/
├── 01_schema.sql
└── 02_command.sql
```

PostgreSQL executes initialization scripts when the database is initialized for the first time.

### Important

These scripts normally run only when PostgreSQL initializes a **new empty data directory**.

If `pgdata` already contains a database, simply running:

```bash
docker compose up
```

will not normally execute the initialization SQL again.

---

# 8. PostgreSQL Healthcheck

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
  interval: 5s
  timeout: 3s
  retries: 10
```

The healthcheck checks whether PostgreSQL is ready.

Command:

```bash
pg_isready
```

means:

```text
"Is PostgreSQL ready to accept connections?"
```

Compose waits for the database to become healthy before starting the backend because:

```yaml
depends_on:
  postgres:
    condition: service_healthy
```

---

# 9. PostgreSQL Port Mapping

```yaml
ports:
  - "5432:5432"
```

Format:

```text
HOST_PORT:CONTAINER_PORT
```

Therefore:

```text
5432:5432
```

means:

```text
EC2/Host machine port 5432
            ↓
PostgreSQL container port 5432
```

---

# 10. Backend Service

```yaml
backend:
  container_name: "devboard-backend"
  build: ./backend

  environment:
    PORT: ${BACKEND_PORT}
    POSTGRES_URL: "postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}?sslmode=disable"

  depends_on:
    postgres:
      condition: service_healthy

  ports:
    - "8080:8080"
```

## Build

```yaml
build: ./backend
```

Docker Compose looks inside:

```text
backend/
```

for a Dockerfile.

Example:

```text
backend/
└── Dockerfile
```

It builds the backend image from that Dockerfile.

---

# 11. Backend Database Connection

The backend uses:

```text
postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}?sslmode=disable
```

The important part is:

```text
@postgres
```

Here `postgres` is the **Compose service name**.

Inside the Compose network:

```text
backend
   │
   │ postgres:5432
   ▼
postgres
```

The backend should **not** normally use:

```text
localhost:5432
```

because inside the backend container:

```text
localhost
```

means:

```text
backend container itself
```

not the PostgreSQL container.

---

# 12. Frontend Service

```yaml
frontend:
  build: ./frontend

  depends_on:
    - backend

  ports:
    - "4173:4173"
```

The frontend image is built from:

```text
frontend/Dockerfile
```

Port mapping:

```text
Host:4173
    ↓
Container:4173
```

Therefore the application can be accessed through:

```text
http://SERVER_IP:4173
```

---

# 13. Docker Compose Default Network

When you run:

```bash
docker compose up
```

Docker Compose automatically creates a network for the project.

For example:

```text
devboard_default
```

Architecture:

```text
                devboard_default
                       │
        ┌──────────────┼──────────────┐
        │              │              │
    frontend        backend        postgres
      :4173           :8080           :5432
```

The containers can communicate with each other using **service names**.

For example:

```text
backend → postgres:5432
```

The backend does not need to know the PostgreSQL container IP address.

---

# 14. Why Does Compose Create a Network?

Without container networking, containers would not have convenient service-to-service communication.

Compose creates a private network so:

```text
frontend → backend
backend  → postgres
```

can communicate using service names.

Example:

```text
postgres
```

resolves to the PostgreSQL container through Docker's internal DNS.

---

# 15. Check Docker Networks

List networks:

```bash
docker network ls
```

Example:

```text
NETWORK ID     NAME              DRIVER
xxxxxx         bridge            bridge
xxxxxx         devboard_default  bridge
```

Inspect the Compose network:

```bash
docker network inspect devboard_default
```

You can see containers connected to the network.

Example concept:

```text
devboard_default
│
├── devboard-db
├── devboard-backend
└── devboard-frontend
```

If your project directory has a different Compose project name, the network name may be different.

Find it with:

```bash
docker network ls
```

---

# 16. Start the Application

From the directory containing `docker-compose.yml`:

```bash
docker compose up
```

This runs containers in the foreground.

You will see logs directly in the terminal.

---

# 17. Start in Background

For production-style/background execution:

```bash
docker compose up -d
```

`-d` means:

```text
detached mode
```

The containers continue running after you leave the terminal.

Check:

```bash
docker compose ps
```

or:

```bash
docker ps
```

---

# 18. Build and Start

If you have changed Dockerfiles or application code and need to rebuild images:

```bash
docker compose up --build
```

Background mode:

```bash
docker compose up -d --build
```

Flow:

```text
Dockerfile
    ↓
docker compose build
    ↓
New image
    ↓
Container
    ↓
Application
```

---

# 19. What Does `--build` Do?

Normal:

```bash
docker compose up -d
```

Compose can reuse an existing image.

With:

```bash
docker compose up -d --build
```

Compose first rebuilds the images when needed.

Use it when you change:

```text
Dockerfile
dependencies
package installation
system packages
build configuration
application files copied during image build
```

---

# 20. Rebuild Only Backend

You can rebuild a specific service:

```bash
docker compose build backend
```

Then:

```bash
docker compose up -d backend
```

Or:

```bash
docker compose up -d --build backend
```

Similarly:

```bash
docker compose up -d --build frontend
```

---

# 21. Stop the Application

```bash
docker compose down
```

This stops and removes the Compose containers and network.

Conceptually:

```text
docker compose up
        ↓
containers + network
        ↓
docker compose down
        ↓
containers removed
network removed
```

The named volume is normally **not removed** by plain:

```bash
docker compose down
```

Therefore PostgreSQL data in `pgdata` remains.

---

# 22. Remove Containers + Volume

If you intentionally want to delete the database data:

```bash
docker compose down -v
```

`-v` means remove volumes created for the Compose project.

### Warning

This can delete PostgreSQL data.

Therefore:

```bash
docker compose down
```

= remove containers/network, keep database volume

```bash
docker compose down -v
```

= remove containers/network + volumes

---

# 23. View Container Status

```bash
docker compose ps
```

Example:

```text
NAME                SERVICE      STATUS
devboard-db         postgres     healthy
devboard-backend    backend      running
devboard-frontend   frontend     running
```

---

# 24. View Logs

All services:

```bash
docker compose logs
```

Follow logs:

```bash
docker compose logs -f
```

Backend only:

```bash
docker compose logs backend
```

PostgreSQL:

```bash
docker compose logs postgres
```

Frontend:

```bash
docker compose logs frontend
```

Follow backend logs:

```bash
docker compose logs -f backend
```

---

# 25. Test Backend Using curl

If the backend exposes an endpoint such as:

```text
GET /health
```

run:

```bash
curl http://localhost:8080/health
```

From another machine:

```bash
curl http://SERVER_IP:8080/health
```

For example:

```bash
curl http://YOUR_EC2_PUBLIC_IP:8080/health
```

Expected response might be:

```json
{
  "status": "ok"
}
```

The exact response depends on your backend implementation.

---

# 26. Test Frontend

Open:

```text
http://YOUR_EC2_PUBLIC_IP:4173
```

Or test from the server:

```bash
curl http://localhost:4173
```

If the frontend server returns HTML, you may see HTML content in the terminal.

---

# 27. Test PostgreSQL

Check the container:

```bash
docker compose ps
```

Check PostgreSQL logs:

```bash
docker compose logs postgres
```

You can also enter the PostgreSQL container:

```bash
docker exec -it devboard-db sh
```

Then:

```bash
pg_isready
```

Exit:

```bash
exit
```

---

# 28. Execute PostgreSQL Client

If `psql` is available:

```bash
docker exec -it devboard-db psql -U devboard -d devboard
```

Inside PostgreSQL:

```sql
\dt
```

Show tables.

Exit:

```sql
\q
```

---

# 29. Check Environment Variables

View the Compose configuration:

```bash
docker compose config
```

You can also inspect the backend container:

```bash
docker exec devboard-backend env
```

Or:

```bash
docker exec devboard-backend env | grep POSTGRES
```

Avoid exposing real passwords in logs or screenshots.

---

# 30. Complete Docker Compose File

```yaml
services:

  postgres:
    container_name: "devboard-db"
    image: postgres:16-alpine

    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}

    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init/postgres:/docker-entrypoint-initdb.d:ro

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 3s
      retries: 10

    ports:
      - "5432:5432"


  backend:
    container_name: "devboard-backend"
    build: ./backend

    environment:
      PORT: ${BACKEND_PORT}
      POSTGRES_URL: "postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}?sslmode=disable"

    depends_on:
      postgres:
        condition: service_healthy

    ports:
      - "8080:8080"


  frontend:
    build: ./frontend

    depends_on:
      - backend

    ports:
      - "4173:4173"


volumes:
  pgdata:
```

---

# 31. Complete Workflow

## Step 1 — Go to project

```bash
cd ~/devboard
```

## Step 2 — Check files

```bash
ls
```

Expected:

```text
backend
frontend
init
docker-compose.yml
.env
```

## Step 3 — Check Compose

```bash
docker compose version
```

## Step 4 — Validate configuration

```bash
docker compose config
```

## Step 5 — Build images

```bash
docker compose build
```

## Step 6 — Start application

```bash
docker compose up -d
```

## Step 7 — Check containers

```bash
docker compose ps
```

## Step 8 — Check logs

```bash
docker compose logs -f
```

Press:

```text
Ctrl + C
```

to stop following logs.

The containers continue running because they were started with `-d`.

## Step 9 — Check network

```bash
docker network ls
```

Then:

```bash
docker network inspect devboard_default
```

## Step 10 — Test backend

```bash
curl http://localhost:8080/health
```

From another machine:

```bash
curl http://YOUR_EC2_PUBLIC_IP:8080/health
```

## Step 11 — Test frontend

Open:

```text
http://YOUR_EC2_PUBLIC_IP:4173
```

## Step 12 — Stop application

```bash
docker compose down
```

---

# 32. Important Commands – Quick Revision

| Command                             | Purpose                                 |
| ----------------------------------- | --------------------------------------- |
| `docker compose version`            | Check Compose version                   |
| `docker compose config`             | Validate/resolved Compose configuration |
| `docker compose build`              | Build images                            |
| `docker compose up`                 | Start services in foreground            |
| `docker compose up -d`              | Start services in background            |
| `docker compose up -d --build`      | Rebuild and start                       |
| `docker compose ps`                 | Show Compose services                   |
| `docker compose logs`               | Show logs                               |
| `docker compose logs -f`            | Follow logs                             |
| `docker compose down`               | Stop/remove containers and network      |
| `docker compose down -v`            | Also remove volumes                     |
| `docker compose restart`            | Restart services                        |
| `docker compose pull`               | Pull newer images                       |
| `docker network ls`                 | List Docker networks                    |
| `docker network inspect NAME`       | Inspect network                         |
| `docker volume ls`                  | List Docker volumes                     |
| `docker volume inspect NAME`        | Inspect volume                          |
| `curl http://localhost:8080/health` | Test backend                            |
| `curl http://localhost:4173`        | Test frontend                           |

---

# 33. Most Important Concept

Remember this architecture:

```text
                    EC2 / Host
                       │
          ┌────────────┴────────────┐
          │     Docker Compose      │
          │                         │
          │   devboard_default      │
          │                         │
          │  ┌───────────────┐      │
Port 4173 ├─►│   frontend    │      │
          │  └───────┬───────┘      │
          │          │              │
          │          ▼              │
Port 8080 ├─►┌───────────────┐      │
          │  │    backend    │      │
          │  └───────┬───────┘      │
          │          │              │
          │          │ postgres:5432│
          │          ▼              │
Port 5432 ├─►┌───────────────┐      │
          │  │   postgres    │      │
          │  └───────────────┘      │
          │                         │
          │       pgdata            │
          │          │              │
          │          ▼              │
          │   PostgreSQL data       │
          └─────────────────────────┘
```

### Key rule

**Host communication:**

```text
SERVER_IP:4173 → frontend
SERVER_IP:8080 → backend
SERVER_IP:5432 → PostgreSQL
```

**Container-to-container communication:**

```text
frontend → backend:8080
backend  → postgres:5432
```

The backend connects to:

```text
postgres
```

—not:

```text
localhost
```

because Docker Compose provides internal DNS/service discovery.

# 34. Production Security Note

For a production deployment, you normally should **not expose PostgreSQL publicly** with:

```yaml
ports:
  - "5432:5432"
```

If only the backend needs PostgreSQL, the database can communicate through the internal Compose network without publishing port `5432` to the host.

Similarly, make sure AWS Security Group rules expose only the ports that are actually required.

---

# 35. One-Line Interview Explanation

> "Docker Compose allows me to define and run my frontend, backend, and PostgreSQL services together. Compose automatically creates a default network so services can communicate using service names such as `postgres:5432`. I use a named volume for PostgreSQL persistence, a healthcheck to ensure the database is ready before the backend starts, and `.env` variables to keep configuration separate from the Compose file."



# Docker-Compose-yml

![Image 1](1.png)
![Image 2](2.png)
![Image 3](3.png)
![Image 4](4.png)
![Image 5](5.png)
![Image 6](6.png)
![Image 7](7.png)
![Image 8](8.png)
![Image 9](9.png)
![Image 10](10.png)
![Image 11](11.png)
![Image 12](12.png)
