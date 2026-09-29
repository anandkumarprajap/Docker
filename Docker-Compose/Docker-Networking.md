# Docker Networking with Docker Compose — Devboard

## 1. What is Docker networking?

Docker networking allows containers to communicate with one another and, when configured, with the host machine or external networks.

In this Devboard lab, Docker Compose runs three services:

- **frontend** — the web UI, listening on port `4173` inside its container.
- **backend** — the Go API, listening on port `8080` inside its container.
- **postgres** — the PostgreSQL database, listening on port `5432` inside its container.

Compose creates a project network named `devboard_default` automatically when no custom network is declared. In this lab, it is a local **bridge** network.

## 2. Devboard network architecture

```mermaid
flowchart TB
    User["Browser / User"]
    Host["EC2 Ubuntu host"]
    subgraph Compose["Docker Compose project: devboard"]
      subgraph Net["Bridge network: devboard_default (172.18.0.0/16)"]
        FE["frontend container<br/>devboard-frontend<br/>4173"]
        BE["backend container<br/>devboard-backend<br/>8080"]
        DB["PostgreSQL container<br/>devboard-db<br/>5432"]
      end
      Vol[("Named volume: devboard_pgdata")]
    end
    User -->|"HTTP :8080 (host)"| Host
    Host -->|"published 8080 → 4173"| FE
    FE -->|"API/proxy via service DNS: backend:8080"| BE
    BE -->|"PostgreSQL URL host: postgres, port: 5432"| DB
    DB --- Vol
```

**Key idea:** containers on the same Compose network use **service names** for DNS discovery. The backend connects to `postgres:5432`, not `localhost:5432`. Inside the backend container, `localhost` means the backend container itself.

The frontend's Vite configuration proxies API requests to `http://backend:8080` according to the `.env.example` comments in this lab.

## 3. Network and port mapping

The observed `docker ps` output showed these published ports:

| Service | Container port | Host port | Access |
|---|---:|---:|---|
| frontend | `4173` | `8080` | Browser: `http://<EC2-PUBLIC-IP>:8080` |
| backend | `8080` | `8081` | Host/debug: `http://<EC2-PUBLIC-IP>:8081` |
| postgres | `5432` | `5432` | Optional database debugging |

Port mapping uses the format `HOST_PORT:CONTAINER_PORT`. For example, `"8080:4173"` means requests arriving at host port `8080` are forwarded to port `4173` in the frontend container.

For an EC2 lab, allow the required web port in the instance security group. Avoid exposing PostgreSQL (`5432`) to the public internet; for routine app traffic, the backend should reach it over the private Compose network. If database host access is not needed, remove the database `ports` mapping.

## 4. Docker Compose file explained

The lab's `docker-compose.yml` defines the services and persistent volume:

```yaml
services:
  postgres:
    container_name: devboard-db
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
    container_name: devboard-backend
    build: ./backend
    environment:
      PORT: ${BACKEND_PORT}
      POSTGRES_URL: "postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}?sslmode=disable"
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "8081:8080"

  frontend:
    container_name: devboard-frontend
    build: ./frontend
    depends_on:
      - backend
    ports:
      - "8080:4173"

volumes:
  pgdata:
```

### Important Compose fields

- `services`: declares the containers Compose should run.
- `image`: uses an existing image, here `postgres:16-alpine`.
- `build`: builds an image from the specified directory (`./backend` or `./frontend`).
- `container_name`: sets a fixed container name for this lab.
- `environment`: injects configuration values from the `.env` file.
- `volumes`: persists PostgreSQL data in the named volume `pgdata`; the initialization scripts are mounted read-only.
- `healthcheck`: tests whether PostgreSQL is ready to accept connections.
- `depends_on` with `condition: service_healthy`: waits for the database health check before starting the backend. The frontend's short `depends_on` expresses startup order, not application-level readiness.
- `ports`: publishes container ports on the EC2 host.

Compose automatically places these services on its default project network, `devboard_default`; no explicit `networks:` section is required for this setup.

## 5. Environment file

The lab copied `.env.example` to `.env`:

```bash
cp .env.example .env
```

The example values were:

```dotenv
POSTGRES_USER=devboard
POSTGRES_PASSWORD=devboard
POSTGRES_DB=devboard
BACKEND_PORT=8080

POSTGRES_HOST_PORT=5432
BACKEND_HOST_PORT=8081
FRONTEND_HOST_PORT=8080
```

Compose substitutes `${VARIABLE}` references in the YAML using the environment file. The shown Compose file directly specifies the port mappings, so the `*_HOST_PORT` variables are documented in `.env.example` but are not used by those literal mappings unless the YAML is changed to reference them.

These are demo credentials only. Use strong, unique secrets for any non-lab deployment, and do not commit `.env`.

