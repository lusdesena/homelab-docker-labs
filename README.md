# homelab-docker-labs

> 7 hands-on Docker labs — security, image signing, networking, Swarm/k3s orchestration, registries and storage — built on a real 3-node homelab cluster while preparing for the Docker Certified Associate exam.

---

## Overview

Each lab targets one or more official DCA exam domains and was executed on physical infrastructure I administer — not a disposable cloud sandbox. The labs progress from single-host isolation primitives to a 3-node Swarm cluster that is later torn down and rebuilt as a k3s (Kubernetes) cluster on the same VMs, so the networking, storage and security concepts can be compared side by side across both orchestrators.

| Lab | DCA domain | Weight | Topics |
| --- | --- | --- | --- |
| [`lab0-security-fundamentals`](labs/lab0-security-fundamentals.md) | Security | 15% | namespaces, capabilities, cgroups, seccomp, AppArmor |
| [`lab1-dct-image-scanning`](labs/lab1-dct-image-scanning.md) | Security | 15% | Docker Content Trust (conceptual), Trivy CVE scanning |
| [`lab2-docker-networking`](labs/lab2-docker-networking.md) | Networking | 15% | CNM, IPAM, bridge/overlay, embedded DNS, troubleshooting |
| [`lab3-swarm-orchestration`](labs/lab3-swarm-orchestration.md) | Orchestration | 25% | 3-node Swarm, Raft quorum, stacks, routing mesh, k3s Pods/ConfigMaps/Secrets |
| [`lab4-image-registry`](labs/lab4-image-registry.md) | Image & Registry | 20% | image lifecycle, private registry operations |
| [`lab5-storage-installation`](labs/lab5-storage-installation.md) | Install & Config / Storage | 15% + 10% | daemon.json, storage drivers, volumes, NFS, k3s PV/PVC |
| [`lab6-security-ucp-cosign`](labs/lab6-security-ucp-cosign.md) | Security | 15% | UCP/Docker EE (conceptual), image signing with cosign |

```text
Proxmox VE homelab (i5-12600H / 32 GB)
┌───────────────────────────────────────────────────────────┐
│  docker-lab-manager      docker-lab-worker1  docker-lab-worker2
│  <MANAGER_IP>            <WORKER1_IP>        <WORKER2_IP>
│  Swarm manager           Swarm worker         Swarm worker
│  k3s server        ◄──►  k3s agent      ◄──►  k3s agent
└───────────────────────────────────────────────────────────┘
        Same 3 VMs, reprovisioned between Swarm and k3s phases
```

**Stack:** Docker Engine 29.x, Docker Swarm, k3s (Kubernetes), Trivy, cosign, NFS, AppArmor, seccomp, Proxmox VE, Debian 12.

---

## Problem

Passing the DCA exam required hands-on fluency across six weighted domains, not just theory. Video-course exercises use disposable, pre-configured sandboxes that hide most of the failure modes an operator actually hits — expired join tokens, storage drivers that silently refuse to start, deprecated CLI subcommands. I wanted labs that:

- Run on infrastructure I provision and own end to end (Proxmox VMs, not a managed lab platform).
- Produce a real execution log — actual command output and actual failures — instead of a clean tutorial.
- Cover every domain in proportion to its exam weight, so study time tracked exam risk.

---

## Design decisions

### 1. Real infrastructure over managed lab platforms

KodeKloud-style sandboxes are fine for spaced-repetition practice, but they reset on every session and can't surface persistence issues (Swarm Raft state across restarts, NFS mounts surviving a reboot, storage driver choice at install time). I provisioned three Debian 12 VMs on my own Proxmox host instead and kept them alive across the whole lab sequence. **Rejected:** cloud free-tier VMs — same persistence benefit, but adds cost and network latency irrelevant to the exam content.

### 2. Execution log, not tutorial

Each lab's `-doc.md` captures the commands actually run and their actual output, including the ones that failed on the first try. A tutorial format would have been faster to write but would hide the debugging reasoning that is the real evidence of competence. **Rejected:** cleaning up the log into a "happy path" writeup — it reads better but proves less.

