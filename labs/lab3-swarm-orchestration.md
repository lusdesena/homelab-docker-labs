# Lab 3 — Swarm Orchestration & Kubernetes (k3s) — Execution Log

## Objective

Consolidate Docker Swarm orchestration mechanisms on a real 3-node cluster
(manager + 2 workers), then deploy a k3s cluster on the same VMs to
introduce Kubernetes concepts from the DCA domain.

I manage a Swarm cluster with real nodes, observe Raft quorum behavior
hands-on, deploy stacks, apply placement constraints across nodes, and
troubleshoot services. Then I transition to k3s to work with pods, deployments,
ConfigMaps, and Secrets using `kubectl`, building a foundation for CKA.

DCA competencies: **1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 1.8, 1.9, 1.10, 1.11, 1.12, 1.13,
1.14, 1.15, 1.16, 1.17** — all hands-on exercises.

---

## Environment

| Role | VMID | IP | Resources | OS |
| --- | --- | --- | --- | --- |
| Swarm manager / k3s server | 200 | <MANAGER_IP> | 4 vCPU / 8 GB | Debian 12 / Docker 29.3.1 |
| Swarm worker1 / k3s agent1 | 201 | <WORKER1_IP> | 2 vCPU / 2 GB | Debian 12 (fresh install) |
| Swarm worker2 / k3s agent2 | 202 | <WORKER2_IP> | 2 vCPU / 2 GB | Debian 12 (fresh install) |
| Proxmox VE | — | <PROXMOX_IP> | — | Proxmox VE 9.1 |

Manager access: `ssh -J <admin_user>@vps -p 2222 <admin_user>@localhost`
Worker access: `ssh <admin_user>@<WORKER1_IP>` / `ssh <admin_user>@<WORKER2_IP>` (from the local
network, or by jumping through the manager with `-J <admin_user>@<MANAGER_IP>`)

---

## Setup — Part 1: Swarm Cluster (3 nodes)

Steps 1A and 1B run **only once** before the lab. If the VMs already exist and
Docker is installed, skip directly to Step 1C.

---

### Step 1A — Provision worker VMs on Proxmox

Connect to the Proxmox host and clone the cloud-init template (VMID 9000):

```bash
# Connect to the Proxmox host
ssh <admin_user>@<PROXMOX_IP>

# --- Create swarm-worker1 (VMID 201) ---
qm clone 9000 201 --name swarm-worker1 --full
qm set 201 --cores 2 --memory 2048
qm set 201 --ipconfig0 ip=<WORKER1_IP>/24,gw=<LAN_GATEWAY_IP>
qm set 201 --nameserver 8.8.8.8
qm set 201 --ciuser <admin_user>
# If the template doesn't have your SSH key, add it:
# qm set 201 --sshkeys ~/.ssh/authorized_keys
qm start 201

# --- Create swarm-worker2 (VMID 202) ---
qm clone 9000 202 --name swarm-worker2 --full
qm set 202 --cores 2 --memory 2048
qm set 202 --ipconfig0 ip=<WORKER2_IP>/24,gw=<LAN_GATEWAY_IP>
qm set 202 --nameserver 8.8.8.8
qm set 202 --ciuser <admin_user>
qm start 202

# Wait ~30s for cloud-init to finish and verify they respond
sleep 30
ping -c 2 <WORKER1_IP>
ping -c 2 <WORKER2_IP>
```

---

### Step 1B — Install Docker on the workers

Apply the `vendor-docker.yaml` snippet (on the Proxmox host) to VMs 201 and 202,
the same way it was done for VM 200 in week 2.

Once applied, verify from the manager:

```bash
for NODE in <WORKER1_IP> <WORKER2_IP>; do
  ssh <admin_user>@$NODE "docker version --format 'Docker {{.Server.Version}} OK on \$(hostname)'"
done
```

Expected result: `Docker 29.x.x OK on swarm-worker1` / `...worker2`.

---

### Step 1C — Form the Swarm cluster

```bash
# --- Connect to the manager (VM 200) ---

# Verify the manager already has Swarm active (from the previous lab)
docker info | grep "Swarm"
# If not active: docker swarm init --advertise-addr <MANAGER_IP>

# Get the join token for workers
JOIN_TOKEN=$(docker swarm join-token worker -q)
echo "Token: $JOIN_TOKEN"

# --- Join workers to the cluster (run on each worker) ---
ssh <admin_user>@<WORKER1_IP> "docker swarm join --token $JOIN_TOKEN <MANAGER_IP>:2377"
ssh <admin_user>@<WORKER2_IP> "docker swarm join --token $JOIN_TOKEN <MANAGER_IP>:2377"

# Verify the cluster from the manager
docker node ls
ID                            HOSTNAME        STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
wk3dz1kxkl39u2iuo5ip49g05 *   docker-labs     Ready     Active         Leader           29.4.0
fnj63blk0xagixwrhtf65i2lq     swarm-worker1   Ready     Active                          29.4.0
mewr10h2extvwwmauyepeapih     swarm-worker2   Ready     Active                          29.4.0

```

---

### Step 1D — Prepare the working directory

```bash
# On the manager
mkdir -p ~/lab3-swarm && cd ~/lab3-swarm

# Clean up state from previous labs
docker service ls --format '{{.Name}}' | xargs -r docker service rm
docker stack ls --format '{{.Name}}' | xargs -r docker stack rm
sleep 5
docker container prune -f
docker volume prune -f
docker network prune -f

# Confirm clean state
docker service ls
docker stack ls
```

---

## Procedure — Swarm

---

### Exercise 1 — Swarm cluster setup: manager and workers (topic 1.1)

**Objective:** manage the real Swarm cluster: inspect nodes, add/remove,
handle availability and promotion/demotion.

```bash
# Full cluster view
docker node ls
# Key columns: ID, HOSTNAME, STATUS, AVAILABILITY, MANAGER STATUS

# Inspect the manager node in detail
docker node inspect self --pretty
# Observe: ID, Hostname, Status, Availability, Manager Status, Platform, Resources
ID                            HOSTNAME        STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
wk3dz1kxkl39u2iuo5ip49g05 *   docker-labs     Ready     Active         Leader           29.4.0
fnj63blk0xagixwrhtf65i2lq     swarm-worker1   Ready     Active                          29.4.0
mewr10h2extvwwmauyepeapih     swarm-worker2   Ready     Active                          29.4.0

# Inspect a specific worker
WORKER1_ID=$(docker node ls --filter "name=swarm-worker1" --format '{{.ID}}')
docker node inspect $WORKER1_ID --pretty
D:			fnj63blk0xagixwrhtf65i2lq
Hostname:              	swarm-worker1
Joined at:             	2026-04-15 14:55:50.922210121 +0000 utc
Status:
 State:			Ready
 Availability:         	Active
 Address:		<WORKER1_IP>
Platform:
 Operating System:	linux
 Architecture:		x86_64
Resources:
 CPUs:			2
 Memory:		1.921GiB
Plugins:
 Log:		awslogs, fluentd, gcplogs, gelf, journald, json-file, local, splunk, syslog
 Network:		bridge, host, ipvlan, macvlan, null, overlay
 Volume:		local
Engine Version:		29.4.0

# See the join tokens (for future reference)
docker swarm join-token worker
docker swarm join-token manager

# Change a node's availability (test on worker1)
docker node update --availability drain $WORKER1_ID
docker node ls
# Output: swarm-worker1 with AVAILABILITY = Drain (does not accept new tasks)
ID                            HOSTNAME        STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
wk3dz1kxkl39u2iuo5ip49g05 *   docker-labs     Ready     Active         Leader           29.4.0
fnj63blk0xagixwrhtf65i2lq     swarm-worker1   Ready     Drain                           29.4.0
mewr10h2extvwwmauyepeapih     swarm-worker2   Ready     Active 
# Restore availability
docker node update --availability active $WORKER1_ID
docker node ls
ID                            HOSTNAME        STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
wk3dz1kxkl39u2iuo5ip49g05 *   docker-labs     Ready     Active         Leader           29.4.0
fnj63blk0xagixwrhtf65i2lq     swarm-worker1   Ready     Active                          29.4.0
mewr10h2extvwwmauyepeapih     swarm-worker2   Ready     Active                          29.4.0

# Add labels for placement (used in exercise 9)
docker node update --label-add env=production --label-add storage=ssd docker-labs
docker node update --label-add env=staging --label-add storage=hdd $WORKER1_ID
WORKER2_ID=$(docker node ls --filter "name=swarm-worker2" --format '{{.ID}}')
docker node update --label-add env=staging --label-add storage=hdd $WORKER2_ID

# Verify labels
docker node inspect docker-labs --format '{{.Spec.Labels}}'
map[env:production storage:ssd]
docker node inspect $WORKER1_ID --format '{{.Spec.Labels}}'
map[env:staging storage:hdd]
docker node inspect $WORKER2_ID --format '{{.Spec.Labels}}'
map[env:staging storage:hdd]
```

