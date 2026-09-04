# Lab 4 — Image & Registry Management — Execution Log

## Objective

Master the complete lifecycle of a Docker image: writing efficient Dockerfiles
with all the key instructions, applying layer optimization and multi-stage builds,
inspecting and manipulating images, managing a private registry, and operating the
image CLI.

This lab builds images from scratch, measures the impact of design decisions
on image size and layer count, and publishes images to a local registry.

DCA competencies: **2.1, 2.2, 2.3, 2.4, 2.5, 2.6, 2.7, 2.8, 2.9, 2.10, 2.11, 2.12,
2.13, 2.14, 2.15, 2.16, 2.17** — all hands-on exercises except 2.16 (conceptual).

---

## DCA topics covered

| Topic | Description | Exercise |
| --- | --- | --- |
| 2.1 | Using the Dockerfile | 1 |
| 2.2 | ADD, COPY, VOLUMES, EXPOSE, ENTRYPOINT | 1 |
| 2.3 | Main parts of a Dockerfile | 1 |
| 2.4 | Efficient image: multi-stage, .dockerignore | 2, 3 |
| 2.5 | Image CLI: list, delete, prune, rmi | 8 |
| 2.6 | Inspecting images: filter and format | 4 |
| 2.7 | Image tagging | 6, 7 |
| 2.8 | Building an image from a Dockerfile | 1, 2, 3 |
| 2.9 | Layers: `docker history` | 4 |
| 2.10 | Flatten (collapsing an image into a single layer) | 5 |
| 2.11 | Registry functions | 7 |
| 2.12 | Deploying a private registry | 7 |
| 2.13 | Registry login | 7 |
| 2.14 | Registry search | 7 |
| 2.15 | Pushing an image to a registry | 7 |
| 2.16 | Image signing (cosign — conceptual) | — |
| 2.17 | Pull and delete from registry | 7, 8 |

---

## Prerequisites

- Docker Labs VM (<MANAGER_IP>) running Docker 29.4.1
- Access: `ssh -J <admin_user>@vps -p 2222 <admin_user>@localhost`
- Swarm inactive or active — does not affect this lab (works only on VM 200)
- Tools available on the VM: `curl`, `python3`

---

## Setup

```bash
# Connect to the VM
ssh <admin_user>@vdocker-labs

# Create the lab working directory
mkdir -p ~/lab4-images && cd ~/lab4-images

# Clean up state from previous labs (without affecting Swarm if active)
docker container prune -f
docker image prune -f
docker volume prune -f

# Verify clean state
docker ps -a
docker image ls
```

---

## Exercises

---

### Exercise 1: Complete Dockerfile — key instructions

**Objective:** build a Python HTTP server image covering all the relevant DCA
instructions: ARG, ENV, LABEL, WORKDIR, COPY, USER, EXPOSE, HEALTHCHECK,
ENTRYPOINT, and CMD.

**Procedure:**

```bash
cd ~/lab4-images
mkdir -p ex1-dockerfile && cd ex1-dockerfile

# Create the Python application (minimal HTTP server)
cat > server.py << 'EOF'
import http.server
import socketserver
import os

PORT = int(os.environ.get("APP_PORT", 8080))
ENV  = os.environ.get("APP_ENV", "development")

class Handler(http.server.SimpleHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        msg = f"Hello from Lab 4 | env={ENV} | port={PORT}\n"
        self.wfile.write(msg.encode())

with socketserver.TCPServer(("", PORT), Handler) as httpd:
    print(f"Server on port {PORT}, env={ENV}")
    httpd.serve_forever()
EOF

# Create the complete Dockerfile
#ARG to use in the FROM line
ARG PYTHON_VERSION=3.12-slim

#Base image: slim reduces attack surface
FROM python:${PYTHON_VERSION}

#LABEL: image metadata
LABEL maintainer="<admin_user>@lab4"
LABEL version="1.0"
LABEL description="Python HTTP server for DCA"

#ARG build-time variable (does not persist in the final image)
ARG APP_USER=appuser
ARG UID=1001

#ENV: environment variable that DOES persist in the image and containers
ENV APP_PORT=8080
ENV APP_ENV=production

#WORKDIR: creates the directory if it doesn't exist and sets the CWD for subsequent instructions
WORKDIR /app

#COPY: copies files from the build context to the image filesystem
#COPY over ADD when automatic tar extraction is not needed
COPY server.py .

#USER: run the server as a non-root user (security best practice)
#Create the user with the UID defined in ARG
RUN groupadd --gid ${UID} ${APP_USER} && \
    useradd --uid ${UID} --gid ${UID} --no-create-home ${APP_USER}

USER ${APP_USER}

#EXPOSE: documents the port the container listens on (does not publish the port)
EXPOSE ${APP_PORT}

#HEALTHCHECK: Docker periodically checks whether the service is alive
#--interval=30s: every 30 seconds
#--timeout=5s: maximum time for the check to respond
#--start-period=5s: doesn't count failures during the first 5s of startup
#--retries=3: 3 consecutive failures → unhealthy
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
  CMD python3 -c "\
import urllib.request; \
urllib.request.urlopen ('http://localhost:${APP_PORT}')\
" || exit 1
`
#ENTRYPOINT: fixed command, always executed. Cannot be overridden with docker run <args>
#CMD: default arguments for the ENTRYPOINT (these can be overridden)
ENTRYPOINT ["python3"]
CMD ["server.py"]


# Build the image with --no-cache to see all layers
docker build --no-cache -t lab4/server:1.0 .

# Verify the image was created
docker image ls lab4/server
# Expected output: lab4/server  1.0  <hash>  <time>  ~50-60 MB
IMAGE             ID             DISK USAGE   CONTENT SIZE   EXTRA
lab4/server:1.0   9d0d78b9dc58        177MB         43.2MB  
# Run the container and verify behavior
docker run -d --name srv1 -p 8080:8080 lab4/server:1.0
sleep 3

# Verify server response
curl -s http://localhost:8080
# Expected output: Hello from Lab 4 | env=production | port=8080
Hello from Lab4 | env=production | port=8080
# Verify healthcheck (may take 35s to go from starting to healthy)
docker inspect srv1 --format '{{.State.Health.Status}}'
# Expected output: healthy (after ~35s) or starting (right after start)
healthy
# Override CMD while keeping ENTRYPOINT (only changes the script executed)
docker run --rm lab4/server:1.0 --version
# Output: Python 3.12.x — python3 --version (ENTRYPOINT=python3, CMD overridden)
Python 3.12.13
# Override ENV with -e
docker run --rm -e APP_ENV=staging -e APP_PORT=9090 \
  -p 9090:9090 --name srv-staging -d lab4/server:1.0
