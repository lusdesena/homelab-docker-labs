# Lab 5 — Storage, Volumes & Installation — Execution Log

## Objective
Cover Domain 6 (Storage & Volumes) in full and the remaining gaps in Domain 3 (Installation & Configuration) in a real Docker 29.x + 3-node Swarm + k3s environment.

Topics covered:
- **Domain 3**: 3.2, 3.3, 3.4, 3.6, 3.9 (hands-on) · 3.1, 3.5, 3.10, 3.11 (Docker EE conceptual)
- **Domain 6**: 6.1, 6.4, 6.5, 6.6, 6.7, 6.8, 6.9 (hands-on) · 6.2, 6.3 (conceptual)

---

## Environment

| Node | IP | Role | Docker |
|---|---|---|---|
| docker-labs | <MANAGER_IP> | Swarm manager / k3s server | 29.4.2 |
| swarm-worker1 | <WORKER1_IP> | Swarm worker / k3s agent | 29.4.2 |
| swarm-worker2 | <WORKER2_IP> | Swarm worker / k3s agent | 29.4.2 |

Prerequisites:
- Active Swarm (3 nodes UP): `docker node ls`
- k3s operational: `kubectl get nodes`
- SSH access to all three nodes

---

## Setup

```bash
# Verify cluster state before starting
docker node ls
kubectl get nodes

# Working directory on docker-lab-manager
mkdir -p ~/lab5 && cd ~/lab5

# Create a test network for the lab
docker network create lab5-net
```

---

## Block A — Installation & Configuration (Domain 3)

### Exercise 1 — Sizing requirements [CONCEPTUAL] (topic 3.1)

No execution required. Memorize for the exam:

**Docker Engine (CE) — practical minimums:**
| Resource | Recommended minimum |
|---|---|
| OS | Linux 64-bit (kernel ≥ 3.10) |
| CPU | 2 cores |
| RAM | 2 GB |
| Disk | 10 GB free in `/var/lib/docker` |

**Docker Enterprise / Mirantis Kubernetes Engine (MKE) — official sizing:**
| Role | CPU | RAM | Disk |
|---|---|---|---|
| Manager (UCP) | 8 cores | 16 GB | 100 GB (SSD recommended) |
| Worker | 4 cores | 4 GB | 25 GB |
| DTR | 8 cores | 16 GB | 100 GB |

Key exam points:
- The storage driver is configured in `daemon.json` — changing it requires restarting the daemon and **loses all existing container/image data**.
- `overlay2` is the recommended driver for all modern filesystems (ext4, xfs with `d_type=true`).
- `devicemapper` in `loop-lvm` mode is for development only — `direct-lvm` for production.

---

### Exercise 2 — Docker Engine installation + storage driver (topics 3.2, 6.1)

#### 2a. Check current installation and storage driver

```bash
# Full daemon version and info
docker version
Client: Docker Engine - Community
 Version:           29.4.2
 API version:       1.54
 Go version:        go1.26.2
 Git commit:        055a478
 Built:             Fri May  1 10:24:04 2026
 OS/Arch:           linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          29.4.2
  API version:      1.54 (minimum version 1.40)
  Go version:       go1.26.2
  Git commit:       d329809
  Built:            Fri May  1 10:24:04 2026
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          v2.2.3
  GitCommit:        77c84241c7cbdd9b4eca2591793e3d4f4317c590
 runc:
  Version:          1.3.5
  GitCommit:        v1.3.5-0-g488fc13e
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
docker info
Client: Docker Engine - Community
 Version:    29.4.2
 Context:    default
 Debug Mode: false
 Plugins:
  buildx: Docker Buildx (Docker Inc.)
    Version:  v0.33.0
    Path:     /usr/libexec/docker/cli-plugins/docker-buildx
  compose: Docker Compose (Docker Inc.)
    Version:  v5.1.3
    Path:     /usr/libexec/docker/cli-plugins/docker-compose

Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 0
 Server Version: 29.4.2
 Storage Driver: overlayfs
# Storage driver in use
docker info | grep -A3 "Storage Driver"
Storage Driver: overlayfs
  driver-type: io.containerd.snapshotter.v1
 Logging Driver: json-file
 Cgroup Driver: systemd
# Docker data directory filesystem
stat -f -c %T /var/lib/docker
ext2/ext3
# Check d_type support (required for overlay2)
tune2fs -l $(df /var/lib/docker | awk 'NR==2{print $1}') 2>/dev/null | grep "Filesystem features" || \
  xfs_info /var/lib/docker 2>/dev/null | grep ftype
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype needs_recovery extent 64bit flex_bg sparse_super large_file huge_file dir_nlink extra_isize metadata_csum

#dir_index = overlay2 compatible
#has_journal = journaling active (ext4 standard)
```

Expected output on Debian 12:
```
Storage Driver: overlay2
 Backing Filesystem: extfs
```

#### 2b. Reference: installing Docker from the official repo (Debian/Ubuntu)

```bash
# Install dependencies
apt-get update && apt-get install -y ca-certificates curl gnupg

# Add Docker's official GPG key
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg \
  | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

# Add repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | tee /etc/apt/sources.list.d/docker.list

# Install Docker Engine
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verify
docker version
```

> **Lab note**: Docker is already installed. This block is exam reference material — practice the command from memory.

#### 2c. Storage drivers by OS/filesystem (topic 6.1)

```bash
# View current driver in detail
docker info | grep -E "Storage Driver|Backing Filesystem|Supports d_type|Native Overlay Diff"
Storage Driver: overlayfs
```

| Driver | Recommended OS / FS | Production | Notes |
|---|---|---|---|
| `overlay2` | Linux / ext4, xfs (d_type=true) | ✅ Recommended | Default in modern Docker CE |
| `devicemapper` | Linux / any block device | ⚠️ direct-lvm only | Legacy, not recommended |
| `vfs` | Any | ❌ Testing only | No CoW, one copy per layer |
| `overlay` | Linux / kernel < 4.0 | ❌ Obsolete | Predecessor of overlay2 |
| `windowsfilter` | Windows | ✅ | Windows containers only |
| `zfs` | Linux / FreeBSD with ZFS | ✅ | Requires ZFS installed |

