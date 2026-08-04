# Lab 2 — Docker Networking — Execution Log

## Objective

Consolidate the Docker networking mechanisms studied in Week 3: the Container Network
Model (CNM), built-in network drivers, port publishing, DNS, overlay networks,
and connectivity troubleshooting.

The goal is to understand network namespaces as the foundation of the CNM, create and manage
bridge and overlay networks, verify connectivity between containers, and diagnose
real issues using engine logs.

DCA competencies covered: 4.1, 4.2, 4.4, 4.5, 4.6, 4.7, 4.8, 4.10, 4.11
(conceptual: 4.3, 4.9, 4.12, 4.13)

---

## Environment

| Element | Value |
|---|---|
| Node | Docker Labs VM — <MANAGER_IP> |
| OS | Debian 12 |
| Docker Engine | 29.3.1 (verify with `docker version`) |
| Access | `ssh -J <admin_user>@vps -p 2222 <admin_user>@localhost` |
| User | <admin_user> (sudo available) |

Prerequisites:
- Docker Engine installed and the daemon active (`systemctl is-active docker`)
- The <admin_user> user belongs to the docker group (`groups | grep docker`)
- `iproute2` tooling available on the VM (`ip` and `bridge` commands)
- For the overlay exercise: Swarm initialized (`docker swarm init`)

---

## Setup

```bash
# 1. Connect to the VM
ssh -J <admin_user>@vps -p 2222 <admin_user>@localhost

# 2. Verify Docker Engine is operational
docker version
docker info | grep -E "Server Version|Swarm|Default Network"

# 3. Verify available network tools
ip -V
ip utility, iproute2-6.1.0, libbpf 1.1.2
bridge --version 2>/dev/null || echo "bridge included in iproute2"

# 4. Create working directory
mkdir -p ~/lab2-networking && cd ~/lab2-networking

# 5. Clean up networks and containers from previous labs
docker container prune -f
docker network prune -f

# 6. Initialize Swarm (needed for overlay in exercise 6)
docker swarm init --advertise-addr <MANAGER_IP>
# If already initialized: docker info | grep "Swarm: active"
docker info | grep "Swarm: active"
 Swarm: active

```

---

## Procedure

---

### Foundation — Network namespaces and veth pairs (CNM basis)

**Goal:** understand the kernel mechanism Docker uses to isolate each container's
network before working with the higher-level drivers.

```bash
# View the network namespaces active on the host (before starting containers)
sudo ip netns list
# Note: Docker manages its own netns outside of /var/run/netns — they don't appear here by default

# Start a container and observe its netns
docker run -d --name ns-test nginx:alpine

# View the container's PID
CPID=$(docker inspect ns-test --format '{{.State.Pid}}')
echo "Container PID: $CPID"
PID: 19210

# Inspect the container's network namespace from the host
sudo nsenter -t $CPID -n ip addr
# Expected output: eth0 interface with its own IP + lo (loopback)
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0@if100: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether e2:e9:fd:38:57:8a brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever

# View the host veth interface that connects to that container
ip addr show type veth
# Each container has a veth pair: vethXXXXXX (host) <-> eth0 (container)
100: veth2b724c7@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
    link/ether 4a:c9:42:26:ee:87 brd ff:ff:ff:ff:ff:ff link-netnsid 2
    inet6 fe80::48c9:42ff:fe26:ee87/64 scope link 
       valid_lft forever preferred_lft forever

# View the docker0 bridge and its connected veths
sudo bridge link show
# Expected output: vethXXXXXX master docker0
100: veth2b724c7@eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master docker0 state forwarding priority 32 cost 2 

# View the host's routing table towards the containers
ip r
# Expected output: 172.17.0.0/16 dev docker0 proto kernel
default via <LAN_GATEWAY_IP> dev eth0 proto static 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 
172.18.0.0/16 dev docker_gwbridge proto kernel scope link src 172.18.0.1 linkdown 
<HOMELAB_LAN>/24 dev eth0 proto kernel scope link src <MANAGER_IP> 

# Clean up
docker rm -f ns-test
```

**Key concept:** each container has its own network namespace with an isolated network
stack. Docker connects that namespace to the `docker0` bridge via a veth pair. This is
exactly what the Container Network Model (CNM) implements.

---

### Exercise 1 — Container Network Model and IPAM (topic 4.1)

**Goal:** explore the CNM architecture and IP address management (IPAM) in Docker.

