# Bus Ticket Booking — Docker, Kubernetes & AWS Deployment Guide

> A detailed, step-by-step guide for engineering students covering three deployment paths for the **Bus Ticket Booking System** (Spring Boot backend + Spring Boot Thymeleaf frontend + MySQL).
>
> Every step, analogy, command, and viva question is written for a reader who has built Spring Boot apps but never containerized or deployed one. Nothing is assumed beyond "I can run `./mvnw spring-boot:run`."

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Application Overview](#application-overview)
3. [Deployment Roadmap](#deployment-roadmap)
4. [SECTION A — Docker](#section-a--docker)
5. [SECTION B — Kubernetes (K8s)](#section-b--kubernetes-k8s)
6. [SECTION C — AWS Deployment](#section-c--aws-deployment)
7. [End-to-End Deployment Paths](#end-to-end-deployment-paths)
8. [Security & Secrets Checklist](#security--secrets-checklist)
9. [Cost Awareness](#cost-awareness)

---

## Prerequisites

You should already have:

| Tool | Why | Install |
|---|---|---|
| JDK 21 | Build the Spring Boot JARs | [adoptium.net](https://adoptium.net) |
| Maven wrapper | Already in each project (`./mvnw`) | — |
| Git | Clone / push / pull | [git-scm.com](https://git-scm.com) |
| A code editor | IntelliJ / VS Code | — |
| AWS Free Tier account | For SECTION C | [aws.amazon.com/free](https://aws.amazon.com/free) |
| ~4 GB free RAM | Docker Desktop + minikube | — |

**Helpful but optional:** basic Linux command-line familiarity, understanding of HTTP requests, ability to read YAML.

---

## Application Overview

Two Spring Boot apps + one database. Two ports + one DB port.

```
+--------------------+     HTTP/REST     +--------------------+    JDBC    +---------+
| bus-ticket-frontend|  ---------------> | bus-ticket-backend |  -------> | MySQL   |
| Thymeleaf + MVC    |  <---  JSON  ---  | REST + JPA         |  <------  | 8.0     |
| :8081              |                   | :8080              |           | :3306   |
+--------------------+                   +--------------------+           +---------+
        ^
        |
    Browser (student/admin)
```

| Component | Responsibility | Stateful? |
|---|---|---|
| `bus-ticket-frontend` | Renders HTML pages using Thymeleaf, posts forms, shows search results | No |
| `bus-ticket-backend` | REST `/api/*` endpoints, JPA persistence, PDF generation, scheduled jobs | No (ephemeral) |
| `MySQL` | Source of truth for trips, bookings, payments, reviews | **Yes** |

**Key property:** the frontend calls the backend via a config value `backend.base-url`. Today it points at `http://localhost:8080`. In Docker Compose it becomes `http://backend:8080`. In Kubernetes it becomes `http://bus-ticket-backend:8080`. In AWS with ALB it becomes `http://<internal-alb>.ap-south-1.elb.amazonaws.com`.

That single property is the hinge that makes all three deployment strategies possible.

---

## Deployment Roadmap

You will deploy the exact same application in three progressively-harder ways. Each level unlocks new production capabilities.

```
+---------------------------------------------------------------+
| Level 1: Docker (local)                                       |
|   - docker build + docker compose up                          |
|   - Three containers on your laptop                           |
|   - SECTION A                                                 |
+---------------------------------------------------------------+
                          ||
                          \/  once you can do this reliably
+---------------------------------------------------------------+
| Level 2: Kubernetes (minikube first, then cloud-ready)        |
|   - Pods, Deployments, Services, Ingress                      |
|   - Self-healing, rolling updates, scaling                    |
|   - SECTION B                                                 |
+---------------------------------------------------------------+
                          ||
                          \/  once your YAMLs work locally
+---------------------------------------------------------------+
| Level 3: AWS (four deployment options)                        |
|   - Single EC2 + Docker Compose                               |
|   - EC2 + RDS                                                 |
|   - ECS Fargate + RDS + ALB                                   |
|   - EKS + RDS + ALB                                           |
|   - SECTION C                                                 |
+---------------------------------------------------------------+
```

**Each section is independently runnable.** You can do Docker only and call it a day, or go all the way to EKS if you want the full DevOps experience.

---

## What Each Section Contains

| Section | What's inside | Viva Qs |
|---|---|---|
| A — Docker | History, architecture, images/containers/volumes/networks, Dockerfile deep-dive, multi-stage builds, docker-compose, troubleshooting | ~60 |
| B — Kubernetes | Control plane architecture, Pods/Deployments/Services/Ingress/StatefulSets, minikube setup, full YAMLs for bus-ticket, troubleshooting | ~70 |
| C — AWS | ECR push, four deployment options (EC2, EC2+RDS, ECS Fargate, EKS), ALB setup, IAM, Secrets Manager, cleanup checklist, cost warnings | ~80 |

**Total: 200+ viva questions** across deep technical, architectural, and jumbled-concept styles.

---

## How to Read This Guide

- **Reading as reference?** Jump directly to the section you need via the TOC.
- **Reading to learn?** Do Section A first (fundamentals), then pick B (if K8s is on your syllabus) or C (if AWS is).
- **Preparing for viva?** Each section ends with a viva-questions subsection organised by difficulty.
- **Doing a project demo?** Use the "End-to-End Deployment Paths" near the end — it stitches sections into a runbook.

---
---
# SECTION A — DOCKER

This section of the deployment guide takes you from "I have never run a `docker` command in my life" to "I can package my Bus Ticket Booking System (backend + frontend + MySQL) into containers and run the full stack with a single command."

We are going to take it slow. If you already know Docker, skim the headers and jump to Part A5. If you have never touched Docker, start at Part A1 and read linearly — every part builds on the previous one.

Throughout the section we will keep coming back to our concrete project:

- Backend: `bus-ticket-booking-system` (Spring Boot 3.3.5, Java 21, REST API under `/api/*`, listens on port 8080)
- Frontend: `bus-ticket-frontend` (Spring Boot 3.3.5, Java 21, Thymeleaf, listens on port 8081, no DB)
- Database: MySQL 8, schema `busticketbooking`, user `root`

Ready? Let's go.

---

## Part A1: What is Docker?

### The One-Sentence Definition

Docker is a tool that lets you package your application and everything it needs to run (the JVM, the OS libraries, configuration files, even a tiny Linux filesystem) into a single portable box called a **container**, so that the box runs the exact same way on your laptop, your teammate's laptop, the college lab PC, and a production server in AWS.

That's it. The rest of this section is just unpacking that sentence.

### The Shipping Container Analogy

Before 1956, shipping goods across the ocean was a nightmare. Every port had its own system. Bananas from India, motorbikes from Japan, cotton bales from Egypt — each was loaded differently, stored differently, and unloaded differently. Ports needed dozens of specialists. A ship's cargo hold was a chaotic jigsaw puzzle.

Then a trucking businessman named Malcolm McLean had an idea: what if every piece of cargo was loaded into a standard-size metal box? The box doesn't care what's inside — bananas, motorbikes, cotton, cars, whatever. The ship doesn't care what's in the box either. It just stacks boxes. The crane at the port just lifts boxes. The truck at the destination just drops the box off.

This one simple standardization completely transformed global trade. Shipping costs collapsed. Loading time went from days to hours. This is literally why globalization is possible.

Software had the same problem. "It works on my machine" is the developer's version of "the banana boat has a different hold than the motorbike boat." Docker is the shipping container for software. You put your app + its dependencies into a standard box (a Docker container), and now it doesn't matter what's underneath — Windows, macOS, Ubuntu, Amazon Linux, AWS ECS, Kubernetes — the box runs.

The Docker logo is literally a whale carrying shipping containers. That's not a coincidence. That is the entire philosophy.

### A Second Analogy: The Tiffin Box

If shipping containers feel too industrial, try this. Imagine a Mumbai dabbawala. Every office worker in Mumbai gets a home-cooked lunch delivered by a man who has never seen their office, using a box that every dabbawala knows how to handle.

The tiffin:
- Has a standard shape (everyone's bicycle rack fits it)
- Contains everything the worker needs for lunch (rice, dal, sabzi, chapati — no outside ingredients needed)
- Is sealed and identified so it doesn't get mixed up
- Is portable between home, train, bicycle, and office

A Docker container is a tiffin for your app. Inside: JVM, your JAR, config files, required system libraries. Outside: just a standard interface. The host doesn't need to know Java is installed — the container brought its own Java.

### The "Works On My Machine" Problem

Every student developer has lived this nightmare. You built a Spring Boot project on your laptop. It runs. You zip it and send it to your teammate. It doesn't run. Why?

Possible reasons:
- You have Java 21, they have Java 17
- You have MySQL 8, they have MySQL 5.7
- Your `application.properties` points to `localhost:3306` but their MySQL is on a different port
- Your `JAVA_HOME` is set, theirs is not
- You installed some native library (like `tesseract` or `poppler`) and forgot to document it
- You're on Windows, they're on Mac — path separators are different

Multiply this by five team members and a submission deadline. Nightmare fuel.

Docker solves this by saying: instead of sending the source code and hoping they have the same setup, send the **entire environment** as one package. Java, MySQL driver, config, your JAR — all inside the box. If it runs on your laptop in Docker, it runs on theirs in Docker, guaranteed.

### A Bit of History

- **2013**: Solomon Hykes demos Docker at PyCon. It takes the internet by storm.
- **2014-2015**: Every startup starts rewriting their deployment to use Docker.
- **2015**: Google open-sources Kubernetes (the orchestrator that manages thousands of containers).
- **2017**: Docker Inc. splits into Docker CE (free) and Docker EE (enterprise, later sold to Mirantis).
- **2020-now**: Docker is the default way to ship software. Every major cloud (AWS, GCP, Azure) has native container services.

Adoption stats you can quote in a viva:
- Over 13 million developers use Docker (per Docker's own 2023 report)
- Docker Hub has over 15 million images
- Around 75% of organizations use containers in production (CNCF survey)
- Most Fortune 500 companies use Docker or Kubernetes somewhere

### What Docker Actually Solves

| Problem | Without Docker | With Docker |
|---------|---------------|-------------|
| Portability | "Works on my machine" | Works identically everywhere |
| Dependency hell | Manually install Java, MySQL, libs | All baked into the image |
| Onboarding new team member | 2-day setup doc | `docker compose up` |
| Running multiple versions | Conflict on the host | Each version in its own container |
| Clean uninstall | Leftover files everywhere | `docker rm` and it's gone |
| Scaling | Build new VM, install stack | Spin up N copies of image |
| Reproducible builds | "What version did we test with?" | Image tag pins everything |

### Containers vs Virtual Machines

This confuses everyone at first. A VM (like VirtualBox, VMware, or an EC2 instance) and a container feel similar — both isolate software. But they are very different.

| Aspect | Virtual Machine | Container |
|--------|----------------|-----------|
| What's virtualized | Entire hardware (CPU, RAM, disk, NIC) | Just the process + filesystem |
| Guest OS | Full OS (Ubuntu, Windows) inside | Shares host kernel, no guest OS |
| Boot time | 30 seconds to minutes | Under 1 second |
| Size | GBs (OS + app) | MBs to hundreds of MBs |
| Isolation | Very strong (hypervisor) | Strong (kernel namespaces + cgroups) |
| Density per host | Tens of VMs | Hundreds or thousands of containers |
| Overhead | High (whole OS running) | Very low |
| Use case | Strong isolation, different OS | Packaging apps, microservices |

Picture it like this:

```
VIRTUAL MACHINES                   CONTAINERS
+-----------+  +-----------+      +-------+ +-------+ +-------+
|  App A    |  |  App B    |      | App A | | App B | | App C |
+-----------+  +-----------+      +-------+ +-------+ +-------+
|  Libs     |  |  Libs     |      |     Shared Host OS Kernel |
+-----------+  +-----------+      +---------------------------+
|  Guest OS |  |  Guest OS |      |    Container Runtime      |
+-----------+  +-----------+      +---------------------------+
|        Hypervisor        |      |         Host OS           |
+--------------------------+      +---------------------------+
|       Host Hardware      |      |       Host Hardware       |
+--------------------------+      +---------------------------+
```

Key insight: **containers share the host OS kernel**. They don't boot their own kernel, so they start instantly and use almost no extra memory. That's why you can run 50 containers on a laptop that could barely handle 3 VMs.

### Core Terminology (Memorize This)

You'll see these words constantly. Learn them now and the rest of Docker becomes easy.

- **Image**: A read-only blueprint. Think "class" in Java. Example: `eclipse-temurin:21-jre-alpine`. Images are built from a `Dockerfile` and stored in a registry.
- **Container**: A running instance of an image. Think "object" in Java. You can run many containers from the same image, each isolated from the others.
- **Dockerfile**: A plain text recipe file with instructions (`FROM`, `COPY`, `RUN`, `CMD`, ...) that tells Docker how to build an image.
- **Registry**: A hosting service for images. Docker Hub is the public one. AWS ECR, GCP GCR, GitHub Container Registry are private alternatives.
- **Layer**: An image is built as a stack of read-only layers, one per Dockerfile instruction. Layers are cached and shared between images — this makes builds fast.
- **Volume**: Persistent storage that lives outside the container's lifecycle. Used for databases, uploaded files, anything you don't want to lose when the container restarts.
- **Network**: A virtual network that containers join. Containers on the same network can talk to each other by name.
- **Tag**: A label attached to an image, usually a version. `bus-ticket-backend:latest` or `mysql:8.0.36`.
- **Docker Daemon (`dockerd`)**: The background service that actually manages containers. You don't talk to it directly.
- **Docker CLI (`docker`)**: The command-line tool you run. It talks to the daemon over a socket.
- **Docker Compose**: A tool for running multi-container apps (exactly our case: backend + frontend + MySQL).

Put a sticky note with these definitions on your monitor for the first week.

---

## Part A2: Docker Architecture Deep Dive

Knowing *how* Docker works under the hood separates students who pass vivas from students who ace them. Let's open the hood.

### The Big Picture

```
+-----------------------------------------------------+
|                   YOUR TERMINAL                     |
|                                                     |
|   $ docker run -p 8080:8080 bus-ticket-backend      |
|            |                                        |
|            |  (HTTP over Unix socket                |
|            |   /var/run/docker.sock)                |
|            v                                        |
|   +---------------------+                           |
|   |   Docker CLI        |                           |
|   |   (docker command)  |                           |
|   +---------------------+                           |
|            |                                        |
+------------|----------------------------------------+
             |
             v
+-----------------------------------------------------+
|               DOCKER DAEMON (dockerd)               |
|                                                     |
|   - Receives API calls from CLI                     |
|   - Manages images, networks, volumes               |
|   - Delegates container ops to containerd           |
|                                                     |
|            |                                        |
|            v                                        |
|   +---------------------+                           |
|   |    containerd       |   <-- CNCF graduated     |
|   |  (high-level        |       project, manages   |
|   |   runtime)          |       container lifecycle|
|   +---------------------+                           |
|            |                                        |
|            v                                        |
|   +---------------------+                           |
|   |       runc          |   <-- OCI reference      |
|   |  (low-level         |       runtime, actually  |
|   |   runtime)          |       creates the proc   |
|   +---------------------+                           |
|            |                                        |
+------------|----------------------------------------+
             |
             v
+-----------------------------------------------------+
|                 LINUX KERNEL                        |
|                                                     |
|   namespaces (isolation)                            |
|   cgroups (resource limits)                         |
|   union filesystem (OverlayFS)                      |
|   seccomp, capabilities, AppArmor (security)        |
+-----------------------------------------------------+
```

Let's walk through each layer.

### The Docker Client

When you type `docker run`, you're invoking the Docker CLI. This is just a thin program that translates your command into an HTTP API call and sends it to the Docker Daemon. The client can even run on a completely different machine from the daemon (you can set `DOCKER_HOST=tcp://...`), though this is rare.

### The Docker Daemon

The daemon (`dockerd`) is the brains. It:
- Listens on a Unix socket (`/var/run/docker.sock` on Linux/Mac) or a named pipe (on Windows)
- Pulls images from registries
- Builds images from Dockerfiles
- Manages networks and volumes
- Starts and stops containers (via containerd)

On Docker Desktop (Windows/Mac), the daemon runs inside a tiny Linux VM, because containers need a Linux kernel. That VM is invisible to you — it just works.

### containerd

containerd is the "container manager." It speaks to the kernel, manages the container's lifecycle (create, start, stop, delete), handles image pulling, and provides a stable API for higher-level tools like Docker, Kubernetes, and Podman.

containerd itself is a graduated project of the CNCF (Cloud Native Computing Foundation). Kubernetes uses containerd directly now (since 1.24, Docker was removed as a runtime — but Docker-built images still run fine because they follow the OCI spec).

### runc

runc is the actual program that creates the container process using kernel features (namespaces, cgroups). It's a reference implementation of the OCI (Open Container Initiative) runtime spec. When runc finishes setting everything up, it `exec`s your program (e.g., `java -jar app.jar`) inside the isolated environment and gets out of the way.

### Kernel Features That Make Containers Possible

A container isn't magic. It's just a normal Linux process with three things wrapped around it by the kernel:

1. **Namespaces** — give the process its own view of the system:
   - `pid` namespace: your process thinks it's PID 1, can't see host processes
   - `net` namespace: its own network interfaces and IP
   - `mnt` namespace: its own filesystem view
   - `uts` namespace: its own hostname
   - `ipc` namespace: its own shared memory segments
   - `user` namespace: its own UIDs/GIDs
2. **cgroups (control groups)** — limit what the process can use:
   - CPU (e.g., max 0.5 cores)
   - Memory (e.g., max 512MB, else OOM killed)
   - Disk I/O, network bandwidth
3. **Union filesystem** — makes the container's filesystem appear as a single tree, but it's actually a stack of read-only layers with a thin writable top layer.

When you hear someone say "containers are just processes," this is literally what they mean. There's no VM, no second OS. Just a normal process with isolation decorations.

### Union Filesystems and Image Layers

This is the single most important concept in Docker. Get this right and everything else makes sense.

An image is not one giant blob. It's a stack of layers, where each layer is a diff from the one below. Every instruction in your Dockerfile creates a new layer.

Example: our backend's Dockerfile.

```
Layer 5: ENTRYPOINT ["java", "-jar", "app.jar"]   (metadata)
Layer 4: COPY --from=build /app/target/*.jar app.jar   (8MB)
Layer 3: WORKDIR /app                               (almost 0 bytes)
Layer 2: eclipse-temurin:21-jre-alpine base         (200MB)
Layer 1: alpine linux                               (5MB)
```

The kernel uses OverlayFS (or AUFS on older systems) to merge these layers into a single filesystem view. Writes go into a thin writable layer on top that's unique to the container. When the container is deleted, the writable layer is deleted, and the read-only layers are preserved for the next container.

Consequences of this design:

- **Images are cached**. If 10 containers share the same base image, the bytes are stored once on disk.
- **Layers are cached across builds**. If you rebuild your image and nothing before layer 4 changed, Docker reuses layers 1-3 from the cache. Your build is fast.
- **Order matters**. Put things that change often (your JAR, your source code) at the bottom of the Dockerfile. Put things that rarely change (installing dependencies) at the top. Otherwise every build invalidates the cache.

We'll come back to this when we write the real Dockerfile.

### Image Layer Caching in Practice

Consider these two Dockerfiles for our backend:

**Bad version:**
```dockerfile
FROM maven:3.9-eclipse-temurin-21
WORKDIR /app
COPY . .
RUN mvn clean package -DskipTests
```

**Good version:**
```dockerfile
FROM maven:3.9-eclipse-temurin-21
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests
```

The good version pulls dependencies (the slow part, hundreds of MB) in a separate layer that only invalidates when `pom.xml` changes. Edits to Java source files only bust the last two layers. A typical rebuild drops from 5 minutes to 20 seconds.

### Registries

A registry is a server that stores images. When you `docker pull mysql:8`, Docker contacts a registry, downloads the image layers, and caches them locally.

- **Docker Hub** (`hub.docker.com`) — the public default. Free for open-source images. Rate-limited for anonymous users (100 pulls per 6 hours).
- **Amazon ECR** — AWS's private registry. Integrates with IAM. We'll use this in Section B or C for production.
- **Google Artifact Registry / GCR** — GCP's equivalent.
- **GitHub Container Registry (`ghcr.io`)** — tied to GitHub repos. Great for open-source projects.
- **Private / self-hosted** — run your own with the `registry` image.

An image name is actually `[registry]/[namespace]/[repository]:[tag]`. Defaults fill in the blanks:

- `nginx` → `docker.io/library/nginx:latest`
- `sarthak/bus-ticket-backend` → `docker.io/sarthak/bus-ticket-backend:latest`
- `12345.dkr.ecr.ap-south-1.amazonaws.com/bus-ticket-backend:v1.0` → explicit ECR URL

### Docker Networks

Every container connects to at least one network. Docker gives you several types out of the box.

| Network Driver | What It Does | When To Use |
|----------------|--------------|-------------|
| bridge | Default. Creates a virtual bridge (`docker0`) on the host; containers get a private IP | Most single-host setups |
| host | Container shares the host's network stack (no isolation) | When you need raw network performance or weird port tricks |
| none | No network at all; container is completely isolated | Batch jobs, security sensitive workloads |
| overlay | Multi-host network across a Docker Swarm cluster | Production swarm clusters |
| macvlan | Container gets its own MAC address on the physical LAN | When you need the container to look like a real machine on your LAN |

For our project we use a **custom bridge network** (created automatically by Docker Compose). This has one magical property: containers on the same custom bridge can reach each other by **service name as a DNS hostname**. So our `frontend` container can say "connect to `http://backend:8080`" and Docker's built-in DNS resolves `backend` to the right container IP.

This is a huge deal. No more hardcoding IPs. Service discovery is free.

### Volumes

Containers are ephemeral. When you stop and remove a container, its writable layer is deleted. That's great for your app code, but catastrophic for your MySQL data.

Volumes are how you persist data.

| Volume Type | What It Is | Use Case |
|-------------|-----------|----------|
| Named volume | Docker-managed directory, lives on host at `/var/lib/docker/volumes/<name>` | Databases, anything you want Docker to manage |
| Bind mount | Maps a host path directly into container | Local development, mounting source code live |
| tmpfs | In-memory only, never hits disk | Secrets you don't want on disk, scratch space |
| Anonymous volume | Like named but with a random name (usually a mistake) | Avoid; you'll forget they exist |

For MySQL in our compose file, we'll use a named volume called `mysql-data` so that killing and recreating the MySQL container doesn't lose all our bookings.

### Container Lifecycle

A container goes through these states:

```
         docker create            docker start
   -->  [Created]  ---------->  [Running]  <------> [Paused]
                                    |          docker pause/unpause
                                    |
                                    | docker stop
                                    v
                                [Stopped/Exited]
                                    |
                                    | docker rm
                                    v
                                 [Removed]
```

- **Created**: Container is set up but not started. `docker create` produces this.
- **Running**: Process is executing. `docker run` = create + start.
- **Paused**: Process is frozen via cgroups freezer. Memory is preserved.
- **Stopped/Exited**: Process has ended (either naturally or via `docker stop`). Writable layer is still around.
- **Removed**: `docker rm` deletes the writable layer and metadata. Gone for good.

Useful commands along the way:

```bash
docker ps          # list running containers
docker ps -a       # list all (including stopped)
docker logs <id>   # see stdout/stderr
docker exec -it <id> sh   # get a shell inside a running container
docker stop <id>   # graceful shutdown (SIGTERM then SIGKILL)
docker kill <id>   # immediate SIGKILL
docker rm <id>     # delete stopped container
docker rmi <img>   # delete image
```

---

## Part A3: Types, Sub-types, Features

Let's catalog the various flavors of the things we just talked about. This is useful reference material — skim, then come back when you need it.

### Image Types (By Base)

Almost every Docker image starts `FROM` some base image. The choice of base affects size, security, and capability.

| Base | Size | Pros | Cons | Good For |
|------|------|------|------|----------|
| `alpine` | ~5MB | Tiny, fast to pull | Uses musl libc (not glibc), some libraries break | Web servers, Spring Boot (with alpine-compatible JRE) |
| `debian:slim` | ~80MB | Full glibc, widely compatible | Larger than Alpine | Apps with native deps |
| `ubuntu` | ~80MB | Familiar, lots of packages | Overkill for most | Dev boxes, complex apps |
| `distroless` (Google) | ~20MB | No shell, no package manager, very secure | Hard to debug inside | Production prod-like |
| `scratch` | 0 bytes | Literally empty | You must provide everything (static binaries only) | Go binaries, ultra-minimal |

For our Spring Boot services, we'll use `eclipse-temurin:21-jre-alpine` — a Java-specific image built on Alpine, about 180MB. It contains only a JRE (not a JDK), which is enough to *run* a JAR but not to *compile* one.

### Multi-Architecture Images

Your laptop might be x86_64 (Intel/AMD) or arm64 (Apple Silicon, Raspberry Pi, AWS Graviton). An image built for one won't run on the other unless you either:

- Pull a **multi-arch** image (Docker picks the right variant for your platform automatically)
- Use `docker buildx` to build for multiple architectures

Official images like `eclipse-temurin`, `mysql`, `nginx` are all multi-arch. You don't need to think about it.

### Container Types (By How You Run Them)

| Type | Flags | Behavior |
|------|-------|----------|
| Foreground | `docker run` | Attaches stdout/stderr to your terminal. Ctrl+C stops it. |
| Detached | `docker run -d` | Runs in background, returns container ID. |
| Interactive | `docker run -it` | Attaches a TTY. Needed for shells. |
| One-shot | `docker run --rm` | Deletes itself on exit. Great for throwaway tasks. |

Example flavors of the same command:

```bash
# Foreground — see logs live, Ctrl+C to stop
docker run -p 8080:8080 bus-ticket-backend

# Detached — run in background
docker run -d -p 8080:8080 --name backend bus-ticket-backend

# Interactive shell inside a temporary container
docker run --rm -it eclipse-temurin:21-jre-alpine sh
```

### Volume Types (Review With Detail)

- **Named volume**
  ```bash
  docker volume create mysql-data
  docker run -v mysql-data:/var/lib/mysql mysql:8
  ```
  Docker manages this. Survives container deletion. Backed up via `docker volume` commands.

- **Anonymous volume**
  ```bash
  docker run -v /var/lib/mysql mysql:8
  ```
  Same as named but with an ugly auto-generated name. You *will* leak these. Avoid.

- **Bind mount**
  ```bash
  docker run -v /home/sarthak/code/app:/app myimage
  ```
  Host path `:` container path. The host path *must* exist. Changes on the host are immediately visible in the container. Perfect for development (live code reload).

- **tmpfs mount**
  ```bash
  docker run --tmpfs /tmp:size=64m myimage
  ```
  In-memory only. Fast. Never written to disk. Good for caches or secrets.

### Network Types (Review)

We covered these earlier. Quick refresher:

- `bridge` — default; use for single-host deployments
- `host` — no network isolation; container uses host's eth0 directly
- `none` — no network; complete isolation
- `overlay` — multi-host (Swarm)
- `macvlan` — container is a first-class device on your LAN

Create a custom bridge:

```bash
docker network create bus-net
docker run --network bus-net --name backend bus-ticket-backend
docker run --network bus-net --name frontend bus-ticket-frontend
```

Now `frontend` can reach `backend` by name.

### Dockerfile Instructions Reference

| Instruction | Purpose | Example |
|-------------|---------|---------|
| `FROM` | Base image (must be first non-ARG) | `FROM eclipse-temurin:21-jre-alpine` |
| `RUN` | Execute command at build time, creates a layer | `RUN apt-get update && apt-get install -y curl` |
| `COPY` | Copy files from build context into image | `COPY target/app.jar /app/app.jar` |
| `ADD` | Like COPY, but also handles URLs and auto-extracts tar files | `ADD https://example.com/x.tar.gz /tmp/` |
| `CMD` | Default command when container starts (can be overridden) | `CMD ["java","-jar","app.jar"]` |
| `ENTRYPOINT` | Command that always runs (CMD becomes its args) | `ENTRYPOINT ["java","-jar"]` |
| `ENV` | Set environment variable | `ENV SPRING_PROFILES_ACTIVE=docker` |
| `ARG` | Build-time variable | `ARG JAR_FILE=target/*.jar` |
| `EXPOSE` | Document which port the app listens on (doesn't publish it!) | `EXPOSE 8080` |
| `VOLUME` | Declare a path as a mount point | `VOLUME /var/lib/mysql` |
| `WORKDIR` | Set working directory for subsequent instructions | `WORKDIR /app` |
| `USER` | Run as this user | `USER appuser` |
| `HEALTHCHECK` | How Docker checks if container is healthy | `HEALTHCHECK CMD curl -f http://localhost:8080/actuator/health` |
| `ONBUILD` | Trigger run when a derived image is built | Rarely used |
| `LABEL` | Attach metadata | `LABEL maintainer="sarthak@college.edu"` |
| `STOPSIGNAL` | Signal to send on stop | `STOPSIGNAL SIGTERM` |
| `SHELL` | Change default shell | `SHELL ["/bin/bash","-c"]` |

### Docker CLI vs Docker Compose vs Docker Swarm vs Kubernetes

People confuse these. Here's the lineup:

| Tool | Scope | When |
|------|-------|------|
| Docker CLI (`docker run`, `docker build`) | One container at a time | Manual runs, CI scripts |
| Docker Compose (`docker compose up`) | Multi-container on one host | Local dev, small single-host prod |
| Docker Swarm (`docker swarm`) | Multi-container across many hosts | Simple cluster, almost dead |
| Kubernetes (`kubectl`) | Massive scale, many hosts, self-healing, rolling updates | Real production, big teams |

For our project: **Docker Compose**. It's exactly the right tool for a 3-service app on one machine.

---

## Part A4: Install Docker

Before we go further, let's actually install Docker. Pick your OS.

### Windows (Docker Desktop)

Docker on Windows runs inside WSL2 (Windows Subsystem for Linux 2) because Docker needs a Linux kernel.

1. Make sure you're on Windows 10 21H2+ or Windows 11.
2. Enable WSL2 (Admin PowerShell):
   ```bash
   wsl --install
   ```
3. Reboot.
4. Download Docker Desktop from https://www.docker.com/products/docker-desktop
5. Run the installer. When asked, enable "Use WSL2 based engine."
6. Start Docker Desktop. You'll see a whale icon in the system tray.
7. Verify:
   ```bash
   docker --version
   docker run hello-world
   ```

**Why WSL2?** Because Docker containers use Linux kernel features (namespaces, cgroups) that don't exist on Windows natively. WSL2 is a real Linux kernel running inside Windows.

### Linux (Ubuntu)

The one-liner (trusted convenience script from Docker):

```bash
curl -fsSL https://get.docker.com | sudo sh
```

Then add your user to the docker group so you don't need `sudo`:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Verify:

```bash
docker --version
docker run hello-world
```

### Linux (Amazon Linux 2 / 2023)

```bash
sudo yum install -y docker
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
# log out and back in for group change to apply
docker run hello-world
```

### macOS (Docker Desktop)

1. Download from https://www.docker.com/products/docker-desktop
2. Drag to Applications. Open it.
3. Docker Desktop runs a tiny Linux VM using Apple's Virtualization.framework.
4. Verify:
   ```bash
   docker --version
   docker run hello-world
   ```

### Verifying Your Install

If `docker run hello-world` prints:

```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

you're done. This command:
1. Contacted Docker Hub
2. Pulled the `hello-world` image
3. Created a container from it
4. Ran the container (which prints the message)
5. Exited cleanly

You've run your first container.

---

## Part A5: Dockerize the Backend Step-by-Step

Now we get to the real work. Our goal: turn `bus-ticket-booking-system` (the backend) into a Docker image.

### Step 1: Understand Your Build

Before Docker, make sure your project builds cleanly on your host. From the backend project root:

```bash
./mvnw clean package -DskipTests
```

After this, you should have:

```
target/bus-ticket-booking-system-0.0.1-SNAPSHOT.jar
```

This is an executable fat JAR — Spring Boot bundles the Tomcat server and all dependencies inside. If `java -jar target/bus-ticket-booking-system-0.0.1-SNAPSHOT.jar` works on your machine, we can wrap it in Docker.

### Step 2: Decide on a Build Strategy

We have two common approaches:

**A. Build the JAR outside, copy it in**
```dockerfile
FROM eclipse-temurin:21-jre-alpine
COPY target/*.jar app.jar
ENTRYPOINT ["java","-jar","app.jar"]
```
Simple, but requires the JAR to already be built on the host. Bad for CI/CD across machines.

**B. Multi-stage build (Docker builds the JAR itself)**

Two stages. Stage 1 has Maven and JDK, compiles the code. Stage 2 has only a JRE, copies the JAR from stage 1. Final image is tiny, and the build is self-contained.

We'll use B. It's the industry standard and works the same on your laptop, your CI server, and production.

### Step 3: Write the Dockerfile

Create a file literally named `Dockerfile` (no extension) at the root of the backend project (same directory as `pom.xml`):

```dockerfile
# ---------- Stage 1: Build ----------
FROM maven:3.9-eclipse-temurin-21 AS build

WORKDIR /app

# Copy only pom.xml first so dependency download is cached
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Now copy source and build
COPY src ./src
RUN mvn clean package -DskipTests -B

# ---------- Stage 2: Runtime ----------
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

# Copy the fat jar from the build stage
COPY --from=build /app/target/*.jar app.jar

# Document the port
EXPOSE 8080

# Set default Spring profile
ENV SPRING_PROFILES_ACTIVE=docker

# Run as a non-root user for security
RUN addgroup -S spring && adduser -S spring -G spring
USER spring

ENTRYPOINT ["java","-jar","/app/app.jar"]
```

### Step 4: Understand Every Line

Let's go through it slowly. This is viva gold.

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
```
`FROM` picks a base image. `maven:3.9-eclipse-temurin-21` is Maven 3.9 on top of Java 21. `AS build` names this stage so stage 2 can refer to it. This base is big (~500MB) but we only use it during build — the final image won't include it.

```dockerfile
WORKDIR /app
```
`WORKDIR` changes into `/app`, creating it if it doesn't exist. All following instructions run from here. Think `cd /app`, but baked into the image.

```dockerfile
COPY pom.xml .
RUN mvn dependency:go-offline -B
```
Copy only `pom.xml` (not source). Then download every dependency Maven will need. Because `pom.xml` changes rarely compared to source code, this layer is cached. Next time you edit a `.java` file and rebuild, Docker jumps straight past this slow step.

```dockerfile
COPY src ./src
RUN mvn clean package -DskipTests -B
```
Copy the source and build. `-DskipTests` skips tests (we run them in CI, not in the Docker build). `-B` is batch mode — quieter output, no ANSI colors. The resulting fat JAR lands in `/app/target/*.jar`.

```dockerfile
FROM eclipse-temurin:21-jre-alpine
```
Start a new stage with a clean slate. This image has only the Java 21 *Runtime* (no compiler, no Maven). `-jre` not `-jdk` keeps it small. `-alpine` means it's built on Alpine Linux (~5MB). The whole runtime image is around 180MB.

**Why not keep using the maven image?** Because we don't need Maven to run. Ship what you need, nothing more. This is called "minimizing attack surface" — less software in your container means fewer vulnerabilities.

```dockerfile
WORKDIR /app
```
Same as before, but in the new stage. Each stage is independent; `/app` from stage 1 doesn't carry over.

```dockerfile
COPY --from=build /app/target/*.jar app.jar
```
`--from=build` copies a file from an earlier stage's filesystem. We grab whatever JAR Maven produced and rename it `app.jar` so we don't need to know the exact version.

```dockerfile
EXPOSE 8080
```
Documentation only! `EXPOSE` doesn't publish the port to the host. It's just a hint that this container listens on 8080. To actually publish, we use `-p 8080:8080` when running.

```dockerfile
ENV SPRING_PROFILES_ACTIVE=docker
```
Sets an environment variable. Spring Boot will look for `application-docker.properties` (or `application-docker.yml`) when this profile is active. Great for Docker-specific config (like MySQL host = `mysql` instead of `localhost`).

```dockerfile
RUN addgroup -S spring && adduser -S spring -G spring
USER spring
```
Create a non-root user named `spring` and switch to it. If your container gets compromised (through a Log4Shell-style bug), an attacker is stuck as a non-privileged user. Never run production containers as root.

```dockerfile
ENTRYPOINT ["java","-jar","/app/app.jar"]
```
The default command. `ENTRYPOINT` using the exec form (JSON array) means the process runs as PID 1 without a shell wrapper, which is important for signal handling. When `docker stop` sends SIGTERM, it reaches Java directly, and Spring Boot can shut down gracefully.

**Why ENTRYPOINT and not CMD?** Both work here, but ENTRYPOINT is better when the container has one job. If you use CMD, someone can override it with `docker run myimage bash` and not run your app. With ENTRYPOINT, extra args get passed *to* your app (`docker run myimage --server.port=9090`).

### Step 5: Add .dockerignore

Create a `.dockerignore` file next to the Dockerfile. It works like `.gitignore` — files listed here are not sent to the Docker daemon during the build.

```
target/
.git/
.idea/
*.iml
.mvn/wrapper/maven-wrapper.jar
*.log
Dockerfile
docker-compose.yml
README.md
.vscode/
.DS_Store
```

**Why does this matter?** When you run `docker build`, Docker packages the entire build directory (the "build context") and sends it to the daemon. If you have a `target/` folder with 100MB of class files, that 100MB is transmitted. Worse, if your `.git` has 500MB of history, that goes too. `.dockerignore` keeps the context small, builds fast, and avoids accidentally baking secrets (`.env`, `.aws/`) into images.

### Step 6: Create application-docker.properties

We told Spring to use the `docker` profile. Create `src/main/resources/application-docker.properties`:

```properties
spring.datasource.url=jdbc:mysql://${DB_HOST:mysql}:${DB_PORT:3306}/${DB_NAME:busticketbooking}?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
spring.datasource.username=${DB_USERNAME:root}
spring.datasource.password=${DB_PASSWORD}
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL8Dialect

server.port=8080

# Actuator for healthchecks
management.endpoints.web.exposure.include=health,info
management.endpoint.health.probes.enabled=true
```

Notice `${DB_HOST:mysql}` — this reads an environment variable named `DB_HOST`, defaulting to `mysql` if unset. That `mysql` string is the *service name* we'll define in `docker-compose.yml` — Docker's DNS will resolve it to the MySQL container's IP.

### Step 7: Build the Image

From the backend project root:

```bash
docker build -t bus-ticket-backend:latest .
```

Breakdown:
- `docker build` — the build command
- `-t bus-ticket-backend:latest` — tag the resulting image with name `bus-ticket-backend` and version `latest`
- `.` — the build context is the current directory (Docker reads `Dockerfile` from here by default)

First build: 3-5 minutes (downloads Maven, Java, dependencies). Later builds: seconds, because of layer caching.

Check the result:

```bash
docker images | grep bus-ticket-backend
```

You'll see something like:

```
bus-ticket-backend   latest   a1b2c3d4e5f6   2 minutes ago   182MB
```

182MB to ship a Spring Boot app. Not bad.

### Step 8: Run It

Quickest smoke test — does the JVM even start?

```bash
docker run --rm bus-ticket-backend:latest
```

You'll see Spring Boot logs fly by, then it'll crash because there's no MySQL. That's expected. The point is: Docker started the container, Java ran, Spring loaded.

To actually connect to a MySQL running on your laptop:

```bash
docker run --rm -p 8080:8080 \
  -e DB_HOST=host.docker.internal \
  -e DB_PORT=3306 \
  -e DB_NAME=busticketbooking \
  -e DB_USERNAME=root \
  -e DB_PASSWORD=yourpassword \
  bus-ticket-backend:latest
```

Breakdown:
- `-p 8080:8080` — publish container's port 8080 to host's port 8080 (`host_port:container_port`)
- `-e NAME=VALUE` — set environment variables inside the container
- `host.docker.internal` — a magic DNS name (on Docker Desktop) that resolves to the host machine's IP from inside a container

Open `http://localhost:8080/api/buses` (or whatever your endpoint is) in a browser. If you get JSON back, congrats — you just ran Spring Boot in Docker against MySQL on your host.

### Step 9: What is host.docker.internal?

When you're inside a container, `localhost` means the container itself, not your laptop. So `jdbc:mysql://localhost:3306` tries to find MySQL *in the same container*, which doesn't exist.

Docker Desktop (Win/Mac) provides `host.docker.internal` — a special DNS name that resolves to the host's IP from inside any container. Use it when you want a container to talk to a service running directly on your laptop.

**On Linux** this magic name doesn't exist by default. Workarounds:
- Add `--add-host=host.docker.internal:host-gateway` to `docker run`
- Or just use the host's actual IP (e.g., `172.17.0.1` — the docker0 bridge gateway)
- Or better, run MySQL in a container too (which is what we do in A7)

### Step 10: Get a Shell Inside the Running Container

If you want to poke around:

```bash
docker run --rm -it --entrypoint sh bus-ticket-backend:latest
```

You get a shell. Try `ls /app`, `java --version`, `cat /etc/os-release`. Then `exit`.

Or into a running container:

```bash
docker ps
docker exec -it <container_id> sh
```

---

## Part A6: Dockerize the Frontend Step-by-Step

The frontend is almost identical, but simpler — no database. Let's do it.

### Step 1: Understand the Frontend

`bus-ticket-frontend` is a Spring Boot + Thymeleaf app that:
- Listens on port 8081
- Has no database of its own
- Calls the backend via HTTP at a URL stored in the property `backend.base-url`

The property is typically injected like:

```java
@Value("${backend.base-url}")
private String backendUrl;
```

And used in `RestTemplate` / `WebClient` calls like `backendUrl + "/api/buses"`.

For Docker, we'll set `backend.base-url=http://backend:8080` so the frontend container talks to the backend container by service name.

### Step 2: Write the Dockerfile

At the root of `bus-ticket-frontend`, create `Dockerfile`:

```dockerfile
# ---------- Stage 1: Build ----------
FROM maven:3.9-eclipse-temurin-21 AS build

WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn clean package -DskipTests -B

# ---------- Stage 2: Runtime ----------
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app
COPY --from=build /app/target/*.jar app.jar

EXPOSE 8081

ENV SPRING_PROFILES_ACTIVE=docker
ENV BACKEND_BASE_URL=http://backend:8080

RUN addgroup -S spring && adduser -S spring -G spring
USER spring

ENTRYPOINT ["java","-jar","/app/app.jar"]
```

Same multi-stage pattern, with two differences:

1. `EXPOSE 8081` — frontend runs on 8081, not 8080.
2. `ENV BACKEND_BASE_URL=http://backend:8080` — default value that Spring will substitute into `backend.base-url`.

### Step 3: application-docker.properties for Frontend

`src/main/resources/application-docker.properties`:

```properties
server.port=8081
backend.base-url=${BACKEND_BASE_URL:http://localhost:8080}

# Thymeleaf config
spring.thymeleaf.cache=true
spring.thymeleaf.prefix=classpath:/templates/
spring.thymeleaf.suffix=.html

# No database, so disable datasource auto-configuration
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration

management.endpoints.web.exposure.include=health
```

The crucial line: `backend.base-url=${BACKEND_BASE_URL:http://localhost:8080}`. At runtime, Spring reads the `BACKEND_BASE_URL` environment variable (set by Docker) and falls back to `localhost:8080` if it's missing (useful when running outside Docker).

### Step 4: .dockerignore for Frontend

Same pattern as backend:

```
target/
.git/
.idea/
*.iml
.mvn/wrapper/maven-wrapper.jar
*.log
Dockerfile
docker-compose.yml
README.md
```

### Step 5: Build the Image

```bash
docker build -t bus-ticket-frontend:latest .
```

### Step 6: Run It (Pointing at Backend)

If the backend is running as a container named `backend` on network `bus-net`:

```bash
docker network create bus-net

docker run -d --name backend --network bus-net \
  -e DB_HOST=host.docker.internal \
  -e DB_PASSWORD=yourpassword \
  bus-ticket-backend:latest

docker run -d --name frontend --network bus-net \
  -p 8081:8081 \
  -e BACKEND_BASE_URL=http://backend:8080 \
  bus-ticket-frontend:latest
```

Notice:
- Both containers are on `bus-net`, so they can reach each other by service name.
- The backend doesn't need `-p 8080:8080` here because only the frontend talks to it, and they're on the same network.
- The frontend publishes port 8081 to the host so you can open `http://localhost:8081` in your browser.

This works, but running three containers manually is tedious. That's why Docker Compose exists. Part A7.

### Step 7: Quick Logs / Debugging

```bash
docker logs -f frontend       # follow frontend logs
docker logs --tail 50 backend # last 50 lines of backend logs
docker exec -it frontend sh   # shell inside frontend
```

If the frontend returns 500 errors, the first thing to check is its log — it probably can't reach the backend. Possible causes:
- Backend container not on the same network
- Backend took too long to start (frontend tried to call it too early)
- Typo in service name

Compose will solve the "too early" problem with healthchecks.

---

## Part A7: docker-compose for Full Stack

Running `docker run` three times with lots of flags is not fun. Docker Compose lets us declare the whole stack in one YAML file.

### The Plan

We want three services:

```
+---------------------+      +---------------------+      +----------------+
|  bus-ticket-        | ---> |  bus-ticket-        | ---> |   mysql        |
|  frontend           |      |  backend            |      |   (8.0)        |
|  (port 8081)        |      |  (port 8080)        |      |   (port 3306)  |
+---------------------+      +---------------------+      +----------------+
         |
         |  port 8081 published to host
         v
   Your browser at http://localhost:8081
```

- `mysql` — official MySQL 8 image, with a named volume for persistence.
- `backend` — built from our backend Dockerfile, waits for MySQL to be healthy, talks to `mysql:3306`.
- `frontend` — built from our frontend Dockerfile, waits for backend healthcheck, talks to `backend:8080`.

### Where Does compose.yml Live?

Put it one directory up from both projects, so you have:

```
bus-ticket-deployment/
├── docker-compose.yml
├── bus-ticket-booking-system/      <-- backend repo
│   └── Dockerfile
└── bus-ticket-frontend/            <-- frontend repo
    └── Dockerfile
```

Or inside one of them, adjusting paths. Either works.

### The docker-compose.yml

```yaml
version: "3.9"

name: bus-ticket

services:
  mysql:
    image: mysql:8.0.36
    container_name: bus-ticket-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: busticketbooking
      TZ: Asia/Kolkata
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
      - ./init-scripts:/docker-entrypoint-initdb.d:ro
    networks:
      - bus-net
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p$$MYSQL_ROOT_PASSWORD"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s

  backend:
    build:
      context: ./bus-ticket-booking-system
      dockerfile: Dockerfile
    image: bus-ticket-backend:latest
    container_name: bus-ticket-backend
    restart: unless-stopped
    depends_on:
      mysql:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: docker
      DB_HOST: mysql
      DB_PORT: 3306
      DB_NAME: busticketbooking
      DB_USERNAME: root
      DB_PASSWORD: ${DB_PASSWORD}
      JAVA_OPTS: "-Xms256m -Xmx512m"
    ports:
      - "8080:8080"
    networks:
      - bus-net
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8080/actuator/health"]
      interval: 15s
      timeout: 5s
      retries: 10
      start_period: 60s

  frontend:
    build:
      context: ./bus-ticket-frontend
      dockerfile: Dockerfile
    image: bus-ticket-frontend:latest
    container_name: bus-ticket-frontend
    restart: unless-stopped
    depends_on:
      backend:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: docker
      BACKEND_BASE_URL: http://backend:8080
      JAVA_OPTS: "-Xms256m -Xmx384m"
    ports:
      - "8081:8081"
    networks:
      - bus-net

volumes:
  mysql-data:
    name: bus-ticket-mysql-data

networks:
  bus-net:
    name: bus-ticket-net
    driver: bridge
```

### Walk Through the YAML

**Top-level keys:**

- `version: "3.9"` — Compose file format. Recent Docker Engines understand this.
- `name: bus-ticket` — project name, used as prefix for everything Compose creates.

**Services are the containers:**

- Each service has either `image:` (pull from registry) or `build:` (build from a Dockerfile here).
- `container_name:` gives the container a friendly name (otherwise Compose auto-generates one like `bus-ticket_backend_1`).
- `restart: unless-stopped` tells Docker to auto-restart the container if it crashes, but not if you explicitly stop it.
- `environment:` sets env vars inside the container. Values like `${DB_PASSWORD}` are read from your shell or from a `.env` file.
- `ports: - "8080:8080"` publishes host:container ports.
- `volumes:` mounts volumes. `mysql-data:/var/lib/mysql` mounts the named volume. `./init-scripts:/docker-entrypoint-initdb.d:ro` mounts a host folder (read-only) containing SQL files that MySQL runs on first boot.
- `networks:` attaches the service to the `bus-net` network.
- `depends_on:` declares startup order. With `condition: service_healthy`, backend waits until mysql's healthcheck passes.
- `healthcheck:` defines how Docker decides whether the container is healthy.

**Volumes and networks are declared at the bottom:**

- `volumes: mysql-data:` creates a named volume. Data in `/var/lib/mysql` inside the container is persisted across container restarts.
- `networks: bus-net:` creates a custom bridge network. All three services join it and can resolve each other by service name.

### The .env File

Create `.env` next to `docker-compose.yml`:

```
DB_PASSWORD=superSecret123!
```

Compose auto-reads `.env`. Never commit it to Git. Add `.env` to your `.gitignore`.

### Optional: init-scripts/ Folder

If you have a schema you want seeded automatically, create `init-scripts/01-init.sql`:

```sql
CREATE DATABASE IF NOT EXISTS busticketbooking;
USE busticketbooking;

-- your initial tables, sample data, etc.
```

On *first* MySQL container boot (when the volume is empty), MySQL runs everything in `/docker-entrypoint-initdb.d/` in alphabetical order. Perfect for seed data.

### Running the Stack

From the directory containing `docker-compose.yml`:

```bash
# Build all images and start all services in background
docker compose up --build -d

# See logs (follow all services)
docker compose logs -f

# See logs for one service
docker compose logs -f backend

# List running services
docker compose ps

# Stop everything, keep data
docker compose stop

# Start again
docker compose start

# Stop and remove containers + networks (keeps volumes)
docker compose down

# Nuke everything including volumes (danger — deletes DB data)
docker compose down -v

# Rebuild only one service
docker compose build backend
docker compose up -d backend
```

After `docker compose up --build -d`:

```
[+] Running 4/4
 ⠿ Network bus-ticket-net           Created
 ⠿ Container bus-ticket-mysql       Healthy
 ⠿ Container bus-ticket-backend     Healthy
 ⠿ Container bus-ticket-frontend    Started
```

Open `http://localhost:8081` — that's your frontend. It'll render Thymeleaf pages that fetch data from the backend, which reads from MySQL. All in containers.

### How Service DNS Works

Inside the `bus-net` network:

- The backend reads `DB_HOST=mysql`. It tries to open a TCP connection to `mysql:3306`. Docker's embedded DNS server sees "mysql" and returns the container IP (e.g., `172.20.0.2`).
- The frontend reads `BACKEND_BASE_URL=http://backend:8080`. Same story — `backend` resolves to the backend container's IP.

From outside the network (your browser), you still go through the host's published ports: `localhost:8081` for frontend, `localhost:8080` for backend.

### Why depends_on With healthcheck?

`depends_on: mysql` without the condition only waits for the *container* to be created, not for MySQL to be *ready to accept queries*. MySQL takes 10-30 seconds to initialize on first boot. If the backend starts at second 5 and hits MySQL, it gets a connection refused and crashes.

`condition: service_healthy` makes Compose wait until the mysql container's healthcheck reports "healthy" before starting the backend. Same applies to backend → frontend.

---

## Part A8: Common Docker Problems + Solutions

Every single one of these has happened to me or my teammates. Bookmark this section.

| # | Problem | Symptom | Fix |
|---|---------|---------|-----|
| 1 | Cannot connect to Docker daemon | `Cannot connect to the Docker daemon at unix:///var/run/docker.sock` | Start Docker Desktop (Win/Mac). Linux: `sudo systemctl start docker` |
| 2 | Port already in use | `bind: address already in use` | Something else is on that port. `netstat -ano \| findstr :8080` (Win) or `lsof -i :8080` (Linux/Mac). Kill it or change the host port: `-p 18080:8080` |
| 3 | Container exits immediately | `docker ps -a` shows "Exited (1)" | `docker logs <container>` — probably a stack trace. Typical causes: wrong env vars, DB unreachable, bad JVM flag |
| 4 | Spring Boot can't connect to MySQL | `Communications link failure` | Inside a container, `localhost` is the container itself. Use the MySQL service name (`mysql`) not `localhost` |
| 5 | Image too big | Image is 1.2 GB, takes forever to push | Use multi-stage build; use alpine or distroless; add `.dockerignore` |
| 6 | Build cache not reusing | Every build rebuilds everything | Reorder Dockerfile — copy `pom.xml` before `src`, install deps before copying code |
| 7 | `permission denied` on Linux | `docker: permission denied while trying to connect to Docker daemon socket` | `sudo usermod -aG docker $USER && newgrp docker` |
| 8 | MySQL data lost on restart | All bookings gone after `docker compose down && up` | You forgot the volume, or used `down -v` which deletes volumes. Use a named volume and don't use `-v` unless you want to reset |
| 9 | Container OOMKilled | Container keeps restarting | Check `docker inspect <container>` — `"OOMKilled": true`. Give it more memory: `--memory=1g` or in compose `mem_limit: 1g` |
| 10 | Slow build | `docker build` takes 10 minutes every time | Missing `.dockerignore`, or large context being sent. Check build context size: the first line of build output says `Sending build context to Docker daemon XXX MB` |
| 11 | `host.docker.internal` not working on Linux | `UnknownHostException` | Add `--add-host=host.docker.internal:host-gateway` or use Compose's `extra_hosts`, or just put your service in a container |
| 12 | CORS error when frontend talks to backend | Browser blocks request | Frontend is at `localhost:8081`, backend at `localhost:8080` — different origins. Add `@CrossOrigin` or CORS config on the backend, or route via a reverse proxy |
| 13 | Logs not showing | `docker logs` returns nothing | App may be writing to a file, not stdout. Configure logback / log4j to write to console. Or use `docker logs -f` to follow |
| 14 | No space left on device | `no space left on device` during build or pull | `docker system prune -a --volumes` (careful — deletes unused stuff). Check disk: `docker system df` |
| 15 | Network name conflict | `network with name X already exists` | `docker network ls`, then `docker network rm <name>`. Or Compose's project name clashing — change `name:` |
| 16 | Env var not picked up | Spring reads default, ignores your value | Check case: Spring property `backend.base-url` maps from env var `BACKEND_BASE_URL`. Use `docker exec container env` to list what's actually set |
| 17 | Service starts but can't reach other service by name | `connection refused` on `mysql:3306` | Both services must be on the *same user-defined network*. Default bridge doesn't give DNS |
| 18 | `docker compose up` starts services in wrong order | Backend crashes at boot because MySQL isn't ready | Use `depends_on` with `condition: service_healthy`, and define a MySQL healthcheck |
| 19 | Windows line endings break scripts | `/bin/sh^M: bad interpreter` | CRLF vs LF. In Git: `git config core.autocrlf input`. In the Dockerfile: `RUN dos2unix entrypoint.sh` or commit with LF |
| 20 | Docker Desktop eats all RAM | Laptop becomes unusable | Settings → Resources → limit RAM. On WSL2, edit `%UserProfile%/.wslconfig` and restart WSL: `wsl --shutdown` |

### Going Deeper on a Few

**Problem 3 in detail: Container exits immediately**

You run `docker run bus-ticket-backend` and it exits in 2 seconds. What to do:

1. Look at logs: `docker logs <container_id>` (get ID from `docker ps -a`).
2. Look at the last line. 90% of the time it's a clear Java stack trace.
3. If logs are empty, override the entrypoint and get a shell:
   ```bash
   docker run --rm -it --entrypoint sh bus-ticket-backend:latest
   ls -la /app
   java -jar /app/app.jar    # now run manually and see what happens
   ```

**Problem 11 in detail: host.docker.internal on Linux**

Docker Desktop on Windows and Mac auto-injects `host.docker.internal` pointing to the host. Docker Engine on Linux does not. Three fixes:

```bash
# Option 1: Add host mapping at run time
docker run --add-host=host.docker.internal:host-gateway ...

# Option 2: In docker-compose
services:
  backend:
    extra_hosts:
      - "host.docker.internal:host-gateway"

# Option 3: Use the host's real IP or the docker0 gateway (172.17.0.1)
```

**Problem 14 in detail: Disk full**

Docker accumulates: old images, dangling images, stopped containers, unused volumes, build cache. After a few months it's GBs.

```bash
docker system df                  # see disk usage
docker system prune               # removes stopped containers, dangling images, unused networks
docker system prune -a            # plus unused images (aggressive)
docker system prune -a --volumes  # plus unused volumes (destructive!)
docker builder prune              # just the build cache
```

Before running prune, do a `docker images` and `docker volume ls` sanity check. You do not want to delete your MySQL volume with a month of test data.

**Problem 4 in detail: MySQL connection from backend**

This one trips up everybody on day one. You run your backend container, MySQL is also running, and Spring Boot still can't connect. Walk through it methodically.

Step 1: Is MySQL actually reachable on the network you think it's on?

```bash
# From the backend container, try to resolve and reach MySQL
docker exec -it bus-ticket-backend sh
# inside the container:
nslookup mysql           # should return an IP
nc -zv mysql 3306        # should say "open" (if nc is installed)
```

If `nslookup mysql` fails, you and the MySQL container are not on the same network. Fix with Compose (all services in one file share a network) or with `--network bus-net` on `docker run`.

Step 2: Is MySQL accepting connections yet?

MySQL takes 20-40 seconds to initialize on first boot. If your backend tries too early, connection refused. Fix: add a healthcheck and use `depends_on: condition: service_healthy`, as shown in the compose file.

Step 3: Is the password right?

Inside the backend container, `echo $DB_PASSWORD` should show the password. Inside MySQL: `mysql -u root -p` with the same password should log in. Mismatches happen when one side reads from `.env` and the other doesn't.

Step 4: Is the URL right?

A typical correct URL in the backend:

```
jdbc:mysql://mysql:3306/busticketbooking?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
```

`allowPublicKeyRetrieval=true` is needed with MySQL 8's default `caching_sha2_password` auth, else Java gets a "Public Key Retrieval is not allowed" error. `useSSL=false` avoids cert warnings in local dev (not for production!). `serverTimezone=UTC` avoids the dreaded "The server time zone value 'IST' is unrecognized" error.

**Problem 18 in detail: Startup ordering**

Some real compose logs when services start too fast:

```
backend    | Caused by: com.mysql.cj.exceptions.CJCommunicationsException: Communications link failure
backend    | The last packet sent successfully to the server was 0 milliseconds ago.
backend    | The driver has not received any packets from the server.
```

Fix with proper healthcheck:

```yaml
mysql:
  healthcheck:
    test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p$$MYSQL_ROOT_PASSWORD"]
    interval: 10s
    timeout: 5s
    retries: 10
    start_period: 30s

backend:
  depends_on:
    mysql:
      condition: service_healthy
```

`start_period: 30s` is the grace window — failures during this time don't count against the retry budget. It stops Docker from marking MySQL unhealthy during its slow init.

### Debugging Strategy: The Five-Step Checklist

Whenever something goes wrong, run these in order:

```bash
# 1. What's running?
docker ps
docker compose ps

# 2. What died, and why?
docker ps -a                         # shows exited too
docker logs <container_id>           # last logs of that container
docker compose logs --tail 100 backend

# 3. What's the container's environment and network?
docker inspect <container_id>        # huge JSON, look for State, Config.Env, NetworkSettings
docker network inspect bus-net       # see who's on the network

# 4. Can I shell in and reproduce the issue?
docker exec -it <container_id> sh
# inside: ls, cat config files, curl other services, env

# 5. Is it a config problem? Try a clean slate.
docker compose down
docker compose build --no-cache
docker compose up -d
docker compose logs -f
```

90% of issues fall out after step 2 or 3. The other 10% come out after step 4.

### One More Subtle Gotcha: Clock Drift

If your container's clock drifts far from the host's clock (can happen on laptops that sleep/resume), JWT tokens with short expirations break, TLS certificate validation fails, and MySQL replication gets confused. On Linux with Docker Desktop, hibernation is a common cause. Restart Docker Desktop or run `hwclock --hctosys` in the host. Not something you hit often, but when you do, it looks like a mystery.

### Windows-Specific Pain

A few quirks you will hit on Windows + Docker Desktop + WSL2:

- **Filesystem performance**: bind-mounting a Windows path (e.g., `C:\Users\Sarthak\code`) into a Linux container is slow. For dev, put your code under WSL2 (`\\wsl$\Ubuntu\home\sarthak\code`) — 10x faster.
- **Line endings**: Git may check out files with CRLF. Shell scripts break. Fix: `git config --global core.autocrlf input` and re-clone.
- **Drive sharing**: Docker Desktop needs permission to share `C:\`. Settings → Resources → File Sharing. Some OneDrive/synced folders behave oddly — prefer plain local folders.
- **WSL2 memory**: by default WSL2 can grab half your RAM. Create `%UserProfile%\.wslconfig`:
  ```
  [wsl2]
  memory=4GB
  processors=4
  swap=2GB
  ```
  Then `wsl --shutdown` and reopen Docker.

---

## Part A9: Docker Viva Questions

60+ questions, grouped. Skim them now, re-read the night before your viva.

### Basics (Q1–Q15)

**Q1. What is Docker?**

Docker is an open-source platform for packaging, shipping, and running applications inside lightweight containers. A container bundles your app with its dependencies into a single portable unit that runs identically on any system with Docker installed. It solves "works on my machine" problems and is the standard for modern deployment. In our project, we use Docker to package the Spring Boot backend, Thymeleaf frontend, and MySQL DB so the whole stack starts with one command.

**Q2. What is a container?**

A container is a runnable instance of an image — essentially a normal Linux process isolated from others using kernel namespaces (for filesystem, network, PIDs) and cgroups (for CPU/memory limits). It shares the host kernel, so it starts in under a second and uses little memory. Unlike a VM, there's no guest OS. Our backend container, for example, is just a `java -jar` process living in its own namespaces.

**Q3. What is the difference between an image and a container?**

An image is a read-only template (like a class in Java); a container is a running instance of that image (like an object). One image can spawn many containers. Images live in registries and on disk; containers have a lifecycle (created, running, stopped, removed). `docker build` makes images; `docker run` makes containers.

**Q4. What is a Dockerfile?**

A Dockerfile is a plain-text recipe that tells Docker how to build an image, line by line. Each instruction (`FROM`, `COPY`, `RUN`, `CMD`, etc.) adds a new layer to the final image. We wrote one for the backend and another for the frontend — both multi-stage, both ending with a JRE-on-Alpine runtime image.

**Q5. What is a container registry?**

A registry is a server that stores and distributes Docker images. Docker Hub is the default public one; AWS ECR, GCR, and GitHub Container Registry are popular private options. You `docker push` to upload an image and `docker pull` to download it. In production we'll push `bus-ticket-backend:latest` to ECR and EC2 will pull it.

**Q6. How is Docker different from a virtual machine?**

A VM virtualizes hardware and runs its own OS kernel; a container shares the host kernel and only isolates processes. VMs take GBs and minutes to boot; containers take MBs and seconds. You can run hundreds of containers where you'd fit a dozen VMs. VMs give stronger isolation (good for multi-tenant); containers give density and speed (good for microservices).

**Q7. What is Docker Hub?**

Docker Hub is the default public registry at `hub.docker.com`. It hosts official images (`mysql`, `nginx`, `eclipse-temurin`) as well as user-pushed ones. It's free for public images with pull-rate limits on anonymous users. We pull `maven:3.9-eclipse-temurin-21` and `eclipse-temurin:21-jre-alpine` from it.

**Q8. What does `docker run` do?**

It creates a new container from an image and starts it. Under the hood it's `docker create` + `docker start` combined. Common flags: `-d` (detached), `-p host:container` (port publish), `-e KEY=VAL` (env var), `-v host:container` (volume), `--name X` (friendly name), `--rm` (auto-remove on exit).

**Q9. What is the difference between `docker stop` and `docker kill`?**

`docker stop` sends SIGTERM, waits 10 seconds (default), then sends SIGKILL. The app gets a chance to clean up — Spring Boot will gracefully shut down its Tomcat, close DB connections, etc. `docker kill` sends SIGKILL immediately — abrupt termination, no cleanup. Prefer `stop` unless the app is stuck.

**Q10. What is Docker Compose?**

Docker Compose is a tool for defining and running multi-container applications via a YAML file. Instead of running three `docker run` commands with long flag lists, you write `docker-compose.yml` once and `docker compose up` handles everything. It's perfect for our 3-service (mysql + backend + frontend) setup.

**Q11. What is a Docker image tag?**

A tag is a label attached to an image, usually denoting a version: `mysql:8.0.36`, `bus-ticket-backend:v1.0`, `bus-ticket-backend:latest`. `latest` is just a conventional default tag — it does NOT automatically mean "newest"; it's just whatever was last tagged `latest`. For production, always pin to specific versions, never `latest`.

**Q12. What is the difference between `COPY` and `ADD`?**

Both copy files into the image. `ADD` additionally handles remote URLs and auto-extracts tar archives. Best practice: use `COPY` for anything local (it's predictable), use `ADD` only when you specifically need tar-extraction. In our backend Dockerfile we used `COPY --from=build ...` to copy the JAR from the build stage.

**Q13. What is `EXPOSE` in a Dockerfile?**

`EXPOSE` is documentation — it tells readers (and tools) which port the container listens on, but it does NOT actually publish the port to the host. To publish, you need `-p 8080:8080` at run time or `ports:` in compose. We wrote `EXPOSE 8080` in the backend Dockerfile purely for readability.

**Q14. What does "image pull" mean?**

Pulling is downloading an image (its layers) from a registry to your local disk. `docker pull mysql:8` fetches every layer and stores it under Docker's data directory. If you `docker run` an image you don't have, Docker auto-pulls first. If you already have it, Docker uses the cache — no network call.

**Q15. What does "detached mode" mean?**

Detached (`-d`) means the container runs in the background. You get the container ID back immediately instead of a stream of logs. Use it for long-running services (our backend, frontend, MySQL). Use foreground mode for short debugging runs where you want to see output live.

### Architecture Deep (Q16–Q30)

**Q16. What is the Docker Daemon?**

`dockerd` is the background process that manages all Docker objects — images, containers, networks, volumes. The Docker CLI doesn't do any real work; it just sends REST API calls to the daemon over a Unix socket (`/var/run/docker.sock` on Linux) or a named pipe (on Windows). The daemon then orchestrates containerd and runc.

**Q17. What is containerd?**

containerd is a high-level container runtime, extracted from Docker into its own CNCF project. It handles image management (pull/push), container lifecycle (create/start/stop/delete), and low-level storage/networking. Docker delegates almost all real container work to containerd. Kubernetes uses containerd directly (since 1.24 deprecated the Docker shim).

**Q18. What is runc?**

runc is the low-level OCI runtime — the program that actually creates the container by calling Linux kernel APIs (namespaces, cgroups). It's a reference implementation of the OCI runtime spec. containerd invokes runc for every container it starts. runc is written in Go and is tiny (~10MB).

**Q19. What is the OCI?**

The Open Container Initiative is a Linux Foundation project that defines open standards for container formats and runtimes. Two main specs: OCI image spec (layers, manifests, digests) and OCI runtime spec (how to run a filesystem bundle). Because Docker, Podman, and Kubernetes all follow OCI, images built with Docker run on any OCI-compliant runtime.

**Q20. What are Linux namespaces?**

Namespaces are a kernel feature that isolates system resources. Each container gets its own namespaces for PIDs, network, mount points, hostname, IPC, and user IDs. Inside the container, `ps` shows only the container's processes; `ifconfig` shows only its network interfaces. This is what makes a container feel like its own machine.

**Q21. What are cgroups?**

Control groups are a kernel feature that limits and measures resource usage per process group. Docker uses cgroups to cap a container's CPU (`--cpus=0.5`), memory (`--memory=512m`), block I/O, and network bandwidth. If a container exceeds its memory limit, the kernel OOM-kills it. Without cgroups, one hungry container could starve the host.

**Q22. What is a union filesystem?**

A union filesystem stacks multiple directories and presents them as one logical filesystem. Docker uses OverlayFS (default on modern Linux) to overlay the image's read-only layers with a thin writable top layer per container. Reads see the merged view; writes go only to the top layer (copy-on-write). That's why images are cached and containers are cheap.

**Q23. What is a Docker layer?**

An image layer is the result of one Dockerfile instruction — a diff against the previous layer. `COPY app.jar /app/` adds a layer containing the JAR. Layers are immutable and content-addressed (identified by SHA256 hash). Multiple images can share layers, saving disk and bandwidth.

**Q24. How does Docker's build cache work?**

Docker caches each layer by the hash of its instruction + its inputs. When rebuilding, if a layer's inputs haven't changed, Docker reuses the cached layer and skips re-executing. Caching is sequential: once one layer is invalidated, all later layers must rebuild. That's why we `COPY pom.xml` before `COPY src` — dependencies rarely change, so the slow `mvn dependency:go-offline` layer stays cached.

**Q25. What is the default Docker network?**

The default network is called `bridge` (`docker0` interface on the host). All containers attach to it if you don't specify another. BUT — the default bridge does NOT provide DNS-based service discovery. You have to create a custom bridge (or let Compose do it) to get automatic hostname resolution between containers.

**Q26. What is the difference between bridge and host networks?**

`bridge` gives the container its own network namespace with a private IP on a virtual bridge. `host` removes network isolation — the container shares the host's network stack, using the host's ports and interfaces directly. Host mode is faster (no NAT) but has zero isolation and can conflict with host ports. Use bridge unless you have a specific reason.

**Q27. What is `docker0`?**

`docker0` is the default bridge network interface that Docker creates on the host. It's a virtual switch at 172.17.0.1 by default. Containers on the default bridge get IPs like 172.17.0.2, 172.17.0.3, etc. The host can reach containers via this bridge; containers reach the host via 172.17.0.1.

**Q28. How does inter-container DNS work?**

On a user-defined bridge network (like our `bus-net`), Docker runs an embedded DNS server at 127.0.0.11 inside each container. When your Spring Boot backend asks for `mysql`, the DNS server returns the MySQL container's IP. This is service discovery built in. The default bridge does not have this feature.

**Q29. What is the lifecycle of a container?**

Created → Running → (optionally Paused) → Stopped/Exited → Removed. `docker create` produces Created. `docker start` moves to Running. `docker pause` freezes to Paused. Process exit or `docker stop` moves to Exited. `docker rm` erases the container's writable layer and metadata. The image is untouched.

**Q30. What happens when a container is deleted?**

The container's writable layer (the thin top layer) is deleted from disk, and its metadata is removed from Docker. Any anonymous volumes created inline are orphaned (they become dangling). Named volumes and bind mounts survive — data in them is safe. The image the container was based on is not touched.

### Dockerfile Technical (Q31–Q40)

**Q31. What is the difference between CMD and ENTRYPOINT?**

`CMD` sets a default command that is easily overridden on `docker run`. `ENTRYPOINT` sets the always-run command; anything after the image name becomes its arguments. Best practice for a single-purpose container like our Spring Boot app is to use `ENTRYPOINT ["java","-jar","/app/app.jar"]` so the process is always the JVM. You can combine them — `ENTRYPOINT` for the binary, `CMD` for default args.

**Q32. What is the difference between ARG and ENV?**

`ARG` is a build-time variable available only during `docker build` — gone after the image is built. `ENV` is a runtime environment variable available inside the running container. Use `ARG` for build parameters (like a version number to pass to `curl`), `ENV` for runtime config (like `SPRING_PROFILES_ACTIVE=docker`).

**Q33. Why prefer COPY over ADD?**

`COPY` has a single clear purpose: copy files into the image. `ADD` does the same but also supports URL fetching and tar auto-extraction — behaviors that can surprise you. Docker's own best-practice guide recommends `COPY` unless you specifically need `ADD`'s extras. It's the principle of least astonishment.

**Q34. What are multi-stage builds and why use them?**

A multi-stage build has multiple `FROM` statements. Each stage can have its own base image; later stages can `COPY --from=<stage>` files from earlier ones. Our backend uses a Maven+JDK stage to build the JAR, then a JRE-Alpine stage to run it. Benefit: the final image has no build tools, no source code — just the JRE and the JAR. Smaller (180MB vs 800MB), faster to push, smaller attack surface.

**Q35. Why use Alpine-based images?**

Alpine is a minimal Linux distro (~5MB) built on musl libc and BusyBox. Images based on Alpine are tiny — our `eclipse-temurin:21-jre-alpine` is ~180MB versus ~280MB for the Debian variant. Smaller means faster pulls, less attack surface, and fewer CVEs. Trade-off: musl isn't glibc, so some native libraries (e.g., certain JNI libs) may have issues. For pure-Java Spring Boot, Alpine is fine.

**Q36. Why are there usually two `java` commands — one for build, one to run?**

A JDK (Java Development Kit) has the compiler (`javac`), JAR tools, debugging tools — ~350MB+. A JRE (Java Runtime Environment) has only what you need to *run* compiled code — ~150MB. We need the JDK to compile (stage 1) but only the JRE to run (stage 2). Using the JRE in the final stage shaves hundreds of MB.

**Q37. What is a distroless image?**

A distroless image (popularized by Google) contains only your application and its runtime dependencies — no shell, no package manager, no debugging tools. Examples: `gcr.io/distroless/java21`. Pros: tiny, highly secure. Cons: you can't `docker exec` a shell to debug. Many enterprises use distroless for production Java apps.

**Q38. What does `WORKDIR` do?**

`WORKDIR /app` creates the directory `/app` if it doesn't exist and sets it as the working directory for all subsequent instructions (`RUN`, `COPY`, `CMD`, etc.). It's equivalent to `cd` in shell, but persistent across layers. Better than `RUN cd /app && ...` which doesn't persist.

**Q39. What's the best way to avoid cache-busting during rebuilds?**

Order your Dockerfile so that rarely-changing things come first and frequently-changing things come last. For a Maven project: copy `pom.xml`, download dependencies, then copy `src/` and build. Source edits won't invalidate the dependency layer. Same idea for Node.js: copy `package.json`, `npm install`, then copy source.

**Q40. What is `HEALTHCHECK` and why use it?**

`HEALTHCHECK` tells Docker how to check if your app is actually healthy (not just running). Docker runs the check periodically; the container's status shows "healthy" or "unhealthy." Compose's `depends_on: condition: service_healthy` uses this to sequence startup. Example: `HEALTHCHECK CMD wget --spider -q http://localhost:8080/actuator/health || exit 1`. Without it, `depends_on` only waits for the process to start, not for the app to be ready.

### Networking & Volumes (Q41–Q50)

**Q41. What is the difference between a bind mount and a volume?**

A bind mount maps a specific host path into the container (`/home/sarthak/code:/app`). A named volume is Docker-managed storage (`mysql-data:/var/lib/mysql`). Bind mounts are great for development (live code editing); volumes are better for data you don't want to touch directly (databases). Volumes are portable across hosts via `docker volume` commands; bind mounts depend on specific host paths.

**Q42. How do you persist MySQL data across container restarts?**

Mount a named volume at `/var/lib/mysql` (where MySQL stores its data files). In our compose file: `volumes: - mysql-data:/var/lib/mysql`. When the container is removed and recreated, it reattaches to the volume and finds all the data. Without this mount, restarting MySQL wipes the database.

**Q43. What happens to volumes when you `docker-compose down`?**

`docker compose down` stops containers and removes them plus the network, but *keeps* named volumes. Data is safe. `docker compose down -v` (with `-v`) *also* removes volumes — use only when you want to reset everything. In our case, `down -v` would delete all bookings, so we avoid it unless we genuinely want a clean slate.

**Q44. What is `host.docker.internal`?**

A magic DNS name provided by Docker Desktop (Win/Mac) that resolves, from inside a container, to the host machine's IP. Useful when your container wants to talk to a service running on your laptop (like a MySQL installed natively). On Linux, you need `--add-host=host.docker.internal:host-gateway` to get the same behavior.

**Q45. Why doesn't the default bridge network give service name DNS?**

Historical reasons. The legacy default bridge dates from Docker's early days; user-defined bridges were added later with DNS built in. Best practice: always create a custom network (or let Compose do it). In our project, Compose auto-creates `bus-net`, and service names (`mysql`, `backend`, `frontend`) become DNS names.

**Q46. What is port mapping? What does `-p 8080:8080` mean?**

Port mapping publishes a container port to a host port. Syntax: `-p HOST_PORT:CONTAINER_PORT`. `-p 8080:8080` means "whatever reaches the host on TCP 8080 is forwarded to the container on TCP 8080." You can remap: `-p 18080:8080` exposes it on host 18080. Without port mapping, the container is reachable only from other containers on its network.

**Q47. What's the difference between a named volume and an anonymous volume?**

Both are Docker-managed storage. A named volume has a chosen name (`mysql-data`) — you can reference, back up, and reuse it. An anonymous volume gets an auto-generated hex name — it still works, but you can't easily find or manage it, and it's a common source of leaked data. Prefer named volumes in production and Compose files.

**Q48. What is `tmpfs` in Docker?**

A tmpfs mount is a memory-backed filesystem inside the container — nothing is written to disk. Syntax: `--tmpfs /tmp:size=64m`. Use cases: storing secrets you don't want on disk, high-speed scratch space for builds. Everything in a tmpfs vanishes when the container stops.

**Q49. How do you inspect container networking?**

`docker network ls` lists networks. `docker network inspect bus-net` shows every container on the network with its IP. `docker inspect <container>` shows the container's networks, IP, and port mappings. Inside a container, `ip addr` (if installed) or `cat /etc/hosts` shows network state.

**Q50. How do containers communicate in Compose?**

All services in a compose file share a default network (named `<project>_default` or whatever you define). Each service is reachable by its name as a DNS hostname. Backend reaches MySQL via `jdbc:mysql://mysql:3306/busticketbooking`; frontend reaches backend via `http://backend:8080`. No IPs, no hosts file edits, just names.

### Compose & Production (Q51–Q60)

**Q51. What is Docker Compose vs Docker Swarm vs Kubernetes?**

Compose runs multi-container apps on one host (great for dev and small prod). Swarm extends Compose across multiple hosts with basic orchestration (but it's effectively deprecated). Kubernetes is a full-featured orchestrator for large clusters — self-healing, rolling updates, autoscaling, ingress, secrets, etc. For our project Compose is ideal; for a national-scale production, Kubernetes.

**Q52. What does `depends_on` actually do?**

Without a condition, `depends_on` only controls *startup order* — ensures the dependency container is created and its main process is running, but not that the app inside is ready. With `condition: service_healthy`, Compose waits for the dependency's healthcheck to pass before starting the dependent. Our backend depends on mysql being healthy; frontend depends on backend being healthy.

**Q53. What is a restart policy?**

A restart policy tells Docker what to do if a container exits. Options: `no` (never restart, default), `always` (restart no matter what), `on-failure` (restart only on non-zero exit), `unless-stopped` (always restart unless *you* explicitly stop it). We use `unless-stopped` — handles crashes and reboots, but respects manual `docker stop`.

**Q54. How do you pass secrets to containers?**

Options in increasing security: environment variables (easy but visible in `docker inspect`), mounted files (secrets as files, e.g., `/run/secrets/db_password`), Docker Swarm secrets or Kubernetes secrets (proper encrypted storage), external secret stores (AWS Secrets Manager, HashiCorp Vault). For our project, we pass `DB_PASSWORD` via `.env` → environment var. Never commit `.env` to Git.

**Q55. What is a healthcheck in Compose?**

A healthcheck is a command Docker runs periodically inside the container to determine health. In our MySQL service:
```yaml
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-p$$MYSQL_ROOT_PASSWORD"]
  interval: 10s
  retries: 10
```
Docker runs this every 10 seconds. If it fails 10 times, the container is marked unhealthy. Healthchecks power `depends_on: condition: service_healthy`.

**Q56. What logging drivers does Docker support?**

Docker supports multiple logging drivers: `json-file` (default, local file), `journald` (systemd), `syslog`, `fluentd`, `gelf`, `awslogs`, `splunk`. You configure per-container: `--log-driver=awslogs --log-opt awslogs-group=bus-ticket`. In prod we'd ship to CloudWatch Logs. In dev, json-file is fine; `docker logs` reads from it.

**Q57. How do you limit a container's resources?**

```yaml
services:
  backend:
    mem_limit: 512m
    cpus: 0.5
```
Or on the command line: `--memory=512m --cpus=0.5`. Without limits, one container can starve others. Our backend probably fits in 384–512MB. MySQL needs more (~1GB for production-size data).

**Q58. What is the difference between `docker compose up` and `docker compose run`?**

`docker compose up` starts all services defined in the file and keeps them running. `docker compose run <service> <command>` starts a *one-off* container for the given service with an overridden command — useful for running migrations or admin tasks without touching the main stack (`docker compose run backend ./mvnw flyway:migrate`).

**Q59. How do you scale a service in Compose?**

`docker compose up -d --scale backend=3` starts 3 instances of the backend. Each gets a different internal IP on the network; all are reachable by the service name (DNS returns multiple IPs — round-robin). You'd still need a reverse proxy (nginx, HAProxy, Traefik) to load-balance external traffic across them. For true scaling at scale, move to Kubernetes.

**Q60. What is a reverse proxy, and why would you use one?**

A reverse proxy (nginx, Traefik, HAProxy) sits in front of your app and routes HTTP traffic. Benefits: TLS termination, load balancing across replicas, compression, caching, virtual hosts (one proxy serving many apps by Host header). For our project you could put Nginx in front and route `/api/*` to backend, `/` to frontend. We'll cover this in the AWS sections.

### Connected / Jumbling (Q61–Q70)

**Q61. Why did we choose multi-stage builds for the Spring Boot services?**

Because the image that *builds* the JAR (needs JDK + Maven + source code + dependency cache, ~800MB) is very different from the image that *runs* the JAR (needs only a JRE, ~180MB). Multi-stage lets us use the heavy toolchain during build and ship only the lean runtime. Pushing 180MB to ECR is much faster than 800MB, and the smaller surface area is more secure.

**Q62. Why alpine? Why not use `eclipse-temurin:21-jre` without alpine?**

The Alpine variant is ~180MB; the default (Ubuntu-based) is ~280MB. For pure-Java apps like Spring Boot, there's no downside — Java runs fine on musl libc. Only when you depend on native libraries compiled for glibc do you hit trouble. Since we're using JDBC (pure Java MySQL driver) and no JNI, Alpine is a clear win.

**Q63. Why do we map `8081:8081` for the frontend but not `8080:8080` for the backend in production?**

We expose the frontend because users (browsers) need to reach it. We don't need to expose the backend publicly because only the frontend (inside the Docker network) talks to it — `http://backend:8080` works internally. Publishing fewer ports reduces attack surface. In our dev compose we do expose 8080 for direct API testing (Postman), but in production we'd remove that mapping.

**Q64. Why does the backend container use the service name `mysql` and not an IP?**

Because IPs inside Docker networks are not guaranteed to stay the same across restarts. Service names, on the other hand, are stable DNS entries managed by Docker's embedded DNS. `jdbc:mysql://mysql:3306/...` works regardless of what IP MySQL has today. This is called service discovery, and it's one of the biggest quality-of-life wins of Docker networking.

**Q65. Why does the frontend wait for the backend to be healthy?**

Because if the frontend starts before the backend is accepting HTTP requests, the first user request will get a 500 error (connection refused or timeout). With `depends_on: backend: condition: service_healthy`, Compose waits until the backend's `/actuator/health` returns 200 before starting the frontend. End result: by the time your browser loads the home page, every request downstream will succeed.

**Q66. What would break if we forgot the MySQL volume?**

On first startup, MySQL would create the `busticketbooking` schema inside the container's ephemeral writable layer. You'd add a few bookings. Then you run `docker compose down && up`. The old container is gone, including its writable layer. The new MySQL container starts with a fresh, empty data directory. All bookings lost. Volumes exist precisely to prevent this.

**Q67. Why do we set `SPRING_PROFILES_ACTIVE=docker`?**

So Spring Boot loads `application-docker.properties` instead of the default. That file contains Docker-specific settings: MySQL host = `mysql` (service name), port = 3306, credentials from env vars. Running outside Docker (e.g., `java -jar` directly), you'd use the default profile with `localhost`. Profiles keep per-environment config clean and version-controlled.

**Q68. If the backend is slow to start, what breaks and how do we fix it?**

Without a proper healthcheck, the frontend's `depends_on` thinks the backend is "ready" the moment its JVM process is running — but Spring Boot still needs 15–45 seconds to initialize. The frontend fires its first request and gets refused. Fix: define a `HEALTHCHECK` on the backend Dockerfile (or in compose) that hits `/actuator/health`, with a `start_period: 60s` grace window. Compose then waits for the health status to flip to healthy.

**Q69. If I run `docker compose up` and it fails at build step, what's the first thing I check?**

The build output. Docker prints the exact failing step ("Step 5/12 : RUN mvn clean package ..."). Next, the stack trace. Usual suspects: network issue during Maven download (fix: retry), missing `pom.xml` (fix: check build context path), broken test (fix: we already pass `-DskipTests`), out-of-memory for Maven (fix: raise Docker Desktop's memory limit to 4GB+).

**Q70. Why is this whole Docker setup better than running all three apps directly on my laptop?**

One command to start the whole stack. Zero "install MySQL" pain for teammates. Identical behavior on my laptop, your laptop, and the submission VM. MySQL is contained — no config drift with other projects. When the project is done I can `docker compose down -v` and my laptop is clean. And most importantly: the same image that ran on my laptop is what we'll push to AWS ECR and run in production, with confidence that it behaves the same. That's the real prize — reproducibility from dev to prod.

---

## Wrap-Up of Section A

At this point you should be able to:

- Explain what Docker is, using the shipping container or tiffin analogy
- Draw the architecture: CLI → daemon → containerd → runc → kernel
- Write a multi-stage Dockerfile for a Spring Boot app
- Write a `docker-compose.yml` with healthchecks, depends_on, named volumes, and a custom network
- Run the full bus-ticket stack with `docker compose up --build -d`
- Debug common issues: port conflicts, missing volumes, DNS between containers, cache busting
- Answer any of the 70 viva questions above cold

Next up in Section B, we'll take these same images and push them to AWS ECR, then run them on EC2 — the first step from "local Docker" to "production cloud." But the Docker knowledge you just built is the foundation for everything else: ECS, EKS, Fargate, App Runner — all of them just run Docker images with different orchestration layers on top.

Keep the muscle memory: `docker build`, `docker run`, `docker compose up`, `docker logs`. You'll use them every day for the rest of your career.

### Quick Reference Card (Print This)

```
BUILD
  docker build -t <name>:<tag> .          # build image from Dockerfile in cwd
  docker build --no-cache -t <name> .     # ignore cache
  docker pull <image>                     # download from registry
  docker push <image>                     # upload to registry
  docker tag <old> <new>                  # re-tag an image

RUN
  docker run <image>                      # foreground, attached
  docker run -d --name <n> <image>        # background
  docker run -p 8080:8080 <image>         # publish port
  docker run -e KEY=VAL <image>           # env var
  docker run -v vol:/path <image>         # mount volume
  docker run --rm -it <image> sh          # one-shot shell
  docker exec -it <container> sh          # shell into running container

INSPECT
  docker ps                               # running containers
  docker ps -a                            # all, including exited
  docker images                           # local images
  docker logs -f <container>              # follow logs
  docker inspect <container>              # full JSON details
  docker stats                            # live CPU/mem per container

LIFECYCLE
  docker start <container>                # start stopped container
  docker stop <container>                 # graceful stop (SIGTERM)
  docker restart <container>              # stop + start
  docker kill <container>                 # immediate SIGKILL
  docker rm <container>                   # delete stopped container
  docker rmi <image>                      # delete image

NETWORKS / VOLUMES
  docker network ls / inspect / create / rm
  docker volume ls / inspect / create / rm

COMPOSE
  docker compose up --build -d            # build + run in background
  docker compose down                     # stop + remove (keep volumes)
  docker compose down -v                  # also remove volumes (destructive)
  docker compose logs -f <service>        # follow service logs
  docker compose exec <service> sh        # shell into service
  docker compose ps                       # list project services
  docker compose restart <service>

HOUSEKEEPING
  docker system df                        # disk usage
  docker system prune                     # clean unused objects
  docker system prune -a --volumes        # aggressive clean (careful!)
  docker builder prune                    # clean build cache
```

Pin this to your wall. You'll stop Googling basic commands within a week.

End of Section A.
# SECTION B — KUBERNETES (K8s)

> "Docker gave us containers. Kubernetes gave us a city where containers live, work, heal, and scale — automatically."

Welcome to Section B! In Section A we packaged our Bus Ticket Booking System into Docker images. Now we graduate to the big leagues — running those images in production-grade orchestration using **Kubernetes (K8s)**.

By the end of this section, you'll be able to:
- Explain Kubernetes architecture with confidence in a viva
- Deploy the full Bus Ticket stack (Backend + Frontend + MySQL) on Minikube
- Debug the 20 most common K8s errors your project will throw
- Answer 70+ interview-style questions

Let's begin.

---

## Part B1: What is Kubernetes? Why does it exist?

### B1.1 The One-Line Definition

**Kubernetes is an open-source system for automating the deployment, scaling, and management of containerized applications.**

That's the textbook answer. Now let's make it stick.

### B1.2 The Orchestra Analogy (Remember This for Viva!)

Imagine a symphony orchestra:

| Orchestra                     | Kubernetes                        |
|-------------------------------|-----------------------------------|
| Conductor                     | Kubernetes Control Plane          |
| Musicians                     | Containers                        |
| Section Leader (violin lead)  | Kubelet (one per node)            |
| Sheet Music                   | YAML manifests (desired state)    |
| Stage                         | Cluster                           |
| Section of stage              | Node                              |
| Music stand holding papers    | Pod (holds one or more musicians) |
| Replacement musician on call  | ReplicaSet (keeps N playing)      |
| Audience-facing speaker       | Service (how outsiders hear music)|

The conductor doesn't play an instrument. The conductor reads the score (your YAML) and signals: "I want 3 violins playing this passage at tempo 120." If one violinist faints, the section leader (kubelet) immediately signals the understudy (ReplicaSet) to pick up — **without the conductor having to do anything**.

That is Kubernetes in a nutshell: **declare what you want, and the system makes it happen — and keeps it happening**.

### B1.3 Why Docker Alone Isn't Enough at Scale

You already know Docker. So why do we need another layer?

Let me paint a picture. Imagine your Bus Ticket app has gone viral. Thousands of users hitting `/api/trips` every second.

**Problem 1: What if a container crashes at 3 AM?**
- Docker: Your app goes down. You wake up. You run `docker start`.
- Kubernetes: It detects the crash in seconds and auto-restarts. You keep sleeping.

**Problem 2: What if one VM can't handle the load?**
- Docker: You manually `docker run` on a second server. Nightmare to coordinate.
- Kubernetes: `kubectl scale --replicas=10`. Done. Pods spread across nodes.

**Problem 3: You deployed a buggy version. Revenue is bleeding.**
- Docker: Stop old, pull old, start old. Downtime.
- Kubernetes: `kubectl rollout undo`. Zero-downtime rollback.

**Problem 4: Frontend needs to find Backend's IP.**
- Docker: Hard-code IPs? Use docker-compose networks? Breaks when containers move.
- Kubernetes: Services. Frontend calls `http://bus-ticket-backend:8080` — K8s resolves it magically via CoreDNS.

**Problem 5: You want zero-downtime updates.**
- Docker: Not trivial.
- Kubernetes: Rolling updates built-in. Old pods die only after new pods are ready.

**Problem 6: Traffic load balancing.**
- Docker: You set up nginx manually.
- Kubernetes: Services distribute traffic automatically across pod replicas.

### B1.4 A Quick History (Great for Viva Intro Questions)

- **2003**: Google builds **Borg** internally to manage billions of containers across its data centers.
- **2014**: Google open-sources a cleaner version called **Kubernetes** (Greek for "helmsman" — the person steering the ship). Written in Go.
- **2015**: Google donates K8s to the newly-formed **CNCF (Cloud Native Computing Foundation)**.
- **2017**: K8s wins the container orchestration war against Docker Swarm and Mesos.
- **Today**: Runs on every major cloud (EKS on AWS, GKE on Google, AKS on Azure). Industry standard.

> **Fun Fact:** The "8" in K8s stands for the 8 letters between "K" and "s" in "Kubernetes". So it's "K-eight-s", not "K-ates".

### B1.5 Kubernetes vs Alternatives (Viva Bait!)

| Feature              | Kubernetes            | Docker Swarm       | HashiCorp Nomad     |
|----------------------|-----------------------|--------------------|---------------------|
| Complexity           | High                  | Low                | Medium              |
| Learning curve       | Steep                 | Gentle             | Moderate            |
| Ecosystem            | Massive (Helm, Istio) | Small              | Decent              |
| Stateful workloads   | StatefulSet, PVs      | Limited            | Good                |
| Rolling updates      | Advanced strategies   | Basic              | Good                |
| Non-container jobs   | No                    | No                 | Yes (any binary)    |
| Managed offerings    | Everywhere            | Rare               | HashiCorp only      |
| Job market           | Huge                  | Shrinking          | Niche               |

**Verdict**: Swarm is simpler but has lost the war. Nomad is elegant but niche. K8s is where the jobs are.

### B1.6 Desired State — The Zen of Kubernetes

This is **the** most important concept in K8s. Internalize it.

**Imperative style** (what you did with Docker):
```bash
docker run mysql
docker run backend
docker run frontend
# If backend dies, you manually run it again.
```
You tell the system *how* to do things, step by step.

**Declarative style** (Kubernetes):
```yaml
# I want 3 backend pods running, always.
replicas: 3
```
You tell the system *what* you want. It figures out *how*.

K8s runs an eternal **control loop**:
```
while (true):
    current_state = observe_cluster()
    if current_state != desired_state:
        take_action_to_reconcile()
    sleep(a few seconds)
```

Your YAML is the **desired state**. K8s is the reconciler. That's the whole game.

### B1.7 Core Terminology (Memorize These!)

| Term           | One-Line Meaning                                                  |
|----------------|-------------------------------------------------------------------|
| **Cluster**    | A group of machines running K8s, acting as one                    |
| **Node**       | One machine (VM or physical) in the cluster                       |
| **Pod**        | The smallest deployable unit — wraps 1 (or a few) containers      |
| **Deployment** | Controller that manages a set of replica pods                     |
| **ReplicaSet** | Ensures N pod copies are running (Deployment uses this under hood)|
| **Service**    | A stable network endpoint for a set of pods                       |
| **Ingress**    | HTTP/HTTPS routing from outside into the cluster                  |
| **ConfigMap**  | Non-secret configuration (URLs, feature flags, properties)        |
| **Secret**     | Sensitive data (passwords, API keys) stored base64-encoded        |
| **Namespace**  | Logical partition of the cluster (like folders for resources)     |
| **Controller** | Anything that watches desired state and reconciles actual state   |
| **Manifest**   | A YAML file that declares a resource                              |
| **kubectl**    | The CLI tool to talk to the cluster                               |

---

## Part B2: Kubernetes Architecture Deep Dive

Now we peek under the hood. This section is **viva gold** — examiners love architecture questions.

### B2.1 The Two Halves of a Cluster

A Kubernetes cluster is split into:

1. **Control Plane** (the brain) — makes decisions
2. **Worker Nodes** (the muscle) — runs your actual workloads

```
+---------------------------------------------------------------+
|                        KUBERNETES CLUSTER                      |
+---------------------------------------------------------------+
|                                                                |
|   +-------------------- CONTROL PLANE ------------------+      |
|   |                                                     |      |
|   |   +------------+   +----------+   +--------------+  |      |
|   |   | kube-api   |<->|   etcd   |   | scheduler    |  |      |
|   |   | -server    |   | (data)   |   |              |  |      |
|   |   +-----^------+   +----------+   +------+-------+  |      |
|   |         |                                 |         |      |
|   |   +-----+--------+   +--------------------+-----+   |      |
|   |   | controller-  |   | cloud-controller-        |   |      |
|   |   | manager      |   | manager                  |   |      |
|   |   +--------------+   +--------------------------+   |      |
|   +---------------------+-------------------------------+      |
|                         |                                      |
|   +---------- WORKER NODE 1 ----------+   +-- WORKER NODE 2 --+|
|   |  +---------+   +-----------+      |   |  ...              ||
|   |  | kubelet |   | kube-proxy|      |   |                   ||
|   |  +----+----+   +-----+-----+      |   |                   ||
|   |       |              |            |   |                   ||
|   |  +----v-------+ +---v---+         |   |                   ||
|   |  | container  | | Pods  |         |   |                   ||
|   |  | runtime    | | (app) |         |   |                   ||
|   |  | (containerd)| |       |         |   |                   ||
|   |  +------------+ +-------+         |   |                   ||
|   +-----------------------------------+   +-------------------+|
+---------------------------------------------------------------+
```

### B2.2 Control Plane Components Explained

#### 1. kube-apiserver — The Front Desk

- The **ONLY** component anyone talks to. All roads lead through the API server.
- Exposes the Kubernetes REST API on port 6443.
- `kubectl` talks to it. Controllers talk to it. Kubelets talk to it.
- Stateless — you can run many copies behind a load balancer for HA.

**Analogy**: The receptionist at a hotel. Nobody deals with the kitchen, housekeeping, or maintenance directly — you tell reception what you want, and they relay it.

#### 2. etcd — The Cluster's Memory

- A **distributed key-value store** that holds **all cluster state**.
- Written in Go by CoreOS (now Red Hat).
- Uses the **Raft consensus algorithm** for strong consistency.
- Stores: every pod spec, every ConfigMap, every secret, every node's health.

**If etcd dies, your cluster forgets everything.** That's why production clusters run etcd with 3 or 5 replicas.

**Analogy**: The hotel's reservation book. Lose it and chaos ensues.

#### 3. kube-scheduler — The Placement Expert

- Watches for new pods that haven't been assigned to a node yet.
- For each unscheduled pod, it picks the best node based on:
  - Available CPU and memory
  - Taints and tolerations
  - Node affinity / pod affinity rules
  - Data locality
- It does **NOT** start the pod. It just writes "pod X → node Y" back to etcd.
- Then the kubelet on node Y notices and starts it.

**Two-phase algorithm**:
1. **Filtering**: Remove nodes that can't run the pod (not enough RAM, wrong label, etc.)
2. **Scoring**: Rank remaining nodes (1-100) based on fit
3. Pick the highest scorer.

#### 4. kube-controller-manager — The Reconciler Army

- Runs **many controllers** in one binary:
  - **Node controller** — notices when a node dies
  - **ReplicaSet controller** — keeps N pods running
  - **Deployment controller** — manages rolling updates
  - **Endpoint controller** — populates Service → Pod mappings
  - **ServiceAccount controller** — creates default SAs
- Each controller runs the same pattern: watch → compare → act.

#### 5. cloud-controller-manager — The Cloud Liaison

- Integrates with cloud providers (AWS, GCP, Azure).
- Manages:
  - LoadBalancers (provisions real AWS ELBs, GCP LBs, etc.)
  - Nodes (talks to EC2 API to detect when a VM is terminated)
  - Routes (sets up VPC routing for pod traffic)
- If you're on bare metal or Minikube, you don't need this.

### B2.3 Worker Node Components

#### 1. kubelet — The Node's Agent

- One **kubelet** runs on every worker node.
- Reads the pod specs assigned to its node.
- Talks to the **container runtime** to start/stop containers.
- Reports node health back to the API server.

**Analogy**: The section leader in our orchestra — takes orders from the conductor and makes sure musicians in their section play correctly.

#### 2. kube-proxy — The Network Plumber

- Runs on every node.
- Implements **Service abstraction** by setting up iptables (or IPVS) rules.
- When a packet arrives for a Service IP, kube-proxy routes it to one of the backing pods.

**Two modes**:
- **iptables mode** (default) — Linux kernel rules, simple but slow with >5000 services
- **IPVS mode** — Uses Linux IPVS, much faster at large scale, better load-balancing algorithms

#### 3. Container Runtime

- The software that actually starts containers.
- Original: Docker Engine.
- Since K8s 1.24: Docker support removed. Now uses:
  - **containerd** (most common, extracted from Docker)
  - **CRI-O** (built specifically for K8s)
  - **Kata Containers** (VM-based isolation)
- They all speak the **CRI (Container Runtime Interface)** — a gRPC API the kubelet uses.

### B2.4 The Life of a Pod — End-to-End Flow

**Viva favorite: "Walk me through what happens when I run `kubectl apply -f pod.yaml`."**

```
You                                                         etcd
 |                                                            ^
 | kubectl apply -f pod.yaml                                  |
 v                                                            |
+-------------+                                               |
| kube-       |   validates, authenticates, writes to etcd    |
| apiserver   |-----------------------------------------------+
+------+------+
       |
       | (watches for unscheduled pods)
       v
+-------------+   picks best node, writes "nodeName" back
| scheduler   |-----> apiserver -----> etcd
+-------------+
       |
       |
       v
+-------------+   kubelet on chosen node is watching
| kubelet     |   sees pod assigned to it
+------+------+
       |
       | calls CRI
       v
+-------------+
| containerd  |   pulls image, creates container
+------+------+
       |
       v
+-------------+
| Your app    |   running!
+-------------+
```

Step-by-step:
1. `kubectl` sends the YAML to **kube-apiserver**.
2. API server **authenticates** (who are you?), **authorizes** (can you do this?), and **validates** (is the YAML well-formed?).
3. API server writes the pod object to **etcd**. Status: `Pending`.
4. **Scheduler** is always watching for `Pending` pods. It picks a node.
5. Scheduler updates the pod object with `spec.nodeName = worker-2`.
6. **Kubelet on worker-2** sees this and pulls the image via the container runtime.
7. Kubelet creates the pod sandbox (network, storage), then starts containers.
8. Kubelet updates pod status to `Running`. Written back to etcd.
9. You see `kubectl get pods` show `Running`.

### B2.5 Networking — The Magic That Makes It Work

Kubernetes networking has three fundamental rules:

1. **Every pod gets its own IP address.** No NAT between pods.
2. **Pods can talk to all other pods** on any node without NAT.
3. **Agents on a node** (kubelet, system daemons) can talk to all pods on that node.

#### CNI (Container Network Interface)

K8s doesn't implement networking itself. It delegates to **CNI plugins**.

| Plugin    | Strength                               | Use Case                          |
|-----------|----------------------------------------|-----------------------------------|
| Flannel   | Simplest, easiest                      | Small clusters, labs              |
| Calico    | Network policies, BGP routing          | Enterprise, security-conscious    |
| Cilium    | eBPF-based, super fast, observability  | Modern, high-performance          |
| Weave Net | Encryption by default                  | Dev/test environments             |

#### The IP Ranges

| Range              | Example           | Purpose                     |
|--------------------|-------------------|-----------------------------|
| Node IP            | 192.168.1.10      | The physical/VM network     |
| Pod CIDR           | 10.244.0.0/16     | IPs for pods                |
| Service CIDR       | 10.96.0.0/12      | Virtual IPs for Services    |
| ClusterDNS         | 10.96.0.10        | CoreDNS lives here          |

### B2.6 DNS — How `http://bus-ticket-backend:8080` Works

Every cluster runs **CoreDNS** as a deployment in the `kube-system` namespace.

When you create a Service named `bus-ticket-backend` in namespace `default`, CoreDNS auto-registers:
- `bus-ticket-backend.default.svc.cluster.local` → the Service's ClusterIP
- Short name `bus-ticket-backend` (within same namespace)

When your frontend pod does a DNS lookup for `bus-ticket-backend`, it hits CoreDNS (10.96.0.10), gets the Service IP, then kube-proxy on the node forwards packets to a real backend pod.

**This is why you never hard-code IPs.** Services give you stable names. Pods come and go.

### B2.7 Storage — PersistentVolumes

Pods are **ephemeral** — if a pod dies, data in its container filesystem is **gone**.

For stateful data (like MySQL), you need persistent storage.

```
+----------+      +------+      +--------------+      +---------------+
|   Pod    |----->| PVC  |----->| PV (backed   |----->| Real storage  |
| (MySQL)  |      |      |      | by cloud or  |      | (EBS, NFS,    |
|          |      |      |      | local disk)  |      | hostPath)     |
+----------+      +------+      +--------------+      +---------------+
```

- **PersistentVolume (PV)** — a chunk of storage, cluster-wide resource.
- **PersistentVolumeClaim (PVC)** — a pod's request: "I need 10Gi of ReadWriteOnce storage."
- **StorageClass** — a template that dynamically creates PVs (so you don't hand-craft them).

### B2.8 RBAC — Role-Based Access Control

Who can do what in the cluster?

- **ServiceAccount** — an identity for pods (like a user, but for apps)
- **Role** — a list of allowed actions in a namespace ("can list pods in default")
- **ClusterRole** — same but cluster-wide
- **RoleBinding / ClusterRoleBinding** — attaches a Role to a user/SA

**Principle of least privilege**: Every pod should have just enough permissions, nothing more.

---

## Part B3: Core Resources — Types and Sub-Types

This is the **cookbook**. Bookmark this section.

### B3.1 Pod — The Atom of Kubernetes

**What**: The smallest deployable unit. Wraps one (or rarely, a few) containers that share network and storage.

**When to use directly**: Almost never. Use Deployments instead. Pods created directly don't get restarted if they die.

**Sample YAML**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bus-ticket-backend-pod
  labels:
    app: bus-ticket-backend
spec:
  containers:
  - name: backend
    image: bus-ticket-backend:latest
    ports:
    - containerPort: 8080
    env:
    - name: DB_URL
      value: "jdbc:mysql://mysql:3306/busticketbooking"
```

**Why multi-container?** Sidecar pattern:
- Main container runs your app
- Sidecar tails logs, syncs files, or acts as a proxy (e.g., Istio sidecar)

### B3.2 ReplicaSet — The Baby-sitter

**What**: Ensures N copies of a pod are always running.

**When to use directly**: Almost never. Deployments create ReplicaSets automatically.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: backend-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: bus-ticket-backend
  template:
    metadata:
      labels:
        app: bus-ticket-backend
    spec:
      containers:
      - name: backend
        image: bus-ticket-backend:latest
```

### B3.3 Deployment — The Workhorse

**What**: Manages ReplicaSets. Gives you rolling updates, rollbacks, and history.

**When to use**: For all **stateless** apps (backend, frontend).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bus-ticket-backend
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1       # can create 1 extra pod during update
      maxUnavailable: 0 # don't take any pod down during update
  selector:
    matchLabels:
      app: bus-ticket-backend
  template:
    metadata:
      labels:
        app: bus-ticket-backend
    spec:
      containers:
      - name: backend
        image: bus-ticket-backend:latest
```

**Rolling update magic**: Deploy a new image, K8s:
1. Creates one new pod with the new image.
2. Waits for it to pass readiness probe.
3. Kills one old pod.
4. Repeat until all pods are new.

Zero downtime — if your probes are set right.

### B3.4 StatefulSet — For Databases

**What**: Like Deployment, but with stable identity.

**When to use**: MySQL, PostgreSQL, Kafka, Zookeeper — anything that needs:
- **Stable pod names**: `mysql-0`, `mysql-1` (not random hashes)
- **Stable storage**: each pod gets its own PVC that sticks with it
- **Ordered startup/shutdown**: mysql-0 starts first, then mysql-1

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
```

### B3.5 DaemonSet — One Pod Per Node

**What**: Runs a pod on every node (or a subset).

**When to use**: Log collectors (Fluentd), monitoring agents (Prometheus node exporter), network plugins.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd:v1.16
```

### B3.6 Job and CronJob — Batch Work

**Job**: Run a pod to completion, once.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: bus-ticket-backend:latest
        command: ["java", "-jar", "app.jar", "--migrate"]
      restartPolicy: OnFailure
```

**CronJob**: Run a Job on a schedule (cron syntax).

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-report
spec:
  schedule: "0 2 * * *"   # 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: report
            image: bus-ticket-backend:latest
            command: ["java", "-jar", "app.jar", "--generate-report"]
          restartPolicy: OnFailure
```

### B3.7 Services — The Network Abstraction

Pods are ephemeral; IPs change. **Services** give you a stable endpoint.

#### ClusterIP (Default)

**What**: A virtual IP reachable **only inside the cluster**.
**Use**: Internal service-to-service communication (frontend → backend).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: bus-ticket-backend
spec:
  type: ClusterIP
  selector:
    app: bus-ticket-backend
  ports:
  - port: 8080         # Service port
    targetPort: 8080   # Container port
```

#### NodePort

**What**: Exposes the service on a **static port on every node**.
**Use**: Quick external access for dev/testing.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: bus-ticket-frontend
spec:
  type: NodePort
  selector:
    app: bus-ticket-frontend
  ports:
  - port: 8081
    targetPort: 8081
    nodePort: 30081   # between 30000-32767
```

Access: `http://<any-node-ip>:30081`

#### LoadBalancer

**What**: Provisions a **real cloud load balancer** (AWS ELB, GCP LB).
**Use**: Production external access on the cloud.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: bus-ticket-frontend
spec:
  type: LoadBalancer
  selector:
    app: bus-ticket-frontend
  ports:
  - port: 80
    targetPort: 8081
```

#### ExternalName

**What**: A DNS CNAME alias.
**Use**: Reference an external DB like `mydb.rds.amazonaws.com` as `mysql` inside the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-mysql
spec:
  type: ExternalName
  externalName: mydb.rds.amazonaws.com
```

#### Headless Service

**What**: Service with `clusterIP: None`. No virtual IP — returns pod IPs directly via DNS.
**Use**: StatefulSets (MySQL) where clients need stable pod-level addressing.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306
```

Gives you: `mysql-0.mysql-headless.default.svc.cluster.local`

### B3.8 Ingress — HTTP Smart Routing

**What**: L7 (HTTP/HTTPS) routing, host-based and path-based.

**Why**: NodePort and LoadBalancer give you one IP per service. Ingress lets **many services share one IP/LB**, routed by URL.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bus-ticket-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: bus-ticket.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: bus-ticket-frontend
            port:
              number: 8081
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: bus-ticket-backend
            port:
              number: 8080
```

Requires an **Ingress Controller** (nginx-ingress, Traefik, HAProxy) installed in the cluster.

### B3.9 ConfigMap — Non-Secret Config

**What**: Key-value pairs for config data.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
data:
  DB_URL: "jdbc:mysql://mysql:3306/busticketbooking"
  DB_USERNAME: "busadmin"
  HIKARI_POOL_SIZE: "10"
  LOG_LEVEL: "INFO"
```

Consume in pod:
```yaml
envFrom:
- configMapRef:
    name: backend-config
```

### B3.10 Secret — Sensitive Data

**What**: Like ConfigMap but base64-encoded. For passwords, tokens.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: backend-secret
type: Opaque
data:
  DB_PASSWORD: YnVzMTIzNDU=    # base64 of "bus12345"
```

Create from command:
```bash
kubectl create secret generic backend-secret \
  --from-literal=DB_PASSWORD=bus12345
```

> **Warning**: Kubernetes Secrets are **base64-encoded, NOT encrypted by default**! Anyone with `get secret` access can decode. For real security use:
> - **Sealed Secrets** (Bitnami)
> - **External Secrets Operator** (pulls from AWS Secrets Manager, Vault)
> - **Encryption at rest** for etcd (kube-apiserver `--encryption-provider-config`)

### B3.11 Namespace — Logical Isolation

**What**: Virtual clusters within a cluster.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: bus-ticket-prod
```

Usage:
```bash
kubectl apply -f deployment.yaml -n bus-ticket-prod
kubectl get pods -n bus-ticket-prod
```

Default namespaces:
- `default` — where your stuff lands if you don't specify
- `kube-system` — K8s internal components (CoreDNS, etc.)
- `kube-public` — world-readable metadata
- `kube-node-lease` — node heartbeats

### B3.12 PersistentVolume and PersistentVolumeClaim

**PV** (cluster admin creates or dynamic):
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data/mysql
```

**PVC** (your pod requests):
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

**Access modes**:
| Mode             | Abbreviation | Meaning                                  |
|------------------|--------------|------------------------------------------|
| ReadWriteOnce    | RWO          | One node can mount read-write            |
| ReadOnlyMany     | ROX          | Many nodes can mount read-only           |
| ReadWriteMany    | RWX          | Many nodes can mount read-write (NFS)    |
| ReadWriteOncePod | RWOP         | One pod only (K8s 1.27+)                 |

### B3.13 HorizontalPodAutoscaler

**What**: Auto-scales pod count based on CPU/memory/custom metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: bus-ticket-backend
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

Requires **metrics-server** installed in the cluster.

---

## Part B4: Install Kubernetes Locally

Time to get hands dirty. You'll install a local K8s cluster on your machine.

### B4.1 Options for Local K8s

| Tool              | How it runs                        | Pros                         | Cons                       |
|-------------------|------------------------------------|------------------------------|----------------------------|
| **Minikube**      | VM (or Docker) running K8s         | Full featured, easy          | Slower startup             |
| **kind**          | K8s inside Docker containers       | Very fast, multi-node easy   | Less stable for long runs  |
| **k3d**           | k3s (lightweight) in Docker        | Super lightweight            | k3s ≠ full K8s             |
| **Docker Desktop**| Built-in K8s toggle                | One click                    | Single node only           |

For this guide we'll use **Minikube** — it's the most feature-complete for students.

### B4.2 Requirements (Viva Q!)

- 2+ CPU cores
- 4+ GB RAM (8 GB recommended)
- 20+ GB free disk
- Virtualization enabled in BIOS (for VM driver) OR Docker installed (for docker driver)

### B4.3 Install `kubectl`

**kubectl** is the Swiss Army knife for talking to any K8s cluster.

**Windows (PowerShell)**:
```bash
curl.exe -LO "https://dl.k8s.io/release/v1.30.0/bin/windows/amd64/kubectl.exe"
# Add kubectl.exe to PATH
```

**macOS**:
```bash
brew install kubectl
```

**Linux**:
```bash
curl -LO "https://dl.k8s.io/release/v1.30.0/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

**Verify**:
```bash
kubectl version --client
```

### B4.4 Install Minikube

**Windows**:
```bash
choco install minikube
```
or download the installer from minikube.sigs.k8s.io.

**macOS**:
```bash
brew install minikube
```

**Linux**:
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

### B4.5 Start Minikube

```bash
minikube start --driver=docker --memory=4096 --cpus=2
```

This:
1. Pulls a K8s node image.
2. Starts a Docker container acting as a node.
3. Sets up kubectl context.

### B4.6 Verify the Cluster

```bash
kubectl cluster-info
```
Expected:
```
Kubernetes control plane is running at https://127.0.0.1:xxxx
CoreDNS is running at ...
```

```bash
kubectl get nodes
```
Expected:
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   1m    v1.30.0
```

```bash
kubectl get pods -A
```
Shows all system pods (CoreDNS, etcd, scheduler, etc.)

### B4.7 Useful Minikube Commands

```bash
minikube dashboard            # Open K8s web UI
minikube ip                   # Get cluster IP
minikube ssh                  # Shell into the node
minikube stop                 # Pause cluster
minikube delete               # Nuke cluster
minikube addons list          # See available addons
minikube addons enable ingress
minikube addons enable metrics-server
```

### B4.8 CRITICAL: Make Minikube See Your Docker Images

If you built `bus-ticket-backend:latest` on your host Docker, Minikube's inner Docker can't see it!

**Fix (run this every time you open a new terminal)**:

**Linux/Mac**:
```bash
eval $(minikube docker-env)
```

**Windows PowerShell**:
```bash
minikube docker-env | Invoke-Expression
```

**Windows Git Bash**:
```bash
eval $(minikube -p minikube docker-env --shell bash)
```

Now when you `docker build -t bus-ticket-backend:latest .`, Minikube can see it.

**Alternative**: Load an existing image:
```bash
minikube image load bus-ticket-backend:latest
minikube image load bus-ticket-frontend:latest
```

---

## Part B5: Deploy Backend Step-by-Step

Now the main event. We deploy the `bus-ticket-backend` Spring Boot 3.3.5 app.

### B5.1 Plan

We'll create 4 resources:
1. `ConfigMap` — DB URL, username, pool size
2. `Secret` — DB password
3. `Deployment` — 2 replicas of the backend
4. `Service` (ClusterIP) — internal endpoint on port 8080

### B5.2 Create a Namespace (Optional but Good Practice)

```yaml
# file: namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: bus-ticket
```

```bash
kubectl apply -f namespace.yaml
kubectl config set-context --current --namespace=bus-ticket
```

### B5.3 Secret for DB Password

```yaml
# file: backend-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: backend-secret
  namespace: bus-ticket
type: Opaque
stringData:        # stringData lets us type plain text; K8s base64s for us
  DB_PASSWORD: "bus12345"
```

Apply:
```bash
kubectl apply -f backend-secret.yaml
kubectl get secret backend-secret -o yaml
```

> **Tip**: `stringData` = plain text (auto-encoded). `data` = already base64-encoded. Don't mix them up.

### B5.4 ConfigMap for Non-Secret Env

```yaml
# file: backend-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: bus-ticket
data:
  SPRING_DATASOURCE_URL: "jdbc:mysql://mysql:3306/busticketbooking?useSSL=false&allowPublicKeyRetrieval=true"
  SPRING_DATASOURCE_USERNAME: "root"
  SPRING_JPA_HIBERNATE_DDL_AUTO: "update"
  SPRING_JPA_SHOW_SQL: "false"
  SERVER_PORT: "8080"
  SPRING_DATASOURCE_HIKARI_MAXIMUM_POOL_SIZE: "10"
  SPRING_PROFILES_ACTIVE: "prod"
```

Apply:
```bash
kubectl apply -f backend-configmap.yaml
```

### B5.5 The Backend Deployment (Heart of This Section)

```yaml
# file: backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bus-ticket-backend
  namespace: bus-ticket
  labels:
    app: bus-ticket-backend
    tier: backend
spec:
  replicas: 2                          # run 2 copies for HA
  revisionHistoryLimit: 5              # keep last 5 rollout versions
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: bus-ticket-backend
  template:
    metadata:
      labels:
        app: bus-ticket-backend
        tier: backend
    spec:
      containers:
      - name: backend
        image: bus-ticket-backend:latest
        imagePullPolicy: IfNotPresent    # use local image first (Minikube!)
        ports:
        - name: http
          containerPort: 8080
        envFrom:
        - configMapRef:
            name: backend-config
        - secretRef:
            name: backend-secret
        resources:
          requests:
            cpu: "250m"          # 0.25 CPU core guaranteed
            memory: "512Mi"
          limits:
            cpu: "1000m"         # max 1 CPU
            memory: "1Gi"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        startupProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          failureThreshold: 30
          periodSeconds: 10
```

### B5.6 Line-by-Line Explanation

| Line                                | Meaning                                                          |
|-------------------------------------|------------------------------------------------------------------|
| `apiVersion: apps/v1`               | Which K8s API group the resource belongs to                      |
| `kind: Deployment`                  | The type of resource                                             |
| `metadata.name`                     | Unique name within the namespace                                 |
| `metadata.labels`                   | Tags used for selection/grouping                                 |
| `spec.replicas: 2`                  | Desired pod count                                                |
| `strategy.type: RollingUpdate`      | Default update strategy (vs `Recreate`)                          |
| `maxSurge: 1`                       | Can add 1 pod above desired during update                        |
| `maxUnavailable: 0`                 | Never go below desired count                                     |
| `selector.matchLabels`              | How the Deployment finds "its" pods                              |
| `template`                          | The pod spec for every replica                                   |
| `imagePullPolicy: IfNotPresent`     | Don't re-pull if image already on node (required for Minikube)   |
| `envFrom`                           | Bulk-import env vars from ConfigMap/Secret                       |
| `resources.requests`                | Minimum guaranteed — scheduler uses this to place pod            |
| `resources.limits`                  | Hard cap — exceeding memory → OOMKilled                          |
| `livenessProbe`                     | "Is the app hung?" — if fails, K8s restarts the container        |
| `readinessProbe`                    | "Is the app ready for traffic?" — if fails, pod out of Service   |
| `startupProbe`                      | Gives slow-starting apps time before liveness kicks in           |

### B5.7 Why Three Probes? (Viva Gold!)

| Probe     | Answers                              | If fails                      |
|-----------|--------------------------------------|-------------------------------|
| Startup   | "Has the app finished booting?"      | Keep waiting (no restart)     |
| Readiness | "Can it serve requests right now?"   | Remove from Service endpoints |
| Liveness  | "Is the app alive or deadlocked?"    | Restart container             |

Spring Boot takes ~30s to start. Without a `startupProbe`, the liveness probe fires after 30s and kills a perfectly-booting app. Classic restart loop.

### B5.8 Backend Service

```yaml
# file: backend-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: bus-ticket-backend
  namespace: bus-ticket
  labels:
    app: bus-ticket-backend
spec:
  type: ClusterIP
  selector:
    app: bus-ticket-backend            # matches Deployment's pod labels
  ports:
  - name: http
    port: 8080                         # Service port (what clients use)
    targetPort: 8080                   # Container port
    protocol: TCP
```

> **Why ClusterIP?** Only the frontend (another in-cluster service) needs to reach it. No reason to expose it to the outside world.

### B5.9 Apply Everything

```bash
kubectl apply -f backend-secret.yaml
kubectl apply -f backend-configmap.yaml
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml
```

Or in one shot:
```bash
kubectl apply -f ./backend/
```

### B5.10 Verify the Deployment

```bash
kubectl get deployments
# NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
# bus-ticket-backend    2/2     2            2           1m

kubectl get pods
# NAME                                  READY   STATUS    RESTARTS   AGE
# bus-ticket-backend-7d4b8f9c-abcde     1/1     Running   0          1m
# bus-ticket-backend-7d4b8f9c-fghij     1/1     Running   0          1m

kubectl get svc
# NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
# bus-ticket-backend    ClusterIP   10.96.120.45    <none>        8080/TCP
```

### B5.11 Debug Commands

```bash
# Tail logs of one pod
kubectl logs -f bus-ticket-backend-7d4b8f9c-abcde

# Tail logs of ALL pods with a label
kubectl logs -f -l app=bus-ticket-backend

# Describe — the best debugging command
kubectl describe pod bus-ticket-backend-7d4b8f9c-abcde

# Shell into a pod
kubectl exec -it bus-ticket-backend-7d4b8f9c-abcde -- /bin/sh
```

### B5.12 Test the Backend Locally via Port-Forward

Since the Service is `ClusterIP`, it's invisible from your laptop. Use port-forward:

```bash
kubectl port-forward svc/bus-ticket-backend 8080:8080
```

Now on your laptop:
```bash
curl http://localhost:8080/api/trips
```

You should see JSON! If not, debug.

### B5.13 Common First-Deploy Errors

**Error**: `ErrImagePull` or `ImagePullBackOff`
**Cause**: Minikube can't see your local image.
**Fix**:
```bash
eval $(minikube docker-env)
docker build -t bus-ticket-backend:latest .
# Or:
minikube image load bus-ticket-backend:latest
```

**Error**: `CrashLoopBackOff`, logs say "Connection refused to mysql:3306"
**Cause**: MySQL isn't deployed yet! Backend tries to connect on startup and fails.
**Fix**: Deploy MySQL first (next section B7) OR add init container.

**Error**: Pod stuck in `Pending`
**Cause**: Not enough CPU/memory in cluster.
**Fix**: Check with `kubectl describe pod <pod>` — the Events section tells you.

---

## Part B6: Deploy Frontend Step-by-Step

The frontend (Spring Boot Thymeleaf on port 8081) needs to reach the backend at `http://bus-ticket-backend:8080`.

### B6.1 The Frontend Deployment

```yaml
# file: frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bus-ticket-frontend
  namespace: bus-ticket
  labels:
    app: bus-ticket-frontend
    tier: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: bus-ticket-frontend
  template:
    metadata:
      labels:
        app: bus-ticket-frontend
        tier: frontend
    spec:
      containers:
      - name: frontend
        image: bus-ticket-frontend:latest
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 8081
        env:
        - name: BACKEND_BASE_URL
          value: "http://bus-ticket-backend:8080"
        - name: SERVER_PORT
          value: "8081"
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        resources:
          requests:
            cpu: "200m"
            memory: "384Mi"
          limits:
            cpu: "500m"
            memory: "768Mi"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8081
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8081
          initialDelaySeconds: 30
          periodSeconds: 5
```

### B6.2 The DNS Magic Explained

The line that deserves a neon highlight:
```yaml
value: "http://bus-ticket-backend:8080"
```

**How does `bus-ticket-backend` resolve?**

1. CoreDNS runs in the `kube-system` namespace.
2. Every pod's `/etc/resolv.conf` is auto-configured to use CoreDNS (10.96.0.10).
3. When any Service is created, CoreDNS auto-registers its name.
4. A DNS query for `bus-ticket-backend` (within the same namespace) returns the Service's ClusterIP.
5. kube-proxy intercepts traffic to that ClusterIP and load-balances to a real pod.

**Full DNS name**: `bus-ticket-backend.bus-ticket.svc.cluster.local`
- `bus-ticket-backend` — the Service name
- `bus-ticket` — the namespace
- `svc.cluster.local` — the cluster DNS zone

You can use just the short name **within the same namespace**. Across namespaces, use `backend-svc.other-ns`.

### B6.3 Frontend Service

```yaml
# file: frontend-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: bus-ticket-frontend
  namespace: bus-ticket
spec:
  type: NodePort             # exposed externally for dev
  selector:
    app: bus-ticket-frontend
  ports:
  - name: http
    port: 8081
    targetPort: 8081
    nodePort: 30081          # fixed port on every node
```

### B6.4 Apply and Access

```bash
kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml

kubectl get pods -l app=bus-ticket-frontend
```

Access via Minikube:
```bash
minikube service bus-ticket-frontend --url
# http://192.168.49.2:30081
```

Or port-forward:
```bash
kubectl port-forward svc/bus-ticket-frontend 8081:8081
# then browse http://localhost:8081
```

### B6.5 Quick Sanity Test

Shell into the frontend pod and test the backend connection:

```bash
kubectl exec -it <frontend-pod-name> -- /bin/sh

# Inside the container:
wget -O- http://bus-ticket-backend:8080/api/trips
```

If that works, DNS and routing are golden.

---

## Part B7: Deploy MySQL with StatefulSet

MySQL is **stateful**. If its data disappears on every pod restart, your bookings vanish. So we use `StatefulSet` + `PersistentVolumeClaim`.

### B7.1 Why StatefulSet Not Deployment?

| Feature                   | Deployment        | StatefulSet                              |
|---------------------------|-------------------|------------------------------------------|
| Pod names                 | Random hash       | Ordered: `mysql-0`, `mysql-1`            |
| Storage                   | Shared or none    | One PVC per pod (via volumeClaimTemplates)|
| Startup order             | Parallel          | Sequential (0 before 1)                  |
| Network ID                | Changes           | Stable DNS: `mysql-0.mysql-headless`     |
| Scale down order          | Any               | Reverse order                            |

**For MySQL**: we absolutely need persistent per-pod storage and stable identity.

### B7.2 Headless Service for MySQL

```yaml
# file: mysql-headless-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
  namespace: bus-ticket
  labels:
    app: mysql
spec:
  clusterIP: None          # <--- makes it headless
  selector:
    app: mysql
  ports:
  - port: 3306
    name: mysql
```

### B7.3 Regular Service for App Access

Apps connect to `mysql:3306` — simple ClusterIP:

```yaml
# file: mysql-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: bus-ticket
spec:
  type: ClusterIP
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
```

### B7.4 Secret for MySQL Root Password

```yaml
# file: mysql-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: bus-ticket
type: Opaque
stringData:
  MYSQL_ROOT_PASSWORD: "bus12345"
  MYSQL_DATABASE: "busticketbooking"
```

### B7.5 The StatefulSet

```yaml
# file: mysql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: bus-ticket
spec:
  serviceName: mysql-headless     # must match the headless service
  replicas: 1                      # single instance for now
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - name: mysql
          containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_ROOT_PASSWORD
        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_DATABASE
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "1500m"
            memory: "2Gi"
        livenessProbe:
          exec:
            command:
            - mysqladmin
            - ping
            - -h
            - localhost
            - -u
            - root
            - -p$(MYSQL_ROOT_PASSWORD)
          initialDelaySeconds: 60
          periodSeconds: 20
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command:
            - mysql
            - -h
            - localhost
            - -u
            - root
            - -p$(MYSQL_ROOT_PASSWORD)
            - -e
            - "SELECT 1"
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
```

### B7.6 Key Concepts

**`serviceName: mysql-headless`** — This is **mandatory** for a StatefulSet. It links the SS to its governing Service.

**`volumeClaimTemplates`** — This is where the magic happens. Each pod gets its OWN PVC named `mysql-data-mysql-0`, `mysql-data-mysql-1`, etc. The PVC sticks with the pod even across restarts.

**DNS naming**:
- Pod: `mysql-0.mysql-headless.bus-ticket.svc.cluster.local`
- If you scale to 3: `mysql-0`, `mysql-1`, `mysql-2`

### B7.7 Handling App-DB Startup Race

When you apply everything, backend pods might start before MySQL is ready → `CrashLoopBackOff`.

**Option 1: Init Container** (waits for MySQL)

```yaml
spec:
  initContainers:
  - name: wait-for-mysql
    image: busybox:1.36
    command:
    - sh
    - -c
    - |
      until nc -z mysql 3306; do
        echo "Waiting for mysql..."
        sleep 2
      done
  containers:
  - name: backend
    # ... main container
```

**Option 2**: Let Spring Boot retry (built-in HikariCP connection retry). Liveness/readiness probes handle the rest. This is the cleaner way for Spring Boot apps.

### B7.8 Verify MySQL

```bash
kubectl apply -f mysql-secret.yaml
kubectl apply -f mysql-headless-svc.yaml
kubectl apply -f mysql-service.yaml
kubectl apply -f mysql-statefulset.yaml

kubectl get pods -l app=mysql
# NAME      READY   STATUS    RESTARTS   AGE
# mysql-0   1/1     Running   0          45s

kubectl get pvc
# NAME                  STATUS   VOLUME       CAPACITY   ACCESS MODES
# mysql-data-mysql-0    Bound    pvc-xxx...   5Gi        RWO
```

### B7.9 Shell Into MySQL

```bash
kubectl exec -it mysql-0 -- mysql -u root -p
# password: bus12345

mysql> SHOW DATABASES;
mysql> USE busticketbooking;
mysql> SHOW TABLES;
```

### B7.10 Deletion Caveat!

Deleting the StatefulSet does **NOT** delete the PVCs (data preserved):
```bash
kubectl delete statefulset mysql
# PVC mysql-data-mysql-0 still exists
```

To nuke data:
```bash
kubectl delete pvc mysql-data-mysql-0
```

### B7.11 Production Recommendation

For real production, **don't run databases in Kubernetes unless your team has deep K8s expertise**. Use managed services:
- **AWS RDS** (MySQL/Postgres)
- **GCP Cloud SQL**
- **Azure Database for MySQL**

Then use an `ExternalName` Service:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  type: ExternalName
  externalName: mydb.cabcdef.us-east-1.rds.amazonaws.com
```

Your app code still hits `mysql:3306`. DB concerns outsourced to AWS. Win-win.

---

## Part B8: Ingress + External Access

Port-forwarding and NodePort are fine for dev. For a real deployment, you want:
- A proper hostname like `bus-ticket.local`
- One single entry point for both frontend and backend
- Path-based routing (`/` → frontend, `/api` → backend, if needed)
- TLS/HTTPS

### B8.1 Enable the Ingress Addon in Minikube

```bash
minikube addons enable ingress
```

This installs the **nginx-ingress-controller** — a deployment of nginx that watches Ingress resources and configures itself.

Verify:
```bash
kubectl get pods -n ingress-nginx
```

### B8.2 Install Ingress via Helm (Production)

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

### B8.3 The Ingress Resource

```yaml
# file: bus-ticket-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bus-ticket-ingress
  namespace: bus-ticket
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  rules:
  - host: bus-ticket.local
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: bus-ticket-backend
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: bus-ticket-frontend
            port:
              number: 8081
```

### B8.4 Order Matters!

**Longer/more specific paths first!** If you put `/` before `/api`, everything goes to the frontend.

### B8.5 Edit `/etc/hosts` for Local Testing

Find Minikube IP:
```bash
minikube ip
# 192.168.49.2
```

**Linux/Mac**: edit `/etc/hosts`:
```
192.168.49.2   bus-ticket.local
```

**Windows**: edit `C:\Windows\System32\drivers\etc\hosts` as Administrator.

### B8.6 Apply and Test

```bash
kubectl apply -f bus-ticket-ingress.yaml
kubectl get ingress
# NAME                  CLASS   HOSTS              ADDRESS         PORTS
# bus-ticket-ingress    nginx   bus-ticket.local   192.168.49.2    80
```

Open browser: `http://bus-ticket.local`

### B8.7 TLS with cert-manager (Overview)

For HTTPS, install **cert-manager**:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml
```

Create a ClusterIssuer:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: ai@multigenesys.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

Add to Ingress:
```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - bus-ticket.example.com
    secretName: bus-ticket-tls
```

cert-manager auto-issues and renews Let's Encrypt certs.

### B8.8 Architecture After Ingress

```
                Internet / Your Laptop
                         |
                         v
              +----------+-----------+
              | bus-ticket.local     |
              +----------+-----------+
                         |
                         v
              +---------------------+
              | Ingress Controller  |   (nginx running as a pod)
              +---------+-----------+
                        |
         +--------------+----------------+
         | Path /api                     | Path /
         v                               v
+-------------------+          +-----------------------+
| bus-ticket-       |          | bus-ticket-frontend   |
| backend Service   |          | Service (ClusterIP)   |
+---------+---------+          +-----------+-----------+
          |                                |
          v                                v
   +------+------+                 +-------+-------+
   | backend pod |                 | frontend pod  |
   | x 2         |                 | x 2           |
   +-------------+                 +---------------+
          |
          v
    +-----+------+
    | mysql svc  |
    +-----+------+
          |
          v
    +-----+------+
    | mysql-0    | (StatefulSet)
    +------------+
```

---

## Part B9: Common Kubernetes Problems + Solutions

**This section will save your viva and your deployment.** Read it twice.

### Master Troubleshooting Table

| # | Error                   | Symptom                                     | Root Cause                             | Fix                                                            |
|---|-------------------------|---------------------------------------------|----------------------------------------|----------------------------------------------------------------|
| 1 | ImagePullBackOff        | `STATUS: ImagePullBackOff`                  | Image name wrong, not in registry      | `kubectl describe pod` → check events. Use `minikube image load` |
| 2 | CrashLoopBackOff        | Pod restarting forever                      | App crashes at startup                 | `kubectl logs <pod>` — read the exception                      |
| 3 | Pending                 | Pod never starts                            | No node has enough CPU/mem             | Lower `resources.requests` or add nodes                        |
| 4 | ContainerCreating       | Stuck in this state >2 min                  | Volume/secret/configmap missing        | `kubectl describe pod` → check events                          |
| 5 | Service not reachable   | Curl fails                                  | Wrong selector labels                  | `kubectl get endpoints <svc>` — must list pod IPs              |
| 6 | Env var undefined       | App logs `null` for a value                 | ConfigMap/Secret key typo              | `kubectl describe configmap` — verify key names                |
| 7 | DNS lookup fails        | "unknown host: mysql"                       | CoreDNS crashed, wrong namespace       | `kubectl get pods -n kube-system` — check CoreDNS              |
| 8 | Data lost on restart    | DB empty after pod restart                  | Forgot PVC                             | Use StatefulSet with `volumeClaimTemplates`                    |
| 9 | OOMKilled               | Pod restarts with reason `OOMKilled`        | Memory limit too low                   | Raise `resources.limits.memory`                                |
| 10| Rolling update stuck    | Old pods not terminating                    | Readiness probe failing on new pods    | Check new pod logs; fix probe path/port                        |
| 11| kubectl connect refused | `connection refused 127.0.0.1:8080`         | Wrong context or kube-config           | `kubectl config get-contexts` and `use-context`                |
| 12| Minikube image not found| `Failed to pull image "..."`                | Built image outside Minikube's Docker  | `eval $(minikube docker-env)` OR `minikube image load`         |
| 13| Ingress 404             | Page shows nginx 404                        | Wrong path or host                     | Check `kubectl get ingress` and `curl -H "Host: ..."`          |
| 14| Secret "invalid base64" | `error: illegal base64 data`                | Used `data:` with plain text           | Use `stringData:` or base64-encode first                       |
| 15| HPA not scaling         | HPA stays at minReplicas                    | metrics-server not installed           | `minikube addons enable metrics-server`                        |
| 16| Liveness restart loop   | Pod restarting every ~30s                   | Liveness probe too aggressive          | Add `startupProbe` or raise `initialDelaySeconds`              |
| 17| Permission denied       | `cannot write to /data`                     | Container runs as non-root             | Set `securityContext.runAsUser: 0` OR fix volume permissions   |
| 18| Node out of disk        | Pods evicted with `DiskPressure`            | Disk full on node                      | `kubectl describe node <n>` — clean up images                 |
| 19| DB password not updated | Changed Secret but app still uses old       | Pods don't auto-reload env vars        | `kubectl rollout restart deployment/backend`                   |
| 20| Ingress not created     | `ingressClassName` errors                   | nginx controller not installed         | `minikube addons enable ingress`                               |
| 21| PVC stuck Pending       | PVC never becomes `Bound`                   | No StorageClass or no PV available     | Check `kubectl get sc`; in Minikube, use `standard`            |
| 22| Wrong namespace         | `No resources found`                        | You're in `default`, resources in other| `kubectl get pods -n bus-ticket`                               |

### B9.1 Deep Dive: ImagePullBackOff

**Check**:
```bash
kubectl describe pod <pod-name>
```

Events will show something like:
```
Failed to pull image "bus-ticket-backend:latest": rpc error: code = NotFound
```

**Fix options**:

1. For Minikube — load image directly:
   ```bash
   minikube image load bus-ticket-backend:latest
   ```

2. Or build inside Minikube's Docker:
   ```bash
   eval $(minikube docker-env)
   docker build -t bus-ticket-backend:latest .
   ```

3. For real clusters with private registries — create an `imagePullSecret`:
   ```bash
   kubectl create secret docker-registry regcred \
     --docker-server=https://index.docker.io/v1/ \
     --docker-username=myuser \
     --docker-password=mypass
   ```
   Reference in pod spec:
   ```yaml
   spec:
     imagePullSecrets:
     - name: regcred
   ```

### B9.2 Deep Dive: CrashLoopBackOff

**Diagnosis**:
```bash
kubectl logs <pod> --previous   # logs from the crashed instance
kubectl describe pod <pod>      # look at "Last State: Terminated"
```

**Common causes for Spring Boot apps**:
- DB unreachable at startup → fix DB Service name or wait for MySQL
- Port conflict → ensure container port matches Service targetPort
- OOMKilled → raise memory limits
- Missing env var → check ConfigMap/Secret mounted correctly
- Bad Java args → check JVM flags in ENTRYPOINT

### B9.3 Deep Dive: Service not Reachable

```bash
# Step 1: does the service have endpoints?
kubectl get endpoints bus-ticket-backend
# If EMPTY — selector doesn't match any pod labels!

# Step 2: check selector vs pod labels
kubectl get svc bus-ticket-backend -o yaml | grep -A2 selector
kubectl get pods --show-labels

# Step 3: test from inside the cluster
kubectl run tester --image=busybox --rm -it -- sh
  wget -O- http://bus-ticket-backend:8080/api/trips
```

### B9.4 Deep Dive: Rolling Update Stuck

You did `kubectl set image deployment/backend backend=bus-ticket-backend:v2`. Now `kubectl rollout status` hangs.

**Check**:
```bash
kubectl get pods
# NAME                              READY   STATUS
# bus-ticket-backend-old-xxx        1/1     Running
# bus-ticket-backend-new-yyy        0/1     Running    <- stuck
```

The new pod is `Running` but not `Ready` (0/1). Readiness probe is failing.

**Fix**:
```bash
kubectl logs bus-ticket-backend-new-yyy
```

Common: new version has a bug on the readiness endpoint. Either fix the bug or:
```bash
kubectl rollout undo deployment/bus-ticket-backend
```

### B9.5 Deep Dive: OOMKilled

```bash
kubectl describe pod <pod>
# State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137
```

**Fix**: Java apps need overhead beyond the heap!

```yaml
resources:
  limits:
    memory: "1Gi"       # container limit
env:
- name: JAVA_OPTS
  value: "-Xmx512m -Xms256m"   # heap only 50% of container limit
```

Rule of thumb: **JVM heap = 50-75% of container memory limit**.

### B9.6 Deep Dive: DNS Not Working

```bash
# Test from any pod
kubectl exec -it <pod> -- nslookup bus-ticket-backend
# Expected output:
#   Server: 10.96.0.10
#   Address: 10.96.0.10#53
#   Name: bus-ticket-backend.bus-ticket.svc.cluster.local
#   Address: 10.96.120.45
```

If it fails:
```bash
kubectl get pods -n kube-system | grep coredns
# coredns-xxx    1/1    Running
```

If CoreDNS is crashing:
```bash
kubectl logs -n kube-system coredns-xxx
kubectl rollout restart deployment/coredns -n kube-system
```

### B9.7 Deep Dive: PVC Stuck Pending

```bash
kubectl describe pvc mysql-data-mysql-0
# Events:
#   no persistent volumes available for this claim and no storage class is set
```

**Fix**:
```bash
kubectl get storageclass
# NAME                 PROVISIONER
# standard (default)   k8s.io/minikube-hostpath
```

If none, in Minikube enable `storage-provisioner`:
```bash
minikube addons enable storage-provisioner
minikube addons enable default-storageclass
```

### B9.8 Deep Dive: Config Change Not Propagating

You updated a ConfigMap. But app still uses old value.

**Why**: `envFrom` injects env vars **at pod startup only**. Changing the ConfigMap doesn't restart pods.

**Fix**:
```bash
kubectl rollout restart deployment/bus-ticket-backend
```

Advanced: add a hash annotation so pods auto-restart on ConfigMap change (done via Helm `checksum/config` or tools like Reloader).

### B9.9 Deep Dive: Liveness Probe Restart Loop

Spring Boot app starting. After 30s, liveness probe fires. App still warming up. Probe fails. K8s kills container. Repeat forever.

**Fix**: Use `startupProbe`:
```yaml
startupProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  failureThreshold: 30       # allow 30 * 10s = 5 min to start
  periodSeconds: 10
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  periodSeconds: 10
```

Startup probe runs **first**. Only when it passes once does liveness probe take over.

### B9.10 Debug Checklist (Memorize!)

When **anything** is wrong:

```bash
kubectl get pods                      # Status overview
kubectl describe pod <pod>            # Events, container state, probe results
kubectl logs <pod>                    # App logs
kubectl logs <pod> --previous         # Prior crash's logs
kubectl get events --sort-by=.lastTimestamp   # Cluster-wide events
kubectl exec -it <pod> -- sh          # Shell in
kubectl get svc, endpoints            # Service wiring
kubectl top pod                       # Resource usage (needs metrics-server)
```

---

## Part B10: Kubectl Cheat Sheet

The **40 commands** you'll actually use 99% of the time.

### Basic Info

| Command                                     | Purpose                                  |
|---------------------------------------------|------------------------------------------|
| `kubectl version`                           | Client + server version                  |
| `kubectl cluster-info`                      | Is the cluster reachable?                |
| `kubectl get nodes`                         | List all nodes                           |
| `kubectl get all`                           | Pods, services, deployments in current ns|
| `kubectl get all -A`                        | Same but all namespaces                  |
| `kubectl api-resources`                     | List every resource type                 |
| `kubectl explain pod.spec.containers`       | Docs for any field                       |

### Get / Describe / Logs / Exec

| Command                                     | Purpose                                  |
|---------------------------------------------|------------------------------------------|
| `kubectl get pods`                          | List pods in current namespace           |
| `kubectl get pods -o wide`                  | Include node and IP                      |
| `kubectl get pods -l app=backend`           | Filter by label                          |
| `kubectl get pod <p> -o yaml`               | Full YAML of a pod                       |
| `kubectl describe pod <p>`                  | Events + full status                     |
| `kubectl logs <pod>`                        | Tail logs                                |
| `kubectl logs <pod> -f`                     | Follow (stream)                          |
| `kubectl logs <pod> --previous`             | Logs from prior crash                    |
| `kubectl logs -l app=backend --tail=100`    | Logs across all matching pods            |
| `kubectl exec -it <pod> -- /bin/sh`         | Shell into a container                   |
| `kubectl cp <pod>:/path/file ./file`        | Copy file out of pod                     |

### Apply / Delete / Edit

| Command                                     | Purpose                                  |
|---------------------------------------------|------------------------------------------|
| `kubectl apply -f file.yaml`                | Create or update resource                |
| `kubectl apply -f ./folder/`                | Apply all YAMLs in a folder              |
| `kubectl delete -f file.yaml`               | Delete by manifest                       |
| `kubectl delete pod <pod>`                  | Delete a specific pod (it'll respawn!)   |
| `kubectl delete deploy <name>`              | Delete deployment                        |
| `kubectl edit deployment/<name>`            | Open in `$EDITOR`                        |

### Deployments / Rollouts / Scaling

| Command                                     | Purpose                                  |
|---------------------------------------------|------------------------------------------|
| `kubectl rollout status deployment/<n>`     | Wait for rollout                         |
| `kubectl rollout history deployment/<n>`    | Past revisions                           |
| `kubectl rollout undo deployment/<n>`       | Roll back                                |
| `kubectl rollout restart deployment/<n>`    | Restart all pods                         |
| `kubectl scale deploy/<n> --replicas=5`     | Scale manually                           |
| `kubectl set image deploy/<n> c=img:tag`    | Update image                             |

### Services / Port-forwarding

| Command                                     | Purpose                                  |
|---------------------------------------------|------------------------------------------|
| `kubectl get svc`                           | List services                            |
| `kubectl get endpoints`                     | Which pods back which service            |
| `kubectl port-forward svc/<s> 8080:8080`    | Forward port to localhost                |
| `kubectl port-forward pod/<p> 8080:8080`    | Forward to a specific pod                |

### Context / Namespaces

| Command                                     | Purpose                                  |
|---------------------------------------------|------------------------------------------|
| `kubectl config get-contexts`               | List available clusters                  |
| `kubectl config use-context <name>`         | Switch cluster                           |
| `kubectl config set-context --current --namespace=foo` | Default namespace          |
| `kubectl get ns`                            | List namespaces                          |
| `kubectl create ns foo`                     | Create namespace                         |

### Create (Imperative Shortcuts)

| Command                                     | Purpose                                  |
|---------------------------------------------|------------------------------------------|
| `kubectl create deploy app --image=nginx`   | Quick deployment                         |
| `kubectl create secret generic s --from-literal=k=v` | Quick secret                    |
| `kubectl create configmap cm --from-file=app.properties` | From file                   |
| `kubectl run tester --image=busybox --rm -it -- sh` | Disposable pod                   |
| `kubectl expose deploy/app --port=8080 --type=ClusterIP` | Create service             |

### Metrics / Top

| Command                                     | Purpose                                  |
|---------------------------------------------|------------------------------------------|
| `kubectl top pod`                           | CPU/mem per pod                          |
| `kubectl top node`                          | CPU/mem per node                         |

### Tips

- `kubectl get pods -w` — watch mode, refreshes on changes
- `kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'` — scripting
- Add alias: `alias k=kubectl`
- Autocomplete: `source <(kubectl completion bash)`

---

## Part B11: Kubernetes Viva Questions

70+ questions grouped by topic. Each includes a model answer and, where relevant, commands or project tie-ins. **Memorize this section before your viva**.

### Group 1 — Basics (Q1–Q15)

**Q1. What is Kubernetes?**
Kubernetes is an open-source container orchestration platform originally built by Google (inspired by Borg), now governed by the CNCF. It automates deployment, scaling, networking, and management of containerized applications across a cluster of machines. In our bus-ticket project, we use K8s to run backend, frontend, and MySQL containers with self-healing and load balancing.

**Q2. Why do we need Kubernetes when we already have Docker?**
Docker runs containers on a single host. Kubernetes manages containers across many hosts, handling failures, scaling, service discovery, and rolling updates — things Docker alone doesn't do. Without K8s, if our backend container dies at night, the app is down until someone manually restarts it.

**Q3. What is a Pod?**
A pod is the smallest deployable unit in Kubernetes. It's a wrapper around one (or sometimes a few) containers that share the same network namespace (same IP and ports) and storage volumes. In our project, each backend pod contains one Spring Boot container.

**Q4. Why a Pod instead of running the container directly?**
Pods add a layer of abstraction that lets K8s schedule, monitor, and heal a group of tightly-coupled containers as a unit. A pod can host a main app container plus sidecars (logging, proxy) that all share the same network — a container alone can't do that.

**Q5. What is the difference between a Pod and a Container?**
A container is a runtime unit (made from a Docker image). A pod is a K8s wrapper that can contain one or more containers and adds shared network/storage and lifecycle management. Pod = 1+ containers + networking + volumes.

**Q6. What is a Deployment?**
A Deployment is a higher-level controller that manages a set of identical pods via ReplicaSets. It gives you declarative updates, rolling updates, rollbacks, and scaling. We use a Deployment for our backend so K8s maintains 2 replicas at all times.

**Q7. What is the difference between Deployment and Pod?**
A Pod is one instance. A Deployment manages many Pods and keeps the desired count alive. If a pod dies, the Deployment's ReplicaSet creates a new one — a standalone pod would just stay dead.

**Q8. What is a ReplicaSet? Do I use it directly?**
A ReplicaSet ensures N pod copies run. You rarely use it directly — Deployments create and manage ReplicaSets for you (one per revision of your app). When you update an image, a new RS is born and the old one is scaled down.

**Q9. What is a Service?**
A Service is a stable network endpoint for a set of pods. Pods have ephemeral IPs, but a Service has a stable ClusterIP and DNS name. In our project, `bus-ticket-backend` Service lets the frontend call the backend without knowing specific pod IPs.

**Q10. What are the main Service types?**
- **ClusterIP** (default): internal only
- **NodePort**: exposes a port on every node (dev)
- **LoadBalancer**: provisions a cloud LB (production)
- **ExternalName**: DNS alias to an external hostname
- **Headless (`clusterIP: None`)**: returns pod IPs directly, for StatefulSets

**Q11. What is an Ingress?**
An Ingress is an L7 (HTTP/HTTPS) routing rule. It lets many services share one public IP, routed by hostname and path. We'd use `bus-ticket.local/` → frontend and `/api` → backend through one Ingress. Requires an Ingress Controller (e.g., nginx) to actually do the routing.

**Q12. What is a Namespace?**
A namespace is a logical partition within a cluster. Resources in different namespaces are isolated (different names, separate RBAC). We'd put our project in namespace `bus-ticket` to keep it separate from `default`.

**Q13. What is a ConfigMap?**
A ConfigMap stores non-sensitive key-value configuration (URLs, feature flags, properties). It's mounted into pods as env vars or files. We use one for `SPRING_DATASOURCE_URL`, pool sizes, etc.

**Q14. What is a Secret?**
A Secret stores sensitive data like passwords and tokens, base64-encoded. Keys are injected into pods as env vars or volumes. We store MySQL's `DB_PASSWORD` in a Secret. Note: it's only base64, not encryption — use Sealed Secrets or external KMS for real secrets.

**Q15. What's the difference between ConfigMap and Secret?**
Both store key-value data. ConfigMap is plain text; Secret is base64-encoded and treated differently (shown as `[hidden]` in logs, stored separately). Neither is truly encrypted at rest unless you configure etcd encryption.

### Group 2 — Architecture Deep (Q16–Q30)

**Q16. What are the components of the Kubernetes control plane?**
The control plane runs the cluster's brain:
- **kube-apiserver** — entry point for all API calls
- **etcd** — distributed key-value store holding cluster state
- **kube-scheduler** — assigns pods to nodes
- **kube-controller-manager** — runs controllers (ReplicaSet, Node, etc.)
- **cloud-controller-manager** — integrates with cloud providers

**Q17. What does the kube-apiserver do?**
It's the **only** component that talks to etcd. Every kubectl command, every controller, every kubelet goes through it. It authenticates, authorizes (RBAC), validates, and persists changes. You can run multiple replicas behind a load balancer for HA.

**Q18. What is etcd and why is it critical?**
etcd is a distributed, consistent key-value store built on the Raft algorithm. It holds every piece of cluster state — pods, services, secrets, everything. If etcd is corrupted or lost, the cluster forgets its desired state. Production clusters run etcd with 3 or 5 replicas and take regular backups.

**Q19. What does the scheduler do?**
The kube-scheduler watches for pods with no assigned node. For each, it runs a two-phase algorithm: first **filtering** nodes that can't host the pod (not enough CPU, wrong labels), then **scoring** the rest (1–100) based on fit. It picks the highest-scoring node and writes the assignment back through the API server.

**Q20. What does the controller-manager do?**
It runs many controllers in one binary — ReplicaSet controller, Deployment controller, Node controller, Endpoint controller, ServiceAccount controller, and more. Each runs the same loop: watch desired state → compare to actual → take action to reconcile.

**Q21. What does the kubelet do?**
Kubelet is the agent on every worker node. It talks to the API server to get pod assignments, then calls the container runtime (containerd) to start/stop containers. It also runs liveness/readiness probes and reports pod/node health back to the API server.

**Q22. What does kube-proxy do?**
kube-proxy runs on every node and implements Service networking. It programs iptables (or IPVS) rules so that traffic to a Service's ClusterIP gets forwarded to one of the backing pods, with built-in load balancing.

**Q23. What's the difference between iptables and IPVS mode in kube-proxy?**
iptables mode (default) is simple but becomes slow at >5000 services because each packet traverses many rules. IPVS mode uses Linux IPVS kernel feature — it's much faster at scale and supports sophisticated load-balancing algorithms (round-robin, least-connection, etc.).

**Q24. Walk me through what happens when I run `kubectl apply -f pod.yaml`.**
1. kubectl sends the YAML to the API server. 2. API server authenticates, authorizes, validates, and writes the pod (status `Pending`) to etcd. 3. The scheduler notices an unassigned pod, picks a node, writes `nodeName` back. 4. The kubelet on that node sees the assignment, pulls the image via containerd, starts the container, and updates status to `Running`.

**Q25. What is a container runtime?**
The software that actually runs containers (creates namespaces, cgroups, mounts layers). K8s uses **CRI (Container Runtime Interface)** to talk to any runtime. Common runtimes: containerd (most popular), CRI-O, and Kata Containers. Since K8s 1.24, Docker is no longer supported directly — but containerd (extracted from Docker) fills the gap.

**Q26. What is CNI?**
CNI (Container Network Interface) is a plugin standard for pod networking. K8s itself doesn't handle networking — it delegates to a CNI plugin like Flannel, Calico, or Cilium. The plugin assigns IPs to pods and sets up routing so pods on any node can reach each other without NAT.

**Q27. What are the three fundamental K8s networking rules?**
1. Every pod gets its own IP. 2. All pods can communicate with all other pods on any node without NAT. 3. Agents on a node can reach all pods on that node. These are enforced by the CNI plugin.

**Q28. What is CoreDNS?**
CoreDNS is the in-cluster DNS server, running as a Deployment in `kube-system`. It auto-registers Service names so pods can do DNS lookups like `bus-ticket-backend` and get the Service's ClusterIP. Without it, services couldn't discover each other by name.

**Q29. What's the full DNS name of our backend service?**
`bus-ticket-backend.bus-ticket.svc.cluster.local` — where `bus-ticket-backend` is the Service name, `bus-ticket` is the namespace, and `svc.cluster.local` is the cluster DNS zone. Within the same namespace, pods can just say `bus-ticket-backend`.

**Q30. What is RBAC?**
Role-Based Access Control defines who can do what in the cluster. You create **Roles** (permissions in a namespace) or **ClusterRoles** (cluster-wide), then bind them to users or ServiceAccounts via RoleBindings / ClusterRoleBindings. Principle of least privilege — give pods only the permissions they need.

### Group 3 — Workload Resources (Q31–Q45)

**Q31. Deployment vs StatefulSet — which one for what?**
Use a **Deployment** for stateless apps (our backend, frontend) — pods are interchangeable, random names, shared storage. Use a **StatefulSet** for stateful apps (our MySQL) — stable pod names (`mysql-0`), per-pod persistent storage, ordered startup/shutdown.

**Q32. Why did we use a StatefulSet for MySQL in the bus-ticket project?**
MySQL needs stable storage (data shouldn't vanish on restart), a stable network identity (for replication/clustering), and ordered startup. StatefulSets provide all three via `volumeClaimTemplates` and the headless service pattern. A Deployment would lose data on pod restart.

**Q33. What's a DaemonSet?**
A DaemonSet runs exactly one pod on every node (or a subset via node selectors). Used for per-node agents like log collectors (Fluentd), monitoring agents (Prometheus node exporter), or network plugins. Unlike Deployments, it doesn't care about replica count — it cares about node count.

**Q34. What's the difference between a Job and a CronJob?**
A **Job** runs a pod to completion once (good for migrations). A **CronJob** is a Job scheduled on cron syntax (e.g., `"0 2 * * *"` = daily 2 AM). We'd use a CronJob for nightly revenue reports or seat-cleanup scripts.

**Q35. What is a rolling update?**
A deployment strategy that replaces old pods with new ones gradually: start a new pod, wait for it to become Ready, kill an old pod, repeat. Controlled by `maxSurge` (extra pods above desired) and `maxUnavailable` (how many can be down). Guarantees zero downtime if readiness probes are correct.

**Q36. What's the Recreate strategy?**
An alternative to RollingUpdate: kill **all** old pods first, then create new ones. Causes downtime but useful when old and new versions can't coexist (e.g., DB schema changes). Specified with `strategy.type: Recreate`.

**Q37. How do you roll back a deployment?**
`kubectl rollout undo deployment/bus-ticket-backend` rolls back to the previous revision. You can also specify `--to-revision=3` for a specific past version. K8s keeps history (configurable via `revisionHistoryLimit`).

**Q38. What is a PodDisruptionBudget (PDB)?**
A PDB tells K8s "don't voluntarily evict more than X pods at once." Protects availability during node drains or upgrades. Example: `minAvailable: 1` for a 2-replica backend ensures at least one pod is always up during maintenance.

**Q39. What are liveness, readiness, and startup probes?**
- **Liveness**: "is the app alive?" Fail → restart container. Detects deadlocks.
- **Readiness**: "is the app ready for traffic?" Fail → remove from Service endpoints. No restart.
- **Startup**: "has the app finished starting?" Runs first; only once it passes do liveness/readiness take over. Essential for slow-booting apps like Spring Boot.

**Q40. What happens if I set a liveness probe too aggressive?**
Your pod enters a restart loop. Each time it starts, liveness fails before the app is ready, K8s kills it, restart counter climbs. This is a top-5 K8s pitfall. Fix: add a `startupProbe` or increase `initialDelaySeconds` on liveness.

**Q41. What's an HPA?**
HorizontalPodAutoscaler auto-scales pod replicas based on metrics (CPU, memory, or custom). Example: target 70% CPU, min 2 replicas, max 10. Requires metrics-server. When traffic spikes, HPA adds pods; when it calms, HPA removes them.

**Q42. What's a VPA?**
VerticalPodAutoscaler adjusts pod `requests`/`limits` automatically based on observed usage. Unlike HPA (which changes pod count), VPA changes pod size. Not installed by default.

**Q43. What are resource requests and limits?**
`requests` — guaranteed minimum; scheduler uses this to place pod on a node with enough free capacity. `limits` — hard cap; if a container exceeds memory limit, it's OOMKilled; if it exceeds CPU limit, it's throttled.

**Q44. What is QoS (Quality of Service) class?**
K8s assigns each pod a QoS class based on requests/limits:
- **Guaranteed**: requests == limits on all resources. Last to be evicted.
- **Burstable**: requests < limits. Evicted if node is under pressure.
- **BestEffort**: no requests or limits. First to be evicted.

**Q45. What's the difference between `kubectl apply` and `kubectl create`?**
`create` is imperative — fails if resource already exists. `apply` is declarative — creates if absent, updates (via 3-way merge) if present. Always use `apply` for production YAMLs. `create` is handy for quick tests.

### Group 4 — Networking and Storage (Q46–Q55)

**Q46. How does the frontend reach the backend in our project?**
Frontend pod sends HTTP to `http://bus-ticket-backend:8080`. DNS (CoreDNS) resolves `bus-ticket-backend` to the backend Service's ClusterIP. kube-proxy's iptables rules on the node intercept the packet and load-balance to one of the backend pods. The pod responds, packet returns. Full service discovery without any hard-coded IPs.

**Q47. Can the frontend talk to MySQL directly in our project?**
Technically yes (MySQL Service is reachable from any pod in the namespace), but architecturally no — the frontend should only talk to the backend. Enforcing that is a design decision; in K8s, you'd use a `NetworkPolicy` to block frontend → MySQL and allow only backend → MySQL.

**Q48. What's the difference between ClusterIP and NodePort?**
**ClusterIP**: internal virtual IP, only reachable within the cluster. Default type. **NodePort**: opens a port (30000–32767) on **every node**, so external traffic to `<any-node-ip>:<nodePort>` gets routed to the service. NodePort is really a ClusterIP + external port exposure.

**Q49. LoadBalancer vs Ingress — when to use which?**
**LoadBalancer** gives one public IP per Service — expensive on cloud (each is a real ELB/ALB). **Ingress** lets many services share one LB, with L7 routing (host/path). Use Ingress for HTTP apps to save money and get smart routing. Use LoadBalancer for non-HTTP (TCP, UDP) or when you want a dedicated IP.

**Q50. What's a PersistentVolume (PV)?**
A PV is a cluster resource representing a piece of storage — an AWS EBS volume, an NFS mount, a local disk. It's provisioned by an admin (static) or automatically via StorageClass (dynamic).

**Q51. What's a PersistentVolumeClaim (PVC)?**
A PVC is a pod's request for storage: "give me 5Gi of ReadWriteOnce." K8s matches it to an existing PV (or dynamically provisions a new one from a StorageClass). Pods mount PVCs, not PVs directly.

**Q52. What's a StorageClass?**
A StorageClass is a template that describes how to dynamically create PVs. Example: `fast-ssd` uses AWS gp3 SSDs, `slow-hdd` uses EBS st1. When a PVC references a StorageClass, K8s provisions a matching PV automatically.

**Q53. What are PV access modes?**
- **ReadWriteOnce (RWO)** — one node mounts read-write (most cloud block storage)
- **ReadOnlyMany (ROX)** — many nodes mount read-only
- **ReadWriteMany (RWX)** — many nodes mount read-write (NFS, CephFS)
- **ReadWriteOncePod (RWOP)** — single pod only (K8s 1.27+)

**Q54. What happens to the PVC if I delete the StatefulSet?**
By default, **PVCs are preserved** — your data stays safe. You must delete PVCs explicitly (`kubectl delete pvc mysql-data-mysql-0`). This is intentional: K8s assumes stateful data is precious.

**Q55. What is a NetworkPolicy?**
A NetworkPolicy is a firewall rule between pods. By default, all pods can talk to all others. A NetworkPolicy restricts ingress/egress based on pod labels, namespaces, or IPs. Requires a CNI that supports it (Calico, Cilium). Example: only allow backend pods to reach MySQL port 3306.

### Group 5 — Security (Q56–Q65)

**Q56. What is a ServiceAccount?**
A ServiceAccount is an identity for pods (like a user, but for apps). Every pod gets one (default if unspecified). Used with RBAC to control what the pod's processes can do via the K8s API. E.g., a deployer pod might need a SA with `create deployments` permission.

**Q57. How do RoleBinding and ClusterRoleBinding differ?**
A **RoleBinding** grants a Role within a single namespace. A **ClusterRoleBinding** grants a ClusterRole across the entire cluster. Use RoleBinding for most cases (principle of least privilege); use ClusterRoleBinding only for truly cluster-wide admins.

**Q58. Are Secrets encrypted in K8s?**
**No, by default Secrets are only base64-encoded**, which is NOT encryption. Anyone with API access can decode them. For real encryption:
- Enable **etcd encryption at rest** via `--encryption-provider-config`
- Use **Sealed Secrets** (Bitnami) for GitOps
- Use **External Secrets Operator** to fetch from AWS Secrets Manager, Vault, etc.

**Q59. What's the difference between encoding and encryption?**
**Encoding** (base64) is reversible with no key — any observer can decode. **Encryption** requires a key; without the key, ciphertext is useless. K8s Secrets use encoding by default, not encryption.

**Q60. What is PodSecurityAdmission (PSA)?**
PSA is K8s's built-in pod security policy enforcer (replaces the deprecated PodSecurityPolicy). It enforces three profiles: **privileged** (anything goes), **baseline** (blocks known-bad), **restricted** (hardened defaults). Applied via namespace labels.

**Q61. What is `runAsNonRoot` in securityContext?**
`spec.securityContext.runAsNonRoot: true` refuses to start the container if it would run as UID 0 (root). Combined with `runAsUser: 1000` (or similar), it enforces non-root execution — a defense-in-depth win against container escapes.

**Q62. What's a Pod SecurityContext vs Container SecurityContext?**
Both configure security:
- **Pod-level** `spec.securityContext` applies to all containers in the pod (e.g., `fsGroup` for volume ownership).
- **Container-level** `spec.containers[].securityContext` overrides for that container (e.g., one container `runAsUser: 0`, another not).

**Q63. What's `allowPrivilegeEscalation: false`?**
Prevents a process from gaining more privileges than its parent (e.g., via setuid binaries). Best practice to set this to `false` plus `readOnlyRootFilesystem: true` in hardened deployments.

**Q64. How do you store TLS certs in K8s?**
Use a Secret of type `kubernetes.io/tls`:
```bash
kubectl create secret tls bus-ticket-tls \
  --cert=path/to/cert.pem \
  --key=path/to/key.pem
```
Referenced in Ingress `spec.tls.secretName`. Cert-manager automates issuance/renewal.

**Q65. How do pods authenticate to the API server?**
Every pod gets a projected ServiceAccount token mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`. When the pod makes an API call, it presents this JWT. RBAC then determines what the SA can do.

### Group 6 — Connected / Project-Specific (Q66–Q75)

**Q66. What happens if a backend pod crashes in our bus-ticket project?**
1. The container exits non-zero. 2. Kubelet detects it and reports to the API server. 3. The ReplicaSet controller sees desired=2, actual=1 and creates a replacement pod. 4. Scheduler places it on a node. 5. Service's endpoints automatically include the new pod once readiness passes. User might see one failed request mid-flight, but the Service load-balances around the dying pod. Total recovery: seconds.

**Q67. What if the node running MySQL dies?**
The kubelet stops heartbeating. After `node-monitor-grace-period` (~40s), the node controller marks it NotReady. StatefulSet schedules `mysql-0` on another node. The PVC follows the pod (if using cloud block storage with region-compatible placement). MySQL restarts, reads its data from the volume, and serves again. Total: ~1-2 min downtime.

**Q68. Can the frontend reach MySQL directly? Should it?**
Technically yes — both are in the same namespace; DNS `mysql:3306` resolves from any pod. Architecturally no — it violates layered architecture. Enforce isolation with a NetworkPolicy:
```yaml
podSelector:
  matchLabels:
    app: mysql
ingress:
- from:
  - podSelector:
      matchLabels:
        app: bus-ticket-backend
  ports:
  - port: 3306
```

**Q69. Trace the DNS path from frontend to backend.**
1. Frontend pod opens TCP to `http://bus-ticket-backend:8080`. 2. `/etc/resolv.conf` points to CoreDNS (10.96.0.10). 3. CoreDNS replies with the Service ClusterIP (say 10.96.120.45). 4. Frontend sends packet to 10.96.120.45:8080. 5. kube-proxy iptables rules on the local node DNAT the packet to a real backend pod IP:8080. 6. Response returns through the same NAT. Zero IPs hardcoded.

**Q70. Why does the rolling update work without downtime?**
Because three mechanisms combine: **(a)** Deployment controller creates new pod first (maxSurge=1). **(b)** Readiness probe gates traffic — new pod not added to Service endpoints until healthy. **(c)** Old pod only removed after new pod is Ready. At every instant, at least 2 healthy pods serve traffic.

**Q71. What would happen if I changed `imagePullPolicy: IfNotPresent` to `Always` on Minikube?**
K8s would try to pull `bus-ticket-backend:latest` from a remote registry on every pod start. But that image only exists in Minikube's local Docker — no registry has it. You'd get `ImagePullBackOff`. `IfNotPresent` tells K8s "use the local copy if you have it" — perfect for local dev.

**Q72. How would you debug: "MySQL pod is Running but backend can't connect"?**
1. `kubectl get endpoints mysql` — does the Service have endpoints? 2. `kubectl exec -it <backend> -- nslookup mysql` — DNS working? 3. `kubectl exec -it <backend> -- nc -zv mysql 3306` — port reachable? 4. Check MySQL logs for connection errors. 5. Verify Secret's `DB_PASSWORD` matches MySQL's `MYSQL_ROOT_PASSWORD`. 6. NetworkPolicy blocking? Check `kubectl get networkpolicy`.

**Q73. If we changed the DB password in the Secret, why doesn't the backend pick it up automatically?**
Environment variables (injected via `envFrom`) are fixed at pod startup. Updating the Secret doesn't signal existing pods. You must restart:
```bash
kubectl rollout restart deployment/bus-ticket-backend
```
For auto-reload, use projected volumes (files, which K8s updates in place) or tools like Reloader that detect Secret changes and trigger rollouts.

**Q74. How would you scale the backend when traffic spikes?**
Quick manual: `kubectl scale deploy/bus-ticket-backend --replicas=10`. Permanent automatic: deploy an HPA with `averageUtilization: 70% CPU, min 2, max 20`. When CPU exceeds 70%, K8s adds pods; when it drops, K8s removes them. Requires metrics-server.

**Q75. In a viva: summarize the full journey of a booking request through our K8s setup.**
1. User hits `http://bus-ticket.local/bookings` (browser). 2. DNS resolves to Minikube/cluster IP. 3. Ingress controller (nginx) receives on port 80, inspects host + path, routes to the `bus-ticket-frontend` Service. 4. kube-proxy DNATs to a frontend pod. 5. Frontend calls `http://bus-ticket-backend:8080/api/bookings` — DNS → Service IP → kube-proxy → backend pod. 6. Backend opens a JDBC connection to `mysql:3306` — same DNS+kube-proxy chain. 7. MySQL pod (`mysql-0`) reads/writes to its PVC-backed volume. 8. Response propagates back. At every hop, K8s load-balances, retries via probes, and self-heals. That's the power of the platform.

### Group 7 — Bonus / Jumbled (Q76–Q80)

**Q76. What's the difference between labels and annotations?**
**Labels** are key-value metadata used for **selection** (Services select pods by label; Deployments manage pods by label). **Annotations** are also key-value but used for **non-identifying metadata** — build commit IDs, last-modified timestamps, tool-specific hints. You can't select by annotation.

**Q77. What's an init container?**
A container that runs to completion **before** the main containers start. Used for setup tasks: waiting for a dependency (MySQL), populating a volume, running migrations. Multiple init containers run sequentially. Main containers start only after all inits succeed.

**Q78. What is a sidecar pattern?**
Two containers in the same pod, sharing network and volumes. The sidecar augments the main container — e.g., log shipper reading from a shared volume, service mesh proxy (Envoy) intercepting traffic, config-reloader watching a file. Same pod = same lifecycle.

**Q79. How do you handle secrets rotation in K8s?**
Options: (a) Manual: update Secret, `kubectl rollout restart`. (b) Sealed Secrets with scheduled re-seal. (c) External Secrets Operator: secrets stored in AWS/Vault, auto-synced. (d) Mount Secrets as projected volumes (files auto-update, but app must re-read). (e) Service mesh mTLS rotates cert secrets automatically.

**Q80. If you could learn only one K8s concept well, what would it be?**
The **controller loop / desired state reconciliation**. Every K8s feature — Deployment, Service, Ingress, HPA, custom operators — is built on the same pattern: declare what you want, controller watches and reconciles. Understanding this makes every other resource intuitive. Combined with kubectl debugging (`get`, `describe`, `logs`), this one concept unlocks 80% of the platform.

---

## Closing Notes

Congratulations — you've read through Section B. You now have:

- A working mental model of Kubernetes (orchestra + reconciliation loop)
- YAML for deploying Backend (Deployment + Service + Config + Secret)
- YAML for deploying Frontend (Deployment + Service)
- YAML for deploying MySQL (StatefulSet + Headless Service + PVC)
- Ingress for single-IP external access
- Troubleshooting table for the 20+ errors you will hit
- 40-command kubectl cheat sheet
- 80 viva questions tied to our Bus Ticket project

**Next up: Section C** will cover observability (logs, metrics, tracing), CI/CD (GitHub Actions → K8s), and production hardening.

Until then: deploy, break, fix, repeat. That's the only real way to learn Kubernetes.

---

*End of Section B — Kubernetes*
# SECTION C — AWS DEPLOYMENT

Welcome to the AWS half of our deployment journey for the **Bus Ticket Booking System**. By now you've already:

- Built two Docker images in Section A: `bus-ticket-backend:latest` (port 8080) and `bus-ticket-frontend:latest` (port 8081).
- Run them locally on Kubernetes in Section B using Deployments, Services, and Ingress.

Now we put the whole thing on the internet. We'll use AWS — Amazon Web Services — because it's the most common cloud in industry, has a generous free tier for students, and is what most of your interviewers will ask about.

Before we dive in, a confession: AWS has **200+ services**. You don't need to know all of them. For this project, you need roughly **12**. That's the list we'll focus on.

---

## Part C1: AWS Refresher (Brief)

### What is AWS, really?

AWS is a pile of **rentable computers, storage, and networks** that Amazon operates inside huge buildings called data centers. Instead of buying a server, plugging it in, and praying the power doesn't trip at 3 AM during exam week, you go to a web console, click "I want a server," type your credit card number, and ninety seconds later you get an IP address.

That's it. That's the whole cloud. Every other AWS service is just a nicer wrapper around "Amazon's computers doing something for you."

**Analogy — Uber vs owning a car:**
Owning a car means you pay upfront, insure it, fuel it, repair it, park it, and use it maybe an hour a day. An Uber ride costs more *per minute* but you only pay when you actually ride, and you never worry about the engine.
AWS is Uber for computers. A physical server in your college lab is the car you own.

### Services we'll actually touch

| Service | One-line purpose |
|---|---|
| EC2 | Rent a Linux VM |
| ECR | Store Docker images privately |
| ECS | Run Docker containers without managing VMs |
| EKS | Run Kubernetes without managing masters |
| RDS | Managed MySQL |
| ALB | Load balancer / URL router |
| Route 53 | DNS — turns `busticket.in` into an IP |
| CloudWatch | Logs, metrics, alarms |
| IAM | Who can do what |
| VPC | Private network around your stuff |
| Parameter Store / Secrets Manager | Passwords, DB URLs, API keys |
| S3 | Object storage for files, backups |

You don't need the rest for now. Don't let the service list scare you.

### Free Tier — what you get for 12 months free

| Resource | Free allowance | Used by us? |
|---|---|---|
| EC2 | 750 hours/month of **t2.micro** or **t3.micro** | Yes (Option 1, 2) |
| RDS | 750 hours/month of **db.t3.micro**, 20 GB storage | Yes (Option 2, 3, 4) |
| S3 | 5 GB storage, 20k GET, 2k PUT per month | Optional |
| ECR | 500 MB private image storage/month | Yes |
| Lambda | 1M requests + 400k GB-seconds per month | Not used here |
| CloudWatch | 5 GB log ingestion, 10 custom metrics | Yes |
| Data transfer | 100 GB egress/month | Watch this |
| ALB | **Not free** (~$16/mo) | Only if you need it |
| NAT Gateway | **Not free** (~$32/mo + data) | Avoid unless required |
| EKS control plane | **Not free** ($73/mo) | Only Option 4 |

**750 hours** = roughly one instance running 24/7 all month. Two instances = you blow past the free tier on one of them.

### Pricing Red Flags (Memorize These)

These are the things that silently drain student wallets:

1. **EKS control plane** — $0.10/hour = $73/month. Runs even if you have zero worker nodes. Forgot to delete it over summer break? $219 bill.
2. **NAT Gateway** — $0.045/hour (~$32/mo) + $0.045/GB. Created automatically by some templates.
3. **Elastic IPs not attached** — Free *while attached* to a running EC2. If the instance is stopped, they cost $0.005/hour (~$3.60/mo each).
4. **RDS Multi-AZ** — Doubles RDS cost. Uncheck for learning.
5. **ALB** — ~$16/month even if nobody uses it.
6. **Data egress** — Traffic leaving AWS to the internet costs ~$0.09/GB after the 100 GB free.
7. **Snapshots** — EBS and RDS snapshots pile up silently.

**The rule:** set a **billing alarm at $5** on day one. We'll do it in Part C9.

### Cost Comparison for Our Bus Ticket App

Let's say we run 24/7 with ~100 users/day. Rough monthly costs after free tier expires:

| Architecture | Monthly Cost | Why |
|---|---|---|
| Single EC2 t2.micro + Docker Compose (MySQL in container) | ~$9 | One VM does everything |
| EC2 + RDS db.t3.micro | ~$24 | VM + managed DB |
| ECS Fargate (2 tasks) + RDS + ALB | ~$55 | No VM to manage, but ALB + Fargate add up |
| EKS + 2 t3.small nodes + RDS + ALB | ~$135 | Control plane is $73 alone |

**Lesson:** for a student project demo, **Option 1 is free**. For a real startup, **Option 3** (Fargate) is the sweet spot. EKS is overkill unless you have a team of five+ engineers.

---

## Part C2: Container-Related AWS Services Deep Dive

We now go one level deeper on each service. For each one, we cover: **What it is, When to use, Pricing, Analogy**.

### ECR — Elastic Container Registry

**What it is:** A private registry for Docker images, hosted by AWS and secured by IAM.

**When to use:** Any time you're deploying container images to ECS, EKS, or EC2 and you don't want them on public Docker Hub.

**Pricing:** $0.10 per GB-month stored + data transfer out. Free tier: 500 MB. Our two images (~300 MB each) fit in 500 MB if we only keep one tag.

**Analogy — the cyber cafe locker:**
Docker Hub is like leaving your bag at the front counter of a cyber cafe — anyone walking past can pick it up. ECR is a **locked locker** with your IAM key; you can share the key with your team but outsiders can't see what's inside.

```text
┌─────────────────────┐
│   Your dev laptop   │
│  docker build …     │
└──────────┬──────────┘
           │  docker push
           ▼
┌─────────────────────┐
│       AWS ECR        │  <-- private, IAM-secured
│  bus-ticket-backend  │
│  bus-ticket-frontend │
└──────────┬──────────┘
           │  docker pull
           ▼
┌─────────────────────┐
│   ECS / EKS / EC2    │
└─────────────────────┘
```

### ECS — Elastic Container Service

**What it is:** AWS's native container orchestrator. Think of it as "Docker Swarm, but made by Amazon, and actually production-ready."

**When to use:** When Kubernetes feels too heavy and you just want to run containers behind a load balancer.

**Two launch types:**

| Launch Type | You manage | AWS manages | Pricing |
|---|---|---|---|
| **EC2** | VM OS, patching, scaling | Orchestrator | EC2 hourly rate |
| **Fargate** | Nothing (just the container) | VM + orchestrator | Per vCPU-second + RAM-second |

**Analogy:**
- **ECS on EC2** = renting a kitchen and cooking yourself. Cheaper but you do dishes.
- **ECS on Fargate** = Zomato cloud kitchen. You ship recipes; someone else handles the kitchen.

### EKS — Elastic Kubernetes Service

**What it is:** Managed Kubernetes. AWS runs the control plane (API server, etcd, scheduler); you bring the worker nodes (or use Fargate).

**When to use:** Your team already knows K8s, you want cloud portability (EKS ↔ GKE ↔ AKS), or you have 20+ microservices.

**Pricing:**
- Control plane: **$73/month flat**. Always. No free tier.
- Worker nodes: Normal EC2 price, or Fargate price.

**Analogy:** K8s is a complex recipe. Running your own K8s cluster is cooking that recipe yourself. EKS hires Amazon as the head chef while you remain responsible for groceries (workers, networking, IAM).

### EC2 + Docker — the "just a VM" option

**What it is:** Launch a Linux VM, install Docker, `docker compose up`. No orchestrator.

**When to use:** Your first deploy. Demo. Learning. Small apps with one instance.

**Pricing:** t2.micro is free tier. After that ~$9/mo.

**Analogy:** The chai stall. One person, one stove, one kettle. Works great until you're serving 200 customers.

### RDS — Relational Database Service

**What it is:** MySQL/Postgres/MariaDB/Oracle/SQL Server as a service. AWS handles backups, patching, Multi-AZ failover, read replicas.

**When to use:** Always. Do **not** run MySQL in a container in production. A container dying = your data disappearing (unless you set up persistent volumes, and even then, backups are on you).

**Pricing:** db.t3.micro free tier. After that ~$15/mo + storage.

**Analogy:** RDS is a fully-staffed hotel kitchen. You hand them a menu (SQL schema); they handle groceries, cooks, cleaning, fire safety. MySQL-in-Docker-on-your-laptop is making maggi on your hostel room heater.

### ALB — Application Load Balancer

**What it is:** L7 HTTP(S) router. Forwards `/api/*` to one target group, `/*` to another. Does health checks. Terminates TLS.

**When to use:** You have 2+ containers or you want a stable public URL.

**Pricing:** ~$16/mo + $0.008/LCU-hour. No free tier.

**Analogy:** The bouncer at a club who checks your ID, routes you to the right room (VIP, dance floor), and turns away fake IDs.

### Route 53

**What it is:** DNS. Turns `busticket.in` into `13.235.52.11`. Supports A, AAAA, CNAME, MX, TXT, ALIAS (AWS-specific), and **health-check-based** failover.

**Pricing:** $0.50/hosted zone/month + $0.40 per million queries.

**Analogy:** The contact list on your phone. "Mumma" → +91 9x... Route 53 is the contact list of the internet.

### CloudWatch

**What it is:** Logs + metrics + alarms.

**When to use:** Always. Your containers' `stdout` should flow into CloudWatch Logs. CPU/memory/network should appear as metrics. Alarm on "unhealthy target count > 0".

**Pricing:** 5 GB free logs ingestion/month; $0.50/GB after. 10 custom metrics free.

**Analogy:** The CCTV cameras and motion sensors in a warehouse. Logs = CCTV footage. Metrics = thermometer on the wall. Alarms = the bell that rings when someone enters after hours.

### IAM — Identity and Access Management

**What it is:** Who (user/role) can do what (action) on what (resource).

**Two main entities:**
- **User** = a person with a password.
- **Role** = a hat that an AWS service or user *temporarily wears* to get permissions.

Example: an EC2 instance wearing a role can read from ECR without anyone typing a password.

**Analogy:** Hostel ID card vs visitor pass. ID card = IAM User (you, permanent). Visitor pass = IAM Role (handed out temporarily to a service).

### VPC — Virtual Private Cloud

**What it is:** Your private network in AWS.

Pieces:
- **VPC** — the whole network (e.g. 10.0.0.0/16).
- **Subnet** — a slice. Public subnet = has internet via IGW. Private subnet = no direct internet.
- **Internet Gateway (IGW)** — door to the public internet.
- **NAT Gateway** — outbound-only door for private subnets (lets your DB download patches but not receive traffic).
- **Security Group (SG)** — virtual firewall attached to instances. Stateful.
- **NACL** — subnet-level firewall. Stateless. Rarely touched.

```text
            Internet
               │
          ┌────┴────┐
          │   IGW   │
          └────┬────┘
               │
    ┌──────────┼──────────┐
    │   Public Subnet     │  <-- EC2 web server, ALB
    │   10.0.1.0/24       │
    └──────────┬──────────┘
               │  (NAT Gateway here for outbound)
    ┌──────────┴──────────┐
    │  Private Subnet     │  <-- RDS MySQL
    │   10.0.2.0/24       │
    └─────────────────────┘
```

**Analogy:** A hostel building. VPC = the building. Subnets = floors. IGW = main gate. NAT = a vending machine that lets you get a cold drink without letting strangers walk into your room. SG = the lock on your door.

### Parameter Store and Secrets Manager

**What they are:** Two different AWS services for storing configuration.

| Feature | Parameter Store | Secrets Manager |
|---|---|---|
| Plain config (e.g. BACKEND_BASE_URL) | Yes | Yes |
| Encrypted secrets | Yes (SecureString) | Yes |
| Automatic rotation | No | Yes |
| Cost | Free for <10k params | $0.40/secret/month |

**Our choice:** Parameter Store for everything. It's free.

**Analogy:** Parameter Store is your locker at the hostel (free). Secrets Manager is a bank locker (costs ₹400/year but automatically rotates the key every quarter).

### S3 — Simple Storage Service

**What it is:** Unlimited object storage. Upload files, get URLs.

**When to use:** Static frontend files (if you split Thymeleaf out later), database backups, user uploads (profile pics, bus photos), log archives.

**Pricing:** ~$0.023/GB/month + request costs. Free tier: 5 GB.

**Analogy:** Google Drive but without a folder tree — every file is just a URL.

---

## Part C3: Deployment Option Comparison

We'll pick **one** of four paths depending on budget, skill, and how much pain you want.

```text
   ┌──────────────────────────────────────────────────────────┐
   │                   Your requirements                        │
   └─────────────────────────┬────────────────────────────────┘
                             │
          "I'm learning / just demo it to HoD"
                             │
                             ▼
                 ┌─────────────────────┐
                 │ Option 1: EC2 +      │
                 │ Docker Compose       │
                 └─────────────────────┘

          "I want real production feel, but on a budget"
                             │
                             ▼
                 ┌─────────────────────┐
                 │ Option 2: EC2 + RDS  │
                 └─────────────────────┘

          "I don't want to touch any VM"
                             │
                             ▼
                 ┌─────────────────────┐
                 │ Option 3: Fargate    │
                 │ + RDS                │
                 └─────────────────────┘

          "I need full Kubernetes because reasons"
                             │
                             ▼
                 ┌─────────────────────┐
                 │ Option 4: EKS + RDS  │
                 └─────────────────────┘
```

### The Big Table

| Option | Approach | Monthly Cost | Complexity | Best For | What you learn |
|---|---|---|---|---|---|
| **1** | Single EC2 + Docker Compose | ~$0 (free tier) | Very easy | Dev, demo, viva | Linux, Docker, SSH |
| **2** | EC2 + Docker + RDS | ~$15 after free tier | Easy | Small prod | Networking, SG, RDS |
| **3** | ECS Fargate + RDS | ~$30 | Medium | Real prod, auto-scale | Task defs, ALB, IAM roles |
| **4** | EKS + RDS | ~$75+ | Hard | Multi-team company | K8s, IRSA, Helm |

### Option 1 — EC2 + Docker Compose

**Pros:**
- Free.
- You can SSH in and poke at things like on a regular Linux box.
- Directly reuses your Section A `docker-compose.yml`.

**Cons:**
- Single point of failure — if the VM dies, site dies.
- No auto-scaling.
- MySQL data lives on the VM; if you terminate the instance, data gone.
- You are responsible for OS patches.

**What you learn:** SSH, Linux package management, SG rules, basic ops.

**What breaks:**
- Reboot without `restart: unless-stopped` in compose → containers don't come back.
- VM disk fills up → Docker fails silently.

### Option 2 — EC2 + Docker + RDS

**Pros:**
- Backend and frontend still run as simple containers on the VM.
- MySQL is managed by RDS — daily backups, minor version patches, failover.
- Realistic setup for a small startup.

**Cons:**
- Still a single VM — no HA for app layer.
- Pay for RDS after free tier.

**What you learn:** VPC, subnets, connecting an app to an external DB, SG-to-SG rules.

**What breaks:**
- SG misconfig → container can't reach RDS (classic).
- Forgot to set `publicly accessible = No` → attackers see your DB.

### Option 3 — ECS Fargate + RDS

**Pros:**
- No VM to patch.
- Auto-scale with one command.
- Rolling deployments built-in.
- ALB integration clean.

**Cons:**
- ALB costs ~$16/mo.
- Fargate hourly cost > EC2 hourly cost per CPU.
- Learning curve: task definitions, cluster, service, target groups.

**What you learn:** Task definitions, IAM roles for tasks, CloudWatch log routing, ALB target groups, zero-downtime deploys.

**What breaks:**
- Task execution role missing `ecr:GetAuthorizationToken` → `CannotPullContainerError`.
- Health check fails because you pointed it at `/` instead of `/actuator/health`.

### Option 4 — EKS + RDS

**Pros:**
- Full Kubernetes. Portable to GCP/Azure.
- Reuse your Section B YAMLs.
- Industry standard for big companies.

**Cons:**
- $73/month minimum even if idle.
- Dozens of new concepts: IRSA, ALB Controller, Helm, ExternalDNS.
- Networking is more complex (CNI, pod CIDR vs service CIDR).

**What you learn:** Managed K8s, IRSA, Helm charts, production ingress patterns.

**What breaks:**
- Control plane version upgrades.
- ALB Controller not installed → Ingress does nothing.
- Nodes can't join cluster → IAM role policies missing.

### Decision for our Bus Ticket System

For a 5-member college team with a 15-day deadline:

1. **Week 1:** Get Option 1 working. Take a screenshot for the report.
2. **Week 2:** Migrate to Option 2 for the "production" story in your viva.
3. **Optional bonus:** Demo Option 3 if you finish early.

Skip Option 4 — it's a bill trap and your panel won't be impressed that you spent ₹6000 of someone's money for the same running app.

---

## Part C4: Push Images to ECR (Step-by-Step)

Before any deployment option, the images from Section A must live in ECR. Let's push them.

### Prerequisites

- AWS account created.
- AWS CLI installed and configured:
  ```bash
  aws configure
  # AWS Access Key ID: AKIA...
  # AWS Secret Access Key: ...
  # Default region: ap-south-1   <-- Mumbai, lowest latency for India
  # Default output format: json
  ```
- Docker installed locally (from Section A).
- Your images tagged:
  ```bash
  docker images | grep bus-ticket
  # bus-ticket-backend    latest   ...
  # bus-ticket-frontend   latest   ...
  ```

### Step 1: Create the ECR repositories

**Via Console:**
1. Sign in to AWS Console.
2. Go to **ECR** service.
3. Click **Create repository**.
4. Visibility: **Private**.
5. Repository name: `bus-ticket-backend`.
6. Tag immutability: **Disabled** (for now — we'll overwrite `latest`).
7. Scan on push: **Enabled** (free basic scan).
8. Click **Create repository**.
9. Repeat for `bus-ticket-frontend`.

**Via CLI:**

```bash
aws ecr create-repository \
  --repository-name bus-ticket-backend \
  --region ap-south-1 \
  --image-scanning-configuration scanOnPush=true

aws ecr create-repository \
  --repository-name bus-ticket-frontend \
  --region ap-south-1 \
  --image-scanning-configuration scanOnPush=true
```

After creation, each repo has a URI like:
```
123456789012.dkr.ecr.ap-south-1.amazonaws.com/bus-ticket-backend
```

The `123456789012` is **your AWS account ID**. Grab it with:

```bash
aws sts get-caller-identity --query Account --output text
```

**Why "URI so long?":** Because AWS is multi-tenant. Your account ID, the region, the service, the registry type, and the repo name all show up so AWS knows exactly whose image you mean.

### Step 2: Authenticate Docker to ECR

Docker needs a temporary password to push to ECR. AWS gives you one that's valid for 12 hours.

```bash
aws ecr get-login-password --region ap-south-1 \
  | docker login \
      --username AWS \
      --password-stdin \
      123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

You should see:

```
Login Succeeded
```

**Why is the username literally `AWS`?** Because the password is the real identifier. The username is a placeholder.

**Troubleshoot:**
- `no basic auth credentials` → you skipped this step or the 12h token expired. Re-run.
- `AccessDeniedException` → your IAM user doesn't have `AmazonEC2ContainerRegistryFullAccess`. Attach it.

### Step 3: Tag your local images with ECR URI

Docker images can have multiple tags pointing to the same SHA. We add an ECR-shaped tag:

```bash
# Backend
docker tag bus-ticket-backend:latest \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/bus-ticket-backend:v1

# Frontend
docker tag bus-ticket-frontend:latest \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/bus-ticket-frontend:v1
```

**Why `v1` instead of `latest`?** In production you want **immutable tags**. `latest` is a moving target — nobody can tell which commit it points to. `v1`, `v2`, `v3-hotfix-july17` are specific. Every interviewer likes this answer.

Verify:

```bash
docker images | grep ecr
```

### Step 4: Push

```bash
docker push \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/bus-ticket-backend:v1

docker push \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/bus-ticket-frontend:v1
```

Each push streams layer uploads like:

```
The push refers to repository [123456789012.dkr.ecr.ap-south-1.amazonaws.com/bus-ticket-backend]
a1b2c3d4: Pushed
e5f6g7h8: Pushed
v1: digest: sha256:abcd... size: 1234
```

**Watch out:** the first push for a big image on home Wi-Fi can take 5–15 minutes. Grab chai.

### Step 5: Verify in Console

1. Go to ECR → Repositories.
2. Click `bus-ticket-backend`.
3. You should see a row with tag `v1`, image size, pushed-at time, and scan status.

**Image scan results:** ECR scans for CVEs. You might see HIGH/CRITICAL findings in your base image. For a student project, note them in the report and move on. In real production you'd rebuild on a patched base.

### Step 6: IAM Permissions — the exact policy

For your human IAM user (the one running CLI locally), attach:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage",
        "ecr:CreateRepository",
        "ecr:DescribeRepositories",
        "ecr:ListImages"
      ],
      "Resource": "*"
    }
  ]
}
```

Or shortcut: attach `AmazonEC2ContainerRegistryFullAccess`.

### Troubleshooting Table (ECR)

| Error | Cause | Fix |
|---|---|---|
| `no basic auth credentials` | Token expired or wrong region | Re-run `aws ecr get-login-password` |
| `denied: User: ... is not authorized` | IAM missing ECR perms | Attach policy above |
| `repository ... does not exist` | Typo or wrong region | Check repo name, region flag |
| Push hangs at 0% | Network / proxy | Check `curl https://ecr.ap-south-1.amazonaws.com` reachable |
| Push extremely slow | Home Wi-Fi | Push from a cloud shell or EC2 in same region |
| `name unknown: The repository with name ... does not exist` | Pushed to wrong account ID | Recheck `aws sts get-caller-identity` |

---

## Part C5: Option 1 — Deploy on Single EC2 with Docker Compose

This is the **simplest** path. One VM, Docker installed, `docker compose up`. Done.

### Step 1: Launch the EC2 instance

1. Console → EC2 → **Launch instance**.
2. Name: `bus-ticket-server`.
3. AMI: **Amazon Linux 2023** (free tier eligible, rolling kernel).
4. Instance type: **t2.micro** (1 vCPU, 1 GB RAM). Free tier.
5. Key pair: Create new → name it `bus-ticket-key` → download `.pem` file. **Do NOT lose this file** — you can't SSH without it.
6. Network settings:
   - VPC: default.
   - Subnet: any public one.
   - Auto-assign public IP: **Enable**.
7. **Firewall (Security Group):** click *Create security group* with rules:

| Type | Port | Source | Purpose |
|---|---|---|---|
| SSH | 22 | My IP | Admin only |
| HTTP | 80 | 0.0.0.0/0 | Future use |
| Custom TCP | 8080 | 0.0.0.0/0 | Backend (testing) |
| Custom TCP | 8081 | 0.0.0.0/0 | Frontend |

**Do NOT open 3306** to the world. If MySQL runs in a container, it only listens inside the VM's Docker network. World access = hacked within hours.

8. Storage: 8 GB gp3 (default, free tier up to 30 GB).
9. Click **Launch instance**.

Wait ~1 minute. Status Check → **2/2 checks passed** → note the **Public IPv4** (e.g. `13.234.56.78`).

### Step 2: SSH in

**On Linux/Mac:**
```bash
chmod 400 bus-ticket-key.pem
ssh -i bus-ticket-key.pem ec2-user@13.234.56.78
```

**On Windows PowerShell:**
```powershell
ssh -i C:\Users\Sarthak\Downloads\bus-ticket-key.pem ec2-user@13.234.56.78
```

First connection will ask "Are you sure you want to continue connecting?" → yes.

**Why `ec2-user`?** Amazon Linux ships with that default username. Ubuntu AMIs use `ubuntu`, Debian uses `admin`.

### Step 3: Install Docker + Compose

Once inside the VM:

```bash
# Update packages
sudo dnf update -y

# Install Docker
sudo dnf install -y docker

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Add ec2-user to docker group so we don't need sudo every time
sudo usermod -aG docker ec2-user

# Log out and back in for group change to take effect
exit
```

Re-SSH and verify:

```bash
docker --version
# Docker version 24.x.x
docker ps
# CONTAINER ID   IMAGE   ...   (empty is fine)
```

Install Docker Compose v2 (plugin):

```bash
DOCKER_COMPOSE_VERSION=v2.24.5
sudo curl -SL \
  https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/docker-compose-linux-x86_64 \
  -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose

# Also link as docker plugin
mkdir -p ~/.docker/cli-plugins
ln -sf /usr/local/bin/docker-compose ~/.docker/cli-plugins/docker-compose

docker compose version
```

### Step 4: Give EC2 permission to pull from ECR

We want our EC2 instance to pull images from ECR **without** putting long-lived access keys on the VM.

**Best practice: attach an IAM Instance Profile.**

1. Console → IAM → Roles → **Create role**.
2. Trusted entity: **AWS Service** → **EC2**.
3. Permissions: search and attach `AmazonEC2ContainerRegistryReadOnly`.
4. Role name: `ec2-ecr-read`.
5. Create.
6. Go back to EC2 → select instance → Actions → Security → **Modify IAM role** → choose `ec2-ecr-read` → Update.

Now back in SSH:

```bash
# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo dnf install -y unzip
unzip awscliv2.zip
sudo ./aws/install

aws --version

# Login Docker to ECR (the role lets this just work)
aws ecr get-login-password --region ap-south-1 \
  | docker login \
      --username AWS \
      --password-stdin \
      123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

You should see `Login Succeeded`.

### Step 5: Copy your `docker-compose.yml` to EC2

From your **laptop** (not the EC2):

```bash
scp -i bus-ticket-key.pem docker-compose.yml \
  ec2-user@13.234.56.78:/home/ec2-user/docker-compose.yml
```

On the EC2, edit the file to point to ECR:

```bash
nano ~/docker-compose.yml
```

Change this sort of structure:

```yaml
version: "3.9"

services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: busticketbooking
      MYSQL_USER: busapp
      MYSQL_PASSWORD: busapppass
    volumes:
      - mysql-data:/var/lib/mysql
    restart: unless-stopped

  backend:
    image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/bus-ticket-backend:v1
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/busticketbooking
      SPRING_DATASOURCE_USERNAME: busapp
      SPRING_DATASOURCE_PASSWORD: busapppass
    depends_on:
      - mysql
    restart: unless-stopped

  frontend:
    image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/bus-ticket-frontend:v1
    ports:
      - "8081:8081"
    environment:
      BACKEND_BASE_URL: http://backend:8080
    depends_on:
      - backend
    restart: unless-stopped

volumes:
  mysql-data:
```

**Why `restart: unless-stopped`?** If the container crashes or the VM reboots, Docker automatically restarts it. Otherwise your site stays down until you SSH back in.

### Step 6: Bring it up

```bash
docker compose up -d
```

`-d` = detached (background). Watch logs:

```bash
docker compose logs -f backend
```

Wait 30–60 seconds for Spring Boot to start. When you see:

```
Started BusTicketApplication in 23.4 seconds
```

Press `Ctrl+C` to stop tailing (containers keep running).

### Step 7: Access from browser

Open:

```
http://13.234.56.78:8081
```

You should see the Thymeleaf home page.

Test backend directly:

```
http://13.234.56.78:8080/actuator/health
```

Should return `{"status":"UP"}`.

**If it doesn't work:**

| Symptom | Check |
|---|---|
| Connection refused | SG rules; is port 8081 open? |
| Page loads but no bus routes | Frontend can't reach backend (inside VM, `backend` must be the service name) |
| 500 Internal Server Error | `docker compose logs backend` |
| "Connection to mysql:3306 refused" | MySQL container not started yet; check `docker ps` |

### Step 8: Elastic IP (so IP doesn't change on reboot)

1. EC2 Console → Elastic IPs → **Allocate Elastic IP address** → Allocate.
2. Select the new IP → **Associate** → choose your instance.
3. New public IP is now stable. Replace the old IP in your bookmarks.

**Warning:** If you **stop** (not terminate) the instance later, the EIP keeps costing ~$3.60/month until you release it.

### Step 9: Auto-start on reboot

Our `restart: unless-stopped` handles per-container restart, but if the VM itself reboots, we also need Docker compose to re-up.

Create a systemd service:

```bash
sudo nano /etc/systemd/system/bus-ticket.service
```

Paste:

```ini
[Unit]
Description=Bus Ticket docker compose
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/ec2-user
ExecStart=/usr/local/bin/docker-compose up -d
ExecStop=/usr/local/bin/docker-compose down

[Install]
WantedBy=multi-user.target
```

Enable:

```bash
sudo systemctl daemon-reload
sudo systemctl enable bus-ticket.service
sudo systemctl start bus-ticket.service
sudo systemctl status bus-ticket.service
```

Now reboot to confirm:

```bash
sudo reboot
```

SSH back in after 30 seconds, `docker ps` should show containers running.

### Step 10: User Data Script (new instance automation)

When you next launch an EC2 you can skip all manual steps by pasting this **User Data** at launch time:

```bash
#!/bin/bash
dnf update -y
dnf install -y docker unzip
systemctl enable --now docker
usermod -aG docker ec2-user

curl -SL "https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64" \
  -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install
```

User Data runs once on first boot as root.

### Step 11: MySQL backup strategy (if you keep DB on the VM)

Our compose uses a **named volume** `mysql-data`. That volume is on the VM's disk. If the VM dies, data dies.

Nightly backup to S3:

```bash
# Create S3 bucket
aws s3 mb s3://bus-ticket-backups-sarthak --region ap-south-1

# Backup script
sudo tee /usr/local/bin/mysql-backup.sh <<'EOF'
#!/bin/bash
DATE=$(date +%Y%m%d-%H%M)
docker exec $(docker ps -qf name=mysql) \
  mysqldump -u root -prootpass busticketbooking > /tmp/busticket-$DATE.sql
aws s3 cp /tmp/busticket-$DATE.sql s3://bus-ticket-backups-sarthak/
rm /tmp/busticket-$DATE.sql
EOF

sudo chmod +x /usr/local/bin/mysql-backup.sh

# Cron it
(crontab -l 2>/dev/null; echo "0 2 * * * /usr/local/bin/mysql-backup.sh") | crontab -
```

Now every night at 2 AM, a SQL dump is pushed to S3. 5 GB free tier covers you for months.

### Step 12: Seeing logs

```bash
# All services
docker compose logs

# Tail last 50 lines of backend
docker compose logs --tail=50 -f backend

# Just errors
docker compose logs backend | grep -i error
```

For production-grade logs → ship to CloudWatch (see Part C7).

### Option 1 — Done

You now have a live bus ticket booking app at `http://<your-ip>:8081`. Take a screenshot for the report.

---

## Part C6: Option 2 — Deploy with RDS for MySQL

Option 1 keeps MySQL inside a container. Fine for demos, but:

- VM crashes → data lost.
- No automated backups.
- Scaling = manual dump/restore.

Option 2: **pull MySQL out of the VM** and put it on RDS.

### Why RDS?

| Concern | DIY MySQL in container | RDS |
|---|---|---|
| Daily backup | You write cron | Automatic, 7-day retention free |
| OS patching | You `yum update mysql` | AWS handles minor versions |
| Failover | Manual | One-click Multi-AZ |
| Monitoring | You install | Built-in CloudWatch metrics |
| Encryption at rest | Manual | Checkbox |
| Performance tuning | Your problem | Performance Insights |

**Analogy:** Running MySQL yourself is cooking your own food in the hostel. RDS is the mess. More expensive per meal, but you don't have to wash the vessels.

### Step 1: Pick a VPC and Subnet strategy

We'll put RDS in a **private subnet** and EC2 in a **public subnet**. EC2 reaches RDS over private networking.

Your default VPC already has two public subnets. For a true private subnet we'd add one with no IGW route. For a student setup, using default public subnets for RDS is fine **as long as `Publicly Accessible = No`**.

### Step 2: Create DB Subnet Group

RDS requires a subnet group spanning ≥2 AZs (for HA potential).

1. RDS Console → **Subnet groups** → **Create DB subnet group**.
2. Name: `bus-ticket-subnets`.
3. VPC: default.
4. Add 2 subnets from 2 AZs.
5. Create.

### Step 3: Create Security Group for RDS

1. EC2 Console → **Security Groups** → **Create security group**.
2. Name: `rds-busticket-sg`.
3. VPC: default.
4. Inbound rule: Type = **MySQL/Aurora (3306)**, Source = **custom** → pick your **EC2's SG** (not an IP). This says "allow anything with the EC2 SG tag to connect on 3306."
5. Outbound: default (all).
6. Create.

**Why SG-to-SG instead of IP?** Because the EC2's IP might change; the SG is a stable identifier.

### Step 4: Create the RDS instance

1. RDS Console → **Databases** → **Create database**.
2. Engine: **MySQL**. Version: **8.0.35**.
3. Template: **Free tier**.
4. Settings:
   - DB instance identifier: `bus-ticket-mysql`.
   - Master username: `admin`.
   - Master password: generate strong, e.g. `S0meL0ng#Passw0rd`. **Save it now.**
5. Instance class: `db.t3.micro`.
6. Storage: 20 GB gp3, disable auto-scaling.
7. Connectivity:
   - VPC: default.
   - Subnet group: `bus-ticket-subnets`.
   - Public access: **No**.
   - VPC SG: existing → `rds-busticket-sg`.
   - Port: 3306.
8. Database authentication: Password authentication.
9. Additional configuration:
   - Initial database name: `busticketbooking`.
   - Backup retention: 7 days (free).
   - Encryption at rest: Enable.
   - Log exports: error, slow query.
   - Minor version auto upgrade: Enable.
10. **Create database**. Takes 5–10 minutes.

Once it says **Available**, copy the **Endpoint** like `bus-ticket-mysql.xyz.ap-south-1.rds.amazonaws.com`.

### Step 5: Connect from EC2 to test

SSH back into EC2:

```bash
sudo dnf install -y mariadb105   # MySQL CLI compatible
mysql -h bus-ticket-mysql.xyz.ap-south-1.rds.amazonaws.com -u admin -p
```

Enter the password. You should get a `mysql>` prompt.

```sql
SHOW DATABASES;
-- information_schema
-- busticketbooking
-- mysql
-- performance_schema
-- sys

USE busticketbooking;
SHOW TABLES;
-- Empty set (good — we'll seed next)
```

### Step 6: Load your schema

The project has `sql/schema.sql` in the repo defining these tables (from the Bus Ticket Booking domain):

- `Address`
- `Agency`
- `AgencyOffice`
- `Driver`
- `Bus`
- `Route`
- `Trip`
- `Customer`
- `Booking`
- `Payment`
- `Review`

Copy the SQL file to EC2:

```bash
# From your laptop
scp -i bus-ticket-key.pem sql/schema.sql ec2-user@13.234.56.78:/home/ec2-user/
```

Load it:

```bash
mysql -h bus-ticket-mysql.xyz.ap-south-1.rds.amazonaws.com \
      -u admin -p busticketbooking < /home/ec2-user/schema.sql
```

Verify:

```bash
mysql -h ... -u admin -p busticketbooking -e "SHOW TABLES;"
```

You should see all 11 tables.

### Step 7: Update `docker-compose.yml`

On the EC2, edit the compose file:

```bash
nano ~/docker-compose.yml
```

Remove the `mysql:` service entirely and update backend:

```yaml
version: "3.9"

services:
  backend:
    image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/bus-ticket-backend:v1
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://bus-ticket-mysql.xyz.ap-south-1.rds.amazonaws.com:3306/busticketbooking
      SPRING_DATASOURCE_USERNAME: admin
      SPRING_DATASOURCE_PASSWORD: S0meL0ng#Passw0rd
    restart: unless-stopped

  frontend:
    image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/bus-ticket-frontend:v1
    ports:
      - "8081:8081"
    environment:
      BACKEND_BASE_URL: http://backend:8080
    depends_on:
      - backend
    restart: unless-stopped
```

But wait — hardcoding the password in a YAML file on the VM is bad. Let's fix that next.

### Step 8: Store the DB password in Parameter Store

```bash
aws ssm put-parameter \
  --name "/busticket/prod/db-password" \
  --value "S0meL0ng#Passw0rd" \
  --type SecureString \
  --region ap-south-1

aws ssm put-parameter \
  --name "/busticket/prod/db-url" \
  --value "jdbc:mysql://bus-ticket-mysql.xyz.ap-south-1.rds.amazonaws.com:3306/busticketbooking" \
  --type String \
  --region ap-south-1

aws ssm put-parameter \
  --name "/busticket/prod/db-username" \
  --value "admin" \
  --type String \
  --region ap-south-1
```

SecureString = KMS-encrypted. Only IAM principals with `kms:Decrypt` can read.

Give the EC2 role permission:

1. IAM → Roles → `ec2-ecr-read`.
2. Attach policy `AmazonSSMReadOnlyAccess` (or a narrower custom one).

### Step 9: Fetch at startup

Create a helper script that reads from SSM and exports to environment:

```bash
cat > ~/fetch-secrets.sh <<'EOF'
#!/bin/bash
export SPRING_DATASOURCE_URL=$(aws ssm get-parameter --name "/busticket/prod/db-url" --region ap-south-1 --query 'Parameter.Value' --output text)
export SPRING_DATASOURCE_USERNAME=$(aws ssm get-parameter --name "/busticket/prod/db-username" --region ap-south-1 --query 'Parameter.Value' --output text)
export SPRING_DATASOURCE_PASSWORD=$(aws ssm get-parameter --name "/busticket/prod/db-password" --with-decryption --region ap-south-1 --query 'Parameter.Value' --output text)

docker compose up -d
EOF

chmod +x ~/fetch-secrets.sh
```

Then in `docker-compose.yml`, use variable substitution:

```yaml
services:
  backend:
    environment:
      SPRING_DATASOURCE_URL: ${SPRING_DATASOURCE_URL}
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD}
```

Deploy:

```bash
~/fetch-secrets.sh
docker compose logs -f backend
```

Look for:

```
HikariPool-1 - Start completed
```

Visit `http://<ec2-ip>:8081`. Book a trip. The entry lands in RDS.

### Step 10: Verify data in RDS

```bash
mysql -h bus-ticket-mysql.xyz.ap-south-1.rds.amazonaws.com \
      -u admin -p busticketbooking \
      -e "SELECT COUNT(*) FROM Booking;"
```

### Option 2 — Done

You've now separated the app from the DB. Enterprise-grade setup on free tier.

---

## Part C7: Option 3 — Deploy on ECS Fargate

### What is Fargate again?

ECS with EC2 launch type means **you** pick the VM size, patch it, and scale it. Fargate means **AWS** picks a VM (invisibly, per task), patches it, and charges you only for the vCPU-seconds and MB-seconds your container uses.

No more SSH. No more `yum update`. Just containers.

**Analogy:** Think of EC2 as renting a flat — you deal with the AC repair guy. Fargate is like a hotel room — the moment you check out, it's their problem.

### Architecture Target

```text
            Internet
               │
               ▼
        ┌─────────────┐
        │     ALB      │  port 80/443
        └──────┬──────┘
         │    │    │
    path /api│    │ path /*
               │    │
   ┌───────────▼────┐  ┌─▼──────────────┐
   │ Backend Target │  │ Frontend Target│
   │  (port 8080)    │  │  (port 8081)    │
   │  Fargate tasks  │  │  Fargate tasks  │
   └───────────┬────┘  └────────────────┘
               │
               ▼
         ┌─────────┐
         │   RDS    │
         └─────────┘
```

### Step 1: Create the ECS cluster

```bash
aws ecs create-cluster --cluster-name bus-ticket --region ap-south-1
```

A "cluster" in Fargate land is just a namespace — no VMs.

### Step 2: Create CloudWatch Log Group

```bash
aws logs create-log-group --log-group-name /ecs/bus-ticket --region ap-south-1
```

### Step 3: Create IAM roles

**Task execution role** — lets ECS pull from ECR and write to CloudWatch.

```bash
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ecs-tasks.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name ecsTaskExecutionRole \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

**Task role** — for your app's AWS API calls (e.g. reading SSM).

```bash
aws iam create-role \
  --role-name busTicketTaskRole \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
  --role-name busTicketTaskRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMReadOnlyAccess
```

### Step 4: Task definition for backend

Save as `backend-taskdef.json`:

```json
{
  "family": "bus-ticket-backend",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/busTicketTaskRole",
  "containerDefinitions": [
    {
      "name": "backend",
      "image": "123456789012.dkr.ecr.ap-south-1.amazonaws.com/bus-ticket-backend:v1",
      "portMappings": [{ "containerPort": 8080, "protocol": "tcp" }],
      "essential": true,
      "environment": [
        { "name": "SPRING_PROFILES_ACTIVE", "value": "prod" }
      ],
      "secrets": [
        {
          "name": "SPRING_DATASOURCE_URL",
          "valueFrom": "arn:aws:ssm:ap-south-1:123456789012:parameter/busticket/prod/db-url"
        },
        {
          "name": "SPRING_DATASOURCE_USERNAME",
          "valueFrom": "arn:aws:ssm:ap-south-1:123456789012:parameter/busticket/prod/db-username"
        },
        {
          "name": "SPRING_DATASOURCE_PASSWORD",
          "valueFrom": "arn:aws:ssm:ap-south-1:123456789012:parameter/busticket/prod/db-password"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/bus-ticket",
          "awslogs-region": "ap-south-1",
          "awslogs-stream-prefix": "backend"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/actuator/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

Register:

```bash
aws ecs register-task-definition --cli-input-json file://backend-taskdef.json
```

### Step 5: Task definition for frontend

Save as `frontend-taskdef.json`:

```json
{
  "family": "bus-ticket-frontend",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/busTicketTaskRole",
  "containerDefinitions": [
    {
      "name": "frontend",
      "image": "123456789012.dkr.ecr.ap-south-1.amazonaws.com/bus-ticket-frontend:v1",
      "portMappings": [{ "containerPort": 8081, "protocol": "tcp" }],
      "essential": true,
      "environment": [
        { "name": "BACKEND_BASE_URL", "value": "http://bus-ticket-alb-1234.ap-south-1.elb.amazonaws.com" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/bus-ticket",
          "awslogs-region": "ap-south-1",
          "awslogs-stream-prefix": "frontend"
        }
      }
    }
  ]
}
```

```bash
aws ecs register-task-definition --cli-input-json file://frontend-taskdef.json
```

**Note:** `BACKEND_BASE_URL` points to the ALB DNS (we'll create below). Round-trip: frontend → ALB → backend target group → backend task.

### Step 6: Create the ALB

1. EC2 Console → **Load Balancers** → **Create Load Balancer** → Application.
2. Name: `bus-ticket-alb`.
3. Scheme: internet-facing.
4. IP: IPv4.
5. VPC: default. Select 2+ subnets in 2 AZs.
6. Security group: create `alb-sg` allowing TCP 80 and 443 from `0.0.0.0/0`.
7. Listener: HTTP:80 → **(we'll finish target groups next)**.

### Step 7: Target groups

**Backend TG:**
1. EC2 → Target groups → Create.
2. Type: **IP addresses** (required for Fargate awsvpc mode).
3. Name: `tg-backend`.
4. Protocol: HTTP, port 8080.
5. VPC: default.
6. Health check path: `/actuator/health`.
7. Create.

**Frontend TG:** same but name `tg-frontend`, port 8081, health check `/`.

### Step 8: Listener rules

Back to ALB → Listeners → HTTP:80 → Rules:

- Rule 1: `Path is /api/*` → Forward to `tg-backend`.
- Rule 2: `Path is /actuator/*` → Forward to `tg-backend`.
- Default: Forward to `tg-frontend`.

**Why two target groups and path routing?** The frontend calls the backend via the same public ALB URL. Cleaner than exposing two domains.

Correction for `BACKEND_BASE_URL`: update it in the frontend task def to the ALB DNS without path — the frontend code should prepend `/api` to its calls. If your frontend expects backend at `http://backend:8080`, change it to `http://<alb-dns>` and route.

### Step 9: SG for ECS tasks

Create `ecs-tasks-sg`:
- Inbound: TCP 8080 from `alb-sg`, TCP 8081 from `alb-sg`.
- Outbound: all.

Also add `ecs-tasks-sg` as source on `rds-busticket-sg` for 3306.

### Step 10: Create ECS Services

```bash
aws ecs create-service \
  --cluster bus-ticket \
  --service-name backend-svc \
  --task-definition bus-ticket-backend \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-aaa,subnet-bbb],securityGroups=[sg-ecs-tasks],assignPublicIp=ENABLED}" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:...:targetgroup/tg-backend/...,containerName=backend,containerPort=8080"
```

```bash
aws ecs create-service \
  --cluster bus-ticket \
  --service-name frontend-svc \
  --task-definition bus-ticket-frontend \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-aaa,subnet-bbb],securityGroups=[sg-ecs-tasks],assignPublicIp=ENABLED}" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:...:targetgroup/tg-frontend/...,containerName=frontend,containerPort=8081"
```

Watch tasks spin up:

```bash
aws ecs list-tasks --cluster bus-ticket --service-name backend-svc
aws ecs describe-tasks --cluster bus-ticket --tasks <task-arn>
```

After a minute:

```
lastStatus: RUNNING
healthStatus: HEALTHY
```

### Step 11: Access via ALB

Browse to the ALB DNS: `http://bus-ticket-alb-1234567.ap-south-1.elb.amazonaws.com`.

The frontend shows up. Network calls to `/api/*` go to backend. 

### Step 12: Scaling

```bash
aws ecs update-service \
  --cluster bus-ticket \
  --service backend-svc \
  --desired-count 4
```

Want **auto-scaling**? Register a scalable target:

```bash
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/bus-ticket/backend-svc \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 10

aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/bus-ticket/backend-svc \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name cpu-target \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 60.0,
    "PredefinedMetricSpecification": {"PredefinedMetricType": "ECSServiceAverageCPUUtilization"}
  }'
```

Now ECS scales backend between 2 and 10 tasks targeting 60% CPU.

### Step 13: Rolling deployments

When you build `v2`:

```bash
docker build -t 123...dkr.ecr.ap-south-1.amazonaws.com/bus-ticket-backend:v2 .
docker push ...:v2
```

Update task def:

```bash
# Edit image line in backend-taskdef.json to v2
aws ecs register-task-definition --cli-input-json file://backend-taskdef.json
aws ecs update-service \
  --cluster bus-ticket \
  --service backend-svc \
  --task-definition bus-ticket-backend
```

ECS starts new tasks with `v2`, waits for them to be healthy, drains old ones. Zero downtime (if min healthy % and max % configured).

### Step 14: CloudWatch alarms

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name backend-unhealthy \
  --metric-name UnHealthyHostCount \
  --namespace AWS/ApplicationELB \
  --statistic Average \
  --period 60 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 2 \
  --dimensions Name=TargetGroup,Value=<tg-backend-arn-suffix> Name=LoadBalancer,Value=<alb-arn-suffix> \
  --alarm-actions <sns-topic-arn>
```

### Cost breakdown (2 backend + 2 frontend, always on)

| Component | Spec | Monthly |
|---|---|---|
| Backend Fargate | 2 × 0.5 vCPU, 1 GB | ~$30 |
| Frontend Fargate | 2 × 0.25 vCPU, 0.5 GB | ~$15 |
| ALB | 1 | ~$16 |
| RDS db.t3.micro | 1 | Free (first year) or ~$15 |
| CloudWatch logs | Few GB | ~$2 |
| **Total** | | **~$63/mo (after free tier)** |

### Option 3 — Done

You've reached the **real production pattern** most startups use.

---

## Part C8: Option 4 — Deploy on EKS (Step-by-Step Overview)

This is the "enterprise" option. Remember: **$73/mo just for the control plane.** Don't leave it running overnight by accident.

### Prerequisites

Install:
- `kubectl`
- `eksctl`
- `helm`
- `aws` CLI (already have)

```bash
# On Amazon Linux 2023 / Ubuntu
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Step 1: Create the cluster

```bash
eksctl create cluster \
  --name bus-ticket \
  --region ap-south-1 \
  --version 1.29 \
  --nodes 2 \
  --node-type t3.small \
  --managed
```

**Expect ~15 minutes.** This command provisions:
- VPC (new)
- EKS control plane ($73/mo starts now)
- 2 worker nodes (t3.small, ~$15/mo each)
- IAM roles and OIDC provider

### Step 2: Configure kubectl

`eksctl` already did this, but if you clear your config:

```bash
aws eks update-kubeconfig --name bus-ticket --region ap-south-1
kubectl get nodes
```

Should list 2 Ready nodes.

### Step 3: Re-use your Section B YAMLs

Section B defined Deployments, Services, ConfigMaps. On EKS, just swap the image references to ECR URIs:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  selector:
    matchLabels: { app: backend }
  template:
    metadata:
      labels: { app: backend }
    spec:
      containers:
      - name: backend
        image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/bus-ticket-backend:v1
        ports: [{ containerPort: 8080 }]
        env:
        - name: SPRING_DATASOURCE_URL
          value: "jdbc:mysql://bus-ticket-mysql.xyz.ap-south-1.rds.amazonaws.com:3306/busticketbooking"
        - name: SPRING_DATASOURCE_USERNAME
          valueFrom: { secretKeyRef: { name: db-creds, key: username } }
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom: { secretKeyRef: { name: db-creds, key: password } }
---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector: { app: backend }
  ports: [{ port: 80, targetPort: 8080 }]
  type: ClusterIP
```

ECR pulls **just work** because EKS managed node groups have `AmazonEC2ContainerRegistryReadOnly` attached.

Create secret:

```bash
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password='S0meL0ng#Passw0rd'
```

Apply:

```bash
kubectl apply -f backend.yaml
kubectl apply -f frontend.yaml
kubectl get pods
```

### Step 4: Install AWS Load Balancer Controller

This lets you create `Ingress` resources that spawn real ALBs.

```bash
# Associate IAM OIDC provider
eksctl utils associate-iam-oidc-provider \
  --cluster bus-ticket \
  --approve \
  --region ap-south-1

# Create IAM policy
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# Create service account bound to IAM role (IRSA)
eksctl create iamserviceaccount \
  --cluster bus-ticket \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::123456789012:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Helm install
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=bus-ticket \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### Step 5: Create an ALB Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bus-ticket-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
  - http:
      paths:
      - path: /api
        pathType: Prefix
        backend: { service: { name: backend, port: { number: 80 } } }
      - path: /
        pathType: Prefix
        backend: { service: { name: frontend, port: { number: 80 } } }
```

```bash
kubectl apply -f ingress.yaml
kubectl get ingress bus-ticket-ingress
# ADDRESS column gives ALB DNS
```

### Step 6: IRSA — briefly

**IAM Roles for Service Accounts** maps K8s service accounts to IAM roles. Used above for the ALB controller. Next use case: your backend pod wants to read SSM.

```bash
eksctl create iamserviceaccount \
  --name backend-sa \
  --namespace default \
  --cluster bus-ticket \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonSSMReadOnlyAccess \
  --approve
```

Then in your Deployment:

```yaml
spec:
  template:
    spec:
      serviceAccountName: backend-sa
      containers: ...
```

Now your Java code using AWS SDK's default credential provider **automatically** gets temporary creds for SSM. No access keys in env.

### Step 7: ExternalDNS (optional)

Maps Ingress/Service to Route 53 records automatically.

```bash
helm install external-dns bitnami/external-dns \
  --set provider=aws \
  --set aws.region=ap-south-1 \
  --set domainFilters={yourdomain.in} \
  --set policy=sync
```

Now adding `external-dns.alpha.kubernetes.io/hostname: busticket.yourdomain.in` annotation to your Ingress auto-creates the DNS record.

### Step 8: Teardown (very important)

```bash
eksctl delete cluster --name bus-ticket --region ap-south-1
```

Wait ~10 minutes. Double-check:
- CloudFormation console — all `eksctl-bus-ticket-*` stacks deleted.
- EC2 — no leftover ALBs.
- Remember to also delete RDS if it was the only user of that cluster.

**If you forget this cluster for a month: $73 gone.** Set a calendar reminder.

### Option 4 — Done (and deleted!)

You've seen the EKS workflow. In interviews, you can now speak confidently about IRSA, ALB Controller, Helm, and why this is overkill for a 2-service app.

---

## Part C9: Common AWS Deployment Problems + Solutions

A buffet of things that **will** go wrong. Memorize the common ones.

| # | Problem | Where | Cause | Fix |
|---|---|---|---|---|
| 1 | `no basic auth credentials` on `docker push` | ECR | 12h token expired | `aws ecr get-login-password` again |
| 2 | EC2 can't reach RDS | Option 2 | RDS SG doesn't allow EC2 SG | Add inbound `3306` from EC2 SG |
| 3 | Container OOMKilled on Fargate | Option 3 | Task memory too low | Bump `memory` in taskdef (e.g. 1024 → 2048) |
| 4 | CloudWatch logs empty | Option 3 | `logs:CreateLogStream` missing | Add `AmazonECSTaskExecutionRolePolicy` to execution role |
| 5 | ALB 502 Bad Gateway | Option 3 | Target group health check path wrong | Change to `/actuator/health` |
| 6 | ALB 503 Service Unavailable | Option 3 | No healthy targets | Check task health in console |
| 7 | EKS nodes not joining | Option 4 | IAM role missing | Ensure `AmazonEKSWorkerNodePolicy`, `AmazonEC2ContainerRegistryReadOnly`, `AmazonEKS_CNI_Policy` |
| 8 | `ImagePullBackOff` on EKS | Option 4 | Node role missing ECR read, or IRSA SA wrong | Attach `AmazonEC2ContainerRegistryReadOnly` |
| 9 | RDS connection timeout | Option 2/3/4 | Wrong VPC, missing SG rule, or `Publicly Accessible=No` from outside | Check route tables + SGs |
| 10 | Bill suddenly $40 | Any | Forgot NAT Gateway, EIP, EKS control plane | Billing Console → Cost Explorer → group by Service |
| 11 | Elastic IP still billed | Any | EIP not associated | Release unused EIPs |
| 12 | Fargate task stuck in PENDING | Option 3 | Task exec role missing, or no subnet capacity | Check ECS events tab |
| 13 | HTTPS cert won't work | Any | ACM cert in wrong region (must match ALB) | Request cert in the ALB's region |
| 14 | Route 53 change not visible | Any | DNS TTL / propagation | Wait up to 48h or lower TTL |
| 15 | Secrets Manager rotation breaks app | Any | App caches old creds | Use `DescribeSecret` VersionStage `AWSCURRENT` on each call |
| 16 | Task def revision not used | Option 3 | Didn't `update-service --task-definition` | Re-run update-service |
| 17 | ECR scan blocks pull | Any | No — scan is informational | But set policy if you want to enforce |
| 18 | `caching_sha2_password` auth fails | RDS | Old MySQL client on EC2 | Use MariaDB client 10.5+ or connector/J 8.x |
| 19 | High latency from users | Any | Region is us-east-1 but users in India | Move to `ap-south-1` (Mumbai) |
| 20 | Billing alarms not firing | Any | Alarm region must be us-east-1 for Billing metrics | Create alarm in `us-east-1` |
| 21 | Terraform drift | Any | Console changes by teammate | `terraform plan` often; lock console access |
| 22 | Data egress charges | Any | Frontend pulls images from S3 cross-region | Keep everything in one region |
| 23 | NAT Gateway bill | Option 3/4 | Private subnet reaches internet via NAT | Use VPC endpoints for ECR, S3, SSM |
| 24 | EKS upgrade stuck | Option 4 | Node AMI version mismatch | Upgrade managed node group version after control plane |
| 25 | ECS service won't deploy new task | Option 3 | Deployment circuit breaker tripped | Check `service events`; may need manual rollback |
| 26 | Pod eviction on EKS | Option 4 | Node disk / mem pressure | Add more nodes or right-size |
| 27 | RDS storage full | Option 2/3/4 | Old binlogs, no storage autoscaling | Enable storage autoscaling (set limit) |
| 28 | Can't SSH to EC2 | Option 1/2 | SG rule removed, wrong key | Check SG and key; use EC2 Instance Connect as fallback |
| 29 | Free tier expired surprise | Any | 12 months passed | Set billing alarm + Cost Explorer |
| 30 | SSM Parameter not found | Any | Wrong region | `--region` must match task region |

### Deep dive: caching_sha2_password

MySQL 8 default auth is `caching_sha2_password`. Old JDBC drivers fail with:

```
Authentication plugin 'caching_sha2_password' cannot be loaded
```

Fix: use MySQL Connector/J 8.0.30+. Our Spring Boot starter-parent already uses 8.x, so this only bites if you downgraded.

Alternative (not recommended): change user in RDS:
```sql
ALTER USER 'admin' IDENTIFIED WITH mysql_native_password BY 'S0meL0ng#Passw0rd';
```

### Deep dive: NAT Gateway cost avoidance

If your ECS tasks are in **private** subnets and need to pull from ECR, you have three choices:

1. **Put them in public subnets with `assignPublicIp=ENABLED`** — no NAT needed. (What we did.)
2. **Use VPC endpoints** for `ecr.dkr`, `ecr.api`, `s3` (for image layers), `logs`, `ssm`. Endpoints cost $0.01/hr each but no egress charge.
3. **Pay for NAT Gateway** at $32/mo + data.

For a learning project, #1 is fine. For real prod with private tasks, #2 is the pattern.

### Deep dive: Billing alarm setup

**Billing metrics only exist in `us-east-1`** — AWS quirk.

```bash
# Enable Billing alerts in Account Preferences first.

aws cloudwatch put-metric-alarm \
  --region us-east-1 \
  --alarm-name billing-5usd \
  --metric-name EstimatedCharges \
  --namespace AWS/Billing \
  --statistic Maximum \
  --period 21600 \
  --threshold 5.0 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --dimensions Name=Currency,Value=USD \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:billing-alerts
```

Set this **before** your first EC2. Future-you will thank present-you.

### Deep dive: Region latency

From Pune/Mumbai, ping times:

| Region | Latency |
|---|---|
| ap-south-1 (Mumbai) | 10–20 ms |
| ap-southeast-1 (Singapore) | 60–70 ms |
| us-east-1 (Virginia) | 230–260 ms |
| eu-west-1 (Ireland) | 150–170 ms |

For Indian users, **always** default to `ap-south-1` unless forced otherwise.

---

## Part C10: Cleanup Checklist

Before you go on vacation, study break, or hand over the project, **run through this list**. Takes 15 minutes, saves thousands of rupees.

```text
[ ] 1. EC2 instances        → terminate (not stop) if not needed
[ ] 2. RDS instances        → take final snapshot or skip, then delete
[ ] 3. ALBs                 → delete (both listeners and target groups)
[ ] 4. Target groups        → delete
[ ] 5. ECR repositories     → delete (or leave if under 500 MB free tier)
[ ] 6. ECS services         → update --desired-count 0, then delete service
[ ] 7. ECS clusters         → delete
[ ] 8. EKS clusters         → eksctl delete cluster (HUGE — $73/mo if forgotten)
[ ] 9. Elastic IPs          → release if unassigned
[ ] 10. EBS volumes         → delete orphaned volumes (left from terminated EC2)
[ ] 11. EBS snapshots       → delete old snapshots
[ ] 12. RDS snapshots       → delete old ones
[ ] 13. S3 buckets          → empty and delete
[ ] 14. CloudWatch log groups → delete (or set retention to 7 days)
[ ] 15. SNS topics          → delete unused
[ ] 16. Secrets Manager     → delete secrets (7-day recovery window default)
[ ] 17. NAT Gateways        → delete if any
[ ] 18. VPC endpoints       → delete
[ ] 19. CloudFormation stacks → delete leftover eksctl stacks
[ ] 20. Billing Console     → check current month charges
[ ] 21. Billing alarms      → keep these! they're free
```

**Verification command blast:**

```bash
# What's still running?
aws ec2 describe-instances --query 'Reservations[].Instances[?State.Name==`running`].[InstanceId,InstanceType]' --output table
aws rds describe-db-instances --query 'DBInstances[].[DBInstanceIdentifier,DBInstanceStatus]' --output table
aws eks list-clusters
aws ecs list-clusters
aws elbv2 describe-load-balancers --query 'LoadBalancers[].[LoadBalancerName,State.Code]' --output table
aws ec2 describe-addresses --query 'Addresses[].[PublicIp,InstanceId]' --output table
```

If any of those return data unexpectedly, investigate before going to sleep.

---

## Part C11: AWS Viva Questions

80+ viva-ready Q&As, grouped by topic. Each answer ties back to the bus-ticket project where possible.

### Group 1 — Basics (Q1–Q15)

**Q1. What is AWS?**
AWS (Amazon Web Services) is Amazon's cloud platform offering on-demand compute, storage, database, networking, and developer services over the internet. You rent resources per hour/second instead of buying hardware. For our bus-ticket project, we rented a VM (EC2), a database (RDS), and an image registry (ECR).

**Q2. Differentiate IaaS, PaaS, and SaaS with examples.**
IaaS ("Infrastructure as a Service") gives raw VMs — you install the OS stack. Example: EC2. PaaS ("Platform as a Service") gives a runtime — you upload code. Example: Elastic Beanstalk. SaaS ("Software as a Service") gives a finished app — you just log in. Example: Gmail. Our Option 1 uses IaaS (EC2). Our Option 3 (Fargate) is arguably PaaS because we stop worrying about VMs.

**Q3. What is a region vs an availability zone?**
A region is a geographic area (e.g. Mumbai = `ap-south-1`). An AZ is an isolated data center within a region (Mumbai has 3 AZs: `ap-south-1a`, `1b`, `1c`). AZs inside a region have low-latency private links. We chose `ap-south-1` for our bus-ticket users in India.

**Q4. What is the AWS Free Tier and what does it cover?**
A 12-month offer giving new accounts 750 hours/month of t2.micro EC2, 750 hours of db.t3.micro RDS, 5 GB S3, 500 MB ECR, 1M Lambda invocations. Used smartly, our entire Option 1 and Option 2 run free. It does **not** cover ALB, NAT Gateway, or EKS control plane.

**Q5. What is the AWS shared responsibility model?**
AWS secures **the cloud** (physical DC, hypervisor, global network). You secure **in the cloud** (OS patches on EC2, IAM policies, encryption config, app security). For managed services like RDS, AWS takes over more — still, DB credentials and SQL injection are your problem.

**Q6. What's the difference between `aws configure` profiles and IAM roles?**
`aws configure` puts long-lived access keys into `~/.aws/credentials`. An IAM role is assumed temporarily — ideal for EC2 instances, ECS tasks, Lambda. Roles are safer because there's no key to leak. In Option 1 we attached role `ec2-ecr-read` to the EC2 so Docker could pull from ECR without any key.

**Q7. What is the root user and should you use it?**
The email you signed up with. It has total control. Use it **only** to create an IAM user for yourself, enable MFA, and then **never** again. Bill alarms and day-to-day work should use the IAM user.

**Q8. Name three ways to interact with AWS.**
Console (web), CLI (`aws`), SDKs (boto3, aws-sdk-java). We used console for quick lookups and CLI for repeatable commands like `aws ecr create-repository`.

**Q9. What is an AMI?**
Amazon Machine Image — a template containing OS, pre-installed software, and config. When you launch an EC2, you pick an AMI. We picked Amazon Linux 2023 for our bus-ticket server.

**Q10. What's an ARN?**
Amazon Resource Name — the unique global identifier for any AWS resource. Format: `arn:aws:<service>:<region>:<account>:<resource>`. Example: `arn:aws:ecr:ap-south-1:123456789012:repository/bus-ticket-backend`.

**Q11. What are AWS SDKs?**
Libraries that wrap AWS APIs. `boto3` (Python), `aws-sdk-java-v2`, `aws-sdk-js`. The backend could use `aws-sdk-java` to fetch secrets from Parameter Store at boot.

**Q12. What is Cost Explorer?**
Dashboard showing where your money is going: by service, region, tag. Enable it in Billing preferences. Check it weekly during the first month.

**Q13. What is AWS Organizations?**
A way to manage multiple AWS accounts under one umbrella (linked billing, SCPs). Not needed for a college project; big companies use it to give each team its own account.

**Q14. What's CloudShell?**
A free browser-based terminal with AWS CLI pre-installed. Great when your laptop Wi-Fi is slow — CloudShell runs inside AWS, so pushes/pulls are fast.

**Q15. Tags — why do they matter?**
Key-value labels on resources (e.g. `Project=bus-ticket`, `Env=prod`). Useful for cost allocation (Cost Explorer groups by tag) and bulk actions. Tag everything.

### Group 2 — EC2 + Networking (Q16–Q30)

**Q16. EC2 vs Fargate vs Lambda — when do you use each?**
EC2 = long-running server, full control, you manage OS. Fargate = container without managing VM, good for web apps. Lambda = per-request compute, max 15 min run, good for small API handlers. Our backend is long-running Spring Boot — EC2 or Fargate fit; Lambda would require cold-start handling and state rework.

**Q17. What are EC2 instance families?**
`t` (burstable, good for dev), `m` (general purpose), `c` (CPU-heavy), `r` (RAM-heavy), `g/p` (GPU), `i` (NVMe-heavy). We used `t2.micro` / `t3.small` — cheap and general.

**Q18. Difference between t2 and t3?**
Both burstable. t3 is newer (Nitro-based), has unlimited mode default, slightly better price/performance. t3.micro is *not* always free-tier in every account; check your free tier eligibility.

**Q19. What's the difference between a Security Group and a NACL?**
SG = per-instance, stateful (return traffic allowed automatically), allow rules only. NACL = per-subnet, stateless, allow and deny rules. For our bus-ticket project, we used only SGs.

**Q20. What is a VPC?**
Virtual Private Cloud — your own isolated network in AWS. Has CIDR (e.g. 10.0.0.0/16), subnets, route tables, IGW/NAT. Every AWS account has a default VPC.

**Q21. Public subnet vs private subnet?**
Public subnet has a route to IGW in its route table → resources get internet. Private has no IGW route, so it's only reachable from inside VPC. RDS lives in private ideally; our EC2 is in public.

**Q22. What's an Internet Gateway?**
A VPC-level component that lets public subnets talk to the internet. Free. One per VPC.

**Q23. What's a NAT Gateway and what does it cost?**
Allows private subnet instances to initiate **outbound** internet traffic (e.g. OS updates) without being reachable from the internet. $32/mo + $0.045/GB. A common cause of bill surprise.

**Q24. What is Elastic IP?**
Static public IPv4 owned by your account. Free while attached to a running instance, charged when unattached. We attached one to our bus-ticket EC2 so the IP survives reboots.

**Q25. EBS vs Instance Store?**
EBS = network-attached block storage, persists independently of instance. Instance Store = physical disks on host, wiped when instance stops. EBS is default for most workloads.

**Q26. What is gp3 vs gp2 storage?**
Both general-purpose SSD. gp3 is newer — lets you pay for IOPS independently of size. Use gp3.

**Q27. How do you SSH to an EC2?**
`ssh -i key.pem ec2-user@<public-ip>`. Key must have `chmod 400`. Your SG must allow 22 from your IP.

**Q28. What is EC2 Instance Connect?**
Browser-based SSH without needing .pem file. Works when SG allows 22 from AWS's IC IP range. Good for emergency access.

**Q29. What is a key pair?**
SSH public/private key. Public part lives on EC2 (`~/.ssh/authorized_keys`). Private (.pem) stays with you. **Cannot be recovered if lost.**

**Q30. What happens if you stop vs terminate an EC2?**
Stop = shutdown, disk preserved, no compute bill (still pay EBS). Terminate = delete, no recovery. We terminated our test instances after the demo.

### Group 3 — Container Services (Q31–Q50)

**Q31. ECR vs Docker Hub?**
ECR is private by default, IAM-secured, same region as your compute (fast pulls), per-GB billed. Docker Hub is public-first, rate-limited for free users. For production, ECR wins.

**Q32. How do you log in to ECR from Docker?**
`aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin <accountid>.dkr.ecr.ap-south-1.amazonaws.com`. Token lasts 12 hours.

**Q33. ECS vs EKS?**
ECS is AWS-native and simpler; tightly integrated with ALB, IAM. EKS is managed Kubernetes; portable across clouds, steeper learning curve, $73/mo control plane. For two services, ECS.

**Q34. Fargate vs EC2 launch type in ECS?**
Fargate = no VM to manage, pay per task CPU/RAM per second. EC2 = you pre-provision EC2 as cluster nodes, cheaper if 24/7 utilization is high. Fargate is the Uber; EC2 is owning the car.

**Q35. What is a Task Definition?**
A JSON blueprint in ECS describing which container image, CPU, memory, ports, env vars, logs, IAM role. Like a Kubernetes Pod spec. Each change produces a new revision.

**Q36. What is a Service in ECS?**
A long-running wrapper around Tasks: ensures desired count is running, does rolling updates, registers tasks with ALB. Analogous to a K8s Deployment.

**Q37. What is a Cluster in ECS?**
A logical namespace for services + tasks. For Fargate, it's just a label; for EC2 launch type, it's the group of registered EC2 instances.

**Q38. What's IRSA?**
IAM Roles for Service Accounts (EKS-specific). Lets a K8s pod assume an IAM role via OIDC. So your code uses AWS SDK which transparently gets temporary AWS creds mapped from the pod's service account.

**Q39. How does Fargate pricing work?**
Per second of vCPU and per second of GB RAM reserved by the task. Smallest unit: 0.25 vCPU + 0.5 GB. Example: 0.5 vCPU + 1 GB running 24/7 ~ $15/mo.

**Q40. What are `executionRoleArn` and `taskRoleArn`?**
Execution role: used by ECS agent to pull image from ECR and write logs to CloudWatch. Task role: used by your app code to call AWS APIs (e.g. read SSM).

**Q41. How do you deploy a new image version on ECS?**
Build new image → push to ECR with new tag → register new task def revision pointing at new tag → `aws ecs update-service --task-definition bus-ticket-backend`. ECS does rolling deploy.

**Q42. What's a rolling deployment?**
New tasks spin up; once healthy, old tasks drain. Zero downtime. ECS default deploy strategy.

**Q43. What's blue/green deployment?**
Two full environments (blue = current, green = new). Switch the ALB listener to green once tested. Easy rollback. ECS supports via CodeDeploy.

**Q44. What does `CannotPullContainerError` mean?**
ECS couldn't pull the image. Causes: ECR tag missing, execution role lacks ECR permissions, networking (no NAT / VPC endpoints), image scan blocking.

**Q45. Explain task `awsvpc` network mode.**
Each task gets its own ENI (elastic network interface) and IP in a subnet. Required for Fargate. Enables per-task security groups.

**Q46. How do you pass secrets to an ECS task?**
In task definition `secrets[]`, referencing SSM Parameter Store or Secrets Manager ARNs. ECS injects them as env vars at task start. Execution role needs `ssm:GetParameters` or `secretsmanager:GetSecretValue`.

**Q47. What is CloudMap / ECS Service Discovery?**
Automatic DNS entries per service so tasks find each other by name (e.g. `backend.bus-ticket.local`). Alternative to ALB-based discovery.

**Q48. How do health checks work in ECS?**
Each container can have a command-based healthcheck. ALB target group also health-checks. If ALB fails 3 consecutive checks, target deregisters.

**Q49. How does EKS differ from vanilla K8s?**
Control plane is managed by AWS (you can't SSH to masters). Networking uses AWS VPC CNI (pods get VPC IPs). IAM integration via IRSA. Otherwise same kubectl experience.

**Q50. How do you decide ECS vs EKS for our bus-ticket app?**
Two containers, small team, no existing K8s knowledge → **ECS Fargate**. If we had 30 microservices and engineers who already wrote Helm charts → EKS.

### Group 4 — Databases & Storage (Q51–Q60)

**Q51. RDS vs running MySQL yourself on EC2?**
RDS: automatic backups, patching, failover, monitoring. EC2-hosted MySQL: cheaper, more control, more ops. For production we always use RDS.

**Q52. What is Multi-AZ RDS?**
A synchronous standby replica in another AZ. On primary failure, DNS flips to standby in ~60s. Doubles cost. Off for free tier; on for prod.

**Q53. What's a read replica?**
Async replica that serves read queries. Used to scale reads. Up to 5 per primary. Different from Multi-AZ (which is for HA, not scale).

**Q54. S3 storage classes?**
Standard (hot), IA (infrequent access, cheaper storage, retrieval fee), Glacier Instant / Flexible / Deep Archive (archive). Lifecycle rules can auto-migrate.

**Q55. EBS vs EFS vs S3?**
EBS = block storage for a single EC2 (like a disk). EFS = shared file system (NFS) mountable by many. S3 = object storage. Our MySQL data on RDS uses EBS underneath.

**Q56. RDS endpoint — what is it?**
DNS name like `bus-ticket-mysql.xyz.ap-south-1.rds.amazonaws.com`. Always use this instead of IP — during failover the DNS flips to standby.

**Q57. How do you back up RDS?**
Automated backups (daily, retention up to 35 days) and manual snapshots. Both go to S3 internally. Deleted on DB delete unless you chose "retain snapshot."

**Q58. What is Aurora?**
AWS's custom MySQL/Postgres-compatible DB engine. Faster, auto-scaling storage, cross-region replicas. More expensive than vanilla RDS MySQL. Overkill for us.

**Q59. RDS Performance Insights?**
Graphical tool showing DB load by wait event. Free for 7-day retention. Turn it on.

**Q60. How do you connect from EC2 to RDS securely?**
Both in same VPC. RDS SG allows 3306 from EC2's SG. RDS has `Publicly Accessible=No`. App uses RDS endpoint + encrypted JDBC (`useSSL=true`).

### Group 5 — Load Balancing & DNS (Q61–Q70)

**Q61. ALB vs NLB vs CLB?**
ALB (Application) = L7, HTTP/HTTPS, path/host routing, WebSockets. NLB (Network) = L4, TCP/UDP, ultra-high performance. CLB (Classic) = legacy, avoid. Our bus-ticket uses ALB.

**Q62. What is a target group?**
A set of targets (IPs or EC2 instances) that an ALB routes to. Has its own health check config. We had `tg-backend` (8080) and `tg-frontend` (8081).

**Q63. What are listener rules?**
Conditions that map to target groups. E.g. `Path /api/*` → `tg-backend`. Evaluated in priority order.

**Q64. ALB health check parameters?**
Path (`/actuator/health`), interval (30s), timeout (5s), healthy threshold (2), unhealthy threshold (3), matcher (200-299). Tune based on app start time.

**Q65. What is Route 53?**
AWS DNS. Hosts zones, answers queries. $0.50/zone/month. Supports ALIAS records pointing to ALBs without an extra hop.

**Q66. Route 53 record types we care about?**
A (IPv4), AAAA (IPv6), CNAME (alias to another name), ALIAS (AWS-only, to ALB/CloudFront/S3), MX (mail), TXT (verification).

**Q67. A vs CNAME vs ALIAS?**
A = name → IP. CNAME = name → another name (can't be on root domain). ALIAS = AWS-specific, acts like A but points to AWS resource, free, supports apex.

**Q68. What are Route 53 health checks?**
Route 53 can ping an endpoint and return different DNS responses based on health — e.g. failover routing to secondary region.

**Q69. Certificate for HTTPS — how?**
AWS Certificate Manager (ACM) issues free certs. Attach to ALB listener on 443. Cert must be in same region as the ALB (with one exception: CloudFront requires us-east-1).

**Q70. What is sticky session on ALB?**
ALB uses a cookie to route a client to the same target repeatedly. Useful for apps with in-memory session state. Our Spring Boot app should use external session (Redis/DB) instead; sticky sessions are a crutch.

### Group 6 — Security & IAM (Q71–Q80)

**Q71. User vs Role in IAM?**
User = long-term identity for a human. Role = temporary identity assumed by users or services. Roles are credentials-free (no access keys).

**Q72. What is a policy?**
JSON document with `Effect`, `Action`, `Resource`, `Condition` statements describing permissions. Attached to users/roles/groups.

**Q73. Managed vs inline policy?**
Managed = reusable, versioned, attached to many principals. Inline = embedded, unique to one principal. Prefer managed.

**Q74. IAM best practices?**
Enable MFA on root. Avoid root for daily use. Grant least privilege. Rotate access keys. Use roles for services. Turn on CloudTrail. Tag users.

**Q75. Secrets Manager vs Parameter Store vs env vars?**
Env vars in code/repo = worst. Parameter Store = free, great for most secrets, manual rotation. Secrets Manager = paid but has automatic rotation (especially for RDS passwords). We used Parameter Store.

**Q76. How did we secure the RDS password?**
Stored as SecureString in Parameter Store. EC2 role has `AmazonSSMReadOnlyAccess`. On startup script pulls with `aws ssm get-parameter --with-decryption`. App only sees the env var.

**Q77. What is KMS?**
Key Management Service — stores and rotates encryption keys. Used under the hood by Secrets Manager, SSM SecureString, S3 encryption, EBS encryption.

**Q78. MFA — what is it and why?**
Multi-Factor Auth — second factor beyond password (e.g. TOTP app). Protects from stolen creds. Enable on root immediately.

**Q79. CloudTrail — what is it?**
Audit log of every AWS API call. Useful for "who deleted the RDS last Tuesday?" 90 days free in console.

**Q80. What is AWS WAF?**
Web Application Firewall that sits in front of ALB / CloudFront. Blocks SQL injection, XSS, bot traffic. Useful if bus-ticket gets scraped.

### Group 7 — Connected / Jumbling (Q81–Q90)

**Q81. How does Section A Dockerfile interact with ECR?**
Dockerfile produces a local image. `docker tag` adds an ECR-shaped tag. `docker login` with ECR token. `docker push` uploads layers to ECR. ECS/EKS/EC2 later pulls those same layers using a role-based authentication handshake.

**Q82. How does Section B Kubernetes Deployment YAML map to an ECS Task Definition?**
Deployment `replicas` = ECS `desiredCount`. Pod `containers[].image` = taskdef `containerDefinitions[].image`. Pod `resources.requests` = taskdef `cpu/memory`. `env.valueFrom.secretKeyRef` = taskdef `secrets[].valueFrom`. `livenessProbe` = container `healthCheck`. Service = ECS service + ALB target group.

**Q83. Which option costs least for the bus-ticket project?**
Option 1 (single EC2 + Docker Compose) — essentially free on free tier. If free tier expired, ~$9/mo.

**Q84. Which option is most production-realistic?**
Option 3 (Fargate + RDS + ALB). Auto-scaling, zero-downtime deploys, no VM patching.

**Q85. Between ECS Fargate and EKS, which do you choose for two services and a 5-person team?**
ECS Fargate. EKS's $73/mo control plane and learning curve isn't justified for two services.

**Q86. You pushed a new backend image tagged v2. How does production pick it up?**
Edit task def → change image to `:v2` → `register-task-definition` → `update-service --task-definition bus-ticket-backend`. ECS performs rolling replacement.

**Q87. In Option 2, the frontend container calls the backend by which URL?**
Inside Docker Compose network it's `http://backend:8080` (service DNS). The frontend container's `BACKEND_BASE_URL` env var controls this.

**Q88. Describe the packet path: browser → bus-ticket app on Option 3.**
Browser → DNS resolves `busticket.in` (Route 53) to ALB IP → ALB evaluates listener rules → forwards `/` to `tg-frontend` → Fargate task (frontend container) receives request → frontend calls `BACKEND_BASE_URL/api/...` → ALB again → `tg-backend` → backend container → JDBC → RDS in private subnet.

**Q89. Explain exactly what happens when an ECS task exits unexpectedly.**
ECS service detects desired count > running count. Schedules a new task. New task pulls image from ECR (using execution role). Task registers with ALB target group. Health check eventually passes → traffic resumes. Typical recovery: 30–60 seconds.

**Q90. One-word answer: what region should the bus-ticket app run in for Indian users?**
`ap-south-1` (Mumbai).

---

### Bonus: Lightning Round Q91–Q100

**Q91. What does `aws configure` store, and where?**
Access key ID, secret access key, default region, output format. In `~/.aws/credentials` and `~/.aws/config`.

**Q92. Difference between `aws sts get-caller-identity` and `aws sts assume-role`?**
`get-caller-identity` returns who you currently are. `assume-role` returns new temporary creds for a different role.

**Q93. What is `AWS_PROFILE`?**
Env var naming which `[profile <name>]` block to use from `~/.aws/config`. Lets you juggle multiple accounts.

**Q94. What is the `aws ecs execute-command` equivalent of `kubectl exec`?**
Yes — you can shell into a Fargate task with `aws ecs execute-command --cluster bus-ticket --task <id> --container backend --interactive --command "/bin/sh"`. Enable `enableExecuteCommand: true` on the service first.

**Q95. What does `kubectl get pods -n kube-system` likely show on a new EKS cluster?**
CoreDNS pods, aws-node (CNI) pods, kube-proxy pods. Plus whatever you installed (ALB controller).

**Q96. What is `aws-auth` ConfigMap on EKS?**
Maps IAM users/roles to K8s RBAC groups. Edit with `eksctl create iamidentitymapping`.

**Q97. What is CDK?**
Cloud Development Kit — write infra in TypeScript/Python, synthesize to CloudFormation. Alternative to Terraform.

**Q98. What is CloudFormation drift?**
When someone changes a resource in the console that CloudFormation manages. `aws cloudformation detect-stack-drift` flags it.

**Q99. Why should we avoid storing AWS access keys in git?**
Scrapers harvest GitHub constantly for leaked keys and spin up crypto miners on your account within minutes. Use IAM roles. Use `git-secrets` pre-commit hook.

**Q100. Final — what single action prevents 90% of student bill horror stories?**
Set a **billing alarm at $5** in us-east-1 the moment you create your AWS account. The email wakes you before the damage scales.

---

## Wrap-up

You now have a complete menu for deploying the Bus Ticket Booking System on AWS:

- **Option 1** for a free demo with Docker Compose on a single EC2.
- **Option 2** upgrading MySQL to RDS for realistic data management.
- **Option 3** on Fargate for hands-off production with auto-scaling.
- **Option 4** on EKS if you're chasing the K8s experience (and budget).

Pair this with your Section A Dockerfiles and Section B Kubernetes manifests and you have an end-to-end deployment story for your viva.

Good luck — now go ship it, then *delete everything when done* so you don't wake up to a $200 bill tomorrow.
