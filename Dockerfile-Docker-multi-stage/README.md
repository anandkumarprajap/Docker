# Devboard Frontend — Docker & Multi-Stage Docker Notes

# Docker File
![Image b](b.png)
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
![Image 13](13.png)
![Image 14](14.png)
![Image 15](15.png)
![Image 16](16.png)
![Image 17](17.png)

# Docker Multi-stage 
![Image 18](18.png)
![Image 19](19.png)
![Image 20](20.png)
![Image 21](21.png)

# Docker & multi-stage combine
![Image 22](22.png)
![Image 23](23.png)
![Image 24](24.png)

## Overview

This lab demonstrates two Docker approaches for the same Vite/React frontend:

1. **Normal Dockerfile** → the Vite development server runs inside the container.
2. **Multi-stage Dockerfile** → the application is built first, then only the production build is copied into a smaller runtime image.

The initial image became large mainly because it used a full `node:22` image together with `node_modules`, development dependencies, source files, build tools, and Docker layers. The multi-stage approach reduced the final runtime image substantially.

---

# 1. Project Structure

```text
Devboard/
│
├── Dockerfile
├── Dockerfile.multistage
├── package.json
├── package-lock.json
├── vite.config.js
├── tailwind.config.js
├── postcss.config.js
├── index.html
├── public/
├── src/
├── README.md
├── .dockerignore
└── .gitignore
```

| File | Purpose |
|---|---|
| `package.json` | Application dependencies and scripts |
| `package-lock.json` | Exact dependency versions |
| `src/` | React/Vite source code |
| `public/` | Static files |
| `vite.config.js` | Vite configuration |
| `Dockerfile` | Normal Docker image |
| `Dockerfile.multistage` | Multi-stage production-style image |
| `.dockerignore` | Files excluded from Docker build context |

---

# 2. Overall Docker Concept

```text
                 EC2 Ubuntu Server
                        │
                        │ docker build
                        ▼
              ┌───────────────────┐
              │    Docker Image   │
              │                   │
              │ devboard-frontend │
              └─────────┬─────────┘
                        │
                        │ docker run
                        ▼
              ┌───────────────────┐
              │     Container     │
              │                   │
Internet ────►│ Vite Application  │
              │       :5173       │
              └─────────┬─────────┘
                        │
                  Docker port map
                        │
                  5173 : 5173
                        │
                        ▼
               EC2 Public IP:5173
```

Example:

```text
http://<EC2-PUBLIC-IP>:5173
```

The AWS Security Group must also allow inbound TCP `5173` from the required source.

---

# 3. Install Docker

Update Ubuntu package information:

```bash
sudo apt-get update
```

This normally updates package metadata; it does not itself install Docker.

Install Docker:

```bash
sudo apt-get install docker.io
```

Docker and required dependencies may include:

```text
docker.io
containerd
runc
bridge-utils
```

Check the version:

```bash
docker -v
```

The lab showed approximately:

```text
Docker version 29.1.3
```

---

# 4. Allow Ubuntu User to Run Docker

Add the current Ubuntu user to the Docker group:

```bash
sudo usermod -aG docker $USER
```

Start a new shell with the updated group:

```bash
newgrp docker
```

Test:

```bash
docker ps
```

If it works without `sudo`, the user can access Docker.

---

# 5. `docker ps`

```bash
docker ps
```

Shows currently running containers.

Example columns:

```text
CONTAINER ID
IMAGE
COMMAND
STATUS
PORTS
NAMES
```

If no containers are running, an empty result is normal.

---

# 6. `docker ps -a`

```bash
docker ps -a
```

Shows:

```text
Running containers
+
Stopped containers
+
Exited containers
```

This is useful when a container starts and then crashes.

---

# 7. First Dockerfile

The normal Dockerfile used for the Vite development server is conceptually:

```dockerfile
FROM node:22

WORKDIR /app

COPY . .

RUN npm install

EXPOSE 5173

CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0", "--port", "5173"]
```

---

# 8. Dockerfile — Step by Step

## Step 1 — Base Image

```dockerfile
FROM node:22
```

Docker starts with the Node.js 22 image.

Conceptually:

```text
Docker Image
     │
     └── node:22
          │
          ├── Linux userspace
          ├── Node.js
          ├── npm
          └── required runtime components
```

