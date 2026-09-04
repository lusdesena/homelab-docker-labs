# Lab 6 — UCP Concepts & Image Signing with cosign — Execution Log

## Objective

Cover the 7 remaining topics in Domain 5 (Security):

- **Conceptual** (no Docker EE environment available): 5.6, 5.7, 5.8, 5.11, 5.12, 5.13 — UCP RBAC, identity roles, external certificates, LDAP/AD, client bundles
- **Practical** (executable on Docker 29.x): 5.10 — image signing and verification with cosign

Topics covered:

- **Domain 5**: 5.6, 5.7, 5.8, 5.10, 5.11, 5.12, 5.13

---

## Environment

| Node | IP | Role | Docker |
| --- | --- | --- | --- |
| docker-labs | <MANAGER_IP> | Swarm manager / k3s server | 29.x |
| swarm-worker1 | <WORKER1_IP> | Swarm worker | 29.x |
| swarm-worker2 | <WORKER2_IP> | Swarm worker | 29.x |

Prerequisites:

- Local private registry running at `localhost:5000` (set up in Lab 4)
- If it's not running: `docker run -d -p 5000:5000 --name registry registry:2`
- Working directory: `~/lab6`

---

## Setup

```bash
# Check the registry
curl -s http://localhost:5000/v2/_catalog
# Expected output: {"repositories":[...]}

# If it's not running:
docker run -d -p 5000:5000 --name registry \
  -v registry-data:/var/lib/registry \
  registry:2

# Working directory
mkdir -p ~/lab6 && cd ~/lab6
```

---

## Block A — Docker EE / UCP Security [CONCEPTUAL]

> These exercises don't require execution. The goal is to nail down the concepts for
> the exam. Each exercise includes a quick-reference table.
> Canonical source: `docs/docker-study-guide_v1-5-jan-2025.pdf`

---

### Exercise 1 — Identity roles in UCP (topic 5.6)

UCP (Universal Control Plane) manages identities through **subjects**, **roles**, and **grants**.

#### Subjects (who)

| Type | Description |
| --- | --- |
| User | Individual account (local or LDAP) |
| Team | Group of users within an organization |
| Service account | Identity for automated services (CI/CD) |
| Organization | Container for teams |

#### Predefined UCP roles (from least to most privilege)

| Role | Can do |
| --- | --- |
| `None` | No access — explicit block |
| `View Only` | View resources, cannot modify |
| `Restricted Control` | Deploy containers, no `--privileged`, no host access |
| `Scheduler` | Schedule tasks on nodes (node role) |
| `Full Control` | Full control over the resources in the assigned collection |

#### Custom roles

```text
# Composition of a custom role in UCP:
# Name → List of allowed operations (API permissions)
# Example: "deploy-only"
#   - container_create
#   - service_create
#   - stack_deploy
# Excluded: node_update, secret_create, config_create
```

#### Exam reference 5.6

| Concept | Key fact |
| --- | --- |
| Most restrictive role | `None` — explicitly denies |
| Role for unprivileged CI/CD | `Restricted Control` |
| Role for full management of a namespace | `Full Control` over the collection |
| `Scheduler` is assigned to | Nodes, not directly to users |
| Custom roles | Defined with individual API permissions |

---

### Exercise 2 — UCP managers vs workers (topic 5.7)

#### UCP architecture

```text
┌─────────────────────────────────────────┐
│              UCP Managers               │
│  (Control Plane — minimum 3 for HA)     │
│                                         │
│  ┌──────────┐  ┌──────────┐  ┌───────┐ │
│  │  UCP     │  │  UCP     │  │  UCP  │ │
│  │ Manager1 │  │ Manager2 │  │ Mgr3  │ │
│  │ (Leader) │  │(Follower)│  │(Foll.)│ │
│  └──────────┘  └──────────┘  └───────┘ │
│       Raft consensus among managers     │
└─────────────────────────────────────────┘
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
┌──────────────┐        ┌──────────────┐
│  UCP Worker  │        │  UCP Worker  │
│ (Data Plane) │        │ (Data Plane) │
│  Runs        │        │  Runs        │
│  containers  │        │  containers  │
└──────────────┘        └──────────────┘
```

#### Manager vs Worker comparison

