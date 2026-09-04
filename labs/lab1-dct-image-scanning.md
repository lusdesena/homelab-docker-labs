# Lab 1 — Docker Content Trust & Image Scanning — Execution Log

## Objective

Implement an image security pipeline: build an image, sign it with
Docker Content Trust (DCT), and scan it with Trivy for CVEs. Verify that DCT
blocks pulling unsigned images when enabled.

DCA competency covered: 5.2 Describe the process of signing an image /
5.9 Image passes a security scan.

---

## Environment

| Element | Value |
| --- | --- |
| Node | Docker Labs VM — <MANAGER_IP> |
| OS | Debian 12 |
| Docker Engine | >= 24.x (verify with `docker version`) |
| Access | `ssh -J <admin_user>@vps -p 2222 <admin_user>@localhost` |
| User | <admin_user> (sudo available) |
| External account | Active Docker Hub account (for push + DCT) |

Prerequisites:

- Docker Engine installed and the daemon active (`systemctl is-active docker`)
- The <admin_user> user belongs to the docker group (`groups | grep docker`)
- Docker Hub account with push access (`docker login` functional)

---

## Setup

```bash
docker login
# Enter username and password/token when prompted

# 5. Create working directory
mkdir -p ~/lab1-dct && cd ~/lab1-dct
```

---

## Procedure

### Exercise 1 — Build a test image

**Objective:** have an image of our own to work DCT and scanning on.

```bash
# 1.1 — Create a minimal Dockerfile
cat > Dockerfile << 'EOF'
FROM alpine:3.19
RUN apk add --no-cache curl
LABEL maintainer="<admin_user>@dca-lab"
CMD ["sh"]
EOF

# 1.2 — Build
export DOCKERHUB_USER=<your-dockerhub-username>
docker build -t ${DOCKERHUB_USER}/dca-lab1:1.0 .

# 1.3 — Verify the built image
docker images ${DOCKERHUB_USER}/dca-lab1
IMAGE                   ID             DISK USAGE   CONTENT SIZE   EXTRA
<dockerhub_user>/dca-lab1:1.0   c1a5921ee252         12MB             0B        
docker inspect ${DOCKERHUB_USER}/dca-lab1:1.0 | grep -E "Id|Size|Created"
        "Id": "sha256:c1a5921ee2529512d7ad28117a1912add1cc5b336ff088677b890b585f7e59f5",
        "Created": "2026-04-04T10:32:59.670989655Z",
        "Size": 12043702,
docker push ${DOCKERHUB_USER}/dca-lab1:1.0
```

---

### Exercise 5 — Trivy: scan the image for CVEs

**Objective:** analyze vulnerabilities in the built image.