---

### Exercise 1.3 — Raft quorum: hands-on practice (topic 1.3)

**Objective:** observe cluster behavior when quorum is lost and recovered,
using real nodes. With 1 manager: 0 fault tolerance.

```bash
# --- Current state: 1 manager, 2 workers ---
docker node ls
# 1 manager → quorum = 1, fault tolerance = 0

# Promote worker1 to manager (cluster becomes 2 managers)
docker node promote $WORKER1_ID
docker node ls
ID                            HOSTNAME        STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
wk3dz1kxkl39u2iuo5ip49g05 *   docker-labs     Ready     Active         Leader           29.4.0
fnj63blk0xagixwrhtf65i2lq     swarm-worker1   Ready     Active         Reachable        29.4.0
mewr10h2extvwwmauyepeapih     swarm-worker2   Ready     Active                          29.4.0

# swarm-worker1 now has MANAGER STATUS = Reachable
# 2 managers → quorum = 2, fault tolerance = 0 (worse than with 1!)

# Promote worker2 to manager (cluster becomes 3 managers)
docker node promote $WORKER2_ID
docker node ls
ID                            HOSTNAME        STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
wk3dz1kxkl39u2iuo5ip49g05 *   docker-labs     Ready     Active         Leader           29.4.0
fnj63blk0xagixwrhtf65i2lq     swarm-worker1   Ready     Active         Reachable        29.4.0
mewr10h2extvwwmauyepeapih     swarm-worker2   Ready     Active         Reachable        29.4.0

# 3 managers → quorum = 2, fault tolerance = 1

# Quorum table for the current nodes:
# Managers | Quorum required | Fault tolerance
#    1     |       1          |        0
#    2     |       2          |        0
#    3     |       2          |        1   ← current configuration
#    5     |       3          |        2

# --- Simulate a manager failure (drain worker2-manager) ---
docker node update --availability drain $WORKER2_ID
docker node ls
# The cluster remains operational: 2 active managers, quorum satisfied (2/3)
ID                            HOSTNAME        STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
wk3dz1kxkl39u2iuo5ip49g05 *   docker-labs     Ready     Active         Leader           29.4.0
fnj63blk0xagixwrhtf65i2lq     swarm-worker1   Ready     Active         Reachable        29.4.0
mewr10h2extvwwmauyepeapih     swarm-worker2   Ready     Drain          Reachable        29.4.0

# Verify the cluster accepts operations with 1 manager "down"
docker service create --name quorum-test --replicas 2 nginx:alpine
docker service ls | grep quorum-test
# Expected output: 2/2 — the cluster operates normally with quorum
ID             NAME          MODE         REPLICAS   IMAGE          PORTS
pktbnokuopm5   quorum-test   replicated   2/2        nginx:alpine  
# Restore worker2
docker node update --availability active $WORKER2_ID
docker service rm quorum-test

# --- Demote workers back to worker (for later exercises) ---
docker node demote $WORKER1_ID
Manager fnj63blk0xagixwrhtf65i2lq demoted in the swarm.
docker node demote $WORKER2_ID
Manager mewr10h2extvwwmauyepeapih demoted in the swarm.
docker node ls
# Back to: 1 manager (Leader) + 2 workers
ID                            HOSTNAME        STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
wk3dz1kxkl39u2iuo5ip49g05 *   docker-labs     Ready     Active         Leader           29.4.0
fnj63blk0xagixwrhtf65i2lq     swarm-worker1   Ready     Active                          29.4.0
mewr10h2extvwwmauyepeapih     swarm-worker2   Ready     Active                          29.4.0

```

> **DCA key point:** 3 managers tolerates 1 failure; 5 managers tolerates 2. Always an odd number.
> If quorum is lost, the cluster enters **read-only** mode — it does not accept state
> changes. Recovery: `docker swarm init --force-new-cluster` on the last surviving
> manager (a destructive action, emergency use only).

---

### Exercise 2 — Container vs. service (topics 1.2 and 1.4)

**Objective:** compare `docker run` with `docker service create` and observe the
self-healing behavior of Swarm services on a real cluster.

```bash
# --- Standalone container ---
docker run -d --name c-standalone nginx:alpine
docker kill c-standalone
docker ps -a | grep c-standalone
# Output: Exited — it does not restart
0a523ea206d1   nginx:alpine   "/docker-entrypoint.…"   12 minutes ago   Exited (137) 8 seconds ago             c-standalone
docker rm c-standalone

# --- Swarm service with 3 replicas on a real cluster ---
docker service create \
  --name svc-demo \
  --replicas 3 \
  nginx:alpine

sleep 5
docker service ps svc-demo
# Observe the distribution: the 3 replicas should spread across the 3 nodes
ID             NAME         IMAGE          NODE            DESIRED STATE   CURRENT STATE            ERROR     PORTS
v5iek0c3r0mb   svc-demo.1   nginx:alpine   swarm-worker2   Running         Running 14 seconds ago             
j9cijdtrl309   svc-demo.2   nginx:alpine   docker-labs     Running         Running 18 seconds ago             
ng1d5rtf935s   svc-demo.3   nginx:alpine   swarm-worker1   Running         Running 18 seconds ago  
# See which node each task runs on
docker service ps svc-demo --format "table {{.Name}}\t{{.Node}}\t{{.CurrentState}}"
NAME         NODE            CURRENT STATE
svc-demo.1   swarm-worker2   Running about a minute ago
svc-demo.2   docker-labs     Running about a minute ago
svc-demo.3   swarm-worker1   Running about a minute ago
# Force a container failure and observe recovery
CONTAINER_ID=$(docker service ps svc-demo -q --filter "desired-state=running" | head -1)
# If the task is running on the manager, kill it directly:
docker container ls --filter "name=svc-demo" --format '{{.ID}}' | head -1 | xargs -r docker kill 2>/dev/null || true
# Kills the first running svc-demo container to simulate a failure.
# -r: skips docker kill if there is no container. 2>/dev/null || true: ignores errors and guarantees exit 0.
sleep 5
docker service ps svc-demo
# Expected output: one Shutdown task and one new Running task — Swarm relaunched it
ID             NAME             IMAGE          NODE            DESIRED STATE   CURRENT STATE            ERROR                         PORTS
v5iek0c3r0mb   svc-demo.1       nginx:alpine   swarm-worker2   Running         Running 9 minutes ago                                  
3sqviofgm45z   svc-demo.2       nginx:alpine   docker-labs     Running         Running 53 seconds ago                                 
j9cijdtrl309    \_ svc-demo.2   nginx:alpine   docker-labs     Shutdown        Failed 59 seconds ago    "task: non-zero exit (137)"   
ng1d5rtf935s   svc-demo.3       nginx:alpine   swarm-worker1   Running         Running 9 minutes ago       
# Clean up
docker service rm svc-demo
```