---

### Exercise 3 — Configure daemon + automatic startup (topics 3.2, 3.6)

#### 3a. Check automatic startup

```bash
# Verify Docker starts with the system
systemctl is-enabled docker
enabled
# Detailed status
systemctl status docker
 docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-05-04 08:47:16 UTC; 19min ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 984101 (dockerd)
      Tasks: 11
     Memory: 48.3M
        CPU: 3.070s
     CGroup: /system.slice/docker.service
             └─984101 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

```

#### 3b. Enable if not active

```bash
systemctl enable docker
systemctl enable containerd
```

#### 3c. Configure daemon.json with exam-relevant parameters

```bash
# View current configuration
cat /etc/docker/daemon.json 2>/dev/null || echo "daemon.json does not exist"
{                                                                             
}
# Back up before modifying
cp /etc/docker/daemon.json /etc/docker/daemon.json.lab5.bak 2>/dev/null || true
```

Create a configuration with the parameters most relevant to the exam:

```bash
cat > /tmp/daemon-lab5.json << 'EOF'
{
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "default-address-pools": [
    {"base": "172.30.0.0/16", "size": 24}
  ],
  "metrics-addr": "127.0.0.1:9323",
  "experimental": false
}
EOF

# Validate JSON before applying
python3 -m json.tool /tmp/daemon-lab5.json
#Pretty-printed output if there are no errors
```

```bash
# Apply and restart (the Swarm survives the daemon restart)
cp /tmp/daemon-lab5.json /etc/docker/daemon.json
systemctl daemon-reload && systemctl restart docker

# Verify the Swarm is still operational after restart
docker info | grep -E "Swarm|Storage Driver|Logging Driver"
 Storage Driver: overlay2
 Logging Driver: json-file
 Swarm: active
docker node ls
ID                            HOSTNAME        STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
l4vjq00wuhxlhh9hsxmulnmk9 *   docker-labs     Ready     Active         Leader           29.4.2
zeyrrpv709jjyyfhao9nwg707     swarm-worker1   Ready     Active                          29.4.2
6mid239fygj45ncse7z5dtwtj     swarm-worker2   Ready     Active                          29.4.2
```

---

### Exercise 4 — Logging drivers (topic 3.3)

#### 4a. Default driver: json-file

```bash
# Check driver in use
docker info | grep "Logging Driver"
 Logging Driver: json-file
# Container with default driver
docker run -d --name log-test-json nginx:alpine

# View logs (works with json-file and journald)
docker logs log-test-json
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2026/05/04 09:27:42 [notice] 1#1: using the "epoll" event method
2026/05/04 09:27:42 [notice] 1#1: nginx/1.29.8
2026/05/04 09:27:42 [notice] 1#1: built by gcc 15.2.0 (Alpine 15.2.0) 
2026/05/04 09:27:42 [notice] 1#1: OS: Linux 6.1.0-44-amd64
2026/05/04 09:27:42 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1024:524288
2026/05/04 09:27:42 [notice] 1#1: start worker processes
2026/05/04 09:27:42 [notice] 1#1: start worker process 30
2026/05/04 09:27:42 [notice] 1#1: start worker process 31
2026/05/04 09:27:42 [notice] 1#1: start worker process 32
2026/05/04 09:27:42 [notice] 1#1: start worker process 33
docker logs --tail 5 log-test-json
2026/05/04 09:27:42 [notice] 1#1: start worker processes
2026/05/04 09:27:42 [notice] 1#1: start worker process 30
2026/05/04 09:27:42 [notice] 1#1: start worker process 31
2026/05/04 09:27:42 [notice] 1#1: start worker process 32
2026/05/04 09:27:42 [notice] 1#1: start worker process 33


# Locate the physical log file
LOG_PATH=$(docker inspect log-test-json --format '{{.LogPath}}')
echo $LOG_PATH
/var/lib/docker/containers/07f2392914a83669adb456fc47f1df3d881c6a636582e20fbd0e3fb849c0422e/07f2392914a83669adb456fc47f1df3d881c6a636582e20fbd0e3fb849c0422e-json.log
ls -lh $LOG_PATH
-rw-r----- 1 root root 2.7K May  4 09:27 /var/lib/docker/containers/07f2392914a83669adb456fc47f1df3d881c6a636582e20fbd0e3fb849c0422e/07f2392914a83669adb456fc47f1df3d881c6a636582e20fbd0e3fb849c0422e-json.log
```

#### 4b. Use journald for a specific container

```bash
# Container with journald
docker run -d --name log-test-journald \
  --log-driver journald \
  --log-opt tag="docker/{{.Name}}" \
  nginx:alpine

# docker logs shows the same output as with json-file
docker logs log-test-journald 2>&1 || true

# View logs via journalctl
journalctl CONTAINER_TAG=docker/log-test-journald -n 20
# Or alternatively:
journalctl -u docker -n 20 --no-pager
```

#### 4c. Other drivers — exam reference

```bash
# View all available drivers
docker info | grep -A1 "Plugins"
```

| Driver | Use | `docker logs` | Notes |
|---|---|---|---|
| `json-file` | Default, development | ✅ Yes | Files under `/var/lib/docker/containers/<id>/` |
| `journald` | Production systemd | ❌ No | Query with `journalctl` |
| `syslog` | Syslog/rsyslog | ❌ No | Sends to remote syslog |
| `splunk` | Production Splunk | ❌ No | Requires HEC token |
| `fluentd` | Production EFK | ❌ No | Requires fluentd daemon |
| `none` | Silence logs | ❌ No | No logging |
| `awslogs` | AWS CloudWatch | ❌ No | EC2/ECS only |

> **Exam**: `docker logs` only works with `json-file` and `journald`.

#### 4d. Configure global logging in daemon.json

```bash
# Already included in the daemon.json from exercise 3:
# "log-driver": "json-file",
# "log-opts": { "max-size": "10m", "max-file": "3" }

# Verify
docker info | grep -A3 "Logging Driver"
 Logging Driver: json-file
 Cgroup Driver: systemd
 Cgroup Version: 2
```

