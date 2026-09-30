# Docker Compose Configuration

The following is the Devboard `docker-compose.yml` with comments explaining what each section does.

```yaml
# ============================================================
# Docker Compose Services
# ============================================================
# Defines all containers required for the Devboard application.
#
# Services:
#   1. postgres  -> Database
#   2. backend   -> REST API
#   3. frontend  -> Web UI
# ============================================================

services:

  # ==========================================================
  # 1. PostgreSQL Database Service
  # ==========================================================

  postgres:

    # Custom name for the PostgreSQL container.
    # Instead of an automatically generated name, Docker
    # will create the container as "devboard-db".
    container_name: devboard-db

    # Use the PostgreSQL 16 image based on Alpine Linux.
    # Docker will pull this image if it is not available locally.
    image: postgres:16-alpine

    # Environment variables used to initialize PostgreSQL.
    # ${VARIABLE} values are read from the .env file.
    environment:

      # PostgreSQL username.
      POSTGRES_USER: ${POSTGRES_USER}

      # PostgreSQL password.
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}

      # PostgreSQL database name.
      POSTGRES_DB: ${POSTGRES_DB}

    # ========================================================
    # PostgreSQL Volumes
    # ========================================================

    volumes:

      # Persistent database storage.
      #
      # pgdata:
      #   Docker named volume
      #
      # /var/lib/postgresql/data:
      #   PostgreSQL data directory inside the container
      #
      # This keeps database data even if the container
      # is removed and recreated.
      - pgdata:/var/lib/postgresql/data

      # Mount SQL initialization files into PostgreSQL.
      #
      # Host:
      #   ./init/postgres
      #
      # Container:
      #   /docker-entrypoint-initdb.d
      #
      # SQL files such as:
      #   01_schema.sql
      #   02_seed.sql
      #
      # can be executed when PostgreSQL initializes
      # the database for the first time.
      #
      # :ro means read-only.
      - ./init/postgres:/docker-entrypoint-initdb.d:ro

    # ========================================================
    # PostgreSQL Health Check
    # ========================================================

    healthcheck:

      # Check whether PostgreSQL is ready to accept connections.
      #
      # pg_isready:
      #   PostgreSQL readiness-check command
      #
      # -U:
      #   PostgreSQL username
      #
      # -d:
      #   PostgreSQL database name
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]

      # Run the health check every 5 seconds.
      interval: 5s

      # Each health check can run for maximum 3 seconds.
      timeout: 3s

      # Retry the health check up to 10 times if it fails.
      retries: 10

    # ========================================================
    # PostgreSQL Port
    # ========================================================

    # Format:
    #   HOST_PORT:CONTAINER_PORT
    #
    # Host port 5432 -> PostgreSQL container port 5432
    #
    # Host can therefore access PostgreSQL through:
    #   localhost:5432
    ports:
      - "5432:5432"


  # ==========================================================
  # 2. Backend Service
  # ==========================================================

  backend:

    # Custom name for the backend container.
    container_name: devboard-backend

    # Build the backend Docker image using the Dockerfile
    # located inside the ./backend directory.
    #
    # Expected structure:
    #
    # backend/
    # ├── Dockerfile
    # └── application source code
    build: ./backend

    # ========================================================
    # Backend Environment Variables
    # ========================================================

    environment:

      # Port on which the backend application listens.
      #
      # The value comes from the .env file.
      #
      # Example:
      #   BACKEND_PORT=8080
      PORT: ${BACKEND_PORT}

      # PostgreSQL connection string.
      #
      # postgres:
      #   Docker Compose service name of the database.
      #
      # 5432:
      #   PostgreSQL container port.
      #
      # IMPORTANT:
      # Do NOT use localhost:5432 here.
      #
      # "postgres:5432" allows the backend container
      # to communicate with the PostgreSQL container
      # through the Docker Compose network.
      POSTGRES_URL: "postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}?sslmode=disable"

    # ========================================================
    # Backend Dependency
    # ========================================================

    # Backend depends on PostgreSQL.
    depends_on:

      postgres:

        # Do not start the backend until PostgreSQL
        # passes its health check.
        condition: service_healthy

    # ========================================================
    # Backend Port
    # ========================================================

    # Format:
    #   HOST_PORT:CONTAINER_PORT
    #
    # Host/EC2 port 8081 -> Backend container port 8080
    #
    # Therefore:
    #
    # Browser/API client:
    #   http://SERVER-IP:8081
    #
    # Backend inside Docker:
    #   :8080
    ports:
      - "8081:8080"


  # ==========================================================
  # 3. Frontend Service
  # ==========================================================

  frontend:

    # Custom name for the frontend container.
    container_name: devboard-frontend

    # Build the frontend image using:
    #
    # ./frontend/Dockerfile
    build: ./frontend

    # Frontend depends on the backend service.
    #
    # This defines the startup dependency:
    #
    # Frontend -> Backend
    depends_on:
      - backend

    # ========================================================
    # Frontend Port
    # ========================================================

    # Format:
    #   HOST_PORT:CONTAINER_PORT
    #
    # Host/EC2 port 8080 -> Frontend container port 4173
    #
    # Therefore the website can be opened using:
    #
    # http://SERVER-IP:8080
    ports:
      - "8080:4173"


# ============================================================
# Docker Named Volumes
# ============================================================

volumes:

  # Create a Docker named volume called "pgdata".
  #
  # It is used by PostgreSQL to store persistent database data.
  #
  # Used above:
  #
  #   pgdata:/var/lib/postgresql/data
  #
  # The volume is managed by Docker.
  pgdata:
```