The full Node image is relatively large.

---

## Step 2 — WORKDIR

```dockerfile
WORKDIR /app
```

Creates/selects:

```text
/app
```

inside the image.

After this, `/app` becomes the working directory for following Dockerfile commands.

---

## Step 3 — COPY

```dockerfile
COPY . .
```

Meaning:

```text
COPY <source> <destination>
```

Here:

```text
. = current build context
. = /app
```

So:

```text
EC2 Devboard/
       │
       │ COPY . .
       ▼
Container:
/app/
├── package.json
├── package-lock.json
├── src/
├── public/
├── index.html
└── ...
```

Important: use:

```dockerfile
COPY . .
```

not:

```dockerfile
COPY.
```

There must be a space.

---

# 9. `.dockerignore`

Recommended example:

```text
node_modules
dist
.git
.gitignore
Dockerfile*
README.md
npm-debug.log
```

Without `.dockerignore`, Docker can send unnecessary files to the build context.

Especially:

```text
node_modules/
.git/
dist/
```

can be unnecessary for the build.

---

# 10. `RUN npm install`

```dockerfile
RUN npm install
```

Docker executes `npm install` while creating the image.

It reads:

```text
package.json
package-lock.json
```

and installs:

```text
/app/node_modules
```

Conceptually:

```text
node:22
   +
application source
   +
node_modules
   ↓
large Docker image
```

---

# 11. Why `npm install` Uses So Much Space

A modern JavaScript project can contain hundreds of packages.

Example:

```text
package.json
      │
      ▼
npm install
      │
      ▼
node_modules/
      │
      ├── React
      ├── Vite
      ├── Tailwind
      ├── esbuild
      ├── PostCSS
      ├── dependencies
      └── dependencies of dependencies
```

The lab showed output such as:

```text
added 246 packages
```

Those packages occupy disk space.

---

# 12. `EXPOSE`

```dockerfile
EXPOSE 5173
```

Documents that the application listens on port `5173`.

Important:

> `EXPOSE` does **not** publish the port to the Internet.

You still need:

```bash
docker run -p 5173:5173 ...
```

And the AWS Security Group must allow the required inbound traffic.

---

# 13. `CMD`

Correct command:

```dockerfile
CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0", "--port", "5173"]
```

Conceptually:

```bash
npm run dev -- --host 0.0.0.0 --port 5173
```

The `--` separates npm arguments from Vite arguments.

---

# 14. First Dockerfile Error

The initial command was similar to:

```dockerfile
CMD ["npm", "run", "-", "-host", "0.0.0.0", "--port", "5173"]
```

The container reported:

```text
npm error Missing script: "-host"
```

### Why?

`npm run` expects a script name.

For example:

```bash
npm run dev
```

But the incorrect command effectively told npm to run a script named:

```text
-host
```

There is no `-host` script in `package.json`.

### Correct command

```dockerfile
CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0", "--port", "5173"]
```

---

# 15. Why `--host 0.0.0.0` Is Important

Vite inside a container should listen on:

```text
0.0.0.0
```

rather than only:

```text
localhost
```

Conceptually:

```text
localhost
   ↓
inside container only
```

Whereas:

```text
0.0.0.0
   ↓
listen on container network interfaces
```

Diagram:

```text
                Container
        ┌──────────────────────┐
        │ Vite                 │
        │                      │
        │ 0.0.0.0:5173        │
        └──────────┬───────────┘
                   │
                   ▼
              Docker network
                   │
                   ▼
              EC2 host :5173
```

---

# 16. Build Docker Image

```bash
docker build -t devboard-frontend .
```

Meaning:

```text
docker build
      │
      ├── -t devboard-frontend
      │       │
      │       └── image name
      │
      └── .
          └── current directory = build context
```

Result:

```text
devboard-frontend:latest
```

---

# 17. Docker Build Process

The build was approximately:

```text
Step 1: FROM node:22
          ↓
Step 2: WORKDIR /app
          ↓
Step 3: COPY .
          ↓
Step 4: RUN npm install
          ↓
Step 5: RUN npm rebuild esbuild
          ↓
Step 6: EXPOSE 5173
          ↓
Step 7: CMD ...
          ↓
devboard-frontend:latest
```