```bash
# Cleanup
docker rm -f log-test-json log-test-journald
```

---

### Exercise 5 — Swarm cluster backup and restore (topic 3.4)

The Swarm state (certificates, Raft configuration, secrets, configs) is stored in `/var/lib/docker/swarm` **only on manager nodes**.

#### 5a. Swarm backup (on docker-labs, <MANAGER_IP>)

```bash
# Verify we are the active manager
docker node ls
docker info | grep "Is Manager"
 Is Manager: true

# Document critical information before the backup
echo "=== Join tokens ===" > ~/lab5/swarm-backup-info.txt
docker swarm join-token manager >> ~/lab5/swarm-backup-info.txt
docker swarm join-token worker >> ~/lab5/swarm-backup-info.txt
echo "=== Current nodes ===" >> ~/lab5/swarm-backup-info.txt
docker node ls >> ~/lab5/swarm-backup-info.txt
echo "=== Active services ===" >> ~/lab5/swarm-backup-info.txt
docker service ls >> ~/lab5/swarm-backup-info.txt

cat ~/lab5/swarm-backup-info.txt
```

```bash
# Stop the Docker daemon (required for a consistent backup)
systemctl stop docker

# Verify /var/lib/docker/swarm exists
ls -la /var/lib/docker/swarm/

# Create backup
BACKUP_FILE=~/lab5/swarm-backup-$(date +%Y%m%d-%H%M%S).tar.gz
tar czf $BACKUP_FILE /var/lib/docker/swarm
echo "Backup created: $BACKUP_FILE"
ls -lh $BACKUP_FILE

# Restart daemon
systemctl start docker

# Verify the Swarm is still operational
docker node ls
```

#### 5b. Explore backup contents

```bash
# View backup structure
tar tzf $BACKUP_FILE | head -30
var/lib/docker/swarm/
var/lib/docker/swarm/raft/
var/lib/docker/swarm/raft/snap-v3-encrypted/
var/lib/docker/swarm/raft/wal-v3-encrypted/
var/lib/docker/swarm/raft/wal-v3-encrypted/0000000000000000-0000000000000000.wal
var/lib/docker/swarm/worker/
var/lib/docker/swarm/worker/tasks.db
var/lib/docker/swarm/docker-state.json
var/lib/docker/swarm/certificates/
var/lib/docker/swarm/certificates/swarm-node.crt
var/lib/docker/swarm/certificates/swarm-node.key
var/lib/docker/swarm/certificates/swarm-root-ca.crt
var/lib/docker/swarm/state.json
# Key directories:
# swarm/certificates/  — cluster CA and certificates
# swarm/docker-state.json — cluster metadata
# swarm/raft/  — Raft log and snapshots
```

#### 5c. Restore procedure (exam reference — do not execute)

```bash
# RESTORE PROCEDURE (only in case of cluster loss):
# 1. Install Docker on the new node
# 2. Stop Docker: systemctl stop docker
# 3. Restore the backup:
#    tar xzf swarm-backup-DATE.tar.gz -C /
# 4. Start with the force flag:
#    dockerd --swarm-default-advertise-addr <IP> &
#    docker swarm init --force-new-cluster --advertise-addr <IP>
# 5. Verify: docker node ls
# 6. Add workers with the documented join token
```

---

### Exercise 6 — Installation troubleshooting (topic 3.9)

#### 6a. Diagnostic commands

```bash
# Full Docker system status
docker info
docker system info  # alias of docker info

# Detailed version (client + server)
docker version

# Real-time daemon events (Ctrl+C to exit)
docker system events --since 10m &
EVENTS_PID=$!
#Captures the PID of the background event stream
# Generate an event
docker pull hello-world:latest
2026-05-04T09:45:06.924055980Z node update l4vjq00wuhxlhh9hsxmulnmk9 (name=docker-labs, state.new=ready, state.old=unknown)
2026-05-04T09:45:07.181559686Z network create ql40opro26yj7li5fps6nta7q (name=ingress, type=overlay)
2026-05-04T09:45:08.636706736Z node update zeyrrpv709jjyyfhao9nwg707 (name=swarm-worker1, state.new=ready, state.old=unknown)
2026-05-04T09:45:12.261291118Z node update 6mid239fygj45ncse7z5dtwtj (name=swarm-worker2, state.new=ready, state.old=unknown)
2026-05-04T09:54:13.314242923Z image pull hello-world:latest (name=hello-world)
# View captured events
sleep 2 && kill $EVENTS_PID 2>/dev/null; true
```

```bash
# Daemon logs via journald
journalctl -u docker -n 50 --no-pager

# Real-time logs (useful when restarting the daemon)
journalctl -u docker -f --no-pager &
JOURNAL_PID=$!
systemctl restart docker
sleep 3 && kill $JOURNAL_PID 2>/dev/null; true
```

#### 6b. Common failure cases

```bash
# Case 1: daemon.json with invalid JSON
echo '{"bad json"' > /tmp/bad-daemon.json
# Verify before applying:
python3 -m json.tool /tmp/bad-daemon.json || echo "Invalid JSON — do not apply"

# Case 2: port in use
ss -tlnp | grep -E "2375|2376"

# Case 3: Docker socket permissions
ls -la /var/run/docker.sock
# The user must be in the docker group:
groups $USER

# Case 4: check container in a problematic state
docker ps -a --filter status=dead --filter status=exited
docker inspect <container_id> | jq '.[0].State'
```

#### 6c. Docker EE conceptual topics (3.5, 3.10, 3.11)

**3.5 — Users and teams in UCP (Docker Enterprise):**
- UCP (Universal Control Plane) has native RBAC
- Roles: `None`, `View Only`, `Restricted Control`, `Scheduler`, `Full Control`
- Teams are organized into organizations
- Integration with LDAP/AD for centralized authentication (topic 5.12)
- No Docker EE environment available — evaluated theoretically on the exam