---

### Exercise 3 — `docker inspect` in depth (topic 1.5)

**Objective:** extract information about services, tasks, containers, and nodes using `--format`.

```bash
docker service create \
  --name inspect-demo \
  --replicas 3 \
  --publish published=8080,target=80 \
  --env APP_ENV=production \
  nginx:alpine

sleep 5

# Inspect the service
docker service inspect inspect-demo --format 'Replicas: {{.Spec.Mode.Replicated.Replicas}}'
Replicas: 3
docker service inspect inspect-demo --format 'Image: {{.Spec.TaskTemplate.ContainerSpec.Image}}'
Image: nginx:alpine@sha256:8aa63af009a39ecd6c28d61da578a5447378c10bb097a069e3a3e0fb42bb6b19

docker service inspect inspect-demo \
  --format '{{range .Spec.TaskTemplate.ContainerSpec.Env}}{{.}}{{"\n"}}{{end}}'
APP_ENV=production
# See task distribution by node
docker service ps inspect-demo \
  --format "table {{.Name}}\t{{.Node}}\t{{.CurrentState}}"
NAME             NODE            CURRENT STATE
inspect-demo.1   swarm-worker2   Running 6 minutes ago
inspect-demo.2   docker-labs     Running 6 minutes ago
inspect-demo.3   swarm-worker1   Running 6 minutes ago

# Inspect a specific task
TASK_ID=$(docker service ps inspect-demo -q | head -1)
docker inspect $TASK_ID --format 'State: {{.Status.State}} | Node: {{.NodeID}}'
State: running | Node: mewr10h2extvwwmauyepeapih
# Inspect all nodes in the cluster
docker node ls -q | xargs -I{} docker inspect {} \
  --format 'Node: {{.Description.Hostname}} | Role: {{.Spec.Role}} | State: {{.Status.State}}'
Node: docker-labs | Role: manager | State: ready
Node: swarm-worker1 | Role: worker | State: ready
Node: swarm-worker2 | Role: worker | State: ready
# Clean up
docker service rm inspect-demo
```

---

### Exercise 4 — Deploy stacks with `docker stack deploy` (topics 1.6 and 1.7)

**Objective:** create a multi-service stack with Compose v3, deploy it to the cluster,
and manage updates.

```bash
cd ~/lab3-swarm

cat > stack-demo.yml << 'EOF'
version: "3.8"

services:
  web:
    image: nginx:alpine
    ports:
      - "8081:80"
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
    networks:
      - frontend

  redis:
    image: redis:alpine
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
    networks:
      - frontend
      - backend

networks:
  frontend:
    driver: overlay
  backend:
    driver: overlay
EOF

docker stack deploy -c stack-demo.yml myapp
sleep 10

# See the actual distribution across the 3 nodes
docker stack ps myapp --format "table {{.Name}}\t{{.Node}}\t{{.CurrentState}}"
NAME            NODE            CURRENT STATE
myapp_redis.1   docker-labs     Running about a minute ago
myapp_web.1     swarm-worker2   Running about a minute ago
myapp_web.2     swarm-worker1   Running about a minute ago
myapp_web.3     docker-labs     Running about a minute ago

# Update image (rolling update — topic 1.7)
sed -i 's/nginx:alpine/nginx:1.27-alpine/' stack-demo.yml
# Imperative form (only for standalone services or quick tests):
# docker service update --image nginx:1.27-alpine myapp
docker stack deploy -c stack-demo.yml myapp

# Observe the rolling update in real time
watch -n 2 'docker service ps myapp_web --format "table {{.Name}}\t{{.Node}}\t{{.CurrentState}}"'
# Ctrl+C once all tasks are Running

# Verify access
curl -s -o /dev/null -w "HTTP %{http_code}\n" http://<MANAGER_IP>:8081
HTTP 200
# Clean up
docker stack rm myapp
sleep 5
```

---

### Exercise 5 — Replicas and scaling (topic 1.8)

**Objective:** scale a service and observe how Swarm distributes tasks across nodes.

```bash
docker service create --name svc-scale --replicas 3 nginx:alpine

# See initial distribution (with 3 nodes it should be 1 per node)
docker service ps svc-scale --format "table {{.Name}}\t{{.Node}}\t{{.CurrentState}}"
NAME          NODE            CURRENT STATE
svc-scale.1   docker-labs     Running 58 seconds ago
svc-scale.2   swarm-worker1   Running 58 seconds ago
svc-scale.3   swarm-worker2   Running 58 seconds ago
# Scale to 6 (2 per node)
docker service scale svc-scale=6
sleep 5
docker service ps svc-scale --format "table {{.Name}}\t{{.Node}}\t{{.CurrentState}}"
NAME          NODE            CURRENT STATE
svc-scale.1   docker-labs     Running 2 minutes ago
svc-scale.2   swarm-worker1   Running 2 minutes ago
svc-scale.3   swarm-worker2   Running 2 minutes ago
svc-scale.4   swarm-worker2   Running 11 seconds ago
svc-scale.5   docker-labs     Running 11 seconds ago
svc-scale.6   swarm-worker1   Running 11 seconds ago
# Scale to 2
docker service scale svc-scale=2
sleep 5
docker service ps svc-scale --format "table {{.Name}}\t{{.Node}}\t{{.CurrentState}}"
# With 2 replicas on 3 nodes: 2 nodes will have 1 task, 1 node will have 0
NAME          NODE            CURRENT STATE
svc-scale.1   docker-labs     Running 2 minutes ago
svc-scale.2   swarm-worker1   Running 2 minutes ago

# Scale with update (equivalent)
docker service update --replicas 4 svc-scale
sleep 5
docker service ls | grep svc-scale
yeey7l7skm37   svc-scale   replicated   4/4        nginx:alpine 
docker service rm svc-scale
```

---

### Exercise 6 — Overlay networks and port publishing (topic 1.9)

**Objective:** create overlay networks spanning the 3 nodes and publish ports using both modes.

```bash
docker network create --driver overlay --subnet 10.30.0.0/24 lab3-overlay

docker service create \
  --name svc-network \
  --network lab3-overlay \
  --publish published=8082,target=80 \
  --replicas 3 \
  nginx:alpine

sleep 5

# Verify the service responds from any node in the cluster
# (routing mesh — port 8082 responds on ANY IP in the cluster)
curl -s -o /dev/null -w "From manager: HTTP %{http_code}\n" http://<MANAGER_IP>:8082
From manager: HTTP 200
curl -s -o /dev/null -w "From worker1: HTTP %{http_code}\n" http://<WORKER1_IP>:8082
From worker1: HTTP 200
curl -s -o /dev/null -w "From worker2: HTTP %{http_code}\n" http://<WORKER2_IP>:8082
From worker2: HTTP 200
# Expected output: HTTP 200 on all 3 — this is Swarm's routing mesh

# Add a second network and a port on the fly
docker network create --driver overlay lab3-overlay-b
docker service update --network-add lab3-overlay-b svc-network
docker service update --publish-add published=8083,target=80 svc-network
curl -s -o /dev/null -w "From worker2: HTTP %{http_code}\n" http://<WORKER2_IP>:8083
From worker2: HTTP 200
docker service inspect svc-network --format '{{range .Endpoint.Ports}}{{.PublishedPort}}->{{.TargetPort}}{{"\n"}}{{end}}'
8082->80
8083->80
docker service rm svc-network
docker network rm lab3-overlay lab3-overlay-b
```

---

### Exercise 7 — Volumes in services (topic 1.10)

**Objective:** mount volumes in Swarm services (named, bind, tmpfs).

