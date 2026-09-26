# Docker Network – DevBoard 3-Tier Application

This document explains how to run the **DevBoard 3-Tier application** using Docker containers connected through a custom Docker bridge network.

The architecture contains:

* Frontend — React/Vite
* Backend — Go API
* Database — PostgreSQL
* Docker custom bridge network — `devboard-net`

---

# 1. Architecture

```text
                         Internet / Browser
                                |
                                |
                         EC2 Host Server
                                |
                       Port 8080 : 4173
                                |
                                v
                  +-------------------------+
                  |   Frontend Container    |
                  |      devboard-fe        |
                  |                          |
                  |   Vite Preview Server   |
                  |       Port 4173         |
                  +------------+------------+
                               |
                               | Docker Network
                               | devboard-net
                               |
                               v
                  +-------------------------+
                  |   Backend Container     |
                  |      devboard-be        |
                  |                          |
                  |       Go API             |
                  |       Port 8080         |
                  +------------+------------+
                               |
                               | PostgreSQL
                               | postgres:5432
                               |
                               v
                  +-------------------------+
                  |   PostgreSQL Container   |
                  |        postgres          |
                  |                          |
                  |       Port 5432          |
                  |      Database: devboard  |
                  +-------------------------+

                     Docker Network
                      devboard-net
                       172.18.0.0/16
```

---

# 2. Container Architecture

Your final running environment is:

| Component | Container  | Image                | Container Port | Host Port |
| --------- | ---------- | -------------------- | -------------: | --------: |
| Frontend  | `frontend` | `devboard-fe`        |           4173 |      8080 |
| Backend   | `backend`  | `devboard-be`        |           8080 |      8081 |
| Database  | `postgres` | `postgres:16-alpine` |           5432 |      5432 |

All three containers are connected to:

```text
devboard-net
```

---

# 3. Docker Network

## Check existing networks

```bash
docker network ls
```

Example:

```text
NETWORK ID     NAME           DRIVER    SCOPE
a1fa89f49387   bridge         bridge    local
26674fe80ec8   host           host      local
4f1c236444d6   none           null      local
```

### Meaning

Docker normally creates three networks:

* `bridge` — default Docker bridge network
* `host` — container uses host networking
* `none` — no network connectivity

---

# 4. Create Custom Docker Network

Create a dedicated network for the DevBoard application:

```bash
docker network create devboard-net
```

Example output:

```text
3bc1ab055a2cb72778f4d66f7c8da811a55520946a1fc1d2c39c13715724f893
```

Check it:

```bash
docker network ls
```

Expected:

```text
NETWORK ID     NAME           DRIVER    SCOPE
3bc1ab055a2c   devboard-net   bridge    local
```

## Why create a custom network?

Instead of putting containers on the default `bridge` network, we create an application-specific network.

```text
devboard-net
       |
       +--- frontend
       |
       +--- backend
       |
       +--- postgres
```

Containers on the same custom Docker network can communicate using **container names**.

For example:

```text
backend -> postgres:5432
```

You do NOT need to use:

```text
172.18.0.2
```

because Docker DNS resolves:

```text
postgres
```

to the PostgreSQL container.

---

# 5. Inspect Docker Network

```bash
docker network inspect devboard-net
```

You can also inspect using the network ID:

```bash
docker network inspect 3bc1ab055a2c
```

Your network had:

```text
Subnet: 172.18.0.0/16
Gateway: 172.18.0.1
```

Containers received IP addresses:

```text
postgres  -> 172.18.0.2
backend   -> 172.18.0.3
frontend  -> 172.18.0.4
```

Important:

**Do not configure your application using these container IP addresses.**

Use container names:

```text
postgres
backend
```

because container IP addresses can change.

---

# 6. Frontend Docker Image

Go to the frontend directory:

```bash
cd ~/devboard/frontend
```

Check files:

```bash
ls
```

Example:

```text
Dockerfile
index.html
package.json
package-lock.json
public
src
tailwind.config.js
vite.config.js
vite.preview.config.js
```

---

# 7. Build Frontend Image

```bash
docker build -t devboard-fe .
```

### Meaning

```text
docker build
```

Build a Docker image.

```text
-t devboard-fe
```

Give the image the name:

```text
devboard-fe
```

```text
.
```

Use the current directory as the Docker build context.

Result:

```text
Successfully built ca18e46f60dc
Successfully tagged devboard-fe:latest
```

Check:

```bash
docker images
```

You should see:

```text
devboard-fe:latest
```