```bash
# 5.1 — Full image scan
trivy image ${DOCKERHUB_USER}/dca-lab1:1.0
# Shows: package table with CVEs by severity (CRITICAL/HIGH/MEDIUM/LOW) — output is longer
<dockerhub_user>/dca-lab1:1.0 (alpine 3.19.9)

Total: 6 (UNKNOWN: 0, LOW: 3, MEDIUM: 3, HIGH: 0, CRITICAL: 0)
───────────────────────────────────────────────────┘

Report Summary

┌───────────────────────────────────────┬────────┬─────────────────┬─────────┐
│                Target                 │  Type  │ Vulnerabilities │ Secrets │
├───────────────────────────────────────┼────────┼─────────────────┼─────────┤
│ <dockerhub_user>/dca-lab1:1.0 (alpine 3.19.9) │ alpine │        6        │    -    │
└───────────────────────────────────────┴────────┴─────────────────┴─────────┘
Legend:
- '-': Not scanned
- '0': Clean (no security findings detected)


<dockerhub_user>/dca-lab1:1.0 (alpine 3.19.9)

Total: 6 (UNKNOWN: 0, LOW: 3, MEDIUM: 3, HIGH: 0, CRITICAL: 0)

┌───────────────┬────────────────┬──────────┬────────┬───────────────────┬───────────────┬──────────────────────────────────────────────────────────────┐
│    Library    │ Vulnerability  │ Severity │ Status │ Installed Version │ Fixed Version │                            Title                             │
├───────────────┼────────────────┼──────────┼────────┼───────────────────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ busybox       │ CVE-2024-58251 │ MEDIUM   │ fixed  │ 1.36.1-r20        │ 1.36.1-r21    │ In netstat in BusyBox through 1.37.0, local users can launch │
│               │                │          │        │                   │               │ of networ...                                                 │
│               │                │          │        │                   │               │ https://avd.aquasec.com/nvd/cve-2024-58251                   │
│               ├────────────────┼──────────┤        │                   │               ├──────────────────────────────────────────────────────────────┤
│               │ CVE-2025-46394 │ LOW      │        │                   │               │ In tar in BusyBox through 1.37.0, a TAR archive can have     │
│               │                │          │        │                   │               │ filenames...                                                 │
│               │                │          │        │                   │               │ https://avd.aquasec.com/nvd/cve-2025-46394                   │
├───────────────┼────────────────┼──────────┤        │                   │               ├──────────────────────────────────────────────────────────────┤
│ busybox-binsh │ CVE-2024-58251 │ MEDIUM   │        │                   │               │ In netstat in BusyBox through 1.37.0, local users can launch │
│               │                │          │        │                   │               │ of networ...                                                 │
│               │                │          │        │                   │               │ https://avd.aquasec.com/nvd/cve-2024-58251                   │
│               ├────────────────┼──────────┤        │                   │               ├──────────────────────────────────────────────────────────────┤
│               │ CVE-2025-46394 │ LOW      │        │                   │               │ In tar in BusyBox through 1.37.0, a TAR archive can have     │
│               │                │          │        │                   │               │ filenames...                                                 │
│               │                │          │        │                   │               │ https://avd.aquasec.com/nvd/cve-2025-46394                   │
├───────────────┼────────────────┼──────────┤        │                   │               ├──────────────────────────────────────────────────────────────┤
│ ssl_client    │ CVE-2024-58251 │ MEDIUM   │        │                   │               │ In netstat in BusyBox through 1.37.0, local users can launch │
│               │                │          │        │                   │               │ of networ...                                                 │
│               │                │          │        │                   │               │ https://avd.aquasec.com/nvd/cve-2024-58251                   │
│               ├────────────────┼──────────┤        │                   │               ├──────────────────────────────────────────────────────────────┤
│               │ CVE-2025-46394 │ LOW      │        │                   │               │ In tar in BusyBox through 1.37.0, a TAR archive can have     │
│               │                │          │        │                   │               │ filenames...                                                 │
│               │                │          │        │                   │               │ https://avd.aquasec.com/nvd/cve-2025-46394                   │
└───────────────┴────────────────┴──────────┴────────┴───────────────────┴───────────────┴──────────────────────────────────────────────────────────────┘

# 5.2 — Filter for HIGH and CRITICAL only
trivy image --severity HIGH,CRITICAL ${DOCKERHUB_USER}/dca-lab1:1.0
Report Summary

┌───────────────────────────────────────┬────────┬─────────────────┬─────────┐
│                Target                 │  Type  │ Vulnerabilities │ Secrets │
├───────────────────────────────────────┼────────┼─────────────────┼─────────┤
│ <dockerhub_user>/dca-lab1:1.0 (alpine 3.19.9) │ alpine │        0        │    -    │
└───────────────────────────────────────┴────────┴─────────────────┴─────────┘
Legend:
- '-': Not scanned
- '0': Clean (no security findings detected)
# 5.3 — Scan the base image to compare the attack surface
trivy image --severity HIGH,CRITICAL alpine:3.19
# Compare CVE count: dca-lab1 vs base alpine
Report Summary

┌─────────────────────────────┬────────┬─────────────────┬─────────┐
│           Target            │  Type  │ Vulnerabilities │ Secrets │
├─────────────────────────────┼────────┼─────────────────┼─────────┤
│ alpine:3.19 (alpine 3.19.9) │ alpine │        0        │    -    │
└─────────────────────────────┴────────┴─────────────────┴─────────┘
# 5.4 — JSON output (for CI/CD integrations)
trivy image --format json -o ~/lab1-dct/trivy-results.json ${DOCKERHUB_USER}/dca-lab1:1.0
cat ~/lab1-dct/trivy-results.json | python3 -m json.tool | head -40
 cat ~/lab1-dct/trivy-results.json | python3 -m json.tool | head -40
{
    "SchemaVersion": 2,
    "Trivy": {
        "Version": "0.69.3"
    },
    "ReportID": "019d595e-19b2-7dab-8db4-39e643cfac4e",
    "CreatedAt": "2026-04-04T16:40:32.434897136Z",
    "ArtifactID": "sha256:5737dc22ea5863ade897da5d04a8837ec9cb801a2d259f0195361e19cd3427b4",
    "ArtifactName": "<dockerhub_user>/dca-lab1:1.0",
    "ArtifactType": "container_image",
    "Metadata": {
        "Size": 12622336,
        "OS": {
            "Family": "alpine",
            "Name": "3.19.9",
            "EOSL": true
        },
        "ImageID": "sha256:c1a5921ee2529512d7ad28117a1912add1cc5b336ff088677b890b585f7e59f5",
        "DiffIDs": [
            "sha256:0b44b2151d78267ab6f2c76208c3be18688f49b2b0afd6852a9533f2cce121c5",
            "sha256:6a27761f8a9a1f065b947196eba89669fac26bd37a36f86580a2b6fa8aa02b6e"
        ],
        "RepoTags": [
            "<dockerhub_user>/dca-lab1:1.0",
            "<dockerhub_user>/dca-lab1:unsigned"
        ],
        "RepoDigests": [
            "<dockerhub_user>/dca-lab1@sha256:d43cb2f7d9111f7b925224c99741d1eb945c02bac9bb3c8636fff6314d1d0fdd"
        ],
        "Reference": "<dockerhub_user>/dca-lab1:1.0",
        "ImageConfig": {
            "architecture": "amd64",
            "created": "2026-04-04T10:32:59.670989655Z",
            "history": [
                {
                    "created": "2025-10-08T11:10:40Z",
                    "created_by": "ADD alpine-minirootfs-3.19.9-x86_64.tar.gz / # buildkit",
                    "comment": "buildkit.dockerfile.v0"
                },
                {

# 5.5 — Exit-code mode: useful in pipelines (fails when there are critical CVEs)
trivy image --exit-code 1 --severity CRITICAL ${DOCKERHUB_USER}/dca-lab1:1.0
echo "Exit code: $?"
# 0 = no critical CVEs, 1 = critical CVEs present
Report Summary

┌───────────────────────────────────────┬────────┬─────────────────┬─────────┐
│                Target                 │  Type  │ Vulnerabilities │ Secrets │
├───────────────────────────────────────┼────────┼─────────────────┼─────────┤
│ <dockerhub_user>/dca-lab1:1.0 (alpine 3.19.9) │ alpine │        0        │    -    │
└───────────────────────────────────────┴────────┴─────────────────┴─────────┘
Legend:
- '-': Not scanned
- '0': Clean (no security findings detected)

Exit code: 0

```