## 6. Install and start the lab

On the Ubuntu EC2 instance, the recorded setup was:

```bash
sudo apt-get update
sudo apt install docker.io
sudo usermod -aG docker "$USER"
newgrp docker
sudo apt-get install docker-compose-v2

docker ps
docker compose version
```

Clone the project and use the `advanced` branch:

```bash
git clone https://github.com/anandkumarprajap/devboard.git
cd devboard
git checkout advanced

cp .env.example .env
```

Start the application in the foreground:

```bash
docker compose up
```

Or run it in the background:

```bash
docker compose up -d
```

Build images and start services in one operation:

```bash
docker compose up -d --build
```

## 7. Docker network commands

List Docker networks:

```bash
docker network ls
```

Inspect the Compose network:

```bash
docker network inspect devboard_default
```

The lab's inspection output showed:

- Driver: `bridge`
- Subnet: `172.18.0.0/16`
- Gateway: `172.18.0.1`
- `devboard-db`: `172.18.0.2`
- `devboard-backend`: `172.18.0.3`
- `devboard-frontend`: `172.18.0.4`

Container IPs are dynamically assigned and can change after recreation. Prefer service DNS names (`postgres`, `backend`, `frontend`) instead of hard-coding container IP addresses.

Show Compose services and status:

```bash
docker compose ps
docker ps
```

View service logs:

```bash
docker compose logs
docker compose logs -f backend
docker compose logs -f postgres
docker compose logs -f frontend
```

Inspect a container's network attachment:

```bash
docker inspect devboard-backend
```

Check PostgreSQL readiness from inside its container:

```bash
docker exec devboard-db pg_isready
```

Open a shell in the database container and connect with `psql`:

```bash
docker exec -it devboard-db sh
psql -U devboard -d devboard
```

In `psql`, useful commands include:

```sql
\l
\dt
\d tasks
SELECT * FROM tasks;
\q
```

## 8. Test the application

From the EC2 host:

```bash
curl http://localhost:8080
curl http://localhost:8081/health
```

The first request should return the frontend HTML. The second targets the backend's published host port and should return the API health response if the backend is healthy.

From a browser, open:

```text
http://<EC2-PUBLIC-IP>:8080
```

Ensure the EC2 security group permits inbound TCP `8080` from your intended client IP range. Keep `8081` and especially `5432` restricted to trusted sources, or avoid publishing them when not needed.

## 9. Start, stop, and clean up

```bash
# Stop and remove project containers and the default network
docker compose down

# Stop containers without removing them
docker compose stop

# Start existing containers
docker compose start

# Rebuild and recreate services
docker compose up -d --build

# Remove containers, network, and named volumes (DELETES database data)
docker compose down -v
```

**Warning:** `docker compose down -v` removes the named `pgdata` volume and deletes the lab database contents. Do not use it if you need to preserve the data.

## 10. Troubleshooting notes

| Symptom | Check |
|---|---|
| `docker: command not found` | Install Docker Engine package (`docker.io`) and verify with `docker --version`. |
| Permission denied on Docker socket | Add the user to the `docker` group, then start a new login session or run `newgrp docker`. |
| `git checkout` says “not a git repository” | Change into the cloned project directory before running Git commands. |
| Backend cannot connect to database | Confirm both services are on `devboard_default`, the database is healthy, and the connection host is `postgres` with port `5432`. |
| Container IP changed | Use Compose service DNS names rather than saved IP addresses. |
| Browser cannot reach frontend | Check `docker compose ps`, host-to-container port mapping (`8080:4173`), EC2 security-group inbound rules, and frontend logs. |
| `docker compose up` stops when SSH exits | Foreground mode is attached to the terminal; use `docker compose up -d` for detached operation. |
| Build warns that Buildx is missing | Compose may still build with its default builder, as in this lab; install the Docker Buildx package if you need Bake/buildx features. |

## 11. What the lab demonstrated

1. Installed Docker Engine and Docker Compose V2 on Ubuntu.
2. Cloned the Devboard repository and checked out `advanced`.
3. Configured the Compose file and copied `.env.example` to `.env`.
4. Built the frontend and backend images and pulled the PostgreSQL image.
5. Compose created `devboard_default` and the `devboard_pgdata` volume.
6. PostgreSQL became healthy; the backend connected to PostgreSQL; all three containers started.
7. Inspected the network and observed the three containers with addresses on the same bridge subnet.
8. Verified published ports and tested the frontend/backend/database with `curl`, `docker exec`, and `psql`.

**Core takeaway:** Docker Compose provides a shared project network and service-name DNS, so containers can communicate by service name. Published ports are for access from outside the container network; they are not needed for service-to-service communication on the Compose network.