```bash
# Named volume
docker volume create lab3-data

docker service create \
  --name svc-volume \
  --mount type=volume,source=lab3-data,target=/data \
  --replicas 1 \
  alpine \
  sh -c 'while true; do date >> /data/log.txt; sleep 5; done'
sleep 8
sudo cat $(docker volume inspect lab3-data --format '{{.Mountpoint}}')/log.txt
# The volume was created on worker1, so this command produces no output
docker service ps svc-volume 
ID             NAME           IMAGE           NODE            DESIRED STATE   CURRENT STATE           ERROR     PORTS
lmp0syl26to9   svc-volume.1   alpine:latest   swarm-worker1   Running         Running 4 minutes ago  
docker service rm  svc-volume                                                
docker volume rm lab3-data
docker volume create lab3-data
docker service create --name svc-volume --mount type=volume,source=lab3-data,target=/data --constraint node.role==manager --replicas 1 alpine sh -c 'while true; do date >> /data/log.txt; sleep 5; done'
# Container created with a manager node constraint
sudo cat $(docker volume inspect lab3-data --format '{{.Mountpoint}}')/log.txt
Thu Apr 16 14:54:16 UTC 2026
Thu Apr 16 14:54:21 UTC 2026
Thu Apr 16 14:54:26 UTC 2026
Thu Apr 16 14:54:31 UTC 2026
Thu Apr 16 14:54:36 UTC 2026
Thu Apr 16 14:54:41 UTC 2026
Thu Apr 16 14:54:46 UTC 2026
Thu Apr 16 14:54:51 UTC 2026
Thu Apr 16 14:54:56 UTC 2026
Thu Apr 16 14:55:01 UTC 2026
Thu Apr 16 14:55:06 UTC 2026
Thu Apr 16 14:55:11 UTC 2026
Thu Apr 16 14:55:16 UTC 2026
Thu Apr 16 14:55:21 UTC 2026
Thu Apr 16 14:55:26 UTC 2026
Thu Apr 16 14:55:31 UTC 2026
Thu Apr 16 14:55:36 UTC 2026
Thu Apr 16 14:55:41 UTC 2026
Thu Apr 16 14:55:46 UTC 2026
Thu Apr 16 14:55:51 UTC 2026
Thu Apr 16 14:55:56 UTC 2026
Thu Apr 16 14:56:01 UTC 2026
Thu Apr 16 14:56:06 UTC 2026
Thu Apr 16 14:56:11 UTC 2026
Thu Apr 16 14:56:16 UTC 2026
Thu Apr 16 14:56:21 UTC 2026
Thu Apr 16 14:56:26 UTC 2026
Thu Apr 16 14:56:31 UTC 2026
Thu Apr 16 14:56:36 UTC 2026
Thu Apr 16 14:56:41 UTC 2026
Thu Apr 16 14:56:46 UTC 2026
Thu Apr 16 14:56:51 UTC 2026
Thu Apr 16 14:56:56 UTC 2026
Thu Apr 16 14:57:01 UTC 2026
Thu Apr 16 14:57:06 UTC 2026
Thu Apr 16 14:57:11 UTC 2026
Thu Apr 16 14:57:16 UTC 2026
Thu Apr 16 14:57:21 UTC 2026
Thu Apr 16 14:57:26 UTC 2026
Thu Apr 16 14:57:31 UTC 2026
Thu Apr 16 14:57:36 UTC 2026
Thu Apr 16 14:57:41 UTC 2026
Thu Apr 16 14:57:46 UTC 2026
Thu Apr 16 14:57:51 UTC 2026
Thu Apr 16 14:57:56 UTC 2026
Thu Apr 16 14:58:01 UTC 2026
# Bind mount — if I change the file on the host, it changes inside the container too
mkdir -p ~/lab3-swarm/html
echo "<h1>Lab 3 Swarm</h1>" > ~/lab3-swarm/html/index.html

docker service create --name svc-bindmount \
--mount type=bind,source=/home/<admin_user>/lab3-swarm/html/,target=/usr/share/nginx/html,readonly \
--publish published=8084,target=80 \
--replicas 1 \
--constraint node.role==manager nginx:alpine
curl -s http://<MANAGER_IP>:8084

# tmpfs (ephemeral, useful for sensitive temporary data)
docker service create \
  --name svc-tmpfs \
  --mount type=tmpfs,target=/tmp/cache \
  --replicas 1 \
  alpine sleep 3600

CONTAINER=$(docker container ls --filter "name=svc-tmpfs" --format '{{.ID}}' | head -1)
docker exec $CONTAINER mount | grep tmpfs
tmpfs on /dev type tmpfs (rw,nosuid,size=65536k,mode=755,inode64)
shm on /dev/shm type tmpfs (rw,nosuid,nodev,noexec,relatime,size=65536k,inode64)
tmpfs on /tmp/cache type tmpfs (rw,nosuid,nodev,noexec,relatime,inode64)
tmpfs on /proc/acpi type tmpfs (ro,relatime,inode64)
tmpfs on /proc/interrupts type tmpfs (rw,nosuid,size=65536k,mode=755,inode64)
tmpfs on /proc/kcore type tmpfs (rw,nosuid,size=65536k,mode=755,inode64)
tmpfs on /proc/keys type tmpfs (rw,nosuid,size=65536k,mode=755,inode64)
tmpfs on /proc/timer_list type tmpfs (rw,nosuid,size=65536k,mode=755,inode64)
tmpfs on /sys/firmware type tmpfs (ro,relatime,inode64)
docker service rm svc-volume svc-bindmount svc-tmpfs
docker volume rm lab3-data
```

---

### Exercise 8 — Replicated vs. global services (topic 1.11)

**Objective:** compare `replicated` mode with `global` mode on a 3-node cluster.

```bash
# Replicated mode
docker service create --name svc-replicated --mode replicated --replicas 2 nginx:alpine
docker service ps svc-replicated --format "table {{.Name}}\t{{.Node}}"
# 2 tasks distributed across 2 of the 3 nodes
NAME               NODE
svc-replicated.1   swarm-worker1
svc-replicated.2   swarm-worker2
# Global mode — EXACTLY 1 task per active node
docker service create --name svc-global --mode global nginx:alpine
sleep 5
docker service ps svc-global --format "table {{.Name}}\t{{.Node}}"
# Output: 3 tasks, one on each node — regardless of how many replicas are specified
NAME                                   NODE
svc-global.fnj63blk0xagixwrhtf65i2lq   swarm-worker1
svc-global.mewr10h2extvwwmauyepeapih   swarm-worker2
svc-global.wk3dz1kxkl39u2iuo5ip49g05   docker-labs
docker service ls --format "table {{.Name}}\t{{.Mode}}\t{{.Replicas}}"
# svc-replicated: replicated 2/2
# svc-global:     global    3/3
NAME             MODE         REPLICAS
svc-global       global       3/3
svc-replicated   replicated   2/2
docker service rm svc-replicated svc-global
```

> **DCA key point:** global mode is ideal for monitoring agents, log shippers, and
> proxies that must run on ALL nodes. It cannot be changed after the service is created.

---

### Exercise 9 — Node labels and placement constraints (topic 1.12)

**Objective:** use the labels added in exercise 1 to restrict service placement
across the nodes of the real cluster.