---

## Verification

```bash
echo "=== LAB 1 VERIFICATION ==="

echo ""
echo "1. DCT enabled on the client:"
echo "DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST"
# Should be 1

echo ""
echo "2. dca-lab1:1.0 signature:"
# NOTE: docker trust inspect --pretty is not available in Docker 29.x (Notary v1
# removed in Docker 25+). On the DCA exam this is a concept to describe, not execute.
# Working alternative: verify the tag digest on Docker Hub or with:
docker manifest inspect ${DOCKERHUB_USER}/dca-lab1:1.0 | grep -E "digest|mediaType"

echo ""
echo "3. Pull a signed image with DCT=1 (should succeed):"
docker rmi ${DOCKERHUB_USER}/dca-lab1:1.0 2>/dev/null || true
docker pull ${DOCKERHUB_USER}/dca-lab1:1.0&& echo "OK: pull succeeded"

echo ""
echo "4. Pull an unsigned image with DCT=1 (should fail):"
docker pull busybox 2>&1 | head -3 || echo "OK: pull blocked by DCT"

echo ""
echo "5. Trivy scan — HIGH/CRITICAL CVEs:"
trivy image --severity HIGH,CRITICAL --exit-code 0 ${DOCKERHUB_USER}/dca-lab1:1.0
echo "Scan completed"

echo ""
echo "=== END VERIFICATION ==="
```