---

# 18. Intermediate Containers

The build output showed messages such as:

```text
Running in ...
Removed intermediate container ...
```

This is normal with the legacy builder.

Conceptually:

```text
Dockerfile instruction
        ↓
temporary layer/container
        ↓
filesystem result
        ↓
next step
```

Modern Docker generally uses BuildKit/buildx.

The Docker output warned that the legacy builder is deprecated.

Check buildx:

```bash
docker buildx version
```

---

# 19. Run Container

Command:

```bash
docker run -d -p 5173:5173 --name devboard-frontend-container devboard-frontend
```

Breakdown:

```text
docker run
```

Creates and starts a container.

```text
-d
```

Detached mode. The container runs in the background.

```text
-p 5173:5173
```

Port mapping:

```text
HOST PORT : CONTAINER PORT
5173      : 5173
```

```text
--name devboard-frontend-container
```

Gives the container a readable name.

```text
devboard-frontend
```

The image used to create the container.

---

# 20. Port Mapping Diagram

```text
Internet
   │
   │ http://EC2-IP:5173
   ▼
AWS Security Group
   │
   │ TCP 5173 allowed
   ▼
EC2 Ubuntu
   │
   │ Host port 5173
   ▼
Docker
   │
   │ -p 5173:5173
   ▼
Container
   │
   │ Container port 5173
   ▼
Vite
   │
   ▼
React Application
```

---

# 21. `docker ps`

After successful startup:

```bash
docker ps
```

The output showed:

```text
0.0.0.0:5173->5173/tcp
```

This means:

```text
EC2 host :5173
       │
       ▼
container :5173
```

---

# 22. `docker logs`

You used:

```bash
docker logs 9d27cfa1407
```

or:

```bash
docker logs devboard-frontend-container
```

This shows application stdout/stderr.

For the Vite container, logs may show:

```text
VITE ready
Local: http://localhost:5173/
Network: http://172.17.0.2:5173/
```

Logs are very useful for debugging.

---

# 23. `docker exec`

Example:

```bash
docker exec -it 78e6e04f5392 bash
```

This failed in the Alpine container with:

```text
bash: executable file not found
```

### Why?

Alpine images commonly contain:

```text
/bin/sh
```

but not necessarily:

```text
/bin/bash
```

Use:

```bash
docker exec -it 322b8205be37 sh
```

This worked in the lab.

---

# 24. Why `curl` Was Missing

Inside the Alpine container:

```bash
curl http://localhost:4173
```

returned:

```text
curl: not found
```

Minimal Alpine images do not necessarily include curl.

For temporary troubleshooting:

```bash
apk add --no-cache curl
```

For production images, avoid installing troubleshooting packages unnecessarily.

---

# 25. Docker Image vs Container

This is an important Docker concept:

```text
             Dockerfile
                 │
                 │ docker build
                 ▼
          ┌──────────────┐
          │ Docker Image │
          └──────┬───────┘
                 │
                 │ docker run
                 ▼
          ┌──────────────┐
          │  Container   │
          └──────────────┘
```

### Image

A template/package used to create containers.

### Container

A running or stopped instance created from an image.

Example:

```bash
docker build -t devboard-frontend .
```

creates an image:

```text
devboard-frontend
```

Then:

```bash
docker run devboard-frontend
```

creates a container.

---

# 26. Why the Docker Image Was Huge

The lab showed approximately:

```text
devboard-frontend     ~2.24 GB disk usage
node:22               ~1.64 GB disk usage
```

Docker image disk usage is not simply the size of the application source files. Images contain layers.

Conceptually:

```text
devboard-frontend
│
├── Node base image
├── application source
├── node_modules
├── npm dependencies
├── esbuild
└── Docker layers
```

Important factors:

### 1. `node:22`

A general-purpose Node image.

### 2. `npm install`

Installs development and runtime dependencies.

### 3. `node_modules`

Can contain hundreds or thousands of dependency files.

### 4. Development environment

The application is running:

```text
Vite development server
```

rather than only compiled production files.

### 5. Docker layers

Dockerfile instructions can create filesystem layers.

---

# 27. `npm rebuild esbuild`

You added:

```dockerfile
RUN npm rebuild esbuild
```

