# Lab 0 — Security Fundamentals — Execution Log

## Objective

Practically verify the Docker Engine security mechanisms from Week 1. Demonstrate namespace isolation, capability control, resource limits, and seccomp/AppArmor profiles.

**DCA Topics:** 3.8 (namespaces, cgroups, capabilities) · 5.1 · 5.3 (seccomp, AppArmor) | **File:** `labs/lab0-security-fundamentals.md`

## Exercises

| # | Exercise | Mechanism | Key commands |
| --- | --- | --- | --- |
| 1 | PID, NET, USER isolation | Namespaces | `docker run --rm alpine ps aux` · `ip addr show` |
| 2 | Least privilege | Capabilities | `--cap-drop ALL` · `--cap-add NET_ADMIN` · `--privileged` |
| 3 | CPU and memory limits | cgroups | `--memory 128m` · `--cpus 0.5` · `docker stats` |
| 4 | Syscall filtering | Seccomp | `docker info \ |
| 5 | Resource access control | AppArmor | `aa-status \ |
| 6 | UID remapping | userns-remap | `ps -o user= -p <PID>` with and without remap |

## Commands
### Setup
```bash
sudo aa-status | head -20 #AppArmor Module info (first 20 lines)
apparmor module is loaded.
12 profiles are loaded.
12 profiles are in enforce mode.
   /usr/bin/man
   /usr/lib/NetworkManager/nm-dhcp-client.action
   /usr/lib/NetworkManager/nm-dhcp-helper
   /usr/lib/connman/scripts/dhclient-script
   /{,usr/}sbin/dhclient
   docker-default
   lsb_release
   man_filter
   man_groff
   nvidia_modprobe
   nvidia_modprobe//kmod
   tcpdump
0 profiles are in complain mode.
0 profiles are in kill mode.
0 profiles are in unconfined mode.
0 processes have profiles defined.
0 processes are in enforce mode.
```
## Exercise 1 - Namespaces: demonstrate PID, NET and USER isolation
**Objective:** verify that each container lives in namespaces separate from the host.
### Namespace: PID
```bash
docker run --rm alpine sh -c "echo 'PID inside:'; ps aux | head -5"
PID inside:
PID   USER     TIME  COMMAND
    1 root      0:00 sh -c echo 'PID inside:'; ps aux | head -5
    7 root      0:00 ps aux
    8 root      0:00 head -5
#The sh -c process shows up as PID 1 inside the container
docker run -d --name ns-test alpine sleep 300
docker inspect ns-test | grep -i pid
            "Pid": 4891,
            "PidMode": "",
            "PidsLimit": null,
#Here we can see the real PID on the host machine is 4891
sudo kill -0 $(docker inspect -f '{{.State.Pid}}' ns-test) && echo "PID exists on host"
PID exists on host
#With this command we can check whether the container's PID exists on the host
#The kill -0 command only checks whether the process exists and whether signals can be sent to it
docker inspect -f '{{.State.Pid}}' ns-test
4891
#We see the container's PID on the host machine, -f only returns the field between ''
docker rm -f ns-test #Forces the rm, even though the container is running
```
### Namespace NET
```bash
docker run --rm alpine ip a
2: eth0@if22: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether ea:df:2c:3a:99:8d brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
#Different IP from the host: 172.17.0.1/16
#With --network host (no isolation)
docker run --rm --network host alpine ip a | grep -E "eth0|ens"
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP qlen 1000
    inet <MANAGER_IP>/24 brd <LAN_BROADCAST_IP> scope global eth0
#Same IP as the host, docker-labs VM
```
### Namespace User
```bash
#without userns-remap UID0 inside = UID 0 on host, hence a risk
docker run --rm alpine id
uid=0(root) gid=0(root)
ps aux | grep "sleep 350" | grep -v grep
root        5425  0.0  0.0   1624     4 ?        Ss   13:53   0:00 sleep 350
#The USER column shows "root" (risk)
   
```