sleep 2
curl -s http://localhost:9090
# Output: Hello from Lab 4 | env=staging | port=9090
Hello from Lab4 | env=staging | port=9090
docker rm -f srv-staging
```

**Verification:**

```bash
# The image exists
docker image ls lab4/server:1.0
# Container srv1 responds on port 8080
curl -s http://localhost:8080 | grep "Lab 4"
Hello from Lab4 | env=production | port=8080
# ENTRYPOINT and CMD are correctly defined
docker inspect lab4/server:1.0 --format 'Entrypoint: {{.Config.Entrypoint}} | Cmd: {{.Config.Cmd}}'
Entrypoint: [python3] | Cmd: [server.py]
# Expected output: Entrypoint: [python3] | Cmd: [server.py]
# The user is non-root
docker exec srv1 whoami
appuser
# Expected output: appuser

docker rm -f srv1
```

---

### Exercise 2: Layer optimization and .dockerignore

**Objective:** compare an unoptimized image with an optimized one, understand the
impact of layer ordering on the build cache, and use `.dockerignore` to reduce the
build context.

**Procedure:**

```bash
cd ~/lab4-images
mkdir ex2-layers && cd ex2-layers

# Create supporting files
cat > requirements.txt << 'EOF'
flask==3.1.0
gunicorn==23.0.0
EOF

cat > app.py << 'EOF'
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return "Lab 4 — Layer optimization\n"
EOF

# Create files that should NOT end up in the image
echo "secret123" > .env
mkdir tests && echo "# test" > tests/test_app.py
echo "*.log" > .gitignore
dd if=/dev/urandom of=dummy.bin bs=1M count=5 2>/dev/null  # 5MB junk file

# === UNOPTIMIZED IMAGE ===
cat > Dockerfile.noopt << 'EOF'
FROM python:3.12-slim

# BAD: copy the ENTIRE context first — any change in app.py invalidates
# the pip install layer, even though requirements.txt hasn't changed
COPY . /app
WORKDIR /app
RUN pip install --no-cache-dir -r requirements.txt

CMD ["python", "app.py"]
EOF

docker build --no-cache -f Dockerfile.noopt -t lab4/noopt:1.0 .
docker image ls lab4/noopt:1.0
IMAGE            ID             DISK USAGE   CONTENT SIZE   EXTRA
lab4/noopt:1.0   9a76008af785        210MB         53.8MB   
# Note the size for comparison

# === .dockerignore: exclude unnecessary files ===
cat > .dockerignore << 'EOF'
.env
.git
.gitignore
tests/
*.bin
*.log
__pycache__/
*.pyc
EOF

# === OPTIMIZED IMAGE ===
cat > Dockerfile.opt << 'EOF'
FROM python:3.12-slim

WORKDIR /app

# GOOD: copy requirements.txt FIRST — this layer is cached as long as
# requirements.txt doesn't change. Changes in app.py don't invalidate the
# dependency installation.
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy the code afterward: changes here only invalidate the final layers
COPY app.py .

# Combine RUN commands into a single layer when they're logically related
# This avoids intermediate layers with temporary files
RUN useradd --uid 1001 --no-create-home appuser

USER appuser

EXPOSE 5000

ENV FLASK_APP=app.py
ENV FLASK_ENV=production

CMD ["python", "-m", "flask", "run", "--host=0.0.0.0", "--port=5000"]
EOF

docker build --no-cache -f Dockerfile.opt -t lab4/opt:1.0 .
docker image ls lab4/opt:1.0
IMAGE          ID             DISK USAGE   CONTENT SIZE   EXTRA
lab4/opt:1.0   ca8b9f983c0d        199MB         48.6MB  
```

**Verification:**

```bash
# Compare sizes
docker image ls lab4/noopt:1.0 lab4/opt:1.0 \
  --format "table {{.Repository}}:{{.Tag}}\t{{.Size}}"
# Even though the sizes are similar in this case (same base and dependencies)
REPOSITORY:TAG    SIZE
lab4/opt:1.0      199MB
lab4/noopt:1.0    210MB
lab4/server:1.0   177MB

# the difference in layers is visible in docker history

# Compare layer count
echo "=== noopt layers ===" && docker history lab4/noopt:1.0 | wc -l
=== noopt layers ===
15
echo "=== opt layers ===" && docker history lab4/opt:1.0 | wc -l
=== opt layers ===
21
# Verify .dockerignore worked (dummy.bin must NOT be in the image)
docker run --rm lab4/opt:1.0 ls /app
# Output: only app.py and requirements.txt — no dummy.bin, .env, tests/
app.py
requirements.txt
# Verify reduced build context — the second line of the build shows the size
# "Sending build context to Docker daemon  X.XXkB" → without dummy.bin, should be < 5 kB, requires BuildKit disabled
docker build -f Dockerfile.opt -t lab4/opt:1.0 . 2>&1 | head -3
```

> **DCA key point:** order layers from least to most frequently changed.
> Instructions that change rarely (dependencies, base configuration) go on top;
> source code goes at the bottom. The build cache is only valid up to the first
> layer that changes.

---

### Exercise 3: Multi-stage build

**Objective:** use a multi-stage build to separate the compile/build phase from
the runtime phase, resulting in a final image with no build tools or intermediate
artifacts.

**Procedure:**

```bash
cd ~/lab4-images
mkdir ex3-multistage && cd ex3-multistage

# Create a simple Go application (static binary — ideal for multi-stage)
cat > main.go << 'EOF'
package main

import (
    "fmt"
    "net/http"
    "os"
)

func main() {
    port := "8090"
    if p := os.Getenv("PORT"); p != "" {
        port = p
    }
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Lab 4 multi-stage build | hostname=%s\n", mustHostname())
    })
    fmt.Printf("Listening on :%s\n", port)
    http.ListenAndServe(":"+port, nil)
}

func mustHostname() string {
    h, _ := os.Hostname()
    return h
}


# === SINGLE-STAGE IMAGE (unoptimized) ===
cat > Dockerfile.single << 'EOF'
FROM golang:1.22-bookworm

WORKDIR /app
COPY main.go .

# Compile the binary inside the final image
RUN go build -o server main.go

EXPOSE 8090
CMD ["./server"]
EOF

docker build --no-cache -f Dockerfile.single -t lab4/go-single:1.0 .
docker image ls lab4/go-single:1.0
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
lab4/go-single:1.0   fc6523b214d9       1.31GB          319MB   
# This image includes the entire Go toolchain (~800 MB)

# === MULTI-STAGE IMAGE ===
cat > Dockerfile.multi << 'EOF'
# ---- Stage 1: builder ----
# Use the full Go image to compile
FROM golang:1.22-bookworm AS builder