This forces npm to rebuild the native `esbuild` dependency for the container environment.

The lab showed:

```text
rebuilt dependencies successfully
```

This can help when native dependency binaries are incompatible with the environment.

However, it is not universally required for every Vite project. If `npm install` already installs the correct platform binary, it may not be necessary.

---

# 28. Swap Memory

You created a 2 GB swap file:

```bash
sudo fallocate -l 2G /swapfile
```

Then:

```bash
sudo chmod 600 /swapfile
```

Then:

```bash
sudo mkswap /swapfile
```

Then:

```bash
sudo swapon /swapfile
```

Check:

```bash
free -h
```

---

# 29. Why Swap Helped

The EC2 instance has limited RAM.

Docker builds can consume significant memory because:

```text
npm install
       +
esbuild
       +
Node.js
       +
Docker
       +
Ubuntu
```

can create memory pressure.

Swap gives Linux additional disk-backed virtual memory.

```text
RAM
 │
 ├── Ubuntu
 ├── Docker
 ├── Node
 └── npm
       │
       │ memory pressure
       ▼
    SWAPFILE
    2 GB
```

Important:

> Swap is not equivalent to RAM. It is much slower because it uses disk.

---

# 30. Check RAM

```bash
free -h
```

Useful columns include:

```text
total
used
free
shared
buff/cache
available
```

For practical monitoring, pay particular attention to:

```text
available
```

and swap usage.

---

# 31. Check Disk

```bash
df -h
```

Shows filesystem storage.

Example:

```text
Filesystem      Size   Used   Avail
/dev/root       19G    7.9G   11G
```

This is disk storage, not RAM.

---

# 32. RAM vs Disk

Very important distinction:

```text
free -h
    ↓
RAM + SWAP

df -h
    ↓
DISK STORAGE

docker images
    ↓
Docker image storage
```

Do not confuse RAM, swap, disk, and Docker image storage.

---

# 33. `docker images`

```bash
docker images
```

Shows locally stored Docker images.

Example:

```text
devboard-frontend
devboard-frontend-multi-stage
node:22
node:24-alpine
```

---

# 34. Remove Container

Remove a stopped container:

```bash
docker rm devboard-frontend-container
```

If the container is running:

```bash
docker rm -f devboard-frontend-container
```

`-f` means force stop/remove.

---

# 35. Remove Image

```bash
docker rmi devboard-frontend:latest
```

This removes the image/tag if it is not required by another container.

---

# 36. Clean Builder Cache

```bash
docker builder prune -a -f
```

Removes unused builder cache.

For normal cleanup:

```bash
docker builder prune
```

Be careful with `-a` because it removes more unused build cache.

---

# 37. Multi-Stage Dockerfile

The lab used the following multi-stage concept:

```dockerfile
# Stage 1
FROM node:22 AS builder

WORKDIR /app

COPY . .

RUN npm install

RUN npm run build


# Stage 2
FROM node:24-alpine AS runner

WORKDIR /app

COPY package.json .

RUN npm install -g vite

COPY --from=builder /app/dist ./dist

EXPOSE 4173

CMD ["vite", "preview", "--host", "0.0.0.0", "--port", "4173"]
```

The concept is correct: build in one stage and copy only the required output into the runtime stage.

---

# 38. Multi-Stage Diagram

```text
                Dockerfile.multistage
                         │
                         ▼
              ┌─────────────────────┐
              │    STAGE 1          │
              │      BUILDER        │
              │                     │
              │ FROM node:22        │
              │                     │
              │ /app                │
              │ source code         │
              │ node_modules        │
              │ npm install         │
              │ npm run build       │
              │                     │
              │       ↓             │
              │      /dist          │
              └─────────┬───────────┘
                        │
                        │ COPY --from=builder
                        │ only required output
                        ▼
              ┌─────────────────────┐
              │    STAGE 2          │
              │      RUNNER         │
              │                     │
              │ node:24-alpine      │
              │                     │
              │ /app                │
              │ package.json        │
              │ dist/               │
              │ Vite                │
              └─────────┬───────────┘
                        │
                        ▼
                  Port 4173
                        │
                        ▼
                Production build
```

---

# 39. What Is `/dist`?

When Vite builds the application:

```bash
npm run build
```

Vite converts source code into production-ready static files.

