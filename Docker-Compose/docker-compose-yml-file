# Docker Compose – Devboard 3-Tier Application

## 1. Introduction

Docker Compose is a tool used to define, configure, build, and run multiple Docker containers using a single YAML file.

In this project, Docker Compose manages three services:

1. **Frontend** – User interface of the Devboard application.
2. **Backend** – Application logic and API.
3. **PostgreSQL** – Database for storing application data.

### Architecture

```text
                 USER / WEB BROWSER
                         |
                         | HTTP :8080
                         v
             +------------------------+
             |       FRONTEND         |
             |   devboard-frontend    |
             |   Container Port 4173 |
             +------------------------+
                         |
                         | API Request
                         | Docker Network
                         v
             +------------------------+
             |        BACKEND         |
             |   devboard-backend     |
             |   Container Port 8080 |
             +------------------------+
                         |
                         | PostgreSQL
                         | postgres:5432
                         v
             +------------------------+
             |       POSTGRES         |
             |     devboard-db        |
             |   Container Port 5432 |
             +------------------------+
                         |
                         v
             +------------------------+
             |     DOCKER VOLUME      |
             |        pgdata          |
             |   Persistent Storage   |
             +------------------------+
```

**Communication flow:**

* Browser → Frontend: `localhost:8080`
* Frontend → Backend: backend API through the Docker network.
* Backend → PostgreSQL: `postgres:5432`
* PostgreSQL → Docker volume: persistent database storage.

> Note: The frontend must be configured with the correct backend API URL. If frontend JavaScript runs in the user's browser, it generally cannot resolve the Docker service name `backend`; use an externally accessible API URL, such as `http://localhost:8081`, for local development.

---

## 2. Project Directory Structure

```text
Devboard/
│
├── docker-compose.yml
├── .env
├── .gitignore
│
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   ├── src/
│   └── public/
│
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│
├── init/
│   └── postgres/
│       └── init.sql
│
└── README.md
```

**Directory explanation:**

| File / Directory     | Purpose                                                |
| -------------------- | ------------------------------------------------------ |
| `docker-compose.yml` | Defines and manages all three services.                |
| `.env`               | Stores environment variables and database credentials. |
| `frontend/`          | Contains frontend source code and its Dockerfile.      |
| `backend/`           | Contains backend source code and its Dockerfile.       |
| `init/postgres/`     | Contains SQL or shell initialization scripts.          |
| `pgdata`             | Docker-managed volume for persistent PostgreSQL data.  |
| `README.md`          | Project documentation.                                 |

---

## 3. Complete Docker Compose YAML File

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

---

## 4. Line-by-Line Explanation

### A. Services

```yaml
services:
```

* Defines the services that Docker Compose will manage.
* Each service represents an application component, typically running in its own container.
* This project defines three services: `postgres`, `backend`, and `frontend`.

### B. PostgreSQL Service

```yaml
  postgres:
```

* Defines the PostgreSQL service.
* The service name is `postgres`.
* Other containers on the same Compose network can connect to it using the hostname `postgres`.

```yaml
    container_name: devboard-db
```

* Sets the explicit container name to `devboard-db`.
* This makes the container easier to identify when running Docker commands.

```yaml
    image: postgres:16-alpine
```

* Specifies the PostgreSQL Docker image.
* `postgres` is the official PostgreSQL image.
* `16` specifies the PostgreSQL major version.
* `alpine` indicates a lightweight Alpine Linux-based image.

```yaml
    environment:
```

* Defines environment variables inside the PostgreSQL container.
* These variables configure the database during its initial setup.

```yaml
      POSTGRES_USER: ${POSTGRES_USER}
```

* Sets the PostgreSQL username.
* `${POSTGRES_USER}` is substituted from the environment or the Compose `.env` file.

```yaml
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

* Sets the PostgreSQL user's password.
* The password should be stored securely in `.env`, not hardcoded in the Compose file.

```yaml
      POSTGRES_DB: ${POSTGRES_DB}