## Exercise 2 - Capabilities: drop ALL, selective add, --privileged
**Objective:** apply the principle of least privilege with capabilities.
```bash
docker run --rm alpine cat /proc/self/status | grep -i cap
CapInh:	0000000000000000
CapPrm:	00000000a80425fb
CapEff:	00000000a80425fb
CapBnd:	00000000a80425fb
CapAmb:	0000000000000000
#CapPrm and CapEff show the default set of capabilities
#Using libcap2-bin and the command capsh --decode=00000000a80425fb
#we see that the default capabilities are:
sudo capsh --decode=00000000a80425fb
0x00000000a80425fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
#Container without capabilities
docker run --rm --cap-drop ALL alpine sh -c "echo '=== No capabilities ==='; \
cat /proc/self/status | grep -i cap; \
ip link add dummy0 type dummy 2>&1 || echo 'EXPECTED: operation denied'"
=== No capabilities ===
CapInh:	0000000000000000
CapPrm:	0000000000000000
CapEff:	0000000000000000
CapBnd:	0000000000000000
CapAmb:	0000000000000000
ip: RTNETLINK answers: Operation not permitted
EXPECTED: operation denied
#Adding only the necessary capability
docker run --rm --cap-drop ALL --cap-add NET_ADMIN alpine sh -c "\
echo '=== NET_ADMIN only ===';\
cat /proc/self/status | grep -i cap; \
ip link add dummy0 type dummy && echo 'NET_ADMIN: operation permitted' \
|| echo 'FAILED'"
=== NET_ADMIN only ===
CapInh:	0000000000000000
CapPrm:	0000000000001000
CapEff:	0000000000001000
CapBnd:	0000000000001000
CapAmb:	0000000000000000
NET_ADMIN: operation permitted
#Privileged container
docker run --rm --privileged alpine sh -c "\
> echo '=== Privileged (all layers)';\
> cat /proc/self/status | grep -i cap;\
> echo 'Can see host devices:';\
> ls /dev | wc -l
> "
=== Privileged (all layers)
CapInh:	0000000000000000
CapPrm:	000001ffffffffff
CapEff:	000001ffffffffff
CapBnd:	000001ffffffffff
CapAmb:	0000000000000000
Can see host devices:
152
#vs same command normal container
Non-privileged container
CapInh:	0000000000000000
CapPrm:	00000000a80425fb
CapEff:	00000000a80425fb
CapBnd:	00000000a80425fb
CapAmb:	0000000000000000
Can see host devices:
15
```
## Exercise 3 - cgroups: CPU and Memory Resource Limits
**Objective:** apply resource limits and verify that the kernel enforces them.
```bash
docker run --rm alpine sh -c "cat /sys/fs/cgroup/memory/memory.\
> limit_in_bytes 2>/dev/null || \
> cat /sys/fs/cgroup/memory.max 2>/dev/null || echo 'check with docker stats'"
max
#Since Debian 12 uses cgroups v2 the 1st part of the command produces no output while the 2nd returns max since the container has no limit
docker run -d --name mem-test --memory 128m alpine sleep 300
docker stats mem-test --no-stream
CONTAINER ID   NAME       CPU %     MEM USAGE / LIMIT   MEM %     NET I/O         BLOCK I/O   PIDS
0435f6ac3a4d   mem-test   0.00%     324KiB / 128MiB     0.25%     1.91kB / 126B   0B / 0B     1
#324KiB/128MiB
docker run --rm --memory 64m --memory-swap 64m --tmpfs /tmp \
> alpine sh -c "dd if=/dev/zero of=/tmp/bigfile bs=1M count=128" 2>&1 || echo "OOM or error"
OOM or error
#Only the echo is printed because the process was killed by the kernel before it could print anything OOM
docker rm -f mem-test
docker run -d --name cpu-test --cpus 0.5 alpine sleep 300
docker inspect cpu-test | grep -E "NanoCpus|CpuPeriod|CpuQuota"
            "NanoCpus": 500000000,
            "CpuPeriod": 0,
            "CpuQuota": 0,
# CpuPeriod and CpuQuota show up as 0 because no value was specified, so this value is taken directly from the default cgroups
docker exec cpu-test cat /sys/fs/cgroup/cpu.max
50000 100000
#Since this is Debian 12 with cgroups v2, this command lets us check the default CpuPeriod and Quota
echo "=== No CPU limit ==="; time docker run --rm alpine sh -c "for i in \$(seq 1 1000000); do echo \$i > /dev/null; done"
=== No CPU limit ===

real	0m2.996s
user	0m0.000s
sys	0m0.013s
echo "=== With 0.1 CPU limit ==="; time docker run --rm --cpus 0.1 alpine sh -c "for i in \$(seq 1 1000000); do echo \$i > /dev/null; done"
=== With 0.1 CPU limit ===

real	0m28.518s
user	0m0.004s
sys	0m0.009s
```
## Exercise 4 — Seccomp: default profile vs unconfined
**Objective:** verify that Docker applies the default seccomp profile and demonstrate it.
```bash
docker info | grep -i seccomp
  seccomp
#Seccomp active
docker run --rm alpine sh -c "                                                
    apk add --quiet libcap keyutils 2>/dev/null;
    echo '=== Process Seccomp state ===';                                   
    grep Seccomp /proc/self/status;
    echo '';                                                                     
    echo '=== Attempting keyctl (blocked by seccomp) ===';
    keyctl show && echo 'SUCCESS (unexpected)' || echo 'BLOCKED by seccomp      
  (expected)'                                                                    
  "
=== Process Seccomp state ===
Seccomp:	2
Seccomp_filters:	1
=== Attempting keyctl (blocked by seccomp) ===
Session Keyring
Unable to dump key: Operation not permitted
BLOCKED by seccomp     
  (expected)
#The command output shows that seccomp has an active filter Seccomp: 2 and keyctl is blocked
docker run --rm --security-opt seccomp=unconfined alpine sh -c "                                                
    apk add --quiet libcap keyutils 2>/dev/null;
    echo '=== Process Seccomp state ===';                                   
    grep Seccomp /proc/self/status;
    echo '';                                                                     
    echo '=== Attempting keyctl (blocked by seccomp) ===';
    keyctl show && echo 'SUCCESS (expected) ' || echo 'BLOCKED by seccomp      
  (unexpected)'                                                                    
  "
=== Process Seccomp state ===
Seccomp:	0
Seccomp_filters:	0

=== Attempting keyctl (blocked by seccomp) ===
Session Keyring
 720790696 --alswrv      0     0  keyring: _ses.ed49e850d17edb301b15168b2424b99842cb56127ee088c24a677bc09581e8e7
SUCCESS (expected) 
#With seccomp unconfined keyctl is allowed. Seccomp: 0
docker run -d --name sec-test alpine sleep 60
docker inspect sec-test | grep -i SecurityOpt
            "SecurityOpt": null,
#Implies default seccomp profile
```
## Exercise 5 — AppArmor: verify the docker-default profile
**Objective:** confirm that AppArmor is active and understand what it restricts.
```bash
sudo aa-status | grep docker
   docker-default
#AppArmor loaded with the docker-default profile
docker run -d --name aa-test --security-opt apparmor=docker-default alpine sleep 60
docker inspect aa-test | grep -i apparmor
        "AppArmorProfile": "docker-default",
                "apparmor=docker-default"
#Default AppArmorProfile: docker-default. Without security-opt, apparmor=docker-default applies by default
docker run --rm --security-opt apparmor=docker-default \    alpine cat /proc/self/attr/current 
docker-default (enforce)
#docker-default assigned by default
docker run --rm \
> --security-opt apparmor=unconfined alpine sh -c "cat /proc/self/attr/current 2>/dev/null || echo 'No AppArmor label'"
unconfined
#Shows unconfined
```
## Exercise 6 — userns-remap: demonstrate UID remapping
**Objective:** contrast the container process UID on the host with and without userns-remap.
```bash
docker run -d --name uid-before alpine sleep 300
HOST_PID=$(docker inspect -f '{{.State.Pid}}' uid-before)
#Store the PID in the HOST_PID variable
echo "PID on host: $HOST_PID"
PID on host: 10465
ps -o user= -p $HOST_PID 
root
#User = root
docker run -d --name uid-after alpine sleep 300
HOST_PID_AFTER=$(docker inspect -f '{{.State.Pid}}' uid-after)
echo "PID on host with userns-remap: $HOST_PID_AFTER"
PID on host with userns-remap: 10930
ps -o user= -p $HOST_PID_AFTER
100000
#user = 100000
cat /proc/$(docker inspect -f '{{.State.Pid}}' $(docker run -d alpine sleep 60))/uid_map
         0     100000      65536
#check the default userns-remap mapping
#Done with a test user created in the Week 1 3.8 section
```
## Quick verification