Example:

```text
src/
   ↓
Vite build
   ↓
dist/
├── index.html
├── assets/
│   ├── index-xxxx.js
│   ├── index-xxxx.css
│   └── ...
```

The basic flow is:

```text
Source Code
     ↓
npm run build
     ↓
dist/
     ↓
Production files
```

---

# 40. Why Multi-Stage Reduces Image Size

Normal Dockerfile:

```text
node:22
   +
source code
   +
node_modules
   +
development dependencies
   +
Vite
   +
everything
        ↓
Large image
```

Multi-stage:

```text
Builder
   │
   ├── source
   ├── dependencies
   └── build tools
          │
          ▼
        dist
          │
          ▼
Runner
   │
   ├── lightweight runtime
   └── dist
          ↓
      Smaller image
```

The experiment showed approximately:

```text
Normal image:
~2.24 GB disk usage

Multi-stage:
~387 MB disk usage
```

Exact values depend on Docker storage accounting and image layers, but the reduction was substantial.

---

# 41. Why Builder Dependencies Don't Go Into the Final Image

Stage 1 can contain:

```text
node_modules
npm
Vite
Tailwind
esbuild
source code
build tools
```

After:

```bash
npm run build
```

the important build output is:

```text
dist/
```

Stage 2 uses:

```dockerfile
COPY --from=builder /app/dist ./dist
```

It does not copy the entire Stage 1 filesystem.

---

# 42. `COPY --from=builder`

```dockerfile
COPY --from=builder /app/dist ./dist
```

Means:

```text
From stage named "builder":

/app/dist

        ↓

Current stage:

/app/dist
```

It does **not** copy the whole builder image.

This is the key multi-stage concept.

---

# 43. Why `node:24-alpine` Is Smaller

The runner uses:

```dockerfile
FROM node:24-alpine AS runner
```

Alpine Linux is designed to be lightweight.

So:

```text
node:22
```

is a general image, while:

```text
node:24-alpine
```

is an Alpine-based lightweight image.

This helps reduce final image size.

---

# 44. Why `bash` Was Missing

The runner uses:

```dockerfile
FROM node:24-alpine
```

Then:

```bash
docker exec -it <container> bash
```

failed because Alpine generally provides:

```text
sh
```

rather than Bash.

Use:

```bash
docker exec -it <container> sh
```

---

# 45. What You Saw Inside the Multi-Stage Container

You ran:

```bash
docker exec -it 322b8205be37 sh
```

and saw approximately:

```text
/app
├── dist
└── package.json
```

This is evidence that the multi-stage concept worked.

Compare with the normal image:

```text
/app
├── node_modules
├── Dockerfile
├── README.md
├── package.json
├── package-lock.json
├── public
├── src
├── vite.config.js
└── ...
```

### Normal image

Contains source and dependencies.

### Multi-stage runtime

Contains only the runtime content required by the chosen approach.

---

# 46. Normal vs Multi-Stage

| Feature | Normal Dockerfile | Multi-stage |
|---|---|---|
| Build stage | No separate stage | Yes |
| Source code in final image | Usually yes | Can be excluded |
| `node_modules` | Usually yes | Can be excluded/reduced |
| Build dependencies | Usually remain | Stay in builder |
| Image size | Usually larger | Usually smaller |
| Build complexity | Simple | Slightly more complex |
| Development | Good | Usually not primary purpose |
| Production | Possible | Very useful |
| Security surface | Can be larger | Can be smaller |
| Deployment | Simple | Cleaner |
| Final runtime | Node dev server | Production-style runtime |

---

# 47. Normal Dockerfile Architecture

```text
                   Dockerfile
                       │
                       ▼
              ┌─────────────────┐
              │    node:22      │
              ├─────────────────┤
              │ Source Code     │
              │ node_modules    │
              │ Vite            │
              │ Tailwind        │
              │ npm             │
              └────────┬────────┘
                       │
                       ▼
                Vite Dev Server
                       │
                       ▼
                    :5173
```

Good for:

```text
Development
Learning
Testing
Quick EC2 deployment
```

---

# 48. Multi-Stage Architecture