```

* Specifies the default database created during initial initialization.
* The database name is obtained from the `.env` file.

**Example `.env` file:**

```dotenv
POSTGRES_USER=devboard
POSTGRES_PASSWORD=change_this_password
POSTGRES_DB=devboard_db
BACKEND_PORT=8080
```

Use a strong password for real deployments and keep `.env` out of Git.

### C. PostgreSQL Volumes

```yaml
    volumes:
```

* Defines storage mounts for the PostgreSQL container.
* Volumes preserve database data beyond the lifecycle of an individual container.

```yaml
      - pgdata:/var/lib/postgresql/data
```

* Mounts the named Docker volume `pgdata` at PostgreSQL's data directory.
* PostgreSQL stores its database files in `/var/lib/postgresql/data`.
* The data remains available when the container is recreated, provided the volume is retained.

```yaml
      - ./init/postgres:/docker-entrypoint-initdb.d:ro
```

* Mounts the project's `init/postgres` directory into the container.
* PostgreSQL's official image processes supported initialization scripts from `/docker-entrypoint-initdb.d` during initial database setup.
* `:ro` means read-only, so the container cannot modify the mounted host directory.
* These initialization scripts generally run only when the database data directory is first initialized, not on every restart.

### D. PostgreSQL Health Check

```yaml
    healthcheck:
```

* Defines a health check to determine whether PostgreSQL is ready to accept connections.

```yaml
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
```

* Runs the PostgreSQL utility `pg_isready`.
* `-U` specifies the database user.
* `-d` specifies the database.
* The health check succeeds when PostgreSQL reports that it is accepting connections.

```yaml
      interval: 5s
```

* Runs the health check every 5 seconds.

```yaml
      timeout: 3s
```

* Allows each health check up to 3 seconds to complete.

```yaml
      retries: 10
```

* Allows 10 consecutive unsuccessful health checks before marking the container unhealthy.

### E. PostgreSQL Ports

```yaml
    ports:
      - "5432:5432"
```

* Publishes the PostgreSQL container port to the host machine.
* The format is `HOST_PORT:CONTAINER_PORT`.
* The first `5432` is the host port.
* The second `5432` is the container port.

**Connection example:**

```text
Host machine: localhost:5432
Docker network: postgres:5432
```

The backend should use `postgres:5432` to access the database over the Compose network, not `localhost:5432`.

For a local development environment, consider removing the published database port or binding it only to `127.0.0.1` if host access is needed.

---

## 5. Backend Service

### A. Backend Definition

```yaml
  backend:
```

* Defines the backend service.
* The backend handles application logic and API requests.
* It communicates with the PostgreSQL service over the Docker network.

```yaml
    container_name: devboard-backend
```

* Sets the backend container name to `devboard-backend`.

```yaml
    build: ./backend
```

* Instructs Docker Compose to build an image using the Dockerfile in the `./backend` directory.
* The path is relative to the Compose file's directory.

**Example backend Dockerfile:**

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 8080

CMD ["npm", "start"]
```

This is an illustrative Dockerfile; adjust it to match the backend application's language, dependencies, build process, and start command.

### B. Backend Environment Variables

```yaml
    environment:
```

* Supplies configuration values to the backend container.

```yaml
      PORT: ${BACKEND_PORT}
```

* Sets the backend application's listening port using the value from `.env`.
* In this example, `BACKEND_PORT=8080`.

```yaml
      POSTGRES_URL: "postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}?sslmode=disable"
```

* Defines the PostgreSQL connection URL used by the backend.
* Its structure is:

```text
postgres://USERNAME:PASSWORD@HOST:PORT/DATABASE?sslmode=disable
```

| Component              | Meaning                                   |
| ---------------------- | ----------------------------------------- |
| `postgres://`          | PostgreSQL connection protocol            |
| `${POSTGRES_USER}`     | Database username                         |
| `${POSTGRES_PASSWORD}` | Database password                         |
| `postgres`             | Docker Compose service hostname           |
| `5432`                 | PostgreSQL container port                 |
| `${POSTGRES_DB}`       | Database name                             |
| `sslmode=disable`      | Disables TLS for this database connection |