```bash
docker info | grep -A5 "Security Options"
# Should show: apparmor, seccomp, Seccomp Profile: builtin

docker run --rm --cap-drop ALL alpine ip link add dummy0 type dummy 2>&1 \
  || echo "CORRECT: operation blocked"

docker run --rm alpine cat /proc/self/status | grep Seccomp
# Seccomp: 2

sudo aa-status | grep -c docker-default && echo "docker-default present"
```

## Findings

### Verifying capabilities

To verify that capabilities were removed, a container is run with `--cap-drop ALL` and an operation that requires them is attempted:

```sh
docker run --rm --cap-drop ALL alpine sh -c "ip link add dummy0 type dummy 2>&1 || echo 'Operation denied: capability missing'"
```

The `2>&1` redirection merges stderr (file descriptor 2) with stdout (file descriptor 1), so the error message shows up in standard output. The `&` before the `1` is essential: without it, `2>1` would create a file literally named `1` in the filesystem instead of redirecting to FD 1.

### Detecting the memory limit: cgroups v1 vs v2

During exercise 3 (resource limits) the following command was run to verify the memory limit applied to the container from inside it:

```bash
docker run --rm alpine sh -c "cat /sys/fs/cgroup/memory/memory.limit_in_bytes 2>/dev/null || cat /sys/fs/cgroup/memory.max 2>/dev/null || echo 'check with docker stats'"
```