WORKDIR /build
COPY main.go .

# CGO_ENABLED=0 produces a static binary (no dynamic libc dependencies)
# GOOS/GOARCH specify the target
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-w -s" -o server main.go
# -w -s: strips debug info → smaller binary

# ---- Stage 2: runtime ----
# We only copy the compiled binary into a minimal image
FROM scratch
# scratch = empty image (0 bytes). Only the binary and whatever we copy.
# Common alternative: FROM gcr.io/distroless/static or FROM alpine:3.20

# Copy ONLY the binary from the builder stage
COPY --from=builder /build/server /server

EXPOSE 8090

# No shell in scratch: ENTRYPOINT in exec form (JSON array), not shell string
ENTRYPOINT ["/server"]
EOF

docker build --no-cache -f Dockerfile.multi -t lab4/go-multi:1.0 .
docker image ls lab4/go-multi:1.0
IMAGE               ID             DISK USAGE   CONTENT SIZE   EXTRA
lab4/go-multi:1.0   2e4a5c2811bf       6.87MB         2.09MB 
# Compare sizes
docker image ls lab4/go* \
> --format "table {{.Repository}}:{{.Tag}}\t{{.Size}}"
REPOSITORY:TAG       SIZE
lab4/go-multi:1.0    6.87MB
lab4/go-single:1.0   1.31GB
# Expected difference: ~800 MB (single) vs ~7-10 MB (multi)
```

**Verification:**

```bash
# Run the multi-stage image
docker run -d --name go-multi -p 8090:8090 lab4/go-multi:1.0
sleep 2
curl -s http://localhost:8090
Lab 4 multi-stage build | hostname=599ac86ebb60
# Output: Lab 4 multi-stage build | hostname=<container-id>

# The multi-stage image has NO shell (it's scratch)
docker run --rm lab4/go-multi:1.0 /bin/sh 2>&1 || echo "No shell — expected in scratch"
Listening on :8090
#Since it's built with ENTRYPOINT, Docker runs /server; /bin/sh is ignored and the server starts normally
docker run --rm -d --entrypoint ls lab4/go-multi:1.0 2>&1 || echo "No ls - minimal image"
73b074955466aeb9ecf7dd2077fb8877ad2b9ee0cc345f988a32c0b4ce0b627f
docker: Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: error during container init: exec: "ls": executable file not found in $PATH
Run 'docker run --help' for more information
No ls - minimal image
#With the --entrypoint flag we override the entrypoint and confirm the image is minimal
# View the stages in history
docker history lab4/go-multi:1.0
IMAGE          CREATED          CREATED BY                              SIZE      COMMENT
2e4a5c2811bf   16 minutes ago   ENTRYPOINT ["/server"]                  0B        buildkit.dockerfile.v0
<missing>      16 minutes ago   EXPOSE [8090/tcp]                       0B        buildkit.dockerfile.v0
<missing>      16 minutes ago   COPY /build/server /server # buildkit   4.78MB    buildkit.dockerfile.v0
# Few layers and minimal size — just the binary

docker rm -f go-multi
```

> **DCA key point:** multi-stage builds are the standard technique for production.
> They separate build-time from runtime. The final artifact contains no compilers,
> headers, or source code — reducing attack surface and image size.

---

### Exercise 4: Image inspection — history, inspect, ls --format

**Objective:** extract detailed information from images: layers, sizes, metadata,
internal configuration, and filter/format listings.

**Procedure:**

```bash
cd ~/lab4-images