> Important: `sslmode=disable` is suitable only for an appropriately secured local Docker network or lab. For production, configure database TLS. If the password contains URL-special characters, encode them correctly before placing them in the connection URL.

### C. Backend Dependency

```yaml
    depends_on:
      postgres:
        condition: service_healthy
```

* Defines the startup dependency between the backend and PostgreSQL.
* The backend waits for PostgreSQL to pass its configured health check before starting.
* `service_healthy` is more precise than merely starting the database container.

**Startup sequence:**

```text
Docker Compose starts
          |
          v
   PostgreSQL starts
          |
          v
    Health check runs
          |
          v
 PostgreSQL is healthy
          |
          v
    Backend starts
```

This controls startup order, but it does not replace application-level retry and reconnection handling if the database becomes unavailable later.

### D. Backend Ports

```yaml
    ports:
      - "8081:8080"
```

* Maps host port `8081` to backend container port `8080`.
* The backend listens on port `8080` inside its container.
* The host can access the backend through port `8081`.

```text
Host machine                  Backend container

localhost:8081  ----------->  devboard-backend:8080
     HOST PORT                   CONTAINER PORT
```

Example API address:

```text
http://localhost:8081
```

The backend application must listen on `0.0.0.0:8080` inside the container to accept connections through the published port.

---

## 6. Frontend Service

### A. Frontend Definition

```yaml
  frontend:
```

* Defines the frontend service.
* It serves the Devboard user interface.

```yaml
    container_name: devboard-frontend
```

* Sets the frontend container name to `devboard-frontend`.

```yaml
    build: ./frontend
```

* Builds the frontend Docker image using the Dockerfile in the `frontend` directory.

**Example frontend Dockerfile:**

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

RUN npm run build

RUN npm install -g vite

EXPOSE 4173

CMD ["vite", "preview", "--host", "0.0.0.0", "--port", "4173"]
```

This is an illustrative Vite production-preview setup. For production deployments, a static web server such as Nginx is another common option.

### B. Frontend Dependency

```yaml
    depends_on:
      - backend
```

* Declares that the backend service should start before the frontend service.
* In this short form, Compose waits for the backend container to start, but does not wait for the backend application to become ready.
* For readiness-based startup, configure a backend health check and use `condition: service_healthy`.

### C. Frontend Ports

```yaml
    ports:
      - "8080:4173"
```

* Maps host port `8080` to frontend container port `4173`.
* Users can access the frontend through the host's port `8080`.

```text
Browser                         Frontend container

localhost:8080  ------------->  devboard-frontend:4173
     HOST PORT                      CONTAINER PORT
```

Example browser URL:

```text
http://localhost:8080
```

The frontend server must listen on `0.0.0.0:4173` inside the container.

---

## 7. Docker Volumes

```yaml
volumes:
  pgdata:
```

* Declares a named Docker volume called `pgdata`.
* Compose creates and manages this volume when needed.
* PostgreSQL uses it to store persistent database files.
* Recreating or removing the database container does not normally remove this volume.

**Container versus volume:**

```text
+---------------------------+
|     PostgreSQL Container  |
|                           |
|  /var/lib/postgresql/data |
+-------------+-------------+
              |
              | Mounted volume
              v
+---------------------------+
|       Docker Volume       |
|          pgdata            |
|                           |
|    Persistent DB files    |
+---------------------------+
```

Useful commands:

```bash
# List Docker volumes
docker volume ls

# Inspect the database volume
docker volume inspect devboard_pgdata