The command implements a cascading OR logic to be compatible with both cgroups v1 and cgroups v2:

- **Command 1** (`cat /sys/fs/cgroup/memory/memory.limit_in_bytes`) — cgroups v1-specific path. If the kernel uses cgroups v2 (unified hierarchy), the path does not exist and the command fails. The `2>/dev/null` redirection silently discards the error so it doesn't pollute the output.
- **Command 2** (`cat /sys/fs/cgroup/memory.max`) — cgroups v2 path. Returns the limit in bytes if `--memory` was applied, or the literal `max` if no limit is configured. This is the active path on Debian 12.
- **Command 3** (`echo 'check with docker stats'`) — final fallback: runs only if the two previous commands fail, indicating that no cgroup path is accessible.

On the Docker Labs VM (Debian 12 with kernel 6.1), command 1 fails because the system uses cgroups v2 exclusively. The result comes from command 2.

**Relation to DCA topics:** 3.8 (resource limits / cgroups).

### Troubleshooting: how to trigger OOM with resource limits (topic 3.8)

During exercise 3 several attempts were made to get the kernel's OOM killer to kill a container process with a memory limit.

**Attempt 1 — destination /dev/null (no effect)**

```bash
docker run --rm --memory 64m --memory-swap 64m alpine sh -c \
  "dd if=/dev/zero of=/dev/null bs=1M count=128" 2>&1 || echo "EXPECTED: OOM or error"
```

Output:
```
128+0 records in
128+0 records out
134217728 bytes (128.0MB) copied, 0.002347 seconds, 53.3GB/s
```

Reason: `of=/dev/null` discards the data immediately. `dd` reuses the same 1 MB buffer (= `bs`) on each iteration. Actual RAM usage is constant at ~1 MB, never exceeding the 64 MB limit.

**Attempt 2 — destination /tmp without --tmpfs (no effect)**

```bash
docker run --rm --memory 64m --memory-swap 64m alpine sh -c \
  "dd if=/dev/zero of=/tmp/bigfile bs=1M count=128" 2>&1 || echo "EXPECTED: OOM or error"
```

Output:
```
128+0 records in
128+0 records out
134217728 bytes (128.0MB) copied, 0.207151 seconds, 617.9MB/s
```

Reason: `/tmp` inside the container is part of the overlay filesystem (disk), not RAM. Without `--tmpfs`, the write goes to disk and does not consume cgroup memory.

**Attempt 3 — /tmp as tmpfs (OOM confirmed)**