---

# 8. Frontend Dockerfile Flow

Your frontend Dockerfile uses a multi-stage build.

```text
node:20-alpine
       |
       v
Install dependencies
       |
       v
npm run build
       |
       v
dist/
       |
       v
Runtime image
       |
       v
Vite Preview
       |
       v
Port 4173
```

The important build command was:

```bash
npm run build
```

which creates:

```text
dist/
```

The container finally runs:

```bash
vite preview --host 0.0.0.0 --port 4173
```

---

# 9. Backend Docker Image

Go to backend:

```bash
cd ~/devboard/backend
```

Check:

```bash
ls
```

Example:

```text
Dockerfile
go.mod
go.sum
main.go
main_test.go
```

Build:

```bash
docker build -t devboard-be .
```

Result:

```text
Successfully built 20f481caf432
Successfully tagged devboard-be:latest
```

Check:

```bash
docker images
```

---

# 10. Backend Dockerfile Flow

The backend image uses Go:

```text
golang:1.22-alpine
        |
        v
COPY go.mod go.sum
        |
        v
go mod download
        |
        v
COPY source code
        |
        v
go build
        |
        v
devboard-backend
```

The application listens inside the container on:

```text
8080
```

Dockerfile:

```dockerfile
EXPOSE 8080
```

---

# 11. PostgreSQL Initialization Files

Your PostgreSQL files are:

```bash
cd ~/devboard/init/postgres
```

```bash
ls
```

Output:

```text
01_schema.sql
02_seed.sql
```

### `01_schema.sql`

Creates the database tables:

```text
projects
tasks
```

It also creates:

* indexes
* constraints
* foreign key
* `updated_at` trigger

### `02_seed.sql`

Adds demo data:

```text
2 projects
10+ tasks
```

---

# 12. Create PostgreSQL Container

First create the network:

```bash
docker network create devboard-net
```

Then run PostgreSQL:

```bash
docker run -d \
  --name postgres \
  --network devboard-net \
  -e POSTGRES_USER=devboard \
  -e POSTGRES_PASSWORD=devboard \
  -e POSTGRES_DB=devboard \
  -v "$PWD/01_schema.sql":/docker-entrypoint-initdb.d/01_schema.sql:ro \
  -v "$PWD/02_seed.sql":/docker-entrypoint-initdb.d/02_seed.sql:ro \
  -p 5432:5432 \
  postgres:16-alpine
```

---

# 13. PostgreSQL Command Explanation

## `docker run`

Creates and starts a container.

```bash
docker run
```

## `-d`

Run in detached/background mode.

```bash
-d
```

## `--name postgres`

Container name:

```text
postgres
```

This name becomes important because the backend can connect to:

```text
postgres:5432
```

## `--network devboard-net`

Connect PostgreSQL to:

```text
devboard-net
```

## Environment variables

```bash
-e POSTGRES_USER=devboard
```

PostgreSQL username.

```bash
-e POSTGRES_PASSWORD=devboard
```

PostgreSQL password.

```bash
-e POSTGRES_DB=devboard
```

Database name.

---

# 14. PostgreSQL Volume Mount

```bash
-v "$PWD/01_schema.sql":/docker-entrypoint-initdb.d/01_schema.sql:ro
```

and:

```bash
-v "$PWD/02_seed.sql":/docker-entrypoint-initdb.d/02_seed.sql:ro
```

This means:

```text
Host
/home/ubuntu/devboard/init/postgres/01_schema.sql
             |
             | volume mount
             v
Container
/docker-entrypoint-initdb.d/01_schema.sql
```

PostgreSQL's official image automatically executes `.sql` files in:

```text
/docker-entrypoint-initdb.d/
```

when the database is initialized for the first time.

`ro` means:

```text
read-only
```

The container can read the file but cannot modify the host file.

---

# 15. PostgreSQL Port Mapping

```bash
-p 5432:5432
```

Format:

```text
-p HOST_PORT:CONTAINER_PORT
```

Therefore:

```text
EC2 Host
5432
  |
  v
PostgreSQL Container
5432
```

---

# 16. Important Volume Mount Mistake

You initially used:

```bash
-v "$PWD/01_schema.sql":/docker-entrypoint-initdb.d:ro
```

This caused:

```text
not a directory

Are you trying to mount a directory onto a file
(or vice-versa)?
```

### Why?

You tried to mount:

```text
HOST FILE
01_schema.sql
```

onto:

```text
CONTAINER DIRECTORY
/docker-entrypoint-initdb.d
```