# Remove unused volumes
docker volume prune
```

> Warning: `docker compose down -v` removes Compose-managed named volumes, including `pgdata`. This can permanently delete the database data. Back up important data before removing volumes.

---

## 8. Docker Compose Networking

Docker Compose automatically creates a default network for the project when no custom network is declared.

The three services can communicate using their service names:

```text
            Docker Compose Network
        +-------------------------------+
        |                               |
        |   frontend                    |
        |      |                        |
        |      | backend:8080           |
        |      v                        |
        |   backend                     |
        |      |                        |
        |      | postgres:5432          |
        |      v                        |
        |   postgres                    |
        |                               |
        +-------------------------------+
```

| Source             | Destination | Address          |
| ------------------ | ----------- | ---------------- |
| Browser            | Frontend    | `localhost:8080` |
| Host               | Backend     | `localhost:8081` |
| Backend container  | PostgreSQL  | `postgres:5432`  |
| Frontend container | Backend     | `backend:8080`   |

**Important distinction:**

* Containers communicate using service names and container ports.
* The host accesses published ports using `localhost` or the machine's IP address.
* A browser running outside Docker cannot ordinarily resolve the internal service hostname `backend`.

---

## 9. Create the `.env` File

Create a `.env` file in the same directory as `docker-compose.yml`.

```dotenv
POSTGRES_USER=devboard
POSTGRES_PASSWORD=change_this_password
POSTGRES_DB=devboard_db
BACKEND_PORT=8080
```

Add `.env` to `.gitignore`:

```gitignore
.env
```

Compose automatically reads a project `.env` file for variable substitution when running commands from the project directory.

Do not commit real database credentials to GitHub.

---

## 10. Docker Compose Commands

Run these commands from the directory containing `docker-compose.yml`.

### Check Docker and Compose

```bash
docker --version

docker compose version
```

### Validate the Compose File

```bash
docker compose config
```

* Checks and renders the resolved Compose configuration.
* Helps identify YAML errors and missing environment variables.
* Be careful: rendered configuration can contain sensitive values.

### Build All Services

```bash
docker compose build
```

Builds the images for services with a `build` configuration.

### Start All Containers

```bash
docker compose up -d
```

* Creates the network and volume if needed.
* Builds images when required.
* Creates and starts the containers in the background.

### Build and Start Together

```bash
docker compose up -d --build
```

Rebuilds images as needed and starts the application.

### Check Running Containers

```bash
docker compose ps
```

Displays the status of the Compose services.

### View Logs

```bash
# All service logs
docker compose logs

# Follow logs in real time
docker compose logs -f

# PostgreSQL logs
docker compose logs -f postgres

# Backend logs
docker compose logs -f backend

# Frontend logs
docker compose logs -f frontend
```

### Restart Services

```bash
docker compose restart
```

Restart all services.

To restart only the backend:

```bash
docker compose restart backend
```

### Stop Containers

```bash
docker compose stop
```

Stops the services while retaining the containers and volumes.

### Stop and Remove Containers

```bash
docker compose down
```

Stops and removes the Compose containers and default network. Named volumes are retained.

### Rebuild a Specific Service

```bash
docker compose build backend

docker compose up -d backend
```

### Access a Container Shell

```bash
# Backend shell
docker compose exec backend sh

# Frontend shell
docker compose exec frontend sh

# PostgreSQL command line
docker compose exec postgres \
  psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"
```

For the PostgreSQL command, ensure the variables are defined in your host shell, or use the actual database username and database name.

---

## 11. Verify the Application

After starting the services:

```bash
docker compose ps
```

Expected service mapping:

| Service    | Container           | Host Port | Container Port |
| ---------- | ------------------- | --------: | -------------: |
| Frontend   | `devboard-frontend` |      8080 |           4173 |
| Backend    | `devboard-backend`  |      8081 |           8080 |
| PostgreSQL | `devboard-db`       |      5432 |           5432 |

Open the frontend in your browser:

```text
http://localhost:8080
```

Test the backend using an endpoint that actually exists in your application:

```bash
curl http://localhost:8081/
```

Check PostgreSQL readiness:

```bash
docker compose exec postgres \
  pg_isready -U devboard -d devboard_db