```bash
# List all networks available by default
docker network ls
# Expected output: bridge, host, none
NETWORK ID     NAME      DRIVER    SCOPE
7cc8d801bddc   bridge    bridge    local
07af409e4e22   host      host      local
da09dde8b611   none      null      local
# Inspect the default bridge network — IPAM, subnet, gateway
docker network inspect bridge
# Observe: "IPAM.Config" with Subnet and Gateway, "Driver": "bridge"
[
    {
        "Name": "bridge",
        "Id": "7cc8d801bddca44bbeab4d8802aa9b8be652dfcd685a3ec0a317b72ca604d0ea",
        "Created": "2026-04-08T12:13:42.03358654Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv4": true,
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {},
        "Containers": {},
        "Status": {
            "IPAM": {
                "Subnets": {
                    "172.17.0.0/16": {
                        "IPsInUse": 3,
                        "DynamicIPsAvailable": 65533
                    }
                }
            }
        }
    }
]

# Create a network with custom IPAM
docker network create \
  --driver bridge \
  --subnet 10.10.0.0/24 \
  --gateway 10.10.0.1 \
  --ip-range 10.10.0.128/25 \
  lab2-ipam

# Verify IPAM configuration
docker network inspect lab2-ipam | grep -A 15 '"IPAM"'
       "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "10.10.0.0/24",
                    "IPRange": "10.10.0.128/25",
                    "Gateway": "10.10.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
--
            "IPAM": {
                "Subnets": {
                    "10.10.0.0/24": {
                        "IPsInUse": 3,
                        "DynamicIPsAvailable": 127
                    }
                }
            }
        }
    }
]

# Start a container with a fixed IP within the range
docker run -d --name ipam-test \
  --network lab2-ipam \
  --ip 10.10.0.200 \
  nginx:alpine

# Verify the assigned IP
 docker inspect ipam-test --format '{{index .NetworkSettings.Networks "lab2-ipam" "IPAddress"}}'
10.10.0.200

# Expected output: 10.10.0.200

# Clean up
docker rm -f ipam-test
```

---

### Exercise 2 — Built-in network drivers (topic 4.2)

**Goal:** compare the behavior of the bridge, host, and none drivers.

```bash
# --- Driver: bridge (default) ---
docker run -d --name c-bridge --network bridge nginx:alpine
docker inspect c-bridge --format 'Driver: {{.HostConfig.NetworkMode}} | IP:{{.NetworkSettings.Networks.bridge.IPAddress}}'
Driver: bridge | IP:172.17.0.2


# --- Driver: host (no network namespace of its own) ---
docker run -d --name c-host --network host --userns=host nginx:alpine
# The container uses the host's network stack directly; --userns=host had to be specified because userns-remap="default" was active by default, so with user namespace isolation enabled we can't use --network host without breaking that isolation.
docker inspect c-host --format 'NetworkMode: {{.HostConfig.NetworkMode}}'
NetworkMode: host
# Verify: port 80 is open directly on the host
ss -tlnp | grep :80
LISTEN 0      511          0.0.0.0:80        0.0.0.0:*          
LISTEN 0      511             [::]:80           [::]:*    
# --- Driver: none (no network) ---
docker run -d --name c-none --network none nginx:alpine
docker inspect c-none --format 'NetworkMode: {{.HostConfig.NetworkMode}}'
NetworkMode: none
# Verify: no network interfaces (loopback only)
docker exec c-none ip addr
# Expected output: only the lo interface
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever

# Comparison
docker inspect c-bridge c-host c-none \
  --format '{{.Name}}: {{.HostConfig.NetworkMode}}'
/c-bridge:bridge
/c-host:host
/c-none:none
# Clean up
docker rm -f c-bridge c-host c-none
```

---

### Exercise 3 — Create and use a custom bridge network (topic 4.4)

**Goal:** create a user-defined bridge network and verify automatic DNS resolution
between containers (a feature the default bridge network does NOT have).

```bash
# Create a custom bridge network
docker network create --driver bridge lab2-bridge

# Start two containers on the same network
docker run -d --name web --network lab2-bridge nginx:alpine
docker run -d --name client --network lab2-bridge alpine sleep 3600

# Verify DNS resolution by name (only available on user-defined networks)
docker exec client ping -c 3 web
# Expected output: PING web (172.x.x.x): successful reply

# Try the same on the default bridge network (should fail by name)
docker run -d --name web-default nginx:alpine
docker run --rm --network bridge alpine ping -c 2 web-default
# Expected output: ping: bad address 'web-default' — no DNS on the default bridge
ping: bad address 'web-default'

# Connect a container to multiple networks
docker network connect bridge web
docker inspect web --format '{{range $k,$v := .NetworkSettings.Networks}}{{$k}}: {{$v.IPAddress}}{{"\n"}}{{end}}'
# Expected output: two entries (lab2-bridge and bridge) with different IPs
bridge: 172.17.0.3
lab2-bridge: 172.18.0.2

# Disconnect and clean up
docker network disconnect bridge web
docker rm -f web client web-default
```