```bash
docker run --rm --memory 64m --memory-swap 64m --tmpfs /tmp \
  alpine sh -c "dd if=/dev/zero of=/tmp/bigfile bs=1M count=128" 2>&1 || echo "OOM or error"
```

Output:
```
OOM or error
```

Reason: `--tmpfs /tmp` mounts `/tmp` in RAM. Accumulating 128 MB in memory with a 64 MB limit causes the kernel's OOM killer to kill the process. The process is killed before writing anything to stdout, which is why only the `echo` from the `||` shows up.

**Summary table:**

| Command | Actual destination | RAM consumed | OOM |
|---|---|---|---|
| `of=/dev/null` | discarded | ~1 MB (buffer) | No |
| `of=/tmp/bigfile` (without `--tmpfs`) | overlay/disk | ~1 MB (buffer) | No |
| `of=/tmp/bigfile` + `--tmpfs /tmp` | RAM (tmpfs) | cumulative | Yes |

**Relation to DCA topics:** 3.8 (resource limits / cgroups), also relevant for understanding `--tmpfs` in volumes.

### Study notes: empirically verifying cgroups with a CPU benchmark (topic 3.8)

To check that the CPU limit is actually enforced and not just configured, you can measure the execution time of a pure CPU load with and without a limit:

```bash
# No limit
time docker run --rm alpine sh -c "for i in \$(seq 1 1000000); do echo \$i > /dev/null; done"

# With a 0.5 CPU limit
time docker run --rm --cpus 0.5 alpine sh -c "for i in \$(seq 1 1000000); do echo \$i > /dev/null; done"
```

Key points about the command:
- `time` measures the `real`, `user`, and `sys` times of the Docker process on the host.
- The `for i in $(seq 1 1000000)` loop generates pure CPU load with no network or disk I/O.
- `echo $i > /dev/null` discards the output so the bottleneck is CPU only.
- With `--cpus 0.5`, the real time should be roughly double that without the limit, confirming that the cgroup is active and throttling.
- The escaped `\$` are essential: without the backslash, the host shell would expand `$i` and `$(seq ...)` before passing the command to the container, resulting in an empty loop or an error.

**Relation to DCA topics:** 3.8 (resource limits / cgroups).

## DCA/SRE Lessons

- **`--cap-drop ALL` as the basis of least privilege.** Removing all capabilities and adding back only what's needed (`--cap-add`) drastically reduces the attack surface. `--privileged` should never be used in production except for justified cases.

- **`2>&1` vs `2>1` in shell.** Without the `&`, `2>1` creates a file literally named `1` instead of redirecting to file descriptor 1. Essential for capturing capability or seccomp errors in verification scripts.

- **cgroups v2 vs v1 on Debian 12+.** The path `/sys/fs/cgroup/memory/memory.limit_in_bytes` does not exist under cgroups v2; the correct one is `/sys/fs/cgroup/memory.max`. Verification scripts must account for both versions.

- **For `--memory` to trigger the OOM killer, consumption must happen in RAM.** Writing to `/tmp` without `--tmpfs` goes to the overlay filesystem, not the memory cgroup. Only with `--tmpfs /tmp` does the write go to RAM and the limit take effect.

- **`Seccomp: 2` in `/proc/self/status` confirms an active filter.** The value `0` indicates `--security-opt seccomp=unconfined`. Docker's default profile blocks dangerous syscalls such as `keyctl`.

- **userns-remap as a privilege escalation mitigation.** Without it, UID 0 inside the container is UID 0 on the host. With userns-remap active, that process shows up on the host as UID 100000 (unprivileged), limiting the impact of any escape.

---

## Pending extensions (post-DCA portfolio)

> Gaps identified during the DCA review (score: 8.5/10). Not blocking for the exam. Extend the portfolio once certified.

- **MNT, IPC, and UTS namespaces** — The lab demonstrates PID, NET, and USER. Complete it with mount namespace exercises (chroot-like isolation), IPC namespace (isolated semaphores/message queues), and UTS namespace (per-container hostname).
- **Custom seccomp profile (JSON)** — Demonstrate creating a `seccomp.json` profile with explicitly blocked/allowed syscalls and applying it with `--security-opt seccomp=<profile>`. This is the most powerful demonstration of the mechanism and a good real-world hardening exercise.