| Feature | Manager | Worker |
| --- | --- | --- |
| Control plane | Yes | No |
| Raft consensus | Participates | No |
| Runs workloads | Possible but not recommended | Yes |
| UCP web UI | Accessible | No |
| UCP API | Accessible | No |
| Fault tolerance | N managers → (N-1)/2 failures | No impact on control plane |
| Recommended hardware | 8 CPU / 16 GB RAM / 100 GB SSD | 4 CPU / 4 GB RAM / 25 GB |

#### UCP ports

| Port | Protocol | Use |
| --- | --- | --- |
| 443 | TCP | UCP web UI and API |
| 2376 | TCP | Docker Engine TLS (swarm) |
| 2377 | TCP | Swarm manager communication |
| 4789 | UDP | Overlay network (VXLAN) |
| 7946 | TCP/UDP | Gossip between nodes |
| 12376 | TCP | TLS proxy for Docker Engine |
| 12379-12381 | TCP | Raft among UCP managers |

#### Exam reference 5.7

| Concept | Key fact |
| --- | --- |
| Minimum managers for HA | 3 (tolerates 1 failure) |
| Recommended max managers | 7 (tolerates 3 failures) |
| Workers without quorum | Do not affect the control plane |
| Manager running workloads | Possible but consumes control-plane resources — avoid in prod |
| UCP web UI port | 443 |
| Swarm management port | 2377 |

---

### Exercise 3 — External certificates with UCP and DTR (topic 5.8)

By default, UCP and DTR generate self-signed certificates. In production these are replaced with certificates from a corporate or public CA.

#### Replacing the UCP certificate

```bash
# Using docker/ucp (official image)
docker container run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  docker/ucp \
  install \
  --host-address <MANAGER_IP> \
  --external-ca-cert /path/to/ca.pem \
  --external-cert /path/to/cert.pem \
  --external-key /path/to/key.pem

# Or post-install, via the UCP web UI:
# Admin Settings → Certificates → Upload Server Certificate Bundle
```

#### Replacing the DTR certificate

```bash
# Via DTR API
curl -X POST \
  -u admin:password \
  "https://<DTR_URL>/api/v0/meta/settings" \
  -H "Content-Type: application/json" \
  -d '{
    "tlsCertificate": "<cert-pem-contents>",
    "tlsKey": "<key-pem-contents>",
    "tlsCACert": "<ca-pem-contents>"
  }'
```

#### Certificate requirements