# Important Concepts

## 1. `services`

```yaml
services:
```

Defines the containers required by the application.

```text
services
   |
   ├── postgres
   ├── backend
   └── frontend
```

---

## 2. `image` vs `build`

### PostgreSQL

```yaml
image: postgres:16-alpine
```

Use an existing image.

### Backend

```yaml
build: ./backend
```

Build an image from your own Dockerfile.

### Frontend

```yaml
build: ./frontend
```

Build an image from your own Dockerfile.

---

## 3. Port Mapping

The format is always:

```text
HOST_PORT:CONTAINER_PORT
```

Your project:

```text
8080:4173
8081:8080
5432:5432
```

Meaning:

```text
Browser
   |
   | :8080
   ▼
Frontend Container
   | :4173


API Client
   |
   | :8081
   ▼
Backend Container
   | :8080


Backend Container
   |
   | postgres:5432
   ▼
PostgreSQL Container
```

---

## 4. Container-to-Container Communication

Inside Docker Compose, services communicate using their **service names**.

```text
Frontend → Backend

backend:8080
```

and:

```text
Backend → PostgreSQL

postgres:5432
```

Do not use:

```text
localhost:5432
```

from the backend to reach PostgreSQL.

`localhost` means **the current container**.

---

## 5. Persistent Database

```yaml
volumes:
  - pgdata:/var/lib/postgresql/data
```

This provides persistent PostgreSQL storage.

```text
PostgreSQL Container
        |
        ▼
/var/lib/postgresql/data
        |
        ▼
    pgdata Volume
```

So:

```bash
docker compose down
```

does not normally delete the database volume.

But:

```bash
docker compose down -v
```

removes the Compose-managed volume and can delete the stored database data.

---

## 6. Startup Dependency

Your startup flow is:

```text
PostgreSQL
    |
    | healthcheck
    ▼
Healthy
    |
    ▼
Backend
    |
    ▼
Frontend
```

The important configuration is:

```yaml
depends_on:
  postgres:
    condition: service_healthy
```

This makes the backend wait for PostgreSQL's healthcheck.

---

## 7. Useful Commands

Start and build:

```bash
docker compose up -d --build
```

Check containers:

```bash
docker compose ps
```

View logs:

```bash
docker compose logs -f
```

Backend logs:

```bash
docker compose logs -f backend
```

Database logs:

```bash
docker compose logs -f postgres
```

Stop containers:

```bash
docker compose down
```

Stop containers and remove volumes:

```bash
docker compose down -v
```

Check volumes:

```bash
docker volume ls
```

## Final Architecture

```text
                    USER / BROWSER
                          |
                          | :8080
                          ▼
                ┌───────────────────┐
                │     FRONTEND      │
                │   Container       │
                │      :4173        │
                └─────────┬─────────┘
                          |
                          | backend:8080
                          ▼
                ┌───────────────────┐
                │      BACKEND      │
                │   Container       │
                │      :8080        │
                └─────────┬─────────┘
                          |
                          | postgres:5432
                          ▼
                ┌───────────────────┐
                │    POSTGRESQL     │
                │   Container       │
                │      :5432        │
                └─────────┬─────────┘
                          |
                          ▼
                ┌───────────────────┐
                │   pgdata Volume   │
                │ Persistent Data   │
                └───────────────────┘
```


# In Hindi