```bash
# Verify existing labels (added in exercise 1)
docker node ls -q | xargs -I{} docker inspect {} \
  --format 'Node: {{.Description.Hostname}} | Labels: {{.Spec.Labels}}'
Node: docker-labs | Labels: map[env:production storage:ssd]
Node: swarm-worker1 | Labels: map[env:staging storage:hdd]
Node: swarm-worker2 | Labels: map[env:staging storage:hdd]
# Service restricted to the node with env=production (manager)
docker service create \
  --name svc-prod \
  --constraint 'node.labels.env == production' \
  --replicas 2 \
  nginx:alpine

docker service ps svc-prod --format "table {{.Name}}\t{{.Node}}\t{{.CurrentState}}"
# All tasks should be on the manager (env=production)
NAME         NODE          CURRENT STATE
svc-prod.1   docker-labs   Running 42 seconds ago
svc-prod.2   docker-labs   Running 42 seconds ago
# Service restricted to workers (env=staging)
docker service create \
  --name svc-staging \
  --constraint 'node.labels.env == staging' \
  --replicas 4 \
  nginx:alpine

sleep 5
docker service ps svc-staging --format "table {{.Name}}\t{{.Node}}\t{{.CurrentState}}"
# 4 tasks spread between worker1 and worker2 (both have env=staging)
# No task on the manager
NAME            NODE            CURRENT STATE
svc-staging.1   swarm-worker1   Running 38 seconds ago
svc-staging.2   swarm-worker2   Running 38 seconds ago
svc-staging.3   swarm-worker1   Running 38 seconds ago
svc-staging.4   swarm-worker2   Running 38 seconds ago
# Placement preference (preferential distribution, not mandatory)
docker service create \
  --name svc-spread \
  --placement-pref 'spread=node.labels.env' \
  --replicas 6 \
  nginx:alpine

sleep 5
docker service ps svc-spread --format "table {{.Name}}\t{{.Node}}"
# ~2 on the manager (production), ~4 on workers (staging, 2 per worker)
NAME           NODE
svc-spread.1   docker-labs
svc-spread.2   docker-labs
svc-spread.3   swarm-worker2
svc-spread.4   swarm-worker1
svc-spread.5   swarm-worker2
svc-spread.6   docker-labs
docker service rm svc-prod svc-staging svc-spread
```

---

### Exercise 10 — Templates in `docker service create` (topic 1.13)

**Objective:** use Go templates so each task receives dynamic values (node,
slot, task).

```bash
docker service create \
  --name svc-template \
  --replicas 3 \
  --env NODE_ID="{{.Node.ID}}" \
  --env NODE_HOSTNAME="{{.Node.Hostname}}" \
  --env TASK_NAME="{{.Task.Name}}" \
  --env TASK_SLOT="{{.Task.Slot}}" \
  alpine \
  sh -c 'echo "Task $TASK_NAME on node $NODE_HOSTNAME (slot $TASK_SLOT)"; sleep 3600'
# useful for processes that need the --env variable information inside the container
sleep 5
docker service logs svc-template 
svc-template.1.sosun1saqc63@docker-labs    | Task svc-template.1.sosun1saqc63tua9k5yo3iccn on node docker-labs (slot 1)
svc-template.2.rcp2mzohcns2@swarm-worker1    | Task svc-template.2.rcp2mzohcns2epsd4d61hmkdk on node swarm-worker1 (slot 2)
svc-template.3.udnxgxt7f7f2@swarm-worker2    | Task svc-template.3.udnxgxt7f7f2p43es168kavf8 on node swarm-worker2 (slot 3)
# Verify that each task has its own context
# On a real cluster: NODE_HOSTNAME and TASK_SLOT differ between containers
for CONTAINER in $(docker container ls --filter "name=svc-template" --format '{{.ID}}');do echo "--- Container $CONTAINER ---"; docker exec $CONTAINER env | grep -E "NODE|TASK"; done
--- Container 29a34c854e12 ---
NODE_ID=wk3dz1kxkl39u2iuo5ip49g05
NODE_HOSTNAME=docker-labs
TASK_NAME=svc-template.1.sosun1saqc63tua9k5yo3iccn
TASK_SLOT=1
#Only shows the container running on docker-labs
# Dynamic hostname per slot
docker service create \
  --name svc-hostname-tpl \
  --hostname "replica-{{.Task.Slot}}.lab3" \
  --replicas 3 \
  alpine sleep 3600

for CONTAINER in $(docker container ls --filter "name=svc-hostname-tpl" --format '{{.ID}}'); do
  docker exec $CONTAINER hostname;
done
# Expected output: replica-1.lab3, replica-2.lab3, replica-3.lab3
replica-1.lab3
#Only shows the container running on the node where the command executes

docker service rm svc-template svc-hostname-tpl
```

---

### Exercise 11 — Service troubleshooting (topic 1.14)

**Objective:** troubleshoot the most common cases: nonexistent image, impossible
constraint, and port conflict.

```bash
# Case 1: image not available
docker service create --name svc-bad-image --replicas 1 nginx:version-que-no-existe
sleep 5
docker service ps svc-bad-image --no-trunc
# Error: "No such image" or "manifest unknown"
ID                          NAME                  IMAGE              NODE            DESIRED STATE   CURRENT STATE             ERROR                               PORTS
yuum9tivcbubxytyaj8aedt4g   svc-bad-image.1       nginx:badversion   swarm-worker2   Ready           Rejected 1 second ago     "No such image: nginx:badversion"   
1ngb450pzwc1gg727of6tf0is    \_ svc-bad-image.1   nginx:badversion   swarm-worker2   Shutdown        Rejected 6 seconds ago    "No such image: nginx:badversion"   
zqheb8dal09lcvuyn9yjkemd6    \_ svc-bad-image.1   nginx:badversion   swarm-worker2   Shutdown        Rejected 11 seconds ago   "No such image: nginx:badversion"   
yvqs2ivoa5z9gwq9p1hmf67l2    \_ svc-bad-image.1   nginx:badversion   swarm-worker2   Shutdown        Rejected 16 seconds ago   "No such image: nginx:badversion"   
m8tvsttbaqg2jle79cfh04g3o    \_ svc-bad-image.1   nginx:badversion   swarm-worker1   Shutdown        Rejected 21 seconds ago   "No such image: nginx:badversion" 
docker service rm svc-bad-image

# Case 2: impossible constraint
docker service create \
  --name svc-bad-constraint \
  --constraint 'node.labels.nonexistent == true' \
  --replicas 1 \
  nginx:alpine

sleep 5
docker service ps svc-bad-constraint --no-trunc
# Task in Pending: "no suitable node (scheduling constraints not satisfied)"
ID             NAME                   IMAGE          NODE      DESIRED STATE   CURRENT STATE            ERROR                              PORTS
wb48a42w4bqq   svc-bad-constraint.1   nginx:alpine             Running         Pending 24 seconds ago   "no suitable node (scheduling …" 
# Solution: add the missing label to a node
docker node update --label-add nonexistent=true $WORKER1_ID
sleep 5
docker service ps svc-bad-constraint
# The task moves to Running on worker1
ID             NAME                   IMAGE          NODE            DESIRED STATE   CURRENT STATE            ERROR     PORTS
wb48a42w4bqq   svc-bad-constraint.1   nginx:alpine   swarm-worker1   Running         Running 15 seconds ago         
docker service rm svc-bad-constraint
docker node update --label-rm nonexistent $WORKER1_ID

# Case 3: port already in use
docker run -d --name busy -p 8085:80 nginx:alpine
docker service create \
  --name svc-port-conflict \
  --publish published=8085,target=80 \
  --replicas 1 \
  --constraint 'node.role == manager' \
  nginx:alpine

sleep 5
docker service ps svc-port-conflict --no-trunc
# Error: bind: address already in use
sudo ss -tlnp | grep 8085
LISTEN 0      4096         0.0.0.0:8085      0.0.0.0:*    users:(("docker-proxy",pid=200591,fd=8)) 
LISTEN 0      4096            [::]:8085         [::]:*    users:(("docker-proxy",pid=200596,fd=8)) 
#docker-proxy = standalone container occupying the port
docker service rm svc-port-conflict
docker rm -f busy

# General troubleshooting flow:
# 1. docker service ps <svc> --no-trunc     → task error
# 2. docker inspect <task-id>               → full detail
# 3. docker logs <container-id>             → container logs
# 4. docker events --since "5m"             → event timeline
# 5. sudo journalctl -u docker.service      → daemon logs
```

---

### Exercise 12 — Dockerized app communicating with a legacy system (topic 1.15)

**Objective:** connect a Swarm service to a legacy process running on the host.