**3.10 — Deploying Docker Engine, UCP, and DTR (HA):**
- Minimum HA UCP: 3 managers (tolerates 1 failure)
- Minimum HA DTR: 3 replicas with shared storage (NFS/S3)
- Process: install Docker Engine → install UCP → install DTR and point it to UCP
- AWS: UCP on EC2 with ELB + RDS for state, S3 for DTR storage

**3.11 — UCP and DTR backup:**
- UCP backup: `docker container run --rm docker/ucp backup > ucp-backup.tar`
- DTR backup: `docker run --rm docker/dtr backup --ucp-url <URL> > dtr-backup.tar`
- Recommended frequency: daily, store outside the cluster

---

## Block B — Storage & Volumes (Domain 6)

### Exercise 7 — Image layers and filesystem (topic 6.4)

#### 7a. Explore image layers

```bash
# Layer history of an image
docker pull python:3.11-slim
docker history python:3.11-slim
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
1eee6fcc4d86   12 days ago   CMD ["python3"]                                 0B        buildkit.dockerfile.v0
<missing>      12 days ago   RUN /bin/sh -c set -eux;  for src in idle3 p…   36B       buildkit.dockerfile.v0
<missing>      12 days ago   RUN /bin/sh -c set -eux;   savedAptMark="$(a…   42MB      buildkit.dockerfile.v0
<missing>      12 days ago   ENV PYTHON_SHA256=272179ddd9a2e41a0fc8e42e33…   0B        buildkit.dockerfile.v0
<missing>      12 days ago   ENV PYTHON_VERSION=3.11.15                      0B        buildkit.dockerfile.v0
<missing>      12 days ago   ENV GPG_KEY=A035C8C19219BA821ECEA86B64E628F8…   0B        buildkit.dockerfile.v0
<missing>      12 days ago   RUN /bin/sh -c set -eux;  apt-get update;  a…   3.81MB    buildkit.dockerfile.v0
<missing>      12 days ago   ENV LANG=C.UTF-8                                0B        buildkit.dockerfile.v0
<missing>      12 days ago   ENV PATH=/usr/local/bin:/usr/local/sbin:/usr…   0B        buildkit.dockerfile.v0
<missing>      13 days ago   # debian.sh --arch 'amd64' out/ 'trixie' '@1…   78.6MB    debuerreotype 0.17
docker history python:3.11-slim --no-trunc --format "table {{.Size}}\t{{.CreatedBy}}"

# Total number of layers
docker history python:3.11-slim --quiet | wc -l
10
```

#### 7b. Locate layers on the filesystem (overlay2)

```bash
# See where the overlay2 for an image is
IMAGE_ID=$(docker inspect python:3.11-slim --format '{{.Id}}' | cut -d: -f2 | cut -c1-12)
docker inspect python:3.11-slim --format '{{json .GraphDriver}}' | python3 -m json.tool
{
    "Data": {
        "LowerDir": "/var/lib/docker/overlay2/bb55cca7d51a5e5fa510a2daa6320ba8d7310afe8eb6bb3b765cab00161d42cb/diff:/var/lib/docker/overlay2/a73784b416e20ce28da873719ceef45519348b1fa0398f40dc94c8649bdcf045/diff:/var/lib/docker/overlay2/117b5fba14267bd59d7fd635487c67d7de7e456f1d3fbaada6dd55476be26517/diff",
        "MergedDir": "/var/lib/docker/overlay2/3be4e2f150f1d92ef4982d0c6fad05e7e73d4f89a1e012f2d902f52e99728a90/merged",
        "UpperDir": "/var/lib/docker/overlay2/3be4e2f150f1d92ef4982d0c6fad05e7e73d4f89a1e012f2d902f52e99728a90/diff",
        "WorkDir": "/var/lib/docker/overlay2/3be4e2f150f1d92ef4982d0c6fad05e7e73d4f89a1e012f2d902f52e99728a90/work"
    },
    "Name": "overlay2"
}
# View the physical layer directory
ls /var/lib/docker/overlay2/ | head -10
03dc3e06abb0af52293885a9cecabd18a17fc177c19c7f7bdcfc28abbfebf90f
0ea396102a463476794569581b1a0afef2bfd84218c0e7383b9c4a4fd4d03e58
117b5fba14267bd59d7fd635487c67d7de7e456f1d3fbaada6dd55476be26517
15db5412ef758dd9fe565ad68ff1ad72b7e0b0ab5e1761663109257ca13dfd5a
3be4e2f150f1d92ef4982d0c6fad05e7e73d4f89a1e012f2d902f52e99728a90
3e9efaf72ba3e58209b83423d7eeb596f39fecc7d5a7ec901a8d4da0567b1da0
46e7b218421cee3a7313e4fcaf57f92d612ad2db6afc9c274fa63abe9f6394ee
719d4a1882704ecac130cb24bf064778342b6499f60790249fcc29b8532b0a7d
a73784b416e20ce28da873719ceef45519348b1fa0398f40dc94c8649bdcf045
b3cea1122a59b638a1bc6c44788f79370b564b317b0a7c1549f0c255f2009c2c
# Shows the physical storage of the layers
ls /var/lib/docker/image/overlay2/layerdb/sha256/ | head -10
198fb1785a3693e2164fcfa5cccfed50c1ec233877f44a78189521a39a7c36e0
1d9a0e936fac6798e368a505eeb532babea0f78a88ccf678f7a7f6119d6af92a
27169268e599542a8efe40bc98eec72801e56c7eec6974fe72287429b5a56a88
29df493baa13de438d6d2ece3a8333032e0b7b9b9d8cce4ee82194da255f61e1
3f1f2c7545807c52541c2414015214c10174d63181a6a6dea381caf2386b23fe
5aed9176333b0418db26b0d9abbb0cccea9e1d6af3f67473a5c5f1890e7815a8
5b8ea4394f94f0f338ef723b4c0539a41aa457d53709c34a6343ea4c9488d016
6d7c150df58d41c351cd9b03f1cda7a9a23d6fc91436e2bf0f098c6dd78c9c55
7e21891e773c24466c13d24a8f0fd9cdcc0310c88dfb9cfd5bd2a0480b48bd3e
897b3f2a7c1bc2f3d02432f7892fe31c6272c521ad4d70257df624504a3238b4
# Shows the logical index — which layers exist and how they are chained. layerdb/sha256/ID/cacheID -> overlay2/<same value>/diff
# Start a container and view its overlay
docker run -d --name overlay-test python:3.11-slim sleep 300
CONTAINER_ID=$(docker inspect overlay-test --format '{{.Id}}')

# Container layers: lower (read-only) + merged + diff (read-write) + work
docker inspect overlay-test --format '{{json .GraphDriver.Data}}' | python3 -m json.tool
{
    "ID": "0476e94d76e2d1ceb562a8e50bf614782047e9403c622dc323b0b536b8e94003",
    "LowerDir": "/var/lib/docker/overlay2/91b9e66926b34a6841b1e4b697d1b2e7495a0fc06a2c06ab075a816e0b51e8af-init/diff:/var/lib/docker/overlay2/3be4e2f150f1d92ef4982d0c6fad05e7e73d4f89a1e012f2d902f52e99728a90/diff:/var/lib/docker/overlay2/bb55cca7d51a5e5fa510a2daa6320ba8d7310afe8eb6bb3b765cab00161d42cb/diff:/var/lib/docker/overlay2/a73784b416e20ce28da873719ceef45519348b1fa0398f40dc94c8649bdcf045/diff:/var/lib/docker/overlay2/117b5fba14267bd59d7fd635487c67d7de7e456f1d3fbaada6dd55476be26517/diff",
    "MergedDir": "/var/lib/docker/overlay2/91b9e66926b34a6841b1e4b697d1b2e7495a0fc06a2c06ab075a816e0b51e8af/merged",
    "UpperDir": "/var/lib/docker/overlay2/91b9e66926b34a6841b1e4b697d1b2e7495a0fc06a2c06ab075a816e0b51e8af/diff",
    "WorkDir": "/var/lib/docker/overlay2/91b9e66926b34a6841b1e4b697d1b2e7495a0fc06a2c06ab075a816e0b51e8af/work"
}
UPPER=$(docker inspect overlay-test --format '{{.GraphDriver.Data.UpperDir}}')
MERGED=$(docker inspect overlay-test --format '{{.GraphDriver.Data.MergedDir}}')
echo "UpperDir (container RW layer): $UPPER"
echo "MergedDir (unified view): $MERGED"

# Create a file in the container and see it in the UpperDir
docker exec overlay-test sh -c "echo 'hello lab5' > /tmp/test-file.txt"
ls $UPPER/tmp/
test-file.txt
cat $UPPER/tmp/test-file.txt
hello lab5
docker rm -f overlay-test
```