### 3. Swarm and k3s on the same three nodes

Domain 1 (Orchestration, 25% of the exam) expects familiarity with Swarm; CKA prep (next certification) needs the same muscle memory in Kubernetes. Rather than standing up a second cluster, I tore Swarm down and reinstalled the same VMs as a k3s cluster (`lab3`), so concepts like service replicas vs. Deployments, or overlay networks vs. CNI, could be compared on identical hardware. **Rejected:** running both concurrently on separate node pools — would have doubled the RAM budget for no additional exam-relevant signal.

### 4. Document what's deprecated, not just what works

Docker Content Trust (`docker trust sign`, `DOCKER_CONTENT_TRUST=1`) was removed with Notary v1 in Docker 25+; the exam still asks about it conceptually. `lab1` and `lab6` document this explicitly — what the topic requires you to know, why it can't be executed on 29.x, and cosign as the practical replacement that pipelines actually use today. **Rejected:** skipping the non-executable topics — they're still graded on the exam.

---

## Implementation

Each lab follows the same shape: environment/prerequisites → setup → numbered exercises mapped to a DCA topic number → an incidents table (real errors hit, root cause, fix) → a "Lessons for DCA/SRE" section mapping back to the exam domain. See [`labs/`](labs/) for the full execution logs; the walkthroughs below are the highlights, not the complete transcript.

---

## Result

- DCA exam passed 30 May 2026.
- 3-node Swarm cluster built, operated (stacks, scaling, overlay networks, placement constraints) and cleanly decommissioned.
- Same 3 nodes reprovisioned as a k3s cluster running Pods, Deployments, ConfigMaps and Secrets — direct prep for CKA (next in the certification sequence).
- 7 labs, ~250 KB of execution logs with real command output, covering every weighted DCA domain.

---

## Lessons learned

**Storage driver choice is a one-way door.** `overlay2` requires a filesystem with `d_type` support; discovering this after images and volumes already exist means reformatting, not reconfiguring. Now I check `xfs -n ftype=1` (or ext4) before provisioning any Docker host, not after.

**Deprecated CLI ≠ deprecated exam topic.** `docker trust` disappeared in Docker 25+, but DCA topic 5.2 (image signing) is still on the exam. Documenting *why* a command no longer works, and what replaced it operationally (cosign, OCI-native signatures), turned out to be more useful study material than the command itself would have been.

**Kubernetes storage defaults don't do what Swarm volumes do.** k3s's `local-path` provisioner is `RWO` only — a shared, multi-pod volume needs NFS or a CSI driver, the same lesson Swarm teaches with named volumes being node-local. Two orchestrators, same underlying constraint, learned once from having to fix a `Pending` PVC.

**An odd number of managers is not a style preference.** Raft quorum tolerance (3 managers survive 1 failure, 5 survive 2) only becomes intuitive after actually killing a manager and watching `docker node ls` — reading the rule is not the same as depending on it working.

---

## Stack & concepts applied

**Orchestration:** Docker Swarm (services, stacks, routing mesh, placement constraints), k3s (Pods, Deployments, ConfigMaps, Secrets).
**Security:** namespaces, capabilities, cgroups, seccomp, AppArmor, Docker Content Trust (conceptual), cosign image signing, UCP authorization model (conceptual).
**Networking:** CNM, IPAM, bridge and overlay drivers, embedded DNS, service discovery.
**Storage:** storage drivers (overlay2), named volumes, NFS, Kubernetes PV/PVC and CSI.
**Operations:** daemon.json configuration, log drivers, Swarm backup/restore, `docker system df` capacity auditing.

---

## Project status

| Milestone | Status |
| --- | --- |
| DCA exam passed | ✅ 2026-05-30 |
| 7 labs executed and documented | ✅ |
| Portfolio repo published | ✅ |

---

## Additional documentation

- [`labs/`](labs/) — full execution logs for all 7 labs, each mapped to its DCA domain and topic numbers.