```bash
# Simulate a legacy system on the manager
python3 -m http.server 9001 --directory /etc/docker &
LEGACY_PID=$!

curl -s -o /dev/null -w "Legacy HTTP: %{http_code}\n" http://127.0.0.1:9001
Legacy HTTP: 200
# Method 1: --network host (global mode on the cluster)
docker service create \
  --name svc-legacy-host \
  --network host \
  --mode global \
  --constraint 'node.role == manager' \
  alpine \
  sh -c 'wget -qO- http://127.0.0.1:9001 | head -3; sleep 3600'
Serving HTTP on 0.0.0.0 port 9001 (http://0.0.0.0:9001/) ...
127.0.0.1 - - [19/Apr/2026 08:45:55] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [19/Apr/2026 08:57:21] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [19/Apr/2026 08:59:05] "GET / HTTP/1.1" 200 -
#The container's requests show up here
sleep 5
CONTAINER=$(docker container ls --filter "name=svc-legacy-host" --format '{{.ID}}' | head -1)
docker logs $CONTAINER 2>/dev/null | head -3
docker service rm svc-legacy-host

# Method 2: host gateway IP
HOST_IP=$(docker network inspect bridge --format '{{(index .IPAM.Config 0).Gateway}}')
docker service create \
  --name svc-legacy-gw \
  --replicas 1 \
  --env LEGACY_URL="http://$HOST_IP:9001" \
  alpine \
  sh -c 'wget -qO- $LEGACY_URL | head -3; sleep 3600'

sleep 5
CONTAINER=$(docker container ls --filter "name=svc-legacy-gw" --format '{{.ID}}' | head -1)
docker logs $CONTAINER 2>/dev/null | head -3
docker service rm svc-legacy-gw

# Method 3: extra_hosts (custom DNS name)
docker service create \
  --name svc-legacy-dns \
  --host legacy-system:$HOST_IP \
  --replicas 1 \
  alpine \
  sh -c 'wget -qO- http://legacy-system:9001 | head -3; sleep 3600'

sleep 5
CONTAINER=$(docker container ls --filter "name=svc-legacy-dns" --format '{{.ID}}' | head -1)
docker exec $CONTAINER cat /etc/hosts | grep legacy
docker service rm svc-legacy-dns

kill $LEGACY_PID 2>/dev/null || true
```

---

## Setup — Part 2: k3s Cluster

Exercises 13 and 14 use the same 3 VMs. Docker can remain installed —
k3s uses its own containerd and does not conflict with Docker.

> **Why k3s instead of kubeadm?** k3s installs a CNCF-certified Kubernetes in < 5 min,
> with no external dependencies (embedded etcd, CNI included). It's ideal for a homelab and for
> learning `kubectl`. For dedicated CKA prep I'll use kubeadm, but the concepts
> are identical: same API, same kubectl, same objects.
>
> **Making good use of the mini-server spec:** The i5-12600H has plenty of headroom for a
> k3s control plane (VM 200, 4 vCPU/8 GB). The k3s worker VMs (2 vCPU/2 GB each)
> comfortably meet k3s's minimum requirements (512 MB RAM / 1 vCPU per node).

---

### Step 2A — Swarm teardown

```bash
# From the manager (200) — remove workers before leaving
docker node rm --force $WORKER1_ID 2>/dev/null || true
docker node rm --force $WORKER2_ID 2>/dev/null || true

# Leave the swarm on the workers
ssh <admin_user>@<WORKER1_IP> "docker swarm leave --force"
ssh <admin_user>@<WORKER2_IP> "docker swarm leave --force"

# Leave the swarm on the manager
docker swarm leave --force

# Verify Swarm is inactive
docker info | grep "Swarm"
# Expected output: Swarm: inactive
```

---

### Step 2B — Install the k3s server (VM 200)

```bash
# Connect to the manager (future k3s server)
# ssh -J <admin_user>@vps -p 2222 <admin_user>@localhost

# Install the k3s server
# --write-kubeconfig-mode 644 allows reading the kubeconfig without sudo
# --node-name sets the node's name in the cluster
curl -sfL https://get.k3s.io | sh -s - \
  --write-kubeconfig-mode 644 \
  --node-name k3s-server \
  --bind-address <MANAGER_IP> \
  --advertise-address <MANAGER_IP> \
  --node-ip <MANAGER_IP>

# Verify installation
sudo systemctl status k3s
kubectl get nodes
# Expected output (may take 30s): k3s-server   Ready   control-plane,master
NAME         STATUS   ROLES           AGE     VERSION
k3s-server   Ready    control-plane   4m41s   v1.34.6+k3s1
# Get the token to join the agents
K3S_TOKEN=$(sudo cat /var/lib/rancher/k3s/server/node-token)
echo "Token saved: ${K3S_TOKEN:0:20}..."
Token saved: K10<redacted>
# kubectl is already available in the PATH
kubectl version
Client Version: v1.34.6+k3s1
Kustomize Version: v5.7.1
Server Version: v1.34.6+k3s1
```

---

### Step 2C — Install the k3s agents (VMs 201 and 202)

```bash
# Get the token if you don't already have it in the variable
K3S_TOKEN=$(sudo cat /var/lib/rancher/k3s/server/node-token)

# Install the k3s agent on worker1
ssh <admin_user>@<WORKER1_IP> "
  curl -sfL https://get.k3s.io | K3S_URL=https://<MANAGER_IP>:6443 \
    K3S_TOKEN=$K3S_TOKEN \
    sh -s - \
    --node-name k3s-agent1 \
    --node-ip <WORKER1_IP>
"

# Install the k3s agent on worker2
ssh <admin_user>@<WORKER2_IP> "
  curl -sfL https://get.k3s.io | K3S_URL=https://<MANAGER_IP>:6443 \
    K3S_TOKEN=$K3S_TOKEN \
    sh -s - \
    --node-name k3s-agent2 \
    --node-ip <WORKER2_IP>
"

# Wait for the nodes to join (may take ~60s)
sleep 60
kubectl get nodes -o wide
NAME         STATUS   ROLES           AGE   VERSION        INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION   CONTAINER-RUNTIME
k3s-agent1   Ready    <none>          59s   v1.34.6+k3s1   <WORKER1_IP>   <none>        Debian GNU/Linux 12 (bookworm)   6.1.0-44-amd64   containerd://2.2.2-bd1.34
k3s-agent2   Ready    <none>          29s   v1.34.6+k3s1   <WORKER2_IP>   <none>        Debian GNU/Linux 12 (bookworm)   6.1.0-44-amd64   containerd://2.2.2-bd1.34
k3s-server   Ready    control-plane   21m   v1.34.6+k3s1   <MANAGER_IP>   <none>        Debian GNU/Linux 12 (bookworm)   6.1.0-44-amd64   containerd://2.2.2-bd1.34
```

---

### Step 2D — Prepare the k3s working directory

```bash
mkdir -p ~/lab3-k8s && cd ~/lab3-k8s

# Verify kubectl works correctly
kubectl cluster-info
kubectl get nodes -o wide
kubectl get namespaces
# Default k3s namespaces: default, kube-system, kube-public, kube-node-lease
NAME              STATUS   AGE
default           Active   25m
kube-node-lease   Active   25m
kube-public       Active   25m
kube-system       Active   25m

```

---

## Procedure — Kubernetes (k3s)

---

### Exercise 13 — Pods and Deployments in Kubernetes (topic 1.16)

**Objective:** create pods and deployments and scale them with kubectl. Compare them
with their Docker Swarm equivalents.