#### 7c. devicemapper and object/block storage [CONCEPTUAL] (topics 6.2, 6.3)

**6.2 — devicemapper:**
- `loop-lvm`: uses files as block devices — development only, slow I/O
- `direct-lvm`: dedicated block device (LVM thin pool) — production
- Configuration in daemon.json:
  ```json
  {
    "storage-driver": "devicemapper",
    "storage-opts": [
      "dm.thinpooldev=/dev/mapper/docker-thinpool",
      "dm.use_deferred_removal=true",
      "dm.use_deferred_deletion=true"
    ]
  }
  ```
- No longer recommended in Docker 20+ — migrate to overlay2

**6.3 — Object vs Block vs File storage:**
| Type | Example | Use in Docker | Protocol |
|---|---|---|---|
| Block | AWS EBS, LVM, iSCSI | devicemapper, raw volumes | SCSI/NVMe |
| File (NFS) | NFS, CIFS/SMB, AWS EFS | Named volumes with NFS driver | NFS/SMB |
| Object | AWS S3, MinIO | DTR storage backend, logs | HTTP/REST |

- Docker volumes use block or file storage (not object directly)
- DTR can use S3 as an image storage backend

---

### Exercise 8 — Volumes: lifecycle and types (topic 6.5)

#### 8a. Named volumes — basic operations

```bash
# Create and list volumes
docker volume create lab5-data
docker volume ls

# Inspect: mountpoint, driver, scope
docker volume inspect lab5-data
[
    {
        "CreatedAt": "2026-05-04T12:36:54Z",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/lab5-data/_data",
        "Name": "lab5-data",
        "Options": null,
        "Scope": "local"
    }
]

# Actual mountpoint on the filesystem
MOUNTPOINT=$(docker volume inspect lab5-data --format '{{.Mountpoint}}')
echo "Mountpoint: $MOUNTPOINT"
ls -la $MOUNTPOINT
```

#### 8b. Demonstrate persistence

```bash
# Write data with a container
docker run --rm \
  -v lab5-data:/data \
  alpine sh -c "echo 'persistent-data' > /data/archivo.txt && cat /data/archivo.txt"

# Verify the data persists without the container
ls $MOUNTPOINT/
cat $MOUNTPOINT/archivo.txt

# A second container reads the same data
docker run --rm -v lab5-data:/data alpine cat /data/archivo.txt
persistent-data
```

#### 8c. Bind mounts vs named volumes vs tmpfs

```bash
# Bind mount — host directory mounted into the container
mkdir -p ~/lab5/bind-data
echo "from the host" > ~/lab5/bind-data/host-file.txt

docker run --rm \
  -v ~/lab5/bind-data:/app \
  alpine ls /app
host-file.txt
# tmpfs — memory only, does not persist
docker run --rm \
  --mount type=tmpfs,destination=/tmpdata,tmpfs-size=64m \
  alpine sh -c "df -h /tmpdata && echo 'in memory' > /tmpdata/ram-file.txt && cat /tmpdata/ram-file.txt"
  Filesystem                Size      Used Available Use% Mounted on
tmpfs                    64.0M         0     64.0M   0% /tmpdata
in memory
# After removing the container, the data disappears
```

| Type | Persistence | Shareable | Performance | Use |
|---|---|---|---|---|
| Named volume | ✅ Yes | Between containers | High | Application data in production |
| Bind mount | ✅ Yes (host) | With the host | High | Dev, config files |
| tmpfs | ❌ RAM only | No | Very high | Sensitive temporary data |

#### 8d. Volumes in Compose/Stack

