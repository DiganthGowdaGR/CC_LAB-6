# CC Lab 6 – Containerized Backend with NGINX Load Balancer and Jenkins CI/CD

**Name:** SHARATH  
**SRN:** PES2UG24CS823

---

## Overview

This project demonstrates how to containerize a C++ HTTP backend server, deploy multiple instances of it behind an NGINX load balancer, and automate the entire build and deployment process using a Jenkins CI/CD pipeline – all using Docker.

---

## Project Structure

```
.
├── backend/
│   ├── app.cpp          # C++ HTTP server (listens on port 8080)
│   └── Dockerfile       # Docker image for the backend
├── nginx/
│   └── default.conf     # NGINX load balancer configuration
├── Dockerfile.jenkins   # Custom Jenkins Docker image with Docker & g++ support
└── Jenkinsfile          # Jenkins declarative pipeline
```

---

## Components

### 1. C++ Backend Server (`backend/app.cpp`)

A minimal HTTP server written in C++ using POSIX sockets. It:
- Listens on **port 8080**
- Responds to every HTTP request with a plain-text message identifying the container hostname:
  ```
  Served by backend: <hostname>
  ```
- Uses `SO_REUSEADDR` and `SO_REUSEPORT` socket options for clean restarts.

### 2. Backend Dockerfile (`backend/Dockerfile`)

Builds the C++ application inside an **Ubuntu 22.04** container:
1. Installs `g++`
2. Copies `app.cpp` into the image
3. Compiles the application (`g++ app.cpp -o app`)
4. Exposes port **8080** and runs the compiled binary

### 3. NGINX Load Balancer (`nginx/default.conf`)

Configures NGINX as a reverse proxy / load balancer:
- Upstream group **`backend_servers`** contains two backend containers: `backend1` and `backend2`
- Uses the **`least_conn`** load-balancing algorithm (routes new requests to the backend with the fewest active connections)
- Listens on **port 80** and forwards all traffic to the upstream group

### 4. Jenkins Pipeline (`Jenkinsfile`)

A three-stage declarative Jenkins pipeline that automates the full deployment:

| Stage | Description |
|-------|-------------|
| **Build Backend Image** | Removes any existing `backend-app` image and rebuilds it from `backend/Dockerfile` |
| **Deploy Backend Containers** | Creates a Docker network (`app-network`), removes stale containers, and starts two fresh backend containers (`backend1`, `backend2`) on that network |
| **Deploy NGINX Load Balancer** | Starts an NGINX container (`nginx-lb`) on the same network, copies the custom config into it, and reloads NGINX |

### 5. Custom Jenkins Dockerfile (`Dockerfile.jenkins`)

Extends the official `jenkins/jenkins:lts` image with:
- `docker.io` – so Jenkins can run Docker commands
- `make`, `g++`, `curl` – build and utility tools

---

## How It Works

```
Client Request (port 80)
        │
        ▼
  ┌─────────────┐
  │  nginx-lb   │  (least_conn load balancing)
  └──────┬──────┘
         │
   ┌─────┴──────┐
   ▼            ▼
backend1     backend2
(port 8080)  (port 8080)
```

1. A client sends an HTTP request to the host on **port 80**.
2. NGINX receives the request and forwards it to whichever backend has the fewest active connections.
3. The selected backend responds with `Served by backend: <container-hostname>`, making it easy to verify load balancing is working by sending multiple requests and observing different hostnames in the responses.

---

## Running Locally (without Jenkins)

> **Prerequisites:** Docker installed and running.

```bash
# 1. Build the backend image
docker build -t backend-app backend/

# 2. Create the Docker network
docker network create app-network

# 3. Start two backend containers
docker run -d --name backend1 --network app-network backend-app
docker run -d --name backend2 --network app-network backend-app

# 4. Start the NGINX load balancer
docker run -d --name nginx-lb --network app-network -p 80:80 nginx
docker cp nginx/default.conf nginx-lb:/etc/nginx/conf.d/default.conf
docker exec nginx-lb nginx -s reload

# 5. Test the load balancer
curl http://localhost
curl http://localhost
```

Each `curl` request may be served by a different backend container, demonstrating load balancing.

---

## Running via Jenkins

1. Build the custom Jenkins image:
   ```bash
   docker build -t my-jenkins -f Dockerfile.jenkins .
   ```
2. Run Jenkins:
   ```bash
   docker run -d -p 8080:8080 -v /var/run/docker.sock:/var/run/docker.sock my-jenkins
   ```
3. Open `http://localhost:8080`, configure Jenkins, create a **Pipeline** job pointing to this repository, and trigger a build.
4. The pipeline will automatically build the backend image, deploy both backend containers, and set up the NGINX load balancer.