```bash
cd ~/lab3-k8s

# --- Imperative pod (the minimal unit in K8s, equivalent to a Swarm task) ---
kubectl run pod-demo --image=nginx:alpine --port=80
kubectl get pods -o wide
# The NODE column shows which cluster node the pod runs on
NAME       READY   STATUS    RESTARTS   AGE   IP          NODE         NOMINATED NODE   READINESS GATES
pod-demo   1/1     Running   0          10s   10.42.0.9   k3s-server   <none>           <none>
kubectl describe pod pod-demo
# Observe: Node, IP, State, Image, Events
Name:             pod-demo
Namespace:        default
Priority:         0
Service Account:  default
Node:             k3s-server/<MANAGER_IP>
Start Time:       Mon, 20 Apr 2026 09:10:29 +0000
Labels:           run=pod-demo
Annotations:      <none>
Status:           Running
IP:               10.42.0.9
IPs:
  IP:  10.42.0.9
Containers:
  pod-demo:
    Container ID:   containerd://32cc5457f2818ad232b37b92c02dd82491e79f96d926fb6065f37c29155ba0dc
    Image:          nginx:alpine
    Image ID:       docker.io/library/nginx@sha256:5616878291a2eed594aee8db4dade5878cf7edcb475e59193904b198d9b830de
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 20 Apr 2026 09:10:32 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-vs4h9 (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       True 
  ContainersReady             True 
  PodScheduled                True 
Volumes:
  kube-api-access-vs4h9:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  2m48s  default-scheduler  Successfully assigned default/pod-demo to k3s-server
  Normal  Pulling    2m48s  kubelet            Pulling image "nginx:alpine"
  Normal  Pulled     2m45s  kubelet            Successfully pulled image "nginx:alpine" in 3.36s (3.36s including waiting). Image size: 26013346 bytes.
  Normal  Created    2m45s  kubelet            Created container: pod-demo
  Normal  Started    2m45s  kubelet            Started container pod-demo
# View the pod's logs
kubectl logs pod-demo
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2026/04/20 09:10:32 [notice] 1#1: using the "epoll" event method
2026/04/20 09:10:32 [notice] 1#1: nginx/1.29.8
2026/04/20 09:10:32 [notice] 1#1: built by gcc 15.2.0 (Alpine 15.2.0) 
2026/04/20 09:10:32 [notice] 1#1: OS: Linux 6.1.0-44-amd64
2026/04/20 09:10:32 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2026/04/20 09:10:32 [notice] 1#1: start worker processes
2026/04/20 09:10:32 [notice] 1#1: start worker process 31
2026/04/20 09:10:32 [notice] 1#1: start worker process 32
2026/04/20 09:10:32 [notice] 1#1: start worker process 33
2026/04/20 09:10:32 [notice] 1#1: start worker process 34
10.42.0.1 - - [20/Apr/2026:09:15:04 +0000] "GET / HTTP/1.1" 200 896 "-" "curl/7.88.1" "-"
# Run a command in the pod (equivalent to docker exec)
kubectl exec -it pod-demo -- /bin/sh -c "nginx -v; exit"
nginx version: nginx/1.29.8
# Delete the pod (if it fails, it does NOT restart — same as docker run without --restart)
kubectl delete pod pod-demo
kubectl get pods
# Output: No resources found — the pod was not recreated
No resources found in default namespace.
# --- Deployment (equivalent to docker service create) ---
kubectl create deployment web-demo \
  --image=nginx:alpine \
  --replicas=3 \
  --port=80

kubectl get deployments
kubectl get pods -o wide
# The 3 replicas (pods) are distributed across the cluster's 3 nodes
# Observe the NODE column
NAME                        READY   STATUS    RESTARTS   AGE   IP           NODE         NOMINATED NODE   READINESS GATES
web-demo-7fd5d59cc4-h74tr   1/1     Running   0          26s   10.42.2.3    k3s-agent2   <none>           <none>
web-demo-7fd5d59cc4-tmmzg   1/1     Running   0          26s   10.42.0.10   k3s-server   <none>           <none>
web-demo-7fd5d59cc4-w68p8   1/1     Running   0          26s   10.42.1.3    k3s-agent1   <none> 
kubectl rollout status deployment/web-demo
# Output: successfully rolled out — shows the real-time deployment (rollout) status
deployment "web-demo" successfully rolled out

# Scale (equivalent to docker service scale)
kubectl scale deployment web-demo --replicas=5
kubectl get pods -o wide
# 5 pods distributed across k3s-server, k3s-agent1, k3s-agent2
NAME                        READY   STATUS    RESTARTS   AGE     IP           NODE         NOMINATED NODE   READINESS GATES
web-demo-7fd5d59cc4-bfxrx   1/1     Running   0          21s     10.42.0.11   k3s-server   <none>           <none>
web-demo-7fd5d59cc4-h74tr   1/1     Running   0          3m45s   10.42.2.3    k3s-agent2   <none>           <none>
web-demo-7fd5d59cc4-t9t4j   1/1     Running   0          21s     10.42.1.4    k3s-agent1   <none>           <none>
web-demo-7fd5d59cc4-tmmzg   1/1     Running   0          3m45s   10.42.0.10   k3s-server   <none>           <none>
web-demo-7fd5d59cc4-w68p8   1/1     Running   0          3m45s   10.42.1.3    k3s-agent1   <none>           <none>

# Verify self-healing: delete a pod
POD_NAME=$(kubectl get pods -l app=web-demo -o name | head -1 | cut -d/ -f2)
kubectl delete pod $POD_NAME
sleep 10
kubectl get pods
# Output: 5 pods Running — the Deployment recreated the deleted pod
NAME                        READY   STATUS    RESTARTS   AGE
web-demo-7fd5d59cc4-dmfnp   1/1     Running   0          22s
web-demo-7fd5d59cc4-h74tr   1/1     Running   0          7m4s
web-demo-7fd5d59cc4-t9t4j   1/1     Running   0          3m40s
web-demo-7fd5d59cc4-tmmzg   1/1     Running   0          7m4s
web-demo-7fd5d59cc4-w68p8   1/1     Running   0          7m4s
# --- Expose the deployment as a NodePort service ---
kubectl expose deployment web-demo \
  --type=NodePort \
  --port=80 \
  --target-port=80

kubectl get svc web-demo
kubectl get svc web-demo
NAME       TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
web-demo   NodePort   10.43.4.101   <none>        80:31790/TCP   4m53s
NODE_PORT=$(kubectl get svc web-demo -o jsonpath='{.spec.ports[0].nodePort}')
echo "NodePort: $NODE_PORT"

# Access from any node in the cluster (routing equivalent to Swarm's routing mesh)
curl -s -o /dev/null -w "From server: HTTP %{http_code}\n" http://<MANAGER_IP>:$NODE_PORT
HTTP: 200
curl -s -o /dev/null -w "From agent1: HTTP %{http_code}\n" http://<WORKER1_IP>:$NODE_PORT
HTTP: 200
curl -s -o /dev/null -w "From agent2: HTTP %{http_code}\n" http://<WORKER2_IP>:$NODE_PORT
HTTP: 200

# --- Rolling update (equivalent to docker service update --image) ---
kubectl set image deployment/web-demo nginx=nginx:1.27-alpine
kubectl rollout status deployment/web-demo
# Swaps pods one at a time — observe the rolling update

kubectl get pods -o wide
# New version: nginx:1.27-alpine

# Rollback
kubectl rollout undo deployment/web-demo
kubectl rollout status deployment/web-demo

# --- Swarm vs K8s comparison ---
echo "=== Comparison table ==="
echo "docker service create  → kubectl create deployment"
echo "docker service scale   → kubectl scale deployment"
echo "docker service ps      → kubectl get pods"
echo "docker stack deploy    → kubectl apply -f"
echo "docker service update  → kubectl set image / kubectl apply"
echo "docker service rollback→ kubectl rollout undo"

# Clean up
kubectl delete deployment web-demo
kubectl delete svc web-demo
kubectl delete pod pod-demo 2>/dev/null || true
```

---

### Exercise 14 — ConfigMaps and Secrets in Kubernetes (topic 1.17)

**Objective:** pass configuration to pods via ConfigMaps (non-sensitive data) and
Secrets (sensitive data), in both modes: environment variables and mounted files.