```yaml
# ~/lab5/compose-volumes.yml
version: "3.9"
services:
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: lab5pass
    volumes:
      - pgdata:/var/lib/postgresql/data
  app:
    image: alpine
    command: sh -c "while true; do sleep 60; done"
    volumes:
      - pgdata:/shared:ro
      - type: bind
        source: ./bind-data
        target: /config
      - type: tmpfs
        target: /tmp/cache
        tmpfs:
          size: 32m

volumes:
  pgdata:
    driver: local
```

```bash
cd ~/lab5
docker compose -f compose-volumes.yml up -d
docker volume ls | grep lab5
docker compose -f compose-volumes.yml down
# The pgdata volume persists after down
docker volume ls | grep lab5
local     lab5-data
local     lab5_pgdata

```

---

### Exercise 9 — Cleaning up images and volumes (topic 6.6)

#### 9a. Audit disk usage

```bash
# Usage summary
docker system df
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          5         0         452MB     443.6MB (98%)
Containers      0         0         0B        0B
Local Volumes   2         0         39.98MB   39.98MB (100%)
Build Cache     0         0         0B        0B

# Detail by type
docker system df -v
Images space usage:

REPOSITORY    TAG         IMAGE ID       CREATED       SIZE      SHARED SIZE   UNIQUE SIZE   CONTAINERS
python        3.11-slim   1eee6fcc4d86   12 days ago   124MB     0B            124.5MB       0
postgres      15-alpine   c1dd58d6cec8   12 days ago   274MB     8.45MB        265.4MB       0
nginx         alpine      812d47f806db   2 weeks ago   62.2MB    8.45MB        53.73MB       0
alpine        latest      3cb067eab609   2 weeks ago   8.45MB    8.45MB        0B            0
hello-world   latest      e2ac70e7319a   5 weeks ago   10.1kB    0B            10.07kB       0

Containers space usage:

CONTAINER ID   IMAGE     COMMAND   LOCAL VOLUMES   SIZE      CREATED   STATUS    NAMES

Local Volumes space usage:

VOLUME NAME   LINKS     SIZE
lab5_pgdata   0         39.98MB
lab5-data     0         20B

Build cache usage: 0B

CACHE ID   CACHE TYPE   SIZE      CREATED   LAST USED   USAGE     SHARED
```

#### 9b. Clean up by type

```bash
# Only dangling images (no tag)
docker image prune -f

# Images with no associated container (more aggressive)
docker image prune -a -f

# Unused volumes
docker volume prune -f

# Stopped containers
docker container prune -f

# Unused networks
docker network prune -f

# Everything at once (excluding volumes)
docker system prune -f

# Everything including volumes (DESTRUCTIVE — asks for confirmation)
docker system prune --volumes -f
```

#### 9c. Filters for selective cleanup

```bash
# Dangling images
docker image ls --filter dangling=true

# Containers stopped more than 24h ago
docker container ls -a --filter status=exited

# Images created before a date
docker image ls --filter "before=nginx:latest"

# Remove images by reference to another image
docker image prune --filter "until=24h"
```

---

### Exercise 10 — Storage in a Swarm cluster (topic 6.7)

#### 10a. Why named volumes are not shared across nodes

```bash
# Create a service with a named volume, with no node restriction
docker service create \
  --name vol-test \
  --replicas 3 \
  --mount type=volume,source=lab5-data,target=/data \
  alpine sleep 3600

# See which nodes each task runs on
docker service ps vol-test

# Write from the task on the manager
docker exec $(docker ps -q -f name=vol-test) \
  sh -c "hostname > /data/hostname.txt && cat /data/hostname.txt"

# Read from another node (SSH to worker1):
# ssh <admin_user>@<WORKER1_IP>
# docker run --rm -v lab5-data:/data alpine cat /data/hostname.txt
# → DIFFERENT or empty — each node has ITS OWN copy of the volume
docker run --rm -v lab5-data:/data alpine cat /data/hostname.txt
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
Digest: sha256:5b10f432ef3da1b8d4c7eb6c487f2f5a8f096bc91145e68878dd4a5019afde11
Status: Downloaded newer image for alpine:latest
cat: can't open '/data/hostname.txt': No such file or directory

docker service rm vol-test
```

#### 10b. Solution 1: placement constraint + named volume

```bash
# Pin the service to the node that has the data
docker service create \
  --name vol-pinned \
  --constraint 'node.hostname == docker-labs' \
  --mount type=volume,source=lab5-data,target=/data \
  --restart-condition on-failure \
  alpine sh -c "cat /data/hostname.txt || echo 'no data'"

docker service ps vol-pinned
docker service logs vol-pinned
vol-pinned.1.3eet3hqpgnxb@docker-labs    | e6ff8100eb4c
docker service rm vol-pinned
```

#### 10c. Solution 2: NFS shared across nodes (real shared storage)

Configure an NFS server on docker-labs (<MANAGER_IP>):

```bash
# On docker-lab-manager — install NFS server
apt-get install -y nfs-kernel-server

# Create shared directory
mkdir -p /mnt/nfs-swarm
chown nobody:nogroup /mnt/nfs-swarm
chmod 777 /mnt/nfs-swarm

# Export to the cluster nodes
echo "/mnt/nfs-swarm <HOMELAB_LAN>/24(rw,sync,no_subtree_check,no_root_squash)" \
  >> /etc/exports
exportfs -ra
systemctl restart nfs-kernel-server

# Verify exports
showmount -e localhost
Export list for localhost:
/mnt/nfs-swarm <HOMELAB_LAN>/24
```

```bash
# On worker1 (<WORKER1_IP>) and worker2 (<WORKER2_IP>):
apt-get install -y nfs-common

# Verify access
showmount -e <MANAGER_IP>
Export list for <MANAGER_IP>:
/mnt/nfs-swarm <HOMELAB_LAN>/24
```

```bash
# On all THREE nodes — create the NFS volume driver
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=<MANAGER_IP>,rw,sync,nfsvers=4 \
  --opt device=:/mnt/nfs-swarm \
  nfs-shared

docker volume inspect nfs-shared
```