---

### Incidents

#### INC-01 — Unencrypted credentials warning on docker login

**Symptom**

During `docker login` the following warning appeared:

```bash
WARNING! Your password will be stored unencrypted in /home/<admin_user>/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store
```

**Cause**

By default, Docker stores credentials in `~/.docker/config.json` as base64 of the
`username:password` string. Base64 is not real encryption: any process with access to the
file can decode it directly.

**Solution applied — docker-credential-pass**

Configured `docker-credential-pass` as the credential helper to delegate
storage to `pass`, which encrypts with GPG.

Steps executed:

```bash
# 1. Install pass and gnupg2
sudo apt-get install -y pass gnupg2

# 2. Download the docker-credential-pass binary
# (replace VERSION with the latest one available in the GitHub release)
VERSION=0.8.2
curl -fsSL https://github.com/docker/docker-credential-helpers/releases/download/v${VERSION}/docker-credential-pass-v${VERSION}.linux-amd64 \
  -o /tmp/docker-credential-pass
chmod +x /tmp/docker-credential-pass
sudo mv /tmp/docker-credential-pass /usr/local/bin/docker-credential-pass

# Verify that Docker can find it
docker-credential-pass version

# 3. Generate a GPG key (empty passphrase for the lab environment)
gpg --batch --passphrase '' --quick-gen-key "docker-lab <admin_user@lab>" default default 0

# Get the fingerprint of the newly created key
GPG_KEY_ID=$(gpg --list-keys --with-colons "docker-lab <admin_user@lab>" | awk -F: '/^fpr/{print $10; exit}')

# 4. Initialize pass with the GPG key
pass init "${GPG_KEY_ID}"

# 5. Add credsStore at the root level of ~/.docker/config.json
# If config.json only contained the auths block, the structure becomes:
# {
#   "auths": { ... },
#   "credsStore": "pass"
# }
# Edit manually or with jq:
jq '. + {"credsStore": "pass"}' ~/.docker/config.json > /tmp/config.json && mv /tmp/config.json ~/.docker/config.json

# 6. Migrate credentials to the new store
docker logout
docker login
# This time the warning does not appear — credentials are encrypted via pass/GPG
```

**Verification**

After login, `~/.docker/config.json` no longer contains the `auth` field in base64
plaintext. Instead:

```json
{
  "auths": {
    "https://index.docker.io/v1/": {}
  },
  "credsStore": "pass"
}
```

Credentials are retrieved at runtime by calling
`docker-credential-pass get`, which decrypts them with GPG.

**Key concepts**

- `config.json` stores the `auth` field as base64 of `username:password` — this is not
  real encryption; any user with access to the file can decode it.
- `credsStore` indicates the suffix of the binary Docker will invoke:
  `docker-credential-<value>`. With `"credsStore": "pass"` Docker calls
  `docker-credential-pass`.
- `docker-credential-pass` delegates to `pass`, which encrypts the secret with GPG and
  stores it in `~/.password-store/`.
- `gpg-agent` caches the GPG key passphrase during the session, avoiding a passphrase
  prompt on every operation.
- In CI/CD pipelines: use `echo $PASS | docker login --username $USER --password-stdin`
  instead of storing credentials in files on the runner.

**DCA topics covered**