These types don't match.

Correct:

```bash
-v "$PWD/01_schema.sql":/docker-entrypoint-initdb.d/01_schema.sql:ro
```

Now:

```text
Host file
      |
      v
01_schema.sql
      |
      v
Container file
/docker-entrypoint-initdb.d/01_schema.sql
```

For two files:

```bash
-v "$PWD/01_schema.sql":/docker-entrypoint-initdb.d/01_schema.sql:ro
-v "$PWD/02_seed.sql":/docker-entrypoint-initdb.d/02_seed.sql:ro
```

---

# 17. Check PostgreSQL

```bash
docker ps
```

Expected:

```text
postgres:16-alpine
0.0.0.0:5432->5432/tcp
```

---

# 18. Enter PostgreSQL Container

```bash
docker exec -it postgres bash
```

Now you are inside the container.

Run:

```bash
psql -U devboard
```

You enter PostgreSQL:

```text
devboard=#
```

---

# 19. Check Databases

```sql
\l
```

You should see:

```text
devboard
postgres
template0
template1
```

---

# 20. Check Tables

```sql
\dt
```

Expected:

```text
projects
tasks
```

---

# 21. Check Table Structure

```sql
\d tasks
```

This shows:

* columns
* data types
* indexes
* primary key
* foreign key
* check constraints
* triggers

---

# 22. Check Data

```sql
SELECT * FROM tasks;
```

Example:

```text
id | title                  | status
---+------------------------+------------
1  | Design the task schema | done
2  | Build the kanban board | in_progress
3  | Wire up dashboard      | in_progress
```

Exit PostgreSQL:

```sql
\q
```

Exit container:

```bash
exit
```

---

# 23. Backend Container

Run the backend:

```bash
docker run -d \
  --name backend \
  --network devboard-net \
  -e PORT=8080 \
  -e POSTGRES_URL="postgres://devboard:devboard@postgres:5432/devboard?sslmode=disable" \
  -p 8081:8080 \
  devboard-be
```

---

# 24. Backend PostgreSQL Connection

The most important part is:

```text
postgres://devboard:devboard@postgres:5432/devboard?sslmode=disable
```

Break it down:

```text
postgres://
      |
      v
Database protocol

devboard
      |
      v
Username

:
      |
      v

devboard
      |
      v
Password

@
      |
      v

postgres
      |
      v
Docker container name

:
      |
      v

5432
      |
      v
PostgreSQL container port

/
      |
      v

devboard
      |
      v
Database name
```

Complete:

```text
backend
   |
   | postgres://devboard:devboard@postgres:5432/devboard
   |
   v
postgres container
```

---

# 25. Why `postgres` Instead of IP Address?

Do NOT use:

```text
172.18.0.2
```

Use:

```text
postgres
```

Docker provides internal DNS on custom networks.

Therefore:

```text
backend
   |
   | DNS lookup: postgres
   v
172.18.0.2
   |
   v
PostgreSQL
```

If PostgreSQL's IP changes, Docker DNS still resolves:

```text
postgres
```

---

# 26. Backend Port Mapping

You used:

```bash
-p 8081:8080
```

Meaning:

```text
EC2 Host
Port 8081
    |
    v
Backend Container
Port 8080
```

Therefore:

```text
http://EC2-IP:8081
```

can reach the backend.

---

# 27. Important Image Naming Mistake

You initially ran:

```bash
docker run ... devboard-backend
```

Docker returned:

```text
Unable to find image 'devboard-backend:latest' locally
```

The reason is that you built:

```bash
docker build -t devboard-be .
```

Therefore your image name is:

```text
devboard-be
```

not:

```text
devboard-backend
```

Correct:

```bash
docker run ... devboard-be
```

---

# 28. Frontend Container

Run:

```bash
docker run -d \
  --name frontend \
  --network devboard-net \
  -p 8080:4173 \
  devboard-fe
```

---

# 29. Frontend Port Mapping

You used:

```bash
-p 8080:4173
```

Meaning:

```text
Browser
   |
   v
EC2 Host :8080
   |
   v
Frontend Container :4173
```

Therefore the application is accessed using:

```text
http://EC2-IP:8080
```

---

# 30. Check All Containers

```bash
docker ps
```

Expected:

```text
CONTAINER       IMAGE                PORTS
frontend        devboard-fe          8080 -> 4173
backend         devboard-be          8081 -> 8080
postgres        postgres:16-alpine   5432 -> 5432
```

---

# 31. Check Network