# === docker history ===
# View layers of the image from exercise 1
docker history lab4/server:1.0
IMAGE          CREATED        CREATED BY                                      SIZE      COMMENT
9d0d78b9dc58   23 hours ago   CMD ["server.py"]                               0B        buildkit.dockerfile.v0
<missing>      23 hours ago   ENTRYPOINT ["python3"]                          0B        buildkit.dockerfile.v0
<missing>      23 hours ago   HEALTHCHECK &{["CMD-SHELL" "python3 -c \"imp…   0B        buildkit.dockerfile.v0
<missing>      23 hours ago   EXPOSE [8080/tcp]                               0B        buildkit.dockerfile.v0
<missing>      23 hours ago   USER appuser                                    0B        buildkit.dockerfile.v0
<missing>      23 hours ago   RUN |2 APP_USER=appuser UID=1001 /bin/sh -c …   49.2kB    buildkit.dockerfile.v0
<missing>      23 hours ago   COPY server.py . # buildkit                     12.3kB    buildkit.dockerfile.v0
<missing>      23 hours ago   WORKDIR /app                                    8.19kB    buildkit.dockerfile.v0
<missing>      23 hours ago   ENV APP_ENV=production                          0B        buildkit.dockerfile.v0
<missing>      23 hours ago   ENV APP_PORT=8080                               0B        buildkit.dockerfile.v0
<missing>      23 hours ago   ARG UID=1001                                    0B        buildkit.dockerfile.v0
<missing>      23 hours ago   ARG APP_USER=appuser                            0B        buildkit.dockerfile.v0
<missing>      23 hours ago   LABEL description=Python HTTP server for DC…   0B        buildkit.dockerfile.v0
<missing>      23 hours ago   LABEL version=1.0                               0B        buildkit.dockerfile.v0
<missing>      23 hours ago   LABEL maintainer=<admin_user>@lab4              0B        buildkit.dockerfile.v0
<missing>      2 days ago     CMD ["python3"]                                 0B        buildkit.dockerfile.v0
<missing>      2 days ago     RUN /bin/sh -c set -eux;  for src in idle3 p…   16.4kB    buildkit.dockerfile.v0
<missing>      2 days ago     RUN /bin/sh -c set -eux;   savedAptMark="$(a…   41.3MB    buildkit.dockerfile.v0
<missing>      2 days ago     ENV PYTHON_SHA256=c08bc65a81971c1dd578318282…   0B        buildkit.dockerfile.v0
<missing>      2 days ago     ENV PYTHON_VERSION=3.12.13                      0B        buildkit.dockerfile.v0
<missing>      2 days ago     ENV GPG_KEY=7169605F62C751356D054A26A821E680…   0B        buildkit.dockerfile.v0
<missing>      2 days ago     RUN /bin/sh -c set -eux;  apt-get update;  a…   4.94MB    buildkit.dockerfile.v0
<missing>      2 days ago     ENV LANG=C.UTF-8                                0B        buildkit.dockerfile.v0
<missing>      2 days ago     ENV PATH=/usr/local/bin:/usr/local/sbin:/usr…   0B        buildkit.dockerfile.v0
<missing>      3 days ago     # debian.sh --arch 'amd64' out/ 'trixie' '@1…   87.4MB    debuerreotype 0.17
# Columns: IMAGE, CREATED, CREATED BY (Dockerfile instruction), SIZE, COMMENT

# Untruncated version — shows the full command of each layer
docker history --no-trunc lab4/server:1.0
# Every RUN, COPY, ADD line appears in full

# Compare history of optimized vs unoptimized image
echo "=== noopt layers ===" && docker history lab4/noopt:1.0
=== noopt layers ===
IMAGE          CREATED        CREATED BY                                      SIZE      COMMENT
9a76008af785   20 hours ago   CMD ["python" "app.py"]                         0B        buildkit.dockerfile.v0
<missing>      20 hours ago   RUN /bin/sh -c pip install --no-cache-dir -r…   16.9MB    buildkit.dockerfile.v0
<missing>      20 hours ago   WORKDIR /app                                    4.1kB     buildkit.dockerfile.v0
<missing>      20 hours ago   COPY . /app # buildkit                          5.28MB    buildkit.dockerfile.v0
<missing>      2 days ago     CMD ["python3"]                                 0B        buildkit.dockerfile.v0
<missing>      2 days ago     RUN /bin/sh -c set -eux;  for src in idle3 p…   16.4kB    buildkit.dockerfile.v0
<missing>      2 days ago     RUN /bin/sh -c set -eux;   savedAptMark="$(a…   41.3MB    buildkit.dockerfile.v0
<missing>      2 days ago     ENV PYTHON_SHA256=c08bc65a81971c1dd578318282…   0B        buildkit.dockerfile.v0
<missing>      2 days ago     ENV PYTHON_VERSION=3.12.13                      0B        buildkit.dockerfile.v0
<missing>      2 days ago     ENV GPG_KEY=7169605F62C751356D054A26A821E680…   0B        buildkit.dockerfile.v0
<missing>      2 days ago     RUN /bin/sh -c set -eux;  apt-get update;  a…   4.94MB    buildkit.dockerfile.v0
<missing>      2 days ago     ENV LANG=C.UTF-8                                0B        buildkit.dockerfile.v0
<missing>      2 days ago     ENV PATH=/usr/local/bin:/usr/local/sbin:/usr…   0B        buildkit.dockerfile.v0
<missing>      3 days ago     # debian.sh --arch 'amd64' out/ 'trixie' '@1…   87.4MB    debuerreotype 0.17
echo "=== opt layers ===" && docker history lab4/opt:1.0
=== opt layers ===
IMAGE          CREATED        CREATED BY                                      SIZE      COMMENT
ca8b9f983c0d   14 hours ago   CMD ["python" "-m" "flask" "run" "--host=0.0…   0B        buildkit.dockerfile.v0
<missing>      14 hours ago   ENV FLASK_ENV=production                        0B        buildkit.dockerfile.v0
<missing>      14 hours ago   ENV FLASK_APP=app.py                            0B        buildkit.dockerfile.v0
<missing>      14 hours ago   EXPOSE [5000/tcp]                               0B        buildkit.dockerfile.v0
<missing>      14 hours ago   USER appuser                                    0B        buildkit.dockerfile.v0
<missing>      14 hours ago   RUN /bin/sh -c useradd --uid 1001 --no-creat…   49.2kB    buildkit.dockerfile.v0
<missing>      14 hours ago   COPY app.py . # buildkit                        12.3kB    buildkit.dockerfile.v0
<missing>      14 hours ago   RUN /bin/sh -c pip install --no-cache-dir -r…   16.9MB    buildkit.dockerfile.v0
<missing>      14 hours ago   COPY requirements.txt . # buildkit              12.3kB    buildkit.dockerfile.v0
<missing>      23 hours ago   WORKDIR /app                                    8.19kB    buildkit.dockerfile.v0
<missing>      2 days ago     CMD ["python3"]                                 0B        buildkit.dockerfile.v0
<missing>      2 days ago     RUN /bin/sh -c set -eux;  for src in idle3 p…   16.4kB    buildkit.dockerfile.v0
<missing>      2 days ago     RUN /bin/sh -c set -eux;   savedAptMark="$(a…   41.3MB    buildkit.dockerfile.v0
<missing>      2 days ago     ENV PYTHON_SHA256=c08bc65a81971c1dd578318282…   0B        buildkit.dockerfile.v0
<missing>      2 days ago     ENV PYTHON_VERSION=3.12.13                      0B        buildkit.dockerfile.v0
<missing>      2 days ago     ENV GPG_KEY=7169605F62C751356D054A26A821E680…   0B        buildkit.dockerfile.v0
<missing>      2 days ago     RUN /bin/sh -c set -eux;  apt-get update;  a…   4.94MB    buildkit.dockerfile.v0
<missing>      2 days ago     ENV LANG=C.UTF-8                                0B        buildkit.dockerfile.v0
<missing>      2 days ago     ENV PATH=/usr/local/bin:/usr/local/sbin:/usr…   0B        buildkit.dockerfile.v0
<missing>      3 days ago     # debian.sh --arch 'amd64' out/ 'trixie' '@1…   87.4MB    debuerreotype 0.17

# === docker inspect ===
# Full information in JSON
docker inspect lab4/server:1.0

# Extract specific fields with --format (Go templates)
docker inspect lab4/server:1.0 --format '{{.Id}}'
sha256:9d0d78b9dc58e8c93abd7c100c76498bbc42600ff6e3ac5fc9f2b2c0cb9e0532
# Output: sha256:<full hash>

docker inspect lab4/server:1.0 --format 'OS: {{.Os}} | Arch: {{.Architecture}} | Author: {{.Config.Labels.maintainer}}'
OS: linux | Arch: amd64 | Author: <admin_user>@lab4

# Container configuration (CMD, ENTRYPOINT, ENV, ExposedPorts, User)
# json .Config vs .Config
docker inspect lab4/server:1.0 --format '{{.Config}}'
{{appuser map[8080/tcp:{}] [PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin LANG=C.UTF-8 GPG_KEY=7169605F62C751356D054A26A821E680E5FA6305 PYTHON_VERSION=3.12.13 PYTHON_SHA256=c08bc65a81971c1dd5783182826503369466c7e67374d1646519adf05207b684 APP_PORT=8080 APP_ENV=production] [python3] [server.py] map[] /app map[description:Python HTTP server for DCA maintainer:<admin_user>@lab4 version:1.0]  true} {0x1509f29b3140 [] []}}

docker inspect lab4/server:1.0 --format '{{json .Config}}' | python3 -m json.tool
{
    "User": "appuser",
    "ExposedPorts": {
        "8080/tcp": {}
    },
    "Env": [
        "PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
        "LANG=C.UTF-8",
        "GPG_KEY=7169605F62C751356D054A26A821E680E5FA6305",
        "PYTHON_VERSION=3.12.13",
        "PYTHON_SHA256=c08bc65a81971c1dd5783182826503369466c7e67374d1646519adf05207b684",
        "APP_PORT=8080",
        "APP_ENV=production"
    ],
    "Entrypoint": [
        "python3"
    ],
    "Cmd": [
        "server.py"
    ],
    "WorkingDir": "/app",
    "Labels": {
        "description": "Python HTTP server for DCA",
        "maintainer": "<admin_user>@lab4",
        "version": "1.0"
    },
    "ArgsEscaped": true,
    "Healthcheck": {
        "Test": [
            "CMD-SHELL",
            "python3 -c \"import urllib.request; urllib.request.urlopen ('http://localhost:${APP_PORT}')\" || exit 1"
        ],
        "Interval": 30000000000,
        "Timeout": 5000000000,
        "StartPeriod": 5000000000,
        "Retries": 3
    }
}
# Makes the docker inspect output readable

docker inspect lab4/server:1.0 \
  --format 'User: {{.Config.User}} | Entrypoint: {{.Config.Entrypoint}} | Cmd: {{.Config.Cmd}}'

# View environment variables
docker inspect lab4/server:1.0 \
  --format '{{range .Config.Env}}{{.}}{{"\n"}}{{end}}'
User: appuser | Entrypoint: [python3] | Cmd: [server.py]
# View labels
docker inspect lab4/server:1.0 \
  --format '{{range $k,$v := .Config.Labels}}{{$k}}={{$v}}{{"\n"}}{{end}}'
description=Python HTTP server for DCA
maintainer=<admin_user>@lab4
version=1.0
# View exposed ports
docker inspect lab4/server:1.0 \
  --format '{{range $p,$_ := .Config.ExposedPorts}}{{$p}} {{end}}'
8080/tcp
# Number of layers (RootFS)
docker inspect lab4/server:1.0 \
  --format 'Layers: {{len .RootFS.Layers}}'
Layers: 7
# === docker image ls with filters and --format ===
# List all lab images with a custom tabular format
docker image ls \
  --format "table {{.Repository}}\t{{.Tag}}\t{{.ID}}\t{{.Size}}\t{{.CreatedSince}}"
REPOSITORY       TAG       IMAGE ID       SIZE      CREATED
lab4/go-multi    1.0       2e4a5c2811bf   6.87MB    About an hour ago
lab4/go-single   1.0       fc6523b214d9   1.31GB    13 hours ago
lab4/opt         1.0       ca8b9f983c0d   199MB     15 hours ago
lab4/noopt       1.0       9a76008af785   210MB     20 hours ago
lab4/server      1.0       9d0d78b9dc58   177MB     23 hours ago

# Filter by name
docker image ls --filter "reference=lab4/*"

# Filter dangling images (untagged — orphaned intermediate layers)
docker image ls --filter "dangling=true"
# If there are orphan images, they show as <none>:<none>

# Filter by label
docker image ls --filter "label=maintainer=<admin_user>@lab4"

# Filter images before/after a reference
docker image ls --filter "before=lab4/server:1.0"

# Get only the IDs (useful for scripting)
docker image ls --format "{{.ID}}" --filter "reference=lab4/*"
```

**Verification:**

```bash
# Verify the entrypoint can be extracted correctly
docker inspect lab4/server:1.0 --format '{{index .Config.Entrypoint }}'
[python3]
# Output: python3

# Verify layer count (must be > 5 for the exercise 1 image)
docker inspect lab4/server:1.0 --format '{{len .RootFS.Layers}}'
7
# Output: integer > 5

# Verify the labels exist
docker inspect lab4/server:1.0 \
  --format '{{index .Config.Labels "maintainer"}}'
<admin_user>@lab4
```

---

### Exercise 5: Flattening an image — export/import to collapse layers

**Objective:** collapse all the layers of an image into a single one using the
`docker export` / `docker import` cycle. Useful for reducing layer overhead or
creating a clean snapshot image.

**Procedure:**

```bash
cd ~/lab4-images

# Starting image: lab4/server:1.0 (multiple layers)
 echo "Real filesystem layers: $(docker inspect lab4/server:1.0 \
--format '{{len .RootFS.Layers}}')"
Real filesystem layers: 7
docker history lab4/server:1.0 | wc -l
Build steps: 26 #25 + 1 header row
# === FLATTEN PROCESS ===

# Step 1: create a container from the image (without running it)
docker create --name flat-src lab4/server:1.0
# A created container has the filesystem ready but is not running

# Step 2: export the container's filesystem to a tar
# docker export exports the container's FILESYSTEM (not the image — no metadata)
docker export flat-src -o /tmp/server-flat.tar

ls -lh /tmp/server-flat.tar
-rw------- 1 <admin_user> <admin_user> 115M Apr 24 08:25 /tmp/server-flat.tar
# Tar file with the container's entire filesystem

# Step 3: import the tar as a new image
# docker import creates a SINGLE-LAYER image from the tar
# Basic configuration can be added with --change
docker import \
  --change 'ENTRYPOINT ["python3"]' \
  --change 'CMD ["server.py"]' \
  --change 'WORKDIR /app' \
  --change 'ENV APP_PORT=8080' \
  --change 'ENV APP_ENV=production' \
  --change 'EXPOSE 8080' \
  /tmp/server-flat.tar \
  lab4/server:flat

# Step 4: compare layers
echo "=== Original image ===" && docker inspect lab4/server:1.0 --format '{{len .RootFS.Layers}} layers' && docker image ls lab4/server:1.0 --format '{{.Size}}'
=== Original image ===
7 layers
177MB

echo "=== Flattened image ==="
docker inspect lab4/server:flat --format '{{len .RootFS.Layers}} layers'
docker image ls lab4/server:flat --format '{{.Size}}'
echo "=== Flat image ===" && docker inspect lab4/server:flat --format '{{len .RootFS.Layers}} layers' && docker image ls lab4/server:flat --format '{{.Size}}'
=== Flat image ===
1 layers
173MB
# Expected output: 1 layer vs N original layers
# Compare sizes (size may be similar — flatten doesn't compress, it just unifies)
```

**Verification:**

```bash
# The flat image has exactly 1 layer
docker inspect lab4/server:flat --format '{{len .RootFS.Layers}}'
1
# Output: 1

# The container starts and serves responses
docker run -d --name srv-flat -p 8081:8080 lab4/server:flat
sleep 3
curl -s http://localhost:8081
Hello from Lab4 | env=production | port=8080
# Output: Hello from Lab 4 | env=production | port=8080

docker rm -f srv-flat flat-src
rm -f /tmp/server-flat.tar
```

> **DCA key point:** `docker export`/`docker import` operates on the **container's
> filesystem** (not the image). It loses the history, build metadata, and
> Dockerfile instructions. Use only when a clean snapshot is needed or for layer
> obfuscation. **It is not a backup mechanism**.
>
> To transfer images between hosts with their full history: use `docker save`/`docker load`.

---

### Exercise 6: Save/load and cp — transferring images and files

**Objective:** package an image into a tar file to transport it without a registry
(`docker save`/`docker load`) and copy files between host and container (`docker cp`).

**Procedure:**

```bash
cd ~/lab4-images

# === TAGGING ===
# Create several tags of the same image (does not duplicate content — just pointers)
docker tag lab4/server:1.0 lab4/server:latest
docker tag lab4/server:1.0 lab4/server:v1.0.0
docker tag lab4/server:1.0 lab4/server:stable

docker image ls lab4/server --format 'table {{.Tag}}\t{{.ID}}\t{{.Size}}'
TAG       IMAGE ID       SIZE
flat      fffa4b4bd1e0   173MB
1.0       9d0d78b9dc58   177MB
latest    9d0d78b9dc58   177MB
stable    9d0d78b9dc58   177MB
v1.0.0    9d0d78b9dc58   177MB
# All 4 tags share the SAME image ID → no additional space used

# === docker save — save image(s) to a tar ===
# Save a single image (all its tags are included automatically)
docker save lab4/server -o /tmp/server-v1.tar

ls -lh /tmp/server-v1.tar
-rw------- 1 <admin_user> <admin_user> 42M Apr 24 09:42 /tmp/server-v1.tar
# Tar file with the complete image: layers, metadata, and manifest

# Save multiple images into a single tar
docker save lab4/server:1.0 lab4/opt:1.0 | gzip > /tmp/lab4-images.tar.gz
ls -lh /tmp/lab4-images.tar.gz
-rw-r--r-- 1 <admin_user> <admin_user> 47M Apr 24 09:43 /tmp/lab4-images.tar.gz
# Internal structure of the tar (to understand the format)
tar -tf /tmp/server-v1.tar | head -20
# You can see: manifest.json, repositories, and per-layer directories (sha256/)

# === Simulate transfer: remove image and reload from tar ===
docker rmi lab4/server:1.0 lab4/server:latest lab4/server:v1.0.0 lab4/server:stable
docker image ls lab4/server
# Output: no images

docker load -i /tmp/server-v1.tar
docker image ls lab4/server
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
lab4/server:1.0      9d0d78b9dc58        177MB         43.2MB        
lab4/server:latest   9d0d78b9dc58        177MB         43.2MB        
lab4/server:stable   9d0d78b9dc58        177MB         43.2MB        
lab4/server:v1.0.0   9d0d78b9dc58        177MB         43.2MB   
# Output: the image is back with all its tags

# === docker cp — copy files host <-> container ===
# Start a container
docker run -d --name srv-cp -p 8082:8080 lab4/server:1.0

# Copy a file FROM the container TO the host
docker cp srv-cp:/app/server.py /tmp/server-from-container.py
ls -l /tmp/server-from-container.py
-rw-r--r-- 1 <admin_user> <admin_user> 512 Apr 22 08:49 /tmp/server-from-container.py
diff /tmp/server-from-container.py ~/lab4-images/ex1-dockerfile/server.py
# Output: no differences (same file)

# Copy a file FROM the host TO the container
echo "# Lab 4 message" > /tmp/readme.txt
docker cp /tmp/readme.txt srv-cp:/app/readme.txt

docker exec srv-cp cat /app/readme.txt
#Lab 4 message
# Output: # Lab 4 message

# Copy an entire directory
docker cp srv-cp:/app /tmp/app-backup
ls -la /tmp/app-backup/
total 16
drwxr-xr-x  2 <admin_user> <admin_user> 4096 Apr 24 09:59 .
drwxrwxrwt 10 root root 4096 Apr 24 10:00 ..
-rw-r--r--  1 <admin_user> <admin_user>   17 Apr 24 09:59 readme.txt
-rw-r--r--  1 <admin_user> <admin_user>  512 Apr 22 08:49 server.py
# Contains: server.py, readme.txt

# docker cp also works with stopped containers
docker stop srv-cp
docker cp srv-cp:/usr/lib/os-release /tmp/os-releaseb.txt
cat /tmp/os-release.txt
PRETTY_NAME="Debian GNU/Linux 13 (trixie)"
NAME="Debian GNU/Linux"
VERSION_ID="13"
VERSION="13 (trixie)"
VERSION_CODENAME=trixie
DEBIAN_VERSION_FULL=13.4
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
docker rm srv-cp
```

**Verification:**

```bash
# Verify the save tar includes the manifest
tar -xOf /tmp/server-v1.tar manifest.json | python3 -m json.tool | head -10
[
    {
        "Config": "blobs/sha256/58b50d29632ba1adbf1a0334b5f481d10423be048944c54f5ff81d7458efaaf3",
        "RepoTags": [
            "lab4/server:1.0",
            "lab4/server:latest",
            "lab4/server:stable",
            "lab4/server:v1.0.0"
        ],
        "Layers":
    }
]
# Output: JSON with RepoTags and layers

# Verify load restores the image identically
docker load -i /tmp/server-v1.tar 2>&1 | grep -E "Loaded|loaded"
docker inspect lab4/server:1.0 --format 'ID: {{.Id}}' | cut -c1-20
# Must match the ID from before deletion

rm -f /tmp/server-v1.tar /tmp/lab4-images.tar.gz /tmp/server-from-container.py \
      /tmp/readme.txt /tmp/os-release.txt
rm -rf /tmp/app-backup
```

> **DCA key point:** `docker save`/`docker load` preserves the full history, tags,
> and metadata of the image — it is the correct mechanism to transport images
> between hosts without a registry. Unlike `export`/`import`, it retains all layers.

---

### Exercise 7: Private registry — deploy, push, pull, REST API

**Objective:** deploy a private registry with `registry:2` on localhost:5000,
publish images, list them via the REST API, and manage the complete
login/push/pull/delete cycle.

**Procedure:**

```bash
cd ~/lab4-images

# === DEPLOY THE PRIVATE REGISTRY ===
# registry:2 = official Docker Distribution v2 image
docker run -d \
  --name lab4-registry \
  --restart always \
  -p 5000:5000 \
  -v registry-data:/var/lib/registry \
  registry:2

# Verify the registry responds
curl -s http://localhost:5000/v2/
# Output: {} → registry operational (empty JSON response = OK)

# === PUSH IMAGES TO THE PRIVATE REGISTRY ===
# The private registry requires the tag to include the registry address
# Format: <host>:<port>/<name>:<tag>

# Tag and push the image from exercise 1
docker tag lab4/server:1.0 localhost:5000/lab4/server:1.0
docker push localhost:5000/lab4/server:1.0
# Output: layers uploaded, final digest

# Tag and push the optimized image
docker tag lab4/opt:1.0 localhost:5000/lab4/opt:1.0
docker push localhost:5000/lab4/opt:1.0

# Tag and push the multi-stage Go image
docker tag lab4/go-multi:1.0 localhost:5000/lab4/go-multi:1.0
docker push localhost:5000/lab4/go-multi:1.0

# === REGISTRY V2 REST API ===
# The Registry V2 API allows managing the registry without the Docker CLI

# List all repositories in the registry
curl -s http://localhost:5000/v2/_catalog
{"repositories":["lab4/go-multi","lab4/opt","lab4/server"]}
# Output: {"repositories":["lab4/go-multi","lab4/opt","lab4/server"]}

# List tags of a specific repository
curl -s http://localhost:5000/v2/lab4/server/tags/list
{"name":"lab4/server","tags":["1.0"]}
# Output: {"name":"lab4/server","tags":["1.0"]}

# Get the manifest of a specific image
curl -s -H "Accept: application/vnd.docker.distribution.manifest.v2+json, application/vnd.oci.image.index.v1+json, application/vnd.oci.image.manifest.v1+json" http://localhost:5000/v2/lab4/server/manifests/1.0 | python3 -m json.tool | head -20
# Output: JSON with mediaType, config, and layers

# === PULL FROM THE PRIVATE REGISTRY ===
# Remove the local image to force a pull from the registry
docker rmi localhost:5000/lab4/server:1.0

# Pull from the private registry
docker pull localhost:5000/lab4/server:1.0
# Output: layers downloaded from the local registry

# Verify the image works after the pull
docker run --rm --entrypoint python3 localhost:5000/lab4/server:1.0 --version
Python 3.12.13


# === REGISTRY SEARCH (docker search) ===
# docker search only works against Docker Hub (requires internet)
docker search --limit 5 nginx
NAME                              DESCRIPTION                                     STARS     OFFICIAL
nginx                             Official build of Nginx.                        21258     [OK]
nginx/nginx-ingress               NGINX and  NGINX Plus Ingress Controllers fo…   120       
nginx/nginx-prometheus-exporter   NGINX Prometheus Exporter for NGINX and NGIN…   51        
nginx/unit                        This repository is retired, use the Docker o…   66        
nginx/nginx-ingress-operator      NGINX Ingress Operator for NGINX and NGINX P…   3         
# Output: list of nginx images on Docker Hub with stars and description

# Filters in docker search
docker search --filter "is-official=true" python
NAME      DESCRIPTION                                     STARS     OFFICIAL
python    Python is an interpreted, interactive, objec…   10427     [OK]
docker search --filter "stars=100" nginx
NAME                  DESCRIPTION                                     STARS     OFFICIAL
nginx                 Official build of Nginx.                        21258     [OK]
nginx/nginx-ingress   NGINX and  NGINX Plus Ingress Controllers fo…   120       
bitnami/nginx         Bitnami Secure Image for nginx                  203       
ubuntu/nginx          Nginx, a high-performance reverse proxy & we…   141       
linuxserver/nginx     An Nginx container, brought to you by LinuxS…   236  
# For private registries: use the REST API (docker search has no support for it)
curl -s http://localhost:5000/v2/_catalog | python3 -m json.tool
{
    "repositories": [
        "lab4/go-multi",
        "lab4/opt",
        "lab4/server"
    ]
}
# === LOGIN/LOGOUT ON THE REGISTRY ===
# The local registry without auth doesn't require login, but the flow is the same
# For a registry with authentication (example):
# docker login localhost:5000 -u <username> -p <password>
# docker logout localhost:5000

# === DELETE FROM THE REGISTRY ===
# Deleting an image from the private registry uses the REST API
# Step 1: get the image digest
DIGEST=$(curl -sI \
  -H "Accept: application/vnd.oci.image.index.v1+json, application/vnd.oci.image.manifest.v1+json" \
  http://localhost:5000/v2/lab4/opt/manifests/1.0 \
  | grep -i "docker-content-digest" | awk '{print $2}' | tr -d '\r')

echo "Digest: $DIGEST"

# Step 2: delete using the digest
curl -s -X DELETE \
  "http://localhost:5000/v2/lab4/opt/manifests/$DIGEST"
# Output: 202 Accepted (no response body)
# Deletion is disabled by default; to enable it the registry must be started with ENV REGISTRY_STORAGE_DELETE_ENABLED=true
# On the docker run command: -e REGISTRY_STORAGE_DELETE_ENABLED=true
# Verify the repository no longer has the tag
curl -s http://localhost:5000/v2/lab4/opt/tags/list
# Output: {"name":"lab4/opt","tags":null} or a 404 error

# Verify the catalog has changed
curl -s http://localhost:5000/v2/_catalog
```

**Verification:**

```bash
# The registry responds correctly
curl -s http://localhost:5000/v2/ && echo "Registry OK"

# The catalog contains images
curl -s http://localhost:5000/v2/_catalog | python3 -m json.tool

# Pull and run an image from the private registry
docker rmi localhost:5000/lab4/server:1.0 2>/dev/null || true
docker pull localhost:5000/lab4/server:1.0
docker run --rm localhost:5000/lab4/server:1.0 python3 -c "print('Pull from private registry OK')"
```

---

### Exercise 8: Cleanup — prune, rmi, and managing dangling images

**Objective:** manage the image cleanup cycle: identify dangling images, remove
specific images, and use the prune commands with filters.

**Procedure:**

```bash
cd ~/lab4-images

# === GENERATE STATE WITH DANGLING IMAGES ===
# Rebuild the same image with --no-cache to create a new layer and leave the previous one untagged
mkdir -p /tmp/dangling-test && cd /tmp/dangling-test
echo "FROM alpine" > Dockerfile
echo "RUN echo v1 > /version.txt" >> Dockerfile
docker build -t dangling-test:latest .
# Modify and rebuild — the :latest tag points to the new image, the previous one becomes dangling
echo "RUN echo v2 > /version.txt" >> Dockerfile
docker build -t dangling-test:latest .

cd ~/lab4-images

# === LIST DANGLING IMAGES ===
docker image ls --filter "dangling=true"
# Output: image(s) with REPOSITORY=<none> TAG=<none>
docker image ls -f "dangling=true" -q
# IDs only — useful for scripting

# === DOCKER IMAGE PRUNE ===
# Remove only dangling images (safe — does not remove tagged images)
docker image prune -f
# Output: "Total reclaimed space: X MB"
# Tagged images are not removed

docker image ls --filter "dangling=true"
# Output empty — all dangling images removed

# Prune with an age filter (remove dangling images older than 24 hours)
docker image prune --filter "until=24h" -f

# === AGGRESSIVE PRUNE — ALL UNUSED IMAGES ===
# docker image prune -a removes ALL images not in use
# by a running container. USE WITH CAUTION.

# First check what would be removed (no dry-run available — review manually)
docker image ls
docker ps -a  # Containers currently using images

# For this lab, only remove the images from the test exercise
docker rmi dangling-test:latest 2>/dev/null || true
rm -rf /tmp/dangling-test

# === RMI: TARGETED REMOVAL ===
# Remove an image by tag
docker rmi lab4/server:v1.0.0 lab4/server:stable

# If an image has multiple tags, rmi only removes the tag
docker image ls lab4/server
# Output: server:1.0, server:latest, server:flat still exist

# Remove an image by ID (removes ALL tags pointing to that ID)
# Only if no containers are using it
IMAGE_ID=$(docker inspect lab4/server:flat --format '{{.Id}}' | cut -c8-19)
docker rmi lab4/server:flat
# If there were more tags with that ID, it would fail unless all tags are used

# === DOCKER SYSTEM PRUNE ===
# Cleans up: stopped containers + unused networks + dangling images + build cache
# Does NOT remove tagged images or volumes (unless -v is used)
docker system prune -f
# Output: total space reclaimed

# With -a: also includes images without an active container
# docker system prune -a -f  # DESTRUCTIVE — do not run in this lab

# === INSPECT DISK USAGE ===
docker system df
# Output: images, containers, volumes — real and reclaimable size

docker system df -v
# Detailed version per image/container/volume

# === FINAL LAB CLEANUP ===
docker rm -f lab4-registry 2>/dev/null || true
docker volume rm registry-data 2>/dev/null || true

docker rmi \
  lab4/server:1.0 lab4/server:latest \
  lab4/server:flat \
  lab4/noopt:1.0 lab4/opt:1.0 \
  lab4/go-single:1.0 lab4/go-multi:1.0 \
  localhost:5000/lab4/server:1.0 \
  localhost:5000/lab4/go-multi:1.0 \
  2>/dev/null || true

docker image prune -f
docker system df
# Output: space reclaimed
```

**Verification:**

```bash
# No lab images remain
docker image ls --filter "reference=lab4/*"
# Output empty

# No dangling images remain
docker image ls --filter "dangling=true"
# Output empty

# Verify reclaimed space
docker system df
```

---

## Cleanup

```bash
# Make sure no lab containers remain
docker rm -f $(docker ps -aq --filter "name=srv") 2>/dev/null || true
docker rm -f lab4-registry 2>/dev/null || true

# Remove the working directory
rm -rf ~/lab4-images

# Clean up lab images and volumes
docker image prune -f
docker volume rm registry-data 2>/dev/null || true
docker system df
```

---

## Key lessons

### Topics covered with a hands-on exercise

| Topic | Main lesson |
| --- | --- |
| **2.1** | Dockerfile as a declarative recipe: each instruction creates an immutable layer |
| **2.2** | COPY > ADD (ADD only for URLs or tar with extraction); ENTRYPOINT fixed + CMD overridable |
| **2.3** | ARG: build-time only. ENV: persists in the image and container. USER sets process identity |
| **2.4** | Order layers from least to most changing; .dockerignore reduces build context and prevents secret leakage |
| **2.5** | `docker image ls`, `rmi`, `prune` — three levels of removal: tag, image, dangling |
| **2.6** | `docker image ls --format` with Go templates for scripting; `--filter` for subsetting |
| **2.7** | Tag = pointer to an image ID. Multiple tags, same ID = zero additional space |
| **2.8** | `docker build --no-cache` forces a full rebuild; without `--no-cache` it uses cached layers |
| **2.9** | `docker history --no-trunc` shows the exact Dockerfile instruction for each layer |
| **2.10** | Flatten (export/import): 1 single layer. Loses history and metadata. Only for snapshots |
| **2.11-2.15** | Registry V2 REST API: `_catalog` and `tags/list`. Push requires a tag with the registry address |
| **2.16** | Signing with cosign (conceptual): `cosign sign --key cosign.key <image>`. Notary v1 removed in Docker 25+ |
| **2.17** | `docker pull` uses the registry from the tag. No registry → uses Docker Hub |

### Key points for the DCA exam

- **COPY vs ADD:** prefer COPY. ADD has implicit magic (decompresses tar, supports URLs) — use only when needed.
- **ENTRYPOINT vs CMD:** ENTRYPOINT is the executable, CMD is the default arguments. `docker run <image> <args>` overrides CMD, not ENTRYPOINT.
- **ARG vs ENV:** ARG does not persist in the final image. ENV does. For passwords: use secrets, not ARG or ENV.
- **Layers and cache:** the cache is invalidated at the first layer that changes. Dependencies on top, code at the bottom.
- **Multi-stage:** `COPY --from=<stage>` is the key instruction. It can reference the stage name (AS builder) or the index (0, 1...).
- **Flatten:** `export` operates on the container (filesystem). `save` operates on the image (with metadata).
- **Private registry:** `localhost:5000` is an insecure registry — Docker requires `--insecure-registry` or configuration in `/etc/docker/daemon.json` for non-HTTPS remote registries. It works by default on localhost.
- **API V2:** `GET /v2/_catalog` lists repos. `GET /v2/<name>/tags/list` lists tags. `DELETE /v2/<name>/manifests/<digest>` deletes a manifest.
- **Dangling images:** images with no tag or reference. They accumulate with rebuilds. `docker image prune` removes them. `docker image prune -a` is more aggressive.

### Notes for Platform Engineer / SRE

- Image size directly impacts deployment times and bandwidth consumption in CI/CD pipelines. Multi-stage builds and slim/distroless base images are standard in production.
- `.dockerignore` is the first line of defense against secret leakage (`.env`, SSH keys) into the build context.
- A private registry (`registry:2`) is the minimum component for an internal pipeline. In production it is complemented with Harbor, ECR, or Artifact Registry for authentication, RBAC, and integrated scanning.
- `docker system df` is the quick diagnostic command for understanding Docker's disk usage on a host — useful in SRE work when responding to disk-full alerts.

DCA domain mapping: **Domain 2 — Image Creation, Management & Registry (20% of the exam)**