```bash
# On the manager — deploy a service with an NFS shared volume
docker service create \
  --name nfs-service \
  --replicas 3 \
  --mount type=volume,source=nfs-shared,target=/shared \
  alpine sh -c "hostname >> /shared/nodes.txt && sleep 3600"

# Wait for the tasks to start
sleep 5
docker service ps nfs-service

# Verify from the manager — all three nodes write to the same file
cat /mnt/nfs-swarm/nodes.txt
4ff0fe502c41
625a8e8f40c2
#1 of the 3 did not write correctly — this is one of the known problems with NFS, lack of synchronization. This is why NFS is used for a single writer, not for concurrent writes
docker service rm nfs-service
```

#### 10d. Volume plugins (exam reference)

```bash
# List available volume plugins
docker plugin ls

# Install a volume plugin (example: sshfs)
# docker plugin install vieux/sshfs
# docker volume create --driver vieux/sshfs \
#   -o sshcmd=user@host:/path \
#   -o password=xxx \
#   sshfs-vol
```

Plugins relevant to the exam:
- `local` (default) — local storage on the node
- `vieux/sshfs` — NAS via SFTP
- `rexray` — cloud volumes (AWS EBS, Azure Disk, etc.)
- CSI (Container Storage Interface) — Kubernetes standard

---

### Exercise 11 — PersistentVolumes in k3s (topics 6.8, 6.9)

k3s includes `local-path-provisioner` as the default StorageClass, which automatically provisions PVs on the node where the pod runs.

#### 11a. Explore StorageClasses

```bash
# Available StorageClasses
kubectl get storageclass
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  14d
kubectl describe storageclass local-path
Name:                  local-path
IsDefaultClass:        Yes
Annotations:           defaultVolumeType=local,objectset.rio.cattle.io/applied=H4sIAAAAAAAA/4yRz47UMAyHXwX53JYpnamqSBxg0V4QEhJoObuJOzVN4ypxi0areXeUMqDhwJ9j8ov9xZ+fARd+ophYAhhIKhHPVE1dqlhebjUUMHFwYODTj+jBY0pQwEyKDhXBPAOGIIrKElI+Ohpw9fokfp3p82UhMODFoocCpP9KVhNpFVkqi6qeMokz4i+5fAsUy/M2gYGpSXfJVhcv3nNwr984J+GfLQLOv/5T3sb9r6K0oM2V09pTmS5JaYbipzCbrVQ5ioGUdnmcypuJco/BgMaV4FqAx5787upP3BHTCAbqrhmak21Pw9Db5tAe20MzHJuhPnUH19m2w1cOe3fMTX+bbEEd8+USZeO8XIpgIGKwI8UMuHtWQMwD8PxRPNsLGHhHnjRr2fYdvuXgOJw/iMuAL8j6KPGRY9IHCWmdKcL1ewAAAP//KQ1Ko0kCAAA,objectset.rio.cattle.io/id=,objectset.rio.cattle.io/owner-gvk=k3s.cattle.io/v1, Kind=Addon,objectset.rio.cattle.io/owner-name=local-storage,objectset.rio.cattle.io/owner-namespace=kube-system,storageclass.kubernetes.io/is-default-class=true
Provisioner:           rancher.io/local-path
Parameters:            <none>
AllowVolumeExpansion:  <unset>
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     WaitForFirstConsumer
Events:                <none>

# Existing PVs (probably empty at the start)
kubectl get persistentvolumes
No resources found
```

#### 11b. Create a PVC and Pod with persistent storage

```bash
# PVC with dynamic provisioning
cat > ~/lab5/pvc-test.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lab5-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 100Mi
EOF

kubectl apply -f ~/lab5/pvc-test.yaml
kubectl get pvc lab5-pvc
NAME       STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
lab5-pvc   Pending                                      local-path     <unset>                 11s
# STATUS: Pending until a Pod uses it (local-path is WaitForFirstConsumer)
```

```bash
# Pod that uses the PVC
cat > ~/lab5/pod-pvc.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: lab5-pod
spec:
  containers:
    - name: app
      image: alpine
      command: ["sh", "-c", "echo pod-$(hostname) > /data/pod.txt && sleep 3600"]
      volumeMounts:
        - name: storage
          mountPath: /data
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: lab5-pvc
EOF

kubectl apply -f ~/lab5/pod-pvc.yml
kubectl get pod lab5-pod
NAME       READY   STATUS    RESTARTS   AGE
lab5-pod   1/1     Running   0          8s
kubectl get pvc lab5-pvc   # Should now be Bound
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
lab5-pvc   Bound    pvc-3f346c52-d725-43fd-b61e-3a2865c3b573   100Mi      RWO            local-path     <unset>                 9m18s
kubectl get pv              # Automatically provisioned PV
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM              STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-3f346c52-d725-43fd-b61e-3a2865c3b573   100Mi      RWO            Delete           Bound    default/lab5-pvc   local-path     <unset>                          36s
```

#### 11c. Verify persistence

```bash
# View the content written by the pod
kubectl exec lab5-pod -- cat /data/pod.txt
pod-lab5-pod
# Delete and recreate the pod — the data persists because the PV remains
kubectl delete pod lab5-pod
kubectl apply -f ~/lab5/pod-pvc.yaml
kubectl wait --for=condition=Ready pod/lab5-pod --timeout=60s
kubectl exec lab5-pod -- cat /data/pod.txt
pod-lab5-pod
pod-lab5-pod
# The previous pod's file is still there
```

#### 11d. PV/PVC lifecycle