```text
             STAGE 1
             BUILDER
        ┌─────────────────┐
        │ node:22         │
        │ source code     │
        │ npm install     │
        │ npm run build   │
        └────────┬────────┘
                 │
                 │ dist
                 ▼
             STAGE 2
             RUNNER
        ┌─────────────────┐
        │ node:24-alpine  │
        │ dist            │
        │ Vite preview    │
        └────────┬────────┘
                 │
                 ▼
               :4173
```

---

# 49. `npm run dev` vs `vite preview`

Normal Dockerfile:

```bash
npm run dev
```

starts the Vite development server.

Multi-stage:

```bash
vite preview
```

serves the already-built application from:

```text
dist/
```

Conceptually:

```text
Development:

Source
  ↓
Vite dev server
  ↓
5173
```

Production-build preview:

```text
Source
  ↓
npm run build
  ↓
dist/
  ↓
Vite preview
  ↓
4173
```

Important:

> `vite preview` is primarily intended to preview the production build. For a true production deployment, a static server such as Nginx is often preferable for a Vite SPA.

---

# 50. Improved Multi-Stage Dockerfile

A cleaner version for the lab is:

```dockerfile
# Stage 1: Build
FROM node:22 AS builder

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

RUN npm run build


# Stage 2: Runtime
FROM node:24-alpine AS runner

WORKDIR /app

RUN npm install -g vite

COPY --from=builder /app/dist ./dist

EXPOSE 4173

CMD ["vite", "preview", "--host", "0.0.0.0", "--port", "4173"]
```

## Why `npm ci`?

For Docker and CI builds, `npm ci` uses the lock file more strictly and is generally preferable when `package-lock.json` is committed.

---

# 51. Even Better Production Architecture

For a Vite frontend, Node does not have to remain in the final runtime.

A common pattern is:

```text
                Stage 1
             Node builder
                  │
             npm ci
                  │
           npm run build
                  │
                  ▼
                dist/
                  │
                  ▼
             Stage 2
             nginx:alpine
                  │
                  ▼
          Static web server
                  │
                  ▼
                :80
```

Example:

```dockerfile
FROM node:22 AS builder

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

RUN npm run build


FROM nginx:alpine

COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80
```

This is a common production pattern for frontend applications.

---

# 52. Production Frontend Diagram

```text
                    Internet
                       │
                       ▼
                AWS Load Balancer
                       │
                       ▼
                 Nginx Container
                 nginx:alpine
                       │
                       ▼
              /usr/share/nginx/html
                       │
                       ▼
                  React/Vite
                  static files
```

Build flow:

```text
             Docker Build
                  │
        ┌─────────▼─────────┐
        │ Node Builder      │
        │                   │
        │ npm ci            │
        │ npm run build     │
        └─────────┬─────────┘
                  │
                  │ COPY dist
                  ▼
        ┌───────────────────┐
        │ nginx:alpine      │
        │                   │
        │ only static files │
        └───────────────────┘
```

This provides a smaller and simpler runtime.

---

# 53. Why `docker images` Can Still Show Large Images

Suppose the host contains:

```text
node:22
node:24-alpine
devboard-frontend
devboard-frontend-multi-stage
```

Even if the multi-stage final image is small, Docker may still have:

```text
node:22
```

locally because it was used by the builder.

Therefore:

```bash
docker images
```

can show multiple images.

Remove unused images:

```bash
docker image prune
```

More aggressive:

```bash
docker image prune -a
```

Be careful because `-a` removes unused images you may want later.

---

# 54. Docker Disk Investigation Commands

Useful commands:

```bash
docker system df
```

Shows Docker disk usage.

```bash
docker images
```

Shows images.

```bash
docker ps -a
```

Shows containers.

```bash
docker volume ls
```

Shows volumes.

```bash
docker network ls
```

Shows networks.

```bash
docker builder prune
```

Removes unused build cache.

```bash
docker system prune
```

Removes unused Docker objects.

More aggressive:

```bash
docker system prune -a
```

Use carefully.

---

# 55. Docker Storage Visualization

```text
Docker Storage
│
├── Images
│    ├── node:22
│    ├── node:24-alpine
│    ├── devboard-frontend
│    └── devboard-frontend-multi-stage
│
├── Containers
│    ├── running
│    └── stopped
│
├── Volumes
│
├── Networks
│
└── Build Cache
```