```bash
docker network inspect devboard-net
```

You should see:

```text
frontend
backend
postgres
```

connected to the same network.

Example:

```text
devboard-net
     |
     +---- postgres
     |       172.18.0.2
     |
     +---- backend
     |       172.18.0.3
     |
     +---- frontend
             172.18.0.4
```

---

# 32. Complete Communication Flow

## User → Frontend

```text
Browser
   |
   | http://EC2-IP:8080
   v
EC2 Host :8080
   |
   v
frontend container :4173
```

## Backend → PostgreSQL

```text
backend container
       |
       | postgres:5432
       v
postgres container
       |
       v
devboard database
```

---

# 33. Three-Tier Architecture

```text
             USER
              |
              |
              v
       +--------------+
       |   FRONTEND   |
       | React / Vite |
       |    :4173     |
       +------+-------+
              |
              | API
              v
       +--------------+
       |   BACKEND    |
       |     Go       |
       |    :8080     |
       +------+-------+
              |
              | SQL
              v
       +--------------+
       |  POSTGRESQL  |
       |    :5432     |
       +--------------+
```

All containers:

```text
             devboard-net
                  |
       +----------+----------+
       |          |          |
   frontend    backend    postgres
```

---

# 34. Host Ports vs Container Ports

This is very important.

```text
HOST PORT : CONTAINER PORT
```

Your application:

```text
Frontend
8080 : 4173

Backend
8081 : 8080

PostgreSQL
5432 : 5432
```

So:

```text
Browser
   |
   | :8080
   v
Frontend :4173


Backend
   |
   | :5432
   v
Postgres :5432
```

---

# 35. `docker ps`

Use:

```bash
docker ps
```

Purpose:

Shows **running containers**.

---

# 36. `docker ps -a`

Use:

```bash
docker ps -a
```

Purpose:

Shows:

* running containers
* stopped containers
* failed containers
* exited containers

---

# 37. Stop a Container

Example:

```bash
docker stop frontend
```

or:

```bash
docker stop b4ad04b3cfa5
```

The container is stopped but not deleted.

---

# 38. Start a Stopped Container

```bash
docker start frontend
```

You successfully practiced:

```bash
docker stop b4ad04b3cfa5
docker start b4ad04b3cfa5
```

After starting it again:

```bash
docker ps
```

shows the frontend running.

---

# 39. Remove a Container

```bash
docker rm postgres
```

If it is running:

```bash
docker rm -f postgres
```

`-f` means force remove.

---

# 40. View Container Logs

Very useful for debugging:

```bash
docker logs frontend
```

Backend:

```bash
docker logs backend
```

PostgreSQL:

```bash
docker logs postgres
```

Follow logs live:

```bash
docker logs -f backend
```

---

# 41. Enter a Running Container

Frontend:

```bash
docker exec -it frontend sh
```

Backend:

```bash
docker exec -it backend sh
```

PostgreSQL:

```bash
docker exec -it postgres bash
```

---

# 42. Test Docker DNS

Because all containers are on:

```text
devboard-net
```

the backend can resolve:

```text
postgres
```

You can inspect the network:

```bash
docker network inspect devboard-net
```

The important concept is:

```text
Container Name = DNS Name
```

Therefore:

```text
backend ---> postgres
```

works.

---

# 43. Check Docker Images

```bash
docker images
```

Your important images:

```text
devboard-fe
devboard-be
postgres:16-alpine
```

The Node and Go images are also present because they were used during image builds.

---

# 44. Difference Between Image and Container

## Image

Template/blueprint:

```text
devboard-fe
devboard-be
postgres:16-alpine
```

## Container

Running instance of an image:

```text
frontend
backend
postgres
```

Relationship:

```text
IMAGE
  |
  | docker run
  v
CONTAINER
```

Example:

```bash
docker run --name backend devboard-be
```

Here:

```text
devboard-be = image
backend      = container
```

---

# 45. Useful Verification Commands

Check containers:

```bash
docker ps
```

Check all containers:

```bash
docker ps -a
```

Check images:

```bash
docker images
```

Check networks:

```bash
docker network ls
```

Inspect network:

```bash
docker network inspect devboard-net
```

Check frontend logs:

```bash
docker logs frontend
```

Check backend logs:

```bash
docker logs backend
```

Check PostgreSQL logs:

```bash
docker logs postgres
```

---

# 46. Complete Clean Deployment

If you want to recreate the complete environment from scratch:

## Step 1 — Create network

```bash
docker network create devboard-net
```