```bash
# Detailed PV: capacity, accessMode, reclaimPolicy, storageClass
kubectl describe pv
Name:              pvc-3f346c52-d725-43fd-b61e-3a2865c3b573
Labels:            <none>
Annotations:       local.path.provisioner/selected-node: k3s-server
                   pv.kubernetes.io/provisioned-by: rancher.io/local-path
Finalizers:        [kubernetes.io/pv-protection]
StorageClass:      local-path
Status:            Bound
Claim:             default/lab5-pvc
Reclaim Policy:    Delete
Access Modes:      RWO
VolumeMode:        Filesystem
Capacity:          100Mi
Node Affinity:     
  Required Terms:  
    Term 0:        kubernetes.io/hostname in [k3s-server]
Message:           
Source:
    Type:  LocalVolume (a persistent volume backed by local storage on a node)
    Path:  /var/lib/rancher/k3s/storage/pvc-3f346c52-d725-43fd-b61e-3a2865c3b573_default_lab5-pvc
Events:    <none>
# Access modes relevant to the exam:
# RWO (ReadWriteOnce): mounted read-write by 1 node
# ROX (ReadOnlyMany): mounted read-only by N nodes
# RWX (ReadWriteMany): mounted read-write by N nodes (NFS/CephFS)

# Reclaim policies:
# Delete: removes the PV when the PVC is deleted (cloud default)
# Retain: keeps the PV when the PVC is deleted (data safe)
# Recycle: deprecated
```

#### 11e. CSI drivers [CONCEPTUAL] (topic 6.9)

```bash
# View installed CSI drivers (k3s)
kubectl get csidrivers 2>/dev/null || echo "No additional CSI drivers installed"
```

CSI (Container Storage Interface) — the standard for storage plugins in Kubernetes:
- Decouples storage code from the Kubernetes core
- Examples: `ebs.csi.aws.com`, `pd.csi.storage.gke.io`, `disk.csi.azure.com`
- `local-path-provisioner` is not CSI — it's a simplified provisioner
- In production: install the provider's CSI driver → create a StorageClass → PVCs use it automatically

# storageclass-ebs.yaml   
apiVersion: storage.k8s.io/v1   
kind: StorageClass                        
metadata:
  name: ebs-sc                       
provisioner: ebs.csi.aws.com    
parameters:
  type: gp3                             
  encrypted: "true"
reclaimPolicy: Delete               
volumeBindingMode: WaitForFirstConsumer                                              

# pvc-ebs.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:                                
  name: ebs-pvc
spec:                             
  accessModes:                              
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 10Gi

# pod-ebs.yaml
apiVersion: v1                     
kind: Pod
metadata:                               
  name: app-pod                                         
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - mountPath: /data
          name: ebs-vol
  volumes:
  - name: ebs-vol                                                       
      persistentVolumeClaim:                               
        claimName: ebs-pvc

```bash
# Clean up k3s resources
kubectl delete pod lab5-pod
kubectl delete pvc lab5-pvc
# The PV does not stay in Released state — the default policy is Delete, so removing the PVC also removes the PV
kubectl get pv
No resources found
kubectl delete pv <pv-name>
```

---

## Final verification

```bash
# Topics practiced — verification summary
echo "=== Domain 3 — Installation & Config ==="
docker info | grep -E "Storage Driver|Logging Driver|Swarm"
systemctl is-enabled docker && echo "Docker: automatic startup OK"
ls ~/lab5/swarm-backup-*.tar.gz && echo "Swarm backup: OK"

echo ""
echo "=== Domain 6 — Storage & Volumes ==="
docker volume ls | grep lab5
docker system df
kubectl get storageclass
kubectl get pv 2>/dev/null || true

echo ""
echo "=== Cluster OK ==="
docker node ls
kubectl get nodes
```

---

## Lab cleanup

```bash
# Docker
docker compose -f ~/lab5/compose-volumes.yml down -v 2>/dev/null || true
docker volume rm lab5-data nfs-shared 2>/dev/null || true
docker network rm lab5-net 2>/dev/null || true
docker container prune -f

# NFS (if installed)
# systemctl stop nfs-kernel-server
# apt-get remove -y nfs-kernel-server nfs-common
# rm -rf /mnt/nfs-swarm

# Restore the original daemon.json
cp /etc/docker/daemon.json.lab5.bak /etc/docker/daemon.json 2>/dev/null && \
  systemctl daemon-reload && systemctl restart docker || true

# k3s
kubectl delete pod lab5-pod 2>/dev/null || true
kubectl delete pvc lab5-pvc 2>/dev/null || true
rm -rf ~/lab5
```

---

## Common issues

| Issue | Cause | Solution |
|---|---|---|
| `docker logs` doesn't work | Log driver ≠ json-file/journald | Use the driver's own tool (journalctl, splunk) |
| PVC stuck in `Pending` | local-path waits for the first consumer | Create the Pod — the PV is provisioned on startup |
| NFS mount fails on worker | nfs-common not installed | `apt-get install nfs-common` on each worker |
| Swarm doesn't start after restore | IP/hostname changed | Use `--force-new-cluster` with the correct IP |
| `overlay2` not supported | FS without `d_type` | Format with `mkfs.xfs -n ftype=1` or use ext4 |
| daemon.json syntax error | Invalid JSON | Validate with `python3 -m json.tool` before applying |

---

## DCA/SRE lessons

### Domain 3 — Installation & Configuration (15%)
- `overlay2` is the correct storage driver for Debian/Ubuntu in production — remember that changing drivers destroys all existing images and containers
- `daemon.json` centralizes all daemon configuration — invalid JSON prevents Docker from starting
- Daemon logs live in `journalctl -u docker` — mandatory first step in troubleshooting
- A Swarm backup requires **stopping the daemon** — a hot backup does not guarantee Raft consistency
- Docker EE (UCP/DTR) — conceptual only: RBAC, organizations/teams, backup via the official image

### Domain 6 — Storage & Volumes (10%)
- Named volumes are **local to the node** — in Swarm, use NFS or placement constraints for real persistence
- `docker system df` is the SRE command for auditing Docker disk usage before a cleanup
- In Kubernetes, `local-path` provisions dynamically but only supports `RWO` — `RWX` (shared) requires NFS/CephFS
- `ReclaimPolicy: Retain` protects data when a PVC is deleted — essential for databases
- CSI is the storage extensibility standard in Kubernetes — the cloud provider supplies the driver

**DCA mapping:**
- Domain 3: Installation & Configuration — **15% of the exam** (topics 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.9, 3.10, 3.11)
- Domain 6: Storage & Volumes — **10% of the exam** (topics 6.1–6.9)