This explains why deleting a container does not necessarily delete its image.

---

# 56. Container Lifecycle

```text
docker build
     │
     ▼
   IMAGE
     │
     │ docker run
     ▼
 CREATED
     │
     ▼
 RUNNING
     │
     │ docker stop
     ▼
 STOPPED
     │
     │ docker start
     ▼
 RUNNING
     │
     │ docker rm
     ▼
 REMOVED
```

Useful commands:

```bash
docker start <container>
docker stop <container>
docker restart <container>
docker rm <container>
docker rm -f <container>
```

---

# 57. Complete Docker Installation Command Sheet

```bash
sudo apt-get update

sudo apt-get install docker.io

docker -v

sudo systemctl status docker
```

---

# 58. Docker User Permission Commands

```bash
sudo usermod -aG docker $USER

newgrp docker
```

---

# 59. Docker Check Commands

```bash
docker ps
docker ps -a
docker images
docker info
```

---

# 60. Normal Image Build

```bash
docker build -t devboard-frontend .
```

---

# 61. Normal Container Run

```bash
docker run -d \
  -p 5173:5173 \
  --name devboard-frontend-container \
  devboard-frontend
```

---

# 62. Check Normal Container

```bash
docker ps

docker logs devboard-frontend-container
```

---

# 63. Enter Normal Container

Use:

```bash
docker exec -it devboard-frontend-container sh
```

If the image has Bash:

```bash
docker exec -it devboard-frontend-container bash
```

---

# 64. Stop Normal Container

```bash
docker stop devboard-frontend-container
```

---

# 65. Remove Normal Container

```bash
docker rm devboard-frontend-container
```

Or:

```bash
docker rm -f devboard-frontend-container
```

---

# 66. Multi-Stage Build Commands

Build:

```bash
docker build \
  -f Dockerfile.multistage \
  -t devboard-frontend-multi-stage .
```

Run:

```bash
docker run -d \
  -p 4173:4173 \
  --name devboard-frontend-multi-stage-container \
  devboard-frontend-multi-stage
```

Check:

```bash
docker ps
```

Logs:

```bash
docker logs devboard-frontend-multi-stage-container
```

Enter:

```bash
docker exec -it devboard-frontend-multi-stage-container sh
```

---

# 67. Normal vs Multi-Stage — Easy Interview Answer

### Question

**What is the difference between a normal Dockerfile and a multi-stage Dockerfile?**

### Answer

> A normal Dockerfile generally contains the application source code, dependencies, build tools, and runtime in one image. A multi-stage Dockerfile separates the build environment from the runtime environment. The application is built in the first stage, and only the required build output is copied into the final stage. This can significantly reduce image size and the runtime attack surface.

---

# 68. Interview Question — Why Was Your Image 2.24 GB?

### Answer

> My initial image used the full `node:22` image and installed all npm dependencies, including development dependencies. Therefore, the image contained the Node runtime, source code, `node_modules`, build tools, and other dependencies. I then used a multi-stage build where the application was compiled in a builder stage and only the required `dist` output was copied into a lightweight Alpine runtime image. This reduced the final image size significantly.

---

# 69. Interview Question — Why Use `.dockerignore`?

### Answer

> `.dockerignore` prevents unnecessary files such as `node_modules`, `.git`, `dist`, logs, and local files from being sent to the Docker build context. This reduces build context size, improves build performance, and avoids accidentally including unnecessary files in the image.

---

# 70. Interview Question — Why `0.0.0.0`?

### Answer

> The Vite server inside the container needs to listen on the container's network interfaces. Binding only to localhost can make the application inaccessible through Docker's published port. Therefore I use `--host 0.0.0.0`.

---

# 71. Interview Question — What Does `-p 5173:5173` Mean?

### Answer

> The first port is the host port and the second port is the container port. So `-p 5173:5173` forwards traffic from EC2 port 5173 to port 5173 inside the Docker container.

```text
EC2 5173
   │
   ▼
Docker 5173
   │
   ▼
Vite
```

---

# 72. Interview Question — Why Multi-Stage?

```text
Build environment
       │
       │ npm install
       │ npm run build
       ▼
     dist/
       │
       ▼
Small runtime image
```

Benefits:

