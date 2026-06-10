
# Linux Concepts Every Kubernetes Engineer Must Know


Since Kubernetes was originally built around Linux, many Kubernetes components directly interact with Linux kernel features. Understanding these concepts helps tremendously in troubleshooting pods, nodes, networking, storage, and security issues.

# Complete Kubernetes-Linux Mapping

| Linux Concept | Kubernetes Usage |
| --- | --- |
| IPTables | Service Networking |
| Filesystem | PV/PVC Storage |
| Mount Points | Volume Mounts |
| Swap | Memory Management |
| Systemd | Kubelet & Container Runtime Services |
| journalctl | Node Troubleshooting |
| Syslog | System Logs |
| SELinux | Container Security |
| AppArmor | Container Security Profiles |

---

# 1. IPTables

## What is IPTables? (Refer next page to know, how to debug iptables)

IPTables is a Linux firewall and packet filtering framework used to control network traffic.

Think of it as:

```
Traffic Police for Linux Network Packets
```

Every incoming or outgoing packet can be:

- Allowed
- Blocked
- Redirected
- Modified

---

## Why Kubernetes Uses IPTables

Kubernetes Services need traffic routing.

Example:

```
Service IP
10.96.0.10
     |
     v
Pod A
Pod B
Pod C
```

When a request reaches the Service IP, Kubernetes must forward it to one of the pods.

This routing is done using IPTables rules (or IPVS).

---

## Real Example

View rules:

```bash
sudo iptables -L
```
Example Output:

```
Chain INPUT
target     prot opt source      destination
ACCEPT     all  --  anywhere    anywhere

Chain FORWARD
target     prot opt source      destination

Chain OUTPUT
target     prot opt source      destination
```

Show NAT rules:

```bash
sudo iptables -t nat -L
```

---

## Kubernetes Usage

Kube Proxy automatically creates IPTables rules.

```
User Request
      |
      v
Service IP
      |
 IPTables Rule
      |
      v
Target Pod
```

---

## Interview Answer

> IPTables is Linux's packet filtering and firewall framework. Kubernetes uses IPTables through Kube Proxy to implement Service networking and route traffic from Services to Pods.
> 

---

# 2. Filesystems

## What is a Filesystem?

A filesystem determines:

```
How files are stored
How files are organized
How files are retrieved
```

Examples:

| Filesystem | Usage |
| --- | --- |
| ext4 | Most Linux systems |
| XFS | Enterprise servers |
| Btrfs | Snapshot-based systems |
| NFS | Shared network storage |

---

## Why Kubernetes Cares

Pods need storage.

When a Persistent Volume is attached:

```
AWS EBS
     |
     v
Linux Filesystem
     |
     v
Mounted Inside Pod
```

The underlying filesystem determines how data is stored.

---

## Check Filesystem

```bash
df -Th
```

Example output:

```
/dev/sda1 ext4
```

---

## Interview Answer

> A filesystem defines how data is stored and managed on disk. Kubernetes storage solutions ultimately rely on Linux filesystems underneath.
> 

---

# 3. Mount Points

## What is a Mount Point?

A mount point is a directory where storage is attached.

Example:

```
Disk
/dev/sdb
     |
Mounted At
/data
```

Now:

```bash
cd /data
```

accesses that disk.

---

## Real Example

View mounted disks:

```bash
mount
```

or

```bash
df -h
```

---

## Why Kubernetes Uses Mounts

When a volume is attached:

```
Persistent Volume
      |
Mounted on Node
      |
Mounted into Pod
```

Example:

```yaml
volumeMounts:
- mountPath: /app/data
```

Kubernetes internally performs mount operations.

---

## Analogy

Think of a mount point like:

```
USB Drive
      |
Plugged Into
      |
Folder
```

---

## Interview Answer

> A mount point is a directory where a filesystem or storage device is attached. Kubernetes uses mount points extensively when attaching Persistent Volumes to Pods.
> 

---

# 4. Swap

## What is Swap?

Swap is disk space used as extra memory when RAM is exhausted.

Example:

```
RAM Full
   |
Move Some Memory Pages
   |
Disk Swap Space
```

---

## Problem with Swap

Disk is much slower than RAM.

```
RAM Access
≈ Nanoseconds

Disk Access
≈ Milliseconds
```

Huge performance difference.

---

## Why Kubernetes Disables Swap

Kubernetes depends heavily on accurate memory management.

If swap is enabled:

```
Pod Memory Calculations
Become Unpredictable
```

Node stability decreases.

---

## Check Swap

```bash
swapon --show
```

---

## Disable Swap

```bash
swapoff -a
```

---

## Interview Question

Why does Kubernetes require swap to be disabled?

Answer:

> Kubernetes relies on accurate memory accounting and resource management. Swap can cause unpredictable performance and resource usage, so Kubernetes traditionally requires it to be disabled.
> 

---

# 5. Systemd

## What is Systemd?

Systemd is the service manager of modern Linux systems.

It starts, stops, and monitors services.

Think of it as:

```
Service Manager of Linux
```

---

## Examples

Start service:

```bash
systemctl start nginx
```

Stop service:

```bash
systemctl stop nginx
```

Status:

```bash
systemctl status nginx
```

---

## Why Kubernetes Uses It

Kubernetes components run as services.

Examples:

```
kubelet
containerd
docker
```

Check Kubelet:

```bash
systemctl status kubelet
```

---

## Real Production Scenario

Pod not starting.

First thing SRE checks:

```bash
systemctl status kubelet
```

Many incidents end here.

---

## Interview Answer

> Systemd is Linux's service manager responsible for starting, stopping, and monitoring services. Kubernetes components such as Kubelet and Container Runtime are managed by Systemd.
> 

---

# 6. journalctl and Syslog

These are used for logging.

---

## journalctl

Used on Systemd-based systems.

View logs:

```bash
journalctl
```

Kubelet logs:

```bash
journalctl -u kubelet
```

Recent logs:

```bash
journalctl -xe
```

---

## Real Kubernetes Usage

Node NotReady.

Check:

```bash
journalctl -u kubelet
```

You might find:

```
Certificate Expired
Container Runtime Down
Disk Pressure
```

---

## Syslog

Traditional Linux logging system.

Logs stored in:

```
/var/log/syslog
```

or

```
/var/log/messages
```

---

## Difference

| journalctl | Syslog |
| --- | --- |
| Modern | Traditional |
| Systemd based | Syslog daemon based |
| Structured logs | Plain logs |
| Preferred today | Legacy systems |

---

## Interview Answer

> journalctl and Syslog are Linux logging systems. Kubernetes administrators frequently use journalctl to troubleshoot kubelet, container runtime, and node-related issues.
> 

---

# 7. SELinux

(Security Enhanced Linux)

---

## What is SELinux?

SELinux provides Mandatory Access Control (MAC).

It restricts what processes can do.

Even root users can be restricted.

---

## Analogy

Without SELinux:

```
Security Guard
Checks Building Entry Only
```

With SELinux:

```
Security Guard
Checks Every Room Access
```

---

## Why Kubernetes Uses It

Container isolation.

Prevent:

```
Pod Accessing Unauthorized Files
Pod Escaping Container
Unauthorized Host Access
```

---

## Check Status

```bash
getenforce
```

Output:

```
Enforcing
Permissive
Disabled
```

---

## Interview Answer

> SELinux provides mandatory access controls that restrict what processes and containers can access, improving node and container security.
> 

---

# 8. AppArmor

## What is AppArmor?

AppArmor is another Linux security framework.

Similar goal as SELinux.

Controls:

```
Files
Network
Capabilities
Processes
```

---

## Example

You can define:

```
Container Can Read:
/app/config

Container Cannot Access:
/etc/passwd
```

---

## Kubernetes Usage

Pod security profiles can use AppArmor.

Example:

```yaml
annotations:
  container.apparmor.security.beta.kubernetes.io/nginx: runtime/default
```

---

## SELinux vs AppArmor

| SELinux | AppArmor |
| --- | --- |
| Label Based | Path Based |
| More Powerful | Easier |
| Common in RHEL | Common in Ubuntu |
| Complex | Simpler |

---

## Interview Answer

> AppArmor is a Linux security module that restricts application capabilities using security profiles. Kubernetes can use AppArmor profiles to improve container isolation.
> 

---

# 30-Second Interview Answer

> Kubernetes is deeply integrated with Linux. IPTables is used for Service networking, filesystems and mount points are used for storage and volumes, swap must typically be disabled for proper resource management, Systemd manages services like kubelet and containerd, journalctl is used for troubleshooting node issues, and SELinux/AppArmor provide additional security and isolation for containers. Understanding these Linux concepts is essential for operating and troubleshooting Kubernetes clusters in production.
>