| Topic | Domain | Description |
|---|---|---|
| 5.4 — Registry authentication / credentials management | Security (15%) | Secure management of Docker Hub credentials with credential helpers |

---

#### INC-02 — docker build fails with GPG credential helper + BuildKit

**Symptom**

Running `docker build -t ${DOCKERHUB_USER}/dca-lab1:1.0 .` with BuildKit enabled
(the default behavior since Docker 23.x), the build fails with:

```bash
ERROR: failed to build: failed to solve: error getting credentials - err: exit status 1, out: `exit status 2: gpg: decryption failed: No secret key`
```

The error occurred even though `docker-credential-pass list` returned the credentials
correctly in the same terminal session.

**Cause**

BuildKit runs its daemon (`buildkitd`) as a process separate from the user's Docker
process. Since it launches in a different process context, it has no access to the
user session's GPG agent (`gpg-agent`). When BuildKit tries to resolve credentials to
pull the base image (`alpine:3.19`), it calls `docker-credential-pass`, which in turn
tries to decrypt the GPG secret — but without access to the GPG agent, decryption fails
with `No secret key`.

The `pass` credential store worked correctly for other Docker commands because those
commands run in the same process that has access to the session's GPG agent.

**Solution applied**

Configure GPG to allow `pinentry` in loopback mode, so BuildKit can access the GPG
agent even when running as a separate process:

```bash
# Enable loopback pinentry in the GPG agent
echo "allow-loopback-pinentry" >> ~/.gnupg/gpg-agent.conf

# Tell GPG to use loopback mode by default
echo "pinentry-mode loopback" >> ~/.gnupg/gpg.conf

# Restart the GPG agent so the changes take effect
gpgconf --kill gpg-agent

# Verify: the build with BuildKit now works
docker build -t ${DOCKERHUB_USER}/dca-lab1:1.0 .
```

**Verification**

```bash
# Confirm BuildKit is active (should appear in the build output)
docker build -t ${DOCKERHUB_USER}/dca-lab1:1.0 . 2>&1 | head -5
# Lines like "#1 [internal] load build definition" confirm BuildKit is active

# Confirm credentials resolve correctly
docker-credential-pass list
# Should return the Docker Hub credentials without errors
```

**Temporary workaround (not recommended for production)**

```bash
# Disable BuildKit so the build uses the classic builder
DOCKER_BUILDKIT=0 docker build -t ${DOCKERHUB_USER}/dca-lab1:1.0 .
```

Not recommended: the classic builder has been deprecated since Docker 23.x and does not
have feature parity with BuildKit. Useful only for diagnostics.

**Key concepts**

- BuildKit runs as a separate daemon (`buildkitd`) — it does not inherit the user's
  session environment, including access to `gpg-agent`.
- `pinentry-mode loopback` allows the GPG agent to respond to decryption requests from
  processes that have no associated terminal (such as BuildKit).
- `allow-loopback-pinentry` in `gpg-agent.conf` is the permission on the agent side;
  `pinentry-mode loopback` in `gpg.conf` is the instruction on the GPG client side.
- This issue does not appear with plaintext credentials in `config.json` (without
  `credsStore`) because no GPG decryption is involved.

**DCA topics covered**

| Topic | Domain | Description |
|---|---|---|
| 5.4 — Registry authentication / credentials management | Security (15%) | Interaction between encrypted credential helpers and BuildKit |

---

#### INC-03 — `docker push` fails with a GPG batchmode error

**Symptom**

```bash
error getting credentials - err: exit status 1, out: `exit status 2: gpg: Sorry, we are in batchmode - can't get input`
```

**Cause**

Docker uses `pass` as the credential store (`credsStore: pass` in `~/.docker/config.json`).
`pass` encrypts with GPG. When `docker push` calls `docker-credential-pass`, GPG needs
the passphrase to decrypt, but no TTY is available (the push runs in a context without
an interactive terminal), so it enters batchmode and aborts the operation.
The `gpg-agent` did not have the passphrase cached in memory.

**Solution applied**

Pre-cache the passphrase by manually running an interactive GPG operation before the
push. Once `gpg-agent` has the passphrase cached, it serves it to `docker-credential-pass`
without needing a TTY:

```bash
# Any of these commands forces the passphrase to be cached in gpg-agent
echo "test" | gpg --clearsign 2>&1
# Or any equivalent interactive GPG operation