- Smaller image
- Faster image transfer
- Fewer unnecessary packages
- Smaller runtime environment
- Better separation of build and runtime
- Potentially smaller security surface

---

# 73. Complete Learning Flow

The first approach:

```text
                    DEVBOARD
                       │
                       ▼
                  React/Vite
                       │
                       ▼
                  package.json
                       │
                       ▼
              ┌────────────────┐
              │ Dockerfile     │
              └───────┬────────┘
                      │
                      ▼
               docker build
                      │
                      ▼
               Docker Image
                      │
                      ▼
                docker run
                      │
                      ▼
                Container
                      │
                      ▼
                Port 5173
                      │
                      ▼
              EC2 Public IP
```

Then the multi-stage approach:

```text
             Dockerfile.multistage
                      │
                      ▼
              ┌───────────────┐
              │ Builder       │
              │ node:22       │
              │ npm ci        │
              │ npm build     │
              └───────┬───────┘
                      │
                      ▼
                    dist
                      │
                COPY --from
                      │
                      ▼
              ┌───────────────┐
              │ Runner        │
              │ node:alpine   │
              │ dist only     │
              └───────┬───────┘
                      │
                      ▼
                    :4173
```

---

# 74. Final Mental Model

Remember these five things:

```text
1. Dockerfile
      ↓
   Instructions

2. docker build
      ↓
   Image

3. docker run
      ↓
   Container

4. -p HOST:CONTAINER
      ↓
   Port mapping

5. Multi-stage
      ↓
   Build → dist → small runtime
```

For the Devboard frontend:

```text
Development Docker
───────────────────
node:22
+ source
+ node_modules
+ Vite
        ↓
   npm run dev
        ↓
      :5173
```

Multi-stage:

```text
───────────────────
node:22 builder
        ↓
   npm run build
        ↓
      dist/
        ↓
node:alpine runner
        ↓
   Vite preview
        ↓
      :4173
```

---

# 75. Key Correction — `EXPOSE` vs AWS Security Group

Do not treat:

```dockerfile
EXPOSE 5173
```

as opening an AWS port.

`EXPOSE` is Docker image metadata.

The actual path to the Internet requires:

```text
Browser
   │
   ▼
EC2 Public IP:5173
   │
   ▼
AWS Security Group
   │
   │ inbound TCP 5173 allowed
   ▼
EC2 Host Port 5173
   │
   ▼
Docker Port Mapping
   │
   │ -p 5173:5173
   ▼
Container Port 5173
   │
   ▼
Vite
```

Therefore:

```text
EXPOSE
  ≠
AWS firewall rule
```

Both Docker port publishing and an appropriate AWS Security Group rule are required for external access.

---

# 76. What I Practiced in This Lab

```text
Docker installation
        ↓
Docker service verification
        ↓
Docker user permissions
        ↓
docker ps / docker ps -a
        ↓
Dockerfile creation
        ↓
Docker image build
        ↓
Docker container run
        ↓
Port mapping
        ↓
Vite host configuration
        ↓
Container logs
        ↓
docker exec
        ↓
Alpine shell troubleshooting
        ↓
Docker image/container cleanup
        ↓
RAM and disk monitoring
        ↓
Swap configuration
        ↓
Multi-stage Docker builds
        ↓
Builder vs runtime separation
        ↓
dist/ production build
        ↓
Image-size optimization
        ↓
Nginx production architecture
```

---

# 77. Final DevOps Takeaway

The main lesson from this lab is:

```text
Application Source
       │
       ▼
   Dockerfile
       │
       ▼
   docker build
       │
       ▼
   Docker Image
       │
       ▼
   docker run
       │
       ▼
   Container
       │
       ▼
   Port Mapping
       │
       ▼
   EC2
```

For development, a simple image can run the Vite development server:

```text
node:22
+
source
+
node_modules
+
Vite
↓
npm run dev
↓
:5173
```

For a production-oriented frontend build, separate the build environment from the runtime:

```text
Node Builder
    │
    │ npm ci
    │ npm run build
    ▼
  dist/
    │
    │ COPY --from
    ▼
Nginx/Alpine Runtime
    │
    ▼
Static React/Vite files
```

The core DevOps principle is:

> **Build with everything you need, but run with only what you need.**

This is the purpose of the multi-stage Docker build.