## Step 2 — Build frontend

```bash
cd ~/devboard/frontend

docker build -t devboard-fe .
```

## Step 3 — Build backend

```bash
cd ~/devboard/backend

docker build -t devboard-be .
```

## Step 4 — Start PostgreSQL

```bash
cd ~/devboard/init/postgres

docker run -d \
  --name postgres \
  --network devboard-net \
  -e POSTGRES_USER=devboard \
  -e POSTGRES_PASSWORD=devboard \
  -e POSTGRES_DB=devboard \
  -v "$PWD/01_schema.sql":/docker-entrypoint-initdb.d/01_schema.sql:ro \
  -v "$PWD/02_seed.sql":/docker-entrypoint-initdb.d/02_seed.sql:ro \
  -p 5432:5432 \
  postgres:16-alpine
```

## Step 5 — Start backend

```bash
cd ~/devboard

docker run -d \
  --name backend \
  --network devboard-net \
  -e PORT=8080 \
  -e POSTGRES_URL="postgres://devboard:devboard@postgres:5432/devboard?sslmode=disable" \
  -p 8081:8080 \
  devboard-be
```

## Step 6 — Start frontend

```bash
docker run -d \
  --name frontend \
  --network devboard-net \
  -p 8080:4173 \
  devboard-fe
```

## Step 7 — Verify

```bash
docker ps
```

Then:

```bash
docker network inspect devboard-net
```

---

# 47. Final Expected Architecture

```text
                         EC2 SERVER
                    Public IP / Private IP
                           |
             +-------------+-------------+
             |             |             |
           :8080         :8081         :5432
             |             |             |
             v             v             v
       +-----------+ +-----------+ +-----------+
       | frontend  | |  backend  | | postgres  |
       |           | |           | |           |
       | Vite      | | Go API    | | PostgreSQL|
       | :4173     | | :8080     | | :5432     |
       +-----+-----+ +-----+-----+ +-----------+
             |             |
             +------+------+
                    |
             devboard-net
             172.18.0.0/16
```

---

# 48. Most Important Commands to Remember

### Create network

```bash
docker network create devboard-net
```

### List networks

```bash
docker network ls
```

### Inspect network

```bash
docker network inspect devboard-net
```

### Build image

```bash
docker build -t devboard-fe .
```

```bash
docker build -t devboard-be .
```

### Run PostgreSQL

```bash
docker run -d --name postgres --network devboard-net ...
```

### Run backend

```bash
docker run -d --name backend --network devboard-net ...
```

### Run frontend

```bash
docker run -d --name frontend --network devboard-net ...
```

### List running containers

```bash
docker ps
```

### List all containers

```bash
docker ps -a
```

### Stop

```bash
docker stop <container>
```

### Start

```bash
docker start <container>
```

### Remove

```bash
docker rm <container>
```

### Force remove

```bash
docker rm -f <container>
```

### Logs

```bash
docker logs <container>
```

### Enter container

```bash
docker exec -it <container> sh
```

### PostgreSQL shell

```bash
docker exec -it postgres psql -U devboard -d devboard
```

---

# 49. Key Concepts Learned

```text
Docker Image
     |
     | docker run
     v
Docker Container
     |
     | attach to
     v
Docker Network
     |
     +-------- frontend
     |
     +-------- backend
     |
     +-------- postgres
```

### Remember these three rules:

**Rule 1 — Same custom network**

```text
frontend
backend
postgres
        |
        v
devboard-net
```

**Rule 2 — Container-to-container communication uses container names**

```text
backend -> postgres:5432
```

not:

```text
backend -> 172.18.0.2
```

**Rule 3 — Port mapping is for host-to-container access**

```text
8080:4173
8081:8080
5432:5432
```

Meaning:

```text
HOST : CONTAINER
```

---

# 50. DevBoard Docker Flow

```text
                  Git Repository
                       |
                       v
                 Source Code
                 /          \
                /            \
               v              v
        Frontend Code     Backend Code
               |              |
               v              v
        docker build      docker build
               |              |
               v              v
        devboard-fe       devboard-be
               |              |
               |              |
               +------+-------+
                      |
                      v
               devboard-net
                      |
             +--------+--------+
             |        |        |
             v        v        v
         frontend  backend  postgres
             |        |        |
             |        +------->|
             |               DB
             |
             v
        User Browser
```

This is the basic Docker foundation for deploying your **DevBoard 3-tier application** before moving to Docker Compose, CI/CD, or Kubernetes/EKS.