services:

  # ============================================================
  # 1. POSTGRES DATABASE SERVICE
  # ============================================================
  postgres:

    # Custom name of the PostgreSQL container
    # इससे container का नाम devboard-db होगा
    container_name: devboard-db

    # PostgreSQL 16 Alpine Linux based image
    # Docker Hub से PostgreSQL image use होगी
    image: postgres:16-alpine

    # Environment variables used to configure PostgreSQL
    environment:

      # Database username
      # Value .env file से आएगी
      POSTGRES_USER: ${POSTGRES_USER}

      # Database password
      # Value .env file से आएगी
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}

      # Database name
      # Value .env file से आएगी
      POSTGRES_DB: ${POSTGRES_DB}

    # Volumes
    volumes:

      # Named volume for persistent PostgreSQL data
      #
      # Container delete/recreate होने पर भी database data
      # volume में सुरक्षित रहता है.
      #
      # pgdata = Docker volume
      # /var/lib/postgresql/data = container के अंदर PostgreSQL data location
      - pgdata:/var/lib/postgresql/data

      # SQL initialization files
      #
      # Host:
      # ./init/postgres
      #
      # Container:
      # /docker-entrypoint-initdb.d
      #
      # PostgreSQL पहली बार database initialize करते समय
      # इस directory की .sql files execute करता है.
      #
      # :ro = read-only
      - ./init/postgres:/docker-entrypoint-initdb.d:ro

    # Health check
    # Docker check करेगा कि PostgreSQL वास्तव में ready है या नहीं.
    healthcheck:

      # pg_isready PostgreSQL की availability check करता है
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]

      # हर 5 seconds में health check
      interval: 5s

      # एक check को maximum 3 seconds मिलेंगे
      timeout: 3s

      # 10 बार तक retry करेगा
      retries: 10

    # Port mapping
    #
    # HOST:CONTAINER
    #
    # Host machine का 5432
    #        ↓
    # PostgreSQL container का 5432
    #
    # इसलिए host से:
    # localhost:5432
    # PostgreSQL तक पहुंच सकता है.
    ports:
      - "5432:5432"


  # ============================================================
  # 2. BACKEND SERVICE
  # ============================================================
  backend:

    # Custom backend container name
    container_name: devboard-backend

    # Backend का Dockerfile ./backend directory में है
    #
    # Docker इस directory में Dockerfile ढूंढकर
    # backend image build करेगा.
    build: ./backend

    # Backend environment variables
    environment:

      # Backend application किस port पर listen करेगी
      #
      # Value .env file से आएगी.
      PORT: ${BACKEND_PORT}

      # PostgreSQL connection string
      #
      # postgres = PostgreSQL service का Docker Compose name
      # 5432    = PostgreSQL container port
      #
      # IMPORTANT:
      # यहाँ localhost नहीं लिखा है.
      #
      # क्योंकि backend और postgres अलग containers हैं.
      POSTGRES_URL: "postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}?sslmode=disable"

    # Backend को PostgreSQL की जरूरत है.
    #
    # service_healthy का मतलब:
    # Backend तब start होगा जब PostgreSQL health check
    # healthy हो जाए.
    depends_on:
      postgres:
        condition: service_healthy

    # Port mapping
    #
    # HOST:CONTAINER
    #
    # EC2/Host का port 8081
    #          ↓
    # Backend container का port 8080
    #
    # इसलिए browser/host से:
    # http://SERVER-IP:8081
    #
    # backend तक पहुंच सकते हैं.
    ports:
      - "8081:8080"


  # ============================================================
  # 3. FRONTEND SERVICE
  # ============================================================
  frontend:

    # Custom frontend container name
    container_name: devboard-frontend

    # Frontend Dockerfile ./frontend directory में है
    build: ./frontend

    # Frontend backend पर depend करता है
    #
    # Backend container पहले start होगा.
    depends_on:
      - backend

    # Port mapping
    #
    # HOST:CONTAINER
    #
    # Host का port 8080
    #       ↓
    # Frontend container का port 4173
    #
    # इसलिए website:
    # http://SERVER-IP:8080
    ports:
      - "8080:4173"


# ============================================================
# 4. NAMED VOLUME
# ============================================================

volumes:

  # Docker managed volume
  #
  # PostgreSQL का permanent data यहाँ store होगा.
  #
  # Container हटाने के बाद भी volume मौजूद रह सकता है.
  pgdata:

  ```txt
                    DOCKER COMPOSE
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
       ▼                 ▼                 ▼

   FRONTEND           BACKEND          POSTGRES
   :4173              :8080             :5432
       ▲                 ▲                 ▲
       │                 │                 │
       │                 │                 │
     8080              8081              5432
       │                 │                 │
       └──── HOST/EC2 ───┴─────────────────┘

```