# The push now works because gpg-agent has the passphrase cached
docker push ${DOCKERHUB_USER}/dca-lab1:1.0
```

**Outstanding note**

The `gpg-agent` cache expires after ~10 minutes by default. If the interval between
login (or the last access to `pass`) and the push exceeds that timeout, the error will
return.

Permanent fix: increase the TTL in `~/.gnupg/gpg-agent.conf` and reload the agent:

```bash
# Example: cache the passphrase for 1 hour (3600 seconds)
echo "default-cache-ttl 3600" >> ~/.gnupg/gpg-agent.conf
echo "max-cache-ttl 7200" >> ~/.gnupg/gpg-agent.conf
gpgconf --kill gpg-agent
```

**Key concepts**

- GPG enters `batchmode` when no TTY is available and interactive input (the
  passphrase) is required. In batchmode it aborts instead of hanging while waiting for
  input.
- `gpg-agent` acts as a passphrase cache: once entered interactively, it serves it to
  processes without a TTY for the time configured in `default-cache-ttl`.
- `default-cache-ttl` (seconds since last use) and `max-cache-ttl` (seconds since first
  caching) control the cache duration.
- This error is a direct consequence of using `credsStore: pass` with a passphrase-
  protected GPG key — it does not appear with a passphrase-less GPG key or with
  plaintext credentials.

---

## DCA/SRE Lessons

### Exam topics reinforced

| DCA Topic | Domain | Exercise | Status |
| --- | --- | --- | --- |
| 5.2 — Signing an image (DCT, Notary, TUF) | Security (15%) | Exercises 2, 3, 4, 6 | Conceptual — not executable on Docker 29.x |
| 5.9 — Image passes a security scan | Security (15%) | Exercise 5 | Completed |

> **Exercises 2, 3, 4, and 6 — Not executed (Docker 29.x)**
> Notary v1 was removed in Docker 25+. The `docker trust sign`,
> `docker trust inspect --pretty` subcommands, and the automatic signing mechanism via
> `DOCKER_CONTENT_TRUST=1` on push are no longer available. Attempting to run them on
> Docker 29.3.1 returns: `docker: 'trust' is not a docker command`.
> Topic 5.2 is covered **conceptually** in this lab, which is what the exam
> requires given the current state of the tooling.

### Key concepts for the exam

- `DOCKER_CONTENT_TRUST=1` — variable that enabled signature verification on pull and
  push. **The most frequently tested aspect of topic 5.2** at a conceptual level. On
  Docker 29.x the variable is accepted but has no effect (Notary v1 removed).
- With DCT enabled (versions < Docker 25), pulling an **unsigned image would fail** —
  an important conceptual behavior for the exam.
- `docker trust sign` — signed an already-pushed tag without needing a rebuild.
  **Deprecated in Docker 25+, not available on Docker 29.x.**
- `docker trust inspect --pretty` — audited signatures for a repository.
  **Deprecated in Docker 25+, not available on Docker 29.x.**
- DCT uses **Notary** (a TUF — The Update Framework — implementation) as its backend —
  a frequent conceptual exam question.
- The **root key** must be generated and kept **offline** — if lost, the repository's
  chain of trust is lost.
- `trivy image --exit-code 1 --severity CRITICAL` is the standard pattern for **CI/CD
  gates** — fully functional and executable.
- Signing (DCT) != Scanning (Trivy/Scout) — complementary mechanisms: DCT guarantees
  origin, scanning guarantees the absence of known CVEs.

### Mapping to the DCA domain

- Security (15% of the exam) — topics 5.2, 5.9

**Estimated execution time:** 45-60 minutes