---

### Exercise 4 — Publish ports (topics 4.5 and 4.6)

**Goal:** expose container services externally and locate the accessible IP/port.

```bash
# Publish a specific host:container port
docker run -d --name pub-test -p 8080:80 nginx:alpine

# Verify the published port
docker port pub-test
# Expected output: 80/tcp -> 0.0.0.0:8080
80/tcp -> 0.0.0.0:8080
80/tcp -> [::]:8080

# Identify the accessible IP and port from outside
docker inspect pub-test --format \
  '{{.NetworkSettings.Networks.bridge.IPAddress}} | Host port: {{(index .NetworkSettings.Ports "80/tcp" 0).HostPort}}'
172.17.0.2 | Host port: 8080



# Verify external access
curl -s -o /dev/null -w "%{http_code}" http://<MANAGER_IP>:8080
# Expected output: 200
200
# Publish on all interfaces vs a specific interface
docker run -d --name pub-specific -p <MANAGER_IP>:8081:80 nginx:alpine
docker port pub-specific
# Expected output: 80/tcp -> <MANAGER_IP>:8081
80/tcp -> <MANAGER_IP>:8081
# View the nftables rules Docker generated
sudo nft list ruleset | grep -E "8080|8081"
    iifname != "docker0" tcp dport 8080 counter packets 4 bytes 240 dnat to 172.17.0.2:80
    iifname != "docker0" ip daddr <MANAGER_IP> tcp dport 8081 counter packets 0 bytes 0 dnat to 172.17.0.3:80

# Clean up
docker rm -f pub-test pub-specific
```

---

### Exercise 5 — Host vs ingress publishing modes (topic 4.7)

**Goal:** compare Swarm publishing modes: host mode (direct to the node)
vs ingress mode (load balancing via the routing mesh).

```bash
# Prerequisite: Swarm active
docker info | grep "Swarm:"

# --- Ingress mode (default in Swarm) ---
# The routing mesh forwards traffic to any replica from any node
docker service create \
  --name svc-ingress \
  --publish published=8090,target=80 \
  --replicas 2 \
  nginx:alpine

# Verify: endpoint mode is vip + routing mesh
docker service inspect svc-ingress --format '{{range .Endpoint.Ports}}Mode: {{.PublishMode}} Port: {{.PublishedPort}}{{end}}'
# Expected output: Mode: ingress Port: 8090
Mode:ingressPort: 8090
curl -s -o /dev/null -w "%{http_code}" http://<MANAGER_IP>:8090
# Expected output: 200
200

# --- Host mode (direct bind to the node running the task) ---
docker service create \
  --name svc-host \
  --publish "published=8091,target=80,mode=host" \
  --replicas 1 \
  nginx:alpine

docker service inspect svc-host --format '{{range .Endpoint.Ports}}Mode: {{.PublishMode}} Port: {{.PublishedPort}}{{end}}'
# Expected output: Mode: host Port: 8091
Mode: host Port: 8091
# Key difference: host mode does NOT use the routing mesh — the port only responds
# on the node running the replica; ingress responds on any node in the swarm

# Clean up
docker service rm svc-ingress svc-host
```

---

### Exercise 6 — External DNS (topic 4.8)

**Goal:** configure Docker to use an external DNS server instead of the
default resolver.