| Field | Requirement |
| --- | --- |
| SAN (Subject Alt Name) | Must include the hostname and all IPs of the UCP cluster |
| Type | X.509 v3 |
| Format | PEM |
| CA | Can be an internal corporate CA or a public one (Let's Encrypt) |
| Renewal | Manual or via automation — UCP does not auto-renew |

#### Exam reference 5.8

| Concept | Key fact |
| --- | --- |
| Default UCP cert | Self-signed — browsers show a warning |
| Critical cert field | SAN must include all cluster IPs/hostnames |
| How to replace in UCP | `docker/ucp install --external-cert` or web UI Admin Settings |
| How to replace in DTR | API POST `/api/v0/meta/settings` |
| Client bundle uses | UCP's CA to validate connections |

---

### Exercise 4 — RBAC with UCP (topic 5.11)

#### UCP authorization model: Grants

A **grant** connects three elements:

```text
Grant = Subject + Role + Collection
          ↓         ↓        ↓
        (who)   (what they  (on what)
                 can do)
```

#### Collections

Collections are hierarchical groupings of Docker resources (nodes, services, volumes, secrets, configs):

```text
/                          ← root collection (admins only)
├── /Shared                ← shared resources
│   └── /System            ← UCP system resources
└── /dev                   ← development team collection
    ├── /dev/frontend
    └── /dev/backend
└── /prod
    └── /prod/api
```

#### RBAC configuration example

```text
Scenario: the "devs" team can deploy to /dev but not to /prod

Grants:
  1. Subject: Team "devs"
     Role: "Full Control"
     Collection: /dev

  2. Subject: Team "devs"
     Role: "View Only"
     Collection: /Shared

  3. Subject: Team "ops"
     Role: "Full Control"
     Collection: /          ← access to everything
```

#### Exam reference 5.11

| Concept | Key fact |
| --- | --- |
| Authorization unit | Grant = Subject + Role + Collection |
| Collection inheritance | A grant on `/dev` applies to `/dev/frontend` and `/dev/backend` |
| UCP admin role | Has access to the root collection `/` |
| No grant | No access — UCP denies by default |
| Native Docker CE RBAC | Doesn't exist — only in Docker EE/UCP |

---

### Exercise 5 — UCP integration with LDAP/AD (topic 5.12)

#### UCP authentication modes

| Mode | Description |
| --- | --- |
| Managed | Local users created in UCP |
| LDAP / Active Directory | UCP delegates authentication to an external directory |

#### LDAP configuration in UCP

```text
Admin Settings → Authentication & Authorization → LDAP

Required parameters:
  LDAP Server URL:     ldap://ad.company.com:389
  Reader DN:           cn=ucp-reader,dc=company,dc=com
  Reader Password:     ****
  Base DN:             dc=company,dc=com
  Username attribute:  sAMAccountName   (AD) / uid (OpenLDAP)
  Full Name attribute: displayName
  
Optional parameters:
  Group Member Attribute:  member
  Sync interval:           24h (periodic group synchronization)
```

#### LDAP authentication flow

```text
User → UCP login → UCP queries LDAP → LDAP validates credentials
                                      → UCP assigns teams based on LDAP groups
                                      → Grant determines final permissions
```

#### Exam reference 5.12

| Concept | Key fact |
| --- | --- |
| Authentication with LDAP | UCP never stores the password — it validates against LDAP |
| Group synchronization | LDAP group → UCP team (automatic at the configured interval) |
| AD user attribute | `sAMAccountName` |
| OpenLDAP user attribute | `uid` |
| Without LDAP | Users are managed locally in UCP |

---

### Exercise 6 — UCP client bundles (topic 5.13)

A client bundle is a package of certificates and scripts that lets the local Docker CLI point directly at a UCP cluster (instead of the local daemon).

#### Client bundle contents

```text
client-bundle.zip
├── ca.pem          ← UCP's CA, used to verify the server
├── cert.pem        ← User certificate
├── key.pem         ← User private key
├── env.sh          ← Environment variables (Linux/Mac)
├── env.cmd         ← Environment variables (Windows CMD)
├── env.ps1         ← Environment variables (PowerShell)
└── docker.env      ← Alternative for docker-compose
```

#### Downloading the client bundle

```bash
# Via UCP web UI:
# User Menu (top right corner) → My Profile → Client Bundles → New Client Bundle

# Via UCP API:
AUTHTOKEN=$(curl -sk -d \
  '{"username":"admin","password":"password"}' \
  https://<UCP_URL>/auth/login | jq -r .auth_token)

curl -sk -H "Authorization: Bearer $AUTHTOKEN" \
  https://<UCP_URL>/api/clientbundle -o bundle.zip

unzip bundle.zip
```

#### Using the client bundle

```bash
# Activate the bundle — redirects the Docker CLI to the UCP cluster
cd bundle/
source env.sh

# Verify it points to UCP
docker info | grep -E "Server Version|Swarm"

# From here on, all docker commands operate against UCP
docker node ls
docker service ls
docker stack deploy -c stack.yml myapp

# Deactivate (return to the local daemon)
unset DOCKER_HOST DOCKER_TLS_VERIFY DOCKER_CERT_PATH
```

#### Exam reference 5.13

| Concept | Key fact |
| --- | --- |
| Purpose | Connect the local CLI to UCP remotely via mTLS |
| How to activate | `source env.sh` (Linux) |
| What `env.sh` configures | `DOCKER_HOST`, `DOCKER_TLS_VERIFY`, `DOCKER_CERT_PATH` |
| Validity | Expires when the user's certificate expires or is revoked |
| Without a bundle | The CLI talks to the local Docker daemon |

---

## Block B — Image Signing with cosign (topic 5.10)

> Notary v1 was removed in Docker 25+. `DOCKER_CONTENT_TRUST=1` has no effect
> on Docker 29.x. The current standard is **cosign** (Sigstore). This block is the
> only one executable in the real environment.

### Exercise 7 — Install cosign

```bash
# Download the official Sigstore binary
curl -O -L https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
chmod +x cosign-linux-amd64
mv cosign-linux-amd64 /usr/local/bin/cosign

# Verify
cosign version
```

---

### Exercise 8 — Generate a key pair

```bash
cd ~/lab6

# Generate a private key (cosign.key) and public key (cosign.pub)
# You'll be prompted for a password — use "lab6pass" or leave it empty for the lab
cosign generate-key-pair

ls -la cosign.*
# cosign.key  ← private key (protect in production)
# cosign.pub  ← public key (distribute for verification)

cat cosign.pub
```

---

### Exercise 9 — Prepare a test image

```bash
# Verify the registry is running
curl -s http://localhost:5000/v2/_catalog
{"repositories":[]}
# Build or use an existing image
docker pull alpine:latest
docker tag alpine:latest localhost:5000/lab6-alpine:v1
docker push localhost:5000/lab6-alpine:v1

# Get the exact digest of the image (needed by cosign)
docker inspect localhost:5000/lab6-alpine:v1 \
  --format '{{index .RepoDigests 0}}'
alpine@sha256:5b10f432ef3da1b8d4c7eb6c487f2f5a8f096bc91145e68878dd4a5019afde11
```

---

### Exercise 10 — Sign an image

```bash
cd ~/lab6

# Sign the image in the private registry (insecure over HTTP)
cosign sign --key cosign.key \
  --allow-insecure-registry \
  localhost:5000/lab6-alpine:v1

# cosign adds the signature as an OCI artifact in the same registry
# Signature tag: localhost:5000/lab6-alpine:<sha256-digest>.sig

# Verify the signature appears in the registry
curl -s http://localhost:5000/v2/lab6-alpine/tags/list
{"name":"lab6-alpine","tags":["v1","sha256-158d0b04afbe634e1b8472e5800599ca04777ed6f911e3bc7aa6adc8ee29ebce"]}
# Should show the image tag plus the signature tag (.sig)
```

---

### Exercise 11 — Verify the signature

```bash
# Verify with the public key
cosign verify --key cosign.pub \
  --allow-insecure-registry \
  localhost:5000/lab6-alpine:v1

[{"critical":{"identity":{"docker-reference":"localhost:5000/lab6-alpine:v1"},"image":{"docker-manifest-digest":"sha256:158d0b04afbe634e1b8472e5800599ca04777ed6f911e3bc7aa6adc8ee29ebce"},"type":"https://sigstore.dev/cosign/sign/v1"},"optional":{}}]

# Expected output: JSON with the verified signature payload
# [{"critical":{"identity":{"docker-reference":"localhost:5000/lab6-alpine"},...}]

# Try verifying an UNSIGNED image — should fail
docker pull nginx:alpine
docker tag nginx:alpine localhost:5000/lab6-nginx:v1
docker push localhost:5000/lab6-nginx:v1

cosign verify --key cosign.pub \
  --allow-insecure-registry \
  localhost:5000/lab6-nginx:v1

Error: no signatures found
error during command execution: no signatures found
# Expected error: no matching signatures found
```

---

### Exercise 12 — Signing with annotations (metadata)

```bash
# Add metadata to the signature (useful in CI/CD pipelines)
cosign sign --key cosign.key \
  --allow-insecure-registry \
  -a "git-sha=$(git rev-parse --short HEAD 2>/dev/null || echo 'lab6')" \ 
  -a "environment=staging" \
  -a "signed-by=<admin_user>" \
  localhost:5000/lab6-alpine:v1
# -a "git-sha... extracts the exact commit to tie the signature to the exact commit, useful for CI/CD pipelines
# Verify and extract the annotations
cosign verify --key cosign.pub \
  --allow-insecure-registry \
  localhost:5000/lab6-alpine:v1 \
  | python3 -m json.tool
  Verification for localhost:5000/lab6-alpine:v1 --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - Existence of the claims in the transparency log was verified offline
  - The signatures were verified against the specified public key
[
    {
        "critical": {
            "identity": {
                "docker-reference": "localhost:5000/lab6-alpine:v1"
            },
            "image": {
                "docker-manifest-digest": "sha256:158d0b04afbe634e1b8472e5800599ca04777ed6f911e3bc7aa6adc8ee29ebce"
            },
            "type": "https://sigstore.dev/cosign/sign/v1"
        },
        "optional": {
            "environment": "staging",
            "git-sha": "lab6",
            "signed-by": "<admin_user>"
        }
    },
    {
        "critical": {
            "identity": {
                "docker-reference": "localhost:5000/lab6-alpine:v1"
            },
            "image": {
                "docker-manifest-digest": "sha256:158d0b04afbe634e1b8472e5800599ca04777ed6f911e3bc7aa6adc8ee29ebce"
            },
            "type": "https://sigstore.dev/cosign/sign/v1"
        },
        "optional": {}
    }
]

```

---

### Exercise 13 — DCT vs cosign comparison [CONCEPTUAL]

| Feature | Docker Content Trust (Notary v1) | cosign (Sigstore) |
| --- | --- | --- |
| Availability | Removed in Docker 25+ | Current standard |
| Environment variable | `DOCKER_CONTENT_TRUST=1` | N/A (explicit CLI) |
| Signature storage | Separate Notary server | In the registry itself (OCI artifact) |
| Infrastructure | Notary server + signer + TUF | None additional — just an OCI registry |
| Verification | Automatic with `docker pull` if DCT is enabled | Explicit `cosign verify` |
| Keyless signing | No | Yes (Fulcio CA + Rekor transparency log) |
| Production use | Deprecated | Recommended (GitHub Actions, k8s admission) |

---

## Final verification

```bash
echo "=== cosign installed ==="
cosign version

echo ""
echo "=== Keys generated ==="
ls -la ~/lab6/cosign.*

echo ""
echo "=== Signed image in registry ==="
curl -s http://localhost:5000/v2/lab6-alpine/tags/list

echo ""
echo "=== Signature verification ==="
cosign verify --key ~/lab6/cosign.pub \
  --allow-insecure-registry \
  localhost:5000/lab6-alpine:v1 | python3 -m json.tool
```

---

## Lab cleanup

```bash
docker rm -f registry 2>/dev/null || true
docker rmi localhost:5000/lab6-alpine:v1 localhost:5000/lab6-nginx:v1 2>/dev/null || true
rm -rf ~/lab6
```

---

## Common issues

| Issue | Cause | Solution |
| --- | --- | --- |
| `cosign: command not found` | Binary not in PATH | Verify `/usr/local/bin/cosign` and permissions |
| `Error: signing localhost:5000/...` | HTTP registry without the insecure flag | Add `--allow-insecure-registry` |
| `no matching signatures found` | Image not signed or wrong key | Verify with the correct public key |
| `Error opening key` | Wrong `cosign.key` password | Re-enter the password used when generating the key pair |
| UCP web UI unreachable | No Docker EE environment | UCP topics are assessed conceptually on the exam |

---

## DCA/SRE lessons

### Domain 5 — Security (15%)

**UCP / Docker EE (conceptual):**

- The UCP authorization model is `Grant = Subject + Role + Collection` — memorize this formula for the exam
- `Restricted Control` is the role for multi-tenant environments where host access can't be granted
- **Client bundles** are the mechanism for using the `docker` CLI against UCP — `source env.sh` → DOCKER_HOST points to UCP
- LDAP doesn't store passwords in UCP — it only validates. Group synchronization maps group → team automatically
- External certificates require a SAN with all cluster IPs/hostnames — this is the most common mistake in UCP installations
- On the exam: "describe" and "compare" questions don't require knowing the exact UCP CLI, just the conceptual model

**cosign / Image signing (practical):**

- Notary v1 + `DOCKER_CONTENT_TRUST=1` were removed in Docker 25+ — they have no effect on Docker 29.x
- cosign stores the signature as an OCI artifact in the same registry — no additional infrastructure required
- `cosign sign --key` (with key) is the most common mode; keyless signing requires OIDC/Fulcio (cloud)
- In CI/CD pipelines: sign on publish (`cosign sign`) and verify before deploying (`cosign verify`)

**DCA mapping:**

- Domain 5: Security — **15% of the exam** (topics 5.6, 5.7, 5.8, 5.10, 5.11, 5.12, 5.13)