```

Expected successful readiness response:

```text
/var/run/postgresql:5432 - accepting connections
```

The root URL and API response depend on the routes implemented by your backend.

---

## 12. Troubleshooting

### Problem 1: PostgreSQL Is Not Healthy

Check logs:

```bash
docker compose logs postgres
```

Check readiness:

```bash
docker compose exec postgres \
  pg_isready -U devboard -d devboard_db
```

Verify `.env` contains the expected database variables and that the health check is using the correct credentials and database name.

### Problem 2: Backend Cannot Connect to PostgreSQL

Check the backend logs:

```bash
docker compose logs backend
```

Verify the database URL uses:

```text
postgres:5432
```

Do not use `localhost:5432` from inside the backend container, because `localhost` refers to the backend container itself.

Also check database credentials, database initialization, and whether the PostgreSQL service is healthy.

### Problem 3: Frontend Cannot Reach Backend

Check the backend container:

```bash
docker compose ps
```

Check backend logs:

```bash
docker compose logs -f backend
```

For browser-based frontend requests, configure the API URL to use an address accessible from the browser, such as:

```text
http://localhost:8081
```

For a remote EC2 deployment, use the appropriate reachable hostname or domain and configure HTTPS and CORS as required.

### Problem 4: Port Already in Use

Check host ports:

```bash
sudo ss -tulpn
```

If another application is using a required host port, change the host side of the mapping.

For example:

```yaml
ports:
  - "8082:8080"
```

The backend remains on container port `8080`, but the host accesses it on `8082`.

### Problem 5: Database Data Is Missing

Check whether the named volume exists:

```bash
docker volume ls
```

Inspect it:

```bash
docker volume inspect devboard_pgdata
```

Ensure that you have not removed the volume with `docker compose down -v` or another volume-removal command.

---

## 13. Important Docker Compose Concepts

| Concept        | Explanation                                              |
| -------------- | -------------------------------------------------------- |
| Service        | An application component defined in Compose.             |
| Container      | A running instance of an image.                          |
| Image          | A packaged template used to create containers.           |
| `build`        | Builds an image from a Dockerfile and its build context. |
| `image`        | Specifies which image a service uses.                    |
| `environment`  | Sets environment variables inside a container.           |
| `ports`        | Publishes container ports on the host.                   |
| `depends_on`   | Defines service startup dependencies.                    |
| `healthcheck`  | Checks whether a service is functioning or ready.        |
| `volumes`      | Provides persistent storage or mounts host directories.  |
| `.env`         | Supplies values for Compose variable substitution.       |
| Docker network | Allows services to communicate with one another.         |

---

## 14. DevOps Learning Summary

This project demonstrates a three-tier application deployed using Docker Compose.

**Frontend**

* Builds and runs the Devboard user interface.
* Exposes container port `4173`.
* Publishes host port `8080`.

**Backend**

* Builds from the `backend` directory.
* Exposes container port `8080`.
* Publishes host port `8081`.
* Connects to PostgreSQL using the Compose service hostname.

**PostgreSQL**

* Uses the `postgres:16-alpine` image.
* Stores database files in the persistent `pgdata` volume.
* Runs a health check before the backend starts.
* Supports initialization scripts on first database initialization.

**Docker Compose**

* Defines the three services in one YAML file.
* Creates and manages the application network.
* Handles image builds, container startup, port publishing, dependencies, and persistent storage.

### Final Architecture

```text
                  DEVBOARD APPLICATION
                           |
                           v
                   Docker Compose
                           |
           +---------------+---------------+
           |               |               |
           v               v               v
       FRONTEND          BACKEND        POSTGRES
           |               |               |
       Port 4173       Port 8080        Port 5432
           |               |               |
       Host 8080       Host 8081       Host 5432
                           |
                           v
                    Docker Network
                           |
                           v
                   Persistent Volume
                        pgdata
```

**Key takeaway:** Docker Compose makes it possible to manage the frontend, backend, and database together using a single YAML configuration, while keeping their application processes in separate containers and their database data in persistent storage.