```bash
# View the daemon's current DNS configuration
docker info | grep -A 3 "DNS"

# Method 1: per-container DNS at runtime
docker run --rm --dns 8.8.8.8 --dns 1.1.1.1 alpine cat /etc/resolv.conf
# Expected output: nameserver 8.8.8.8, nameserver 1.1.1.1
# Generated by Docker Engine.
# This file can be edited; Docker Engine will not make further changes once it
# has been modified.

nameserver 8.8.8.8
nameserver 1.1.1.1

# Method 2: global DNS in /etc/docker/daemon.json
# (illustration only — requires restarting the daemon)
cat << 'EOF' | sudo tee -a /etc/docker/daemon.json
# Configuration to add to /etc/docker/daemon.json:
{
  "dns": ["8.8.8.8", "1.1.1.1"],
  "dns-search": ["lab.local"]
}
EOF

# Verify the container resolves external names
docker run --rm alpine nslookup docker.com
# Expected output: correct resolution via 8.8.8.8 since DNS is now configured globally in daemon.json
docker run --rm --dns 8.8.8.8 alpine nslookup docker.com
# Forces 8.8.8.8 as the DNS for that specific container

# View internal DNS resolution between containers (Docker's embedded DNS)
docker network create lab2-dns-test
docker run -d --name dns-server --network lab2-dns-test nginx:alpine
docker run --rm --network lab2-dns-test alpine ping -c 3 dns-test
# We can see it resolves, but we don't know via what
docker run --rm --network lab2-dns-test alpine nslookup dns-server 127.0.0.11
Server:		127.0.0.11
Address:	127.0.0.11:53

Non-authoritative answer:

Non-authoritative answer:
Name:	dns-server
Address: 172.20.0.2
# Expected output: name resolved by Docker's embedded DNS (127.0.0.11)

# Clean up
docker rm -f dns-server
docker network rm lab2-dns-test
```

---

### Exercise 7 — Overlay network (topic 4.10)

**Goal:** deploy a Swarm service on an overlay network and verify connectivity
between tasks on the VXLAN-encapsulated network.

```bash
# Prerequisite: Swarm active
docker node ls

# Create overlay network (requires Swarm)
docker network create \
  --driver overlay \
  --subnet 10.20.0.0/24 \
  lab2-overlay

# Verify it's overlay and has swarm scope
docker network inspect lab2-overlay --format \
  'Driver: {{.Driver}} | Scope: {{.Scope}} | Subnet: {{(index .IPAM.Config 0).Subnet}}'
# Expected output: Driver: overlay | Scope: swarm | Subnet: 10.20.0.0/24
Driver: overlay | Scope: swarm | Subnet: 10.20.0.0/24

# Deploy service on the overlay
docker service create \
  --name overlay-web \
  --network lab2-overlay \
  --replicas 2 \
  --publish published=8095,target=80 \
  nginx:alpine

# Verify tasks and nodes
docker service ps overlay-web
ID             NAME            IMAGE          NODE          DESIRED STATE   CURRENT STATE           ERROR     PORTS
l5aqbilq6b25   overlay-web.1   nginx:alpine   docker-labs   Running         Running 2 minutes ago             
tqistteu38b3   overlay-web.2   nginx:alpine   docker-labs   Running         Running 2 minutes ago
# Verify the tasks have IPs within the overlay range
docker service inspect overlay-web --format \
  '{{range .Endpoint.VirtualIPs}}VIP: {{.Addr}}{{end}}'
VIP: 10.0.0.3/24VIP: 10.20.0.2/24

# Access the service
curl -s -o /dev/null -w "%{http_code}" http://<MANAGER_IP>:8095
# Expected output: 200
200
# Clean up
docker service rm overlay-web
docker network rm lab2-overlay
```

---

### Exercise 8 — Connectivity troubleshooting (topic 4.11)

**Goal:** diagnose network issues using engine logs and inspection
tools.

```bash
# Scenario 1: container without connectivity due to wrong network
docker network create lab2-net-a
docker network create lab2-net-b
docker run -d --name svc-a --network lab2-net-a nginx:alpine
docker run -d --name svc-b --network lab2-net-b alpine sleep 3600

# Try pinging between containers on different networks (should fail)
docker exec svc-b ping -c 2 svc-a 2>&1 || echo "Expected failure: isolated networks"
ping: bad address 'svc-a'
Expected failure: isolated networks

# Diagnosis: check which network each container belongs to
docker inspect svc-a --format '{{range $k,$v := .NetworkSettings.Networks}}{{$k}}{{end}}'
lab2-net-a
docker inspect svc-b --format '{{range $k,$v := .NetworkSettings.Networks}}{{$k}}{{end}}'
lab2-net-b
# Fix: connect svc-b to the same network as svc-a
docker network connect lab2-net-a svc-b
docker exec svc-b ping -c 3 svc-a
# Expected output: successful ping after connecting to the same network
PING svc-a (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.060 ms
64 bytes from 172.18.0.2: seq=1 ttl=64 time=0.057 ms
64 bytes from 172.18.0.2: seq=2 ttl=64 time=0.063 ms

--- svc-a ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.057/0.060/0.063 ms

# Scenario 2: review Docker daemon logs for network errors
sudo journalctl -u docker.service --since "10 minutes ago" | grep -E "network|error|failed" | tail -20
# or
sudo journalctl -u docker --since "10 minutes ago" | grep -i"network\|error\|failed" | tail -20

# Scenario 3: inspect network events in real time
docker events --filter type=network &
EVENTS_PID=$!
docker network create lab2-event-test
docker network rm lab2-event-test
sleep 2
kill $EVENTS_PID 2>/dev/null
# Expected output: create and destroy events for the network
2026-04-09T13:55:16.581897299Z network create c223886b7e0bb2cb0c870874417bf1dbb3f246abf0e00c5af54ac90d68131914 (name=lab2-event-test, type=bridge)
2026-04-09T13:55:31.420960007Z network destroy c223886b7e0bb2cb0c870874417bf1dbb3f246abf0e00c5af54ac90d68131914 (name=lab2-event-test, type=bridge)

# Clean up
docker rm -f svc-a svc-b
docker network rm lab2-net-a lab2-net-b 2>/dev/null || true
```