```bash
cd ~/lab3-k8s

# === ConfigMap ===

# Create a ConfigMap from literals (equivalent to --env in Swarm)
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_PORT=8080 \
  --from-literal=LOG_LEVEL=info

kubectl get configmap app-config
NAME         DATA   AGE
app-config   3      8s
kubectl describe configmap app-config
Name:         app-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
APP_ENV:
----
production

APP_PORT:
----
8080

LOG_LEVEL:
----
info


BinaryData
====

Events:  <none>

# Create a ConfigMap from a file (equivalent to --mount bind of a config file)
cat > app.properties << 'EOF'
database.host=postgres.internal
database.port=5432
database.name=myapp_db
max_connections=100
EOF

kubectl create configmap db-config --from-file=app.properties
kubectl describe configmap db-config
Name:         db-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
app.properties:
----
database.host=postgres.internal
database.port=5432
database.name=myapp_db
max_connections=100



BinaryData
====

Events:  <none>
# Pod that uses the ConfigMap as environment variables
cat > pod-configmap-env.yml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-cm-env
spec:
  containers:
  - name: app
    image: alpine
    command: ["sh", "-c", "env | grep -E 'APP_|LOG_'; sleep 3600"]
    envFrom:
    - configMapRef:
        name: app-config
EOF

kubectl apply -f pod-configmap-env.yml
sleep 10
kubectl logs pod-cm-env
# Output: APP_ENV=production, APP_PORT=8080, LOG_LEVEL=info
LOG_LEVEL=info
APP_PORT=8080
APP_ENV=production
# Pod that uses the ConfigMap as a mounted file
cat > pod-configmap-file.yml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-cm-file
spec:
  containers:
  - name: app
    image: alpine
    command: ["sh", "-c", "cat /config/app.properties; sleep 3600"]
    volumeMounts:
    - name: config-volume
      mountPath: /config
  volumes:
  - name: config-volume
    configMap:
      name: db-config
EOF

kubectl apply -f pod-configmap-file.yml
sleep 10
kubectl logs pod-cm-file
# Output: contents of app.properties
database.host=postgres.internal
database.port=5432
database.name=myapp_db
max_connections=100
# === Secret ===

# Create a Secret from literals (equivalent to docker secret create)
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=S3cur3P@ssw0rd

kubectl get secret db-credentials
kubectl describe secret db-credentials
# The values appear as [3 bytes] / [16 bytes] — not in plaintext

# View the value (base64 encoded) — for lab verification only
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 --decode
echo ""

# Pod that uses the Secret as environment variables
cat > pod-secret-env.yml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret-env
spec:
  containers:
  - name: app
    image: alpine
    command: ["sh", "-c", "echo User: $DB_USER; echo Pass: $DB_PASS; sleep 3600"]
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
EOF

kubectl apply -f pod-secret-env.yml
sleep 10
kubectl logs pod-secret-env
# Output: User: admin / Pass: S3cur3P@ssw0rd
User: admin
Pass: S3cur3P@ssw0rd
# Pod that uses the Secret as a mounted file (at /run/secrets — same path as Swarm)
cat > pod-secret-file.yml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret-file
spec:
  containers:
  - name: app
    image: alpine
    command: ["sh", "-c", "ls /app/secrets/; cat /app/secrets/password; echo; sleep 3600"]
    volumeMounts:
    - name: secret-volume
      mountPath: /app/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
EOF

kubectl apply -f pod-secret-file.yml
sleep 10
kubectl logs pod-secret-file
# Output: file list (username, password) + contents of password
password
username
S3cur3P@ssw0rd

# --- Swarm vs K8s comparison ---
echo "=== Secrets Swarm vs K8s ==="
echo "docker secret create   → kubectl create secret generic"
echo "docker service --secret → spec.volumes[secret] + volumeMounts"
echo "Mount path in Swarm:  /run/secrets/<name>"
echo "Mount path in K8s:    configurable (mountPath)"
echo "Encryption in Swarm:  encrypted Raft store"
echo "Encryption in K8s:    etcd (k3s: SQLite) — encryption at rest optional"

# Clean up
kubectl delete pod pod-cm-env pod-cm-file pod-secret-env pod-secret-file
kubectl delete configmap app-config db-config
kubectl delete secret db-credentials
rm -f app.properties pod-configmap-env.yml pod-configmap-file.yml pod-secret-env.yml pod-secret-file.yml
```

---

## Verification

```bash
# === Swarm ===
# Active 3-node cluster
docker node ls | grep -E "Leader|worker"

# Create and scale a distributed service
docker service create --name verify-svc --replicas 3 nginx:alpine
sleep 5
docker service ps verify-svc --format "table {{.Name}}\t{{.Node}}\t{{.CurrentState}}"
# Expected output: 3 tasks Running on different nodes
docker service rm verify-svc

# === k3s ===
# Active 3-node cluster
kubectl get nodes
# Expected output: 3 nodes in Ready state

# Basic deployment works
kubectl create deployment verify-k8s --image=nginx:alpine --replicas=3
sleep 10
kubectl get pods -o wide | grep verify-k8s
# 3 pods Running, distributed across the nodes
kubectl delete deployment verify-k8s

echo "=== Verification complete ==="
```

---

## Incidents

| Incident | Probable cause | Solution |
| --- | --- | --- |
| `docker swarm join` fails with "token invalid" | Expired token or incorrect IP | Regenerate: `docker swarm join-token --rotate worker` |
| Task stuck in `pending` on a specific node | Constraint not satisfied on that node | `docker service ps <svc> --no-trunc` → adjust labels |
| `--network host` fails with replicas >1 | Host networking incompatible with routing mesh | Use `global` mode with a role constraint |
| k3s nodes in `NotReady` after joining | Flannel CNI still initializing | Wait 60-90s; `kubectl describe node <name>` to see events |
| kubectl: `connection refused` on port 6443 | k3s server hasn't fully started | `sudo systemctl status k3s`; `sudo journalctl -u k3s -n 50` |
| Pod in `Pending` due to insufficient resources | Node without available RAM/CPU | `kubectl describe pod <pod>` → Events section |
| K8s Secret not mounting | Wrong namespace | Verify the pod and the secret are in the same namespace |
| `docker swarm leave` fails with "manager" | The node is an active manager | `--force` only if it's the last manager or it has already been demoted |
| kubectl autocomplete not available | Autocompletion not configured in bashrc | Add to ~/.bashrc:  echo 'source <(kubectl completion bash)' >> ~/.bashrc; source ~/.bashrc

---

## DCA/SRE Lessons

### Key points for the DCA exam

- **Quorum:** always odd. 3 managers tolerates 1 failure; 5 tolerates 2.
- **Global mode:** exactly 1 task per node. Cannot be changed after creation.
- **Constraints:** MUST. If no node satisfies it → the task stays pending.
- **Preferences:** PREFER. If it can't be honored, Swarm still distributes the task.
- **Routing mesh:** a service published on port X is reachable on ANY IP in the cluster.
- **Templates:** `{{.Task.Slot}}` is unique per replica; `{{.Node.Hostname}}` varies between nodes.
- **K8s pod vs. Swarm task:** pod = minimal unit. If the pod fails without a Deployment → it does not recover.
- **K8s Deployment vs. Swarm service:** both guarantee N replicas and recover them on failure.
- **ConfigMap:** non-sensitive data. Secret: sensitive data (base64, not real encryption by default).
- **docker secret:** encrypted in the Raft store. K8s secret: in etcd/SQLite, encryption at rest optional.

### Notes for CKA (pre-training)

- Exercises 13 and 14 use the real Kubernetes API — the same objects and commands as CKA.
- k3s is CNCF-certified Kubernetes: `kubectl`, YAML manifests, rolling updates, and RBAC are identical to kubeadm.
- For CKA: practice `kubectl apply -f` instead of imperative commands, and know the YAML manifests for Pod, Deployment, Service, ConfigMap, and Secret.
- Official CKA reference: kubernetes.io/docs (allowed during the exam).

Mapping to the DCA domain: **Domain 1 — Orchestration (25% of the exam)**