---

### Conceptual topics (no executable exercise)

**4.3 — Traffic types between Docker Engine, Registry, and UCP**

Docker manages three types of traffic in a cluster:
- **Management plane:** communication between Swarm nodes (port 2377/tcp for Raft)
- **Data plane:** container network traffic (overlay VXLAN 4789/udp, gossip 7946/tcp+udp)
- **Registry traffic:** image pull/push (HTTPS 443/tcp to the registry)

In environments with UCP (Universal Control Plane, Docker EE), UCP control traffic
adds further ports. Separating these network planes is a security best practice.

**4.9 — L7 HTTP/HTTPS load balancing with Docker EE**

Docker Enterprise Edition includes UCP with an Interlock proxy (based on NGINX or HAProxy)
that enables L7 routing by hostname and path. Not available in Docker CE.
In Docker CE, load balancing is L4 (VIP + Swarm routing mesh).

**4.12 — Routing traffic to Kubernetes pods (ClusterIP / NodePort)**

In Kubernetes, the equivalent of Swarm's overlay is the CNI (Container Network Interface).
ClusterIP exposes the service only within the cluster. NodePort opens a port on each node.
Docker Desktop includes Kubernetes integrated for local testing.

**4.13 — Kubernetes Container Network Model**

Kubernetes requires that all pods communicate with each other without NAT. Each pod has
its own IP. The CNI (Flannel, Calico, Cilium) implements this model. Conceptually
equivalent to Docker's CNM, but without libnetwork drivers.

---

## Common issues

| `cannot share the host's network namespace when user namespaces are enabled` | `userns-remap: default` active in daemon.json — incompatible with `--network host` | Add `--userns=host` to the specific container's command; never disable userns-remap globally |
| `iptables -t nat -L DOCKER` returns empty even though ports are published | Docker 29.x on Debian 12 uses nftables as its backend. `iptables` is an alias for `iptables-nft`, and DNAT rules are written directly into nftables. | Use `sudo nft list ruleset` to inspect the NAT rules. `0.0.0.0:port` rules don't include a destination IP filter; `specific_IP:port` rules add `ip daddr <IP>` to the rule. |

---

## DCA/SRE Lessons

### Topics covered with hands-on exercises
- **4.1** Container Network Model (CNM) and IPAM drivers
- **4.2** Built-in drivers: bridge, host, none — use cases
- **4.4** Create and manage user-defined bridge networks + automatic DNS
- **4.5** Publish ports (`-p` and `--publish` flags)
- **4.6** Identify a container's accessible IP and port
- **4.7** Publishing modes: ingress (routing mesh) vs host (direct to node)
- **4.8** Configure external DNS per container and at the daemon level
- **4.10** Deploy services on an overlay network (Swarm)
- **4.11** Connectivity troubleshooting with logs and inspection

### Conceptual topics (no environment available)
- **4.3** Traffic types (management, data, registry)
- **4.9** L7 HTTP/HTTPS load balancing (Docker EE / UCP)
- **4.12** Routing to Kubernetes pods (ClusterIP / NodePort)
- **4.13** Kubernetes Container Network Model

### Key exam takeaways
- The **default** bridge network does NOT have DNS between containers. **User-defined** networks do.
- `--network host` removes network isolation — the container uses the host's stack.
- In Swarm, **ingress** uses the routing mesh (any node responds); **host** mode binds the port only to the node running the replica.
- Docker's embedded DNS listens on `127.0.0.11` inside every container.
- Every container has its own network namespace with a veth pair connected to the bridge.
- `--network host` is incompatible with `userns-remap` active on the daemon. The correct fix is `--userns=host` per container, not disabling userns-remap globally — doing so would remove user namespace isolation for every container on the host.

Mapped to DCA domain: **Domain 4 — Networking (15% of the exam)**
