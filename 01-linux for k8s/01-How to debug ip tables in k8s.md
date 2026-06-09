# How to debug ip tables in k8s? 

This is exactly where many DevOps candidates struggle. They memorize commands but don't know:

1. **Why we run the command**
2. **How to read the output**
3. **What problem it helps solve**
4. **What action to take after seeing the output**

Let's learn these from a Kubernetes/DevOps/SRE troubleshooting perspective.

---

# 1. IPTables

## Command

```bash
iptables -L
```

## Why Do We Run It?

To check firewall rules on a Linux server.

Common scenarios:

- Pod cannot reach another Pod
- Service not accessible
- Port blocked
- Network troubleshooting

---

![alt text](image.png)

---

## How To Read It

### Rule 1

```
ACCEPT tcp anywhere anywhere tcp dpt:ssh
```

Means:

```
Allow SSH traffic
Port 22
```

---

### Rule 2

```
DROP all 10.0.0.5 anywhere
```

Means:

```
Block all traffic
coming from 10.0.0.5
```

---

## Real Production Issue

Developer says:

```
Application unreachable
```

You check:

```bash
iptables -L
```

Output:

```
DROP tcp anywhere anywhere tcp dpt:8080
```

Problem found:

```
Port 8080 blocked
```

Fix:

```bash
iptables -D INPUT rule-number
```

or

```bash
iptables -F
```

(Only if safe)

Here's an example of how to use the `iptables -D` command to delete a specific rule from the INPUT chain:

### Example Scenario

Let's say you have the following rules in your INPUT chain:

1. Accept established connections
2. Drop invalid packets
3. Allow SSH traffic on port 22
4. Allow HTTP traffic on port 80

If you want to delete the rule that allows SSH traffic (which is rule number 3), you would first list the rules to confirm the rule number:

```bash
iptables -L INPUT --line-numbers
```

The output might look something like this:

```
Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination
1    ACCEPT     all  --  anywhere             anywhere             state ESTABLISHED
2    DROP       all  --  anywhere             anywhere             state INVALID
3    ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:22
4    ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:80
```

### Deleting the Rule

To delete the rule allowing SSH traffic, you would run:

```bash
iptables -D INPUT 3
```

### Explanation

- This command deletes the rule at position 3 in the INPUT chain, which allows SSH traffic on port 22.
- After executing this command, you can verify that the rule has been removed by listing the rules again:

```bash
iptables -L INPUT --line-numbers
```

Now, the output should no longer include the rule for SSH traffic.

---

# 2. Filesystem

## Command

```bash
df -Th
```

---

## Why Do We Run It?

To check:

- Disk usage
- Filesystem type
- Available storage

Very common in Kubernetes node troubleshooting.

---

## Sample Output

```
Filesystem     Type  Size Used Avail Use%
/dev/sda1      ext4   50G  48G   2G  96%
```

---

## How To Read It

| Column | Meaning |
| --- | --- |
| Filesystem | Disk |
| Type | Filesystem Type |
| Size | Total Disk |
| Used | Used Space |
| Avail | Free Space |
| Use% | Disk Usage |

---

## Problem

```
96% Full
```

Very dangerous.

At:

```
100%
```

Node may become:

```
NotReady
```

Pods may fail.

---

## Solution

Find large files:

```bash
du -sh /*
```

or

```bash
du -sh /var/*
```

Cleanup:

```bash
docker system prune
```

or

```bash
journalctl --vacuum-time=7d
```
## Breakdown of the Command 

- journalctl: This is a command-line utility for querying and displaying messages from the journal, which is a component of systemd that collects and stores log data. and 
- --vacuum-time=7d: This option instructs journalctl to delete journal entries that are older than 7 days.

---

# 3. Mount Points

## Command

```bash
mount
```

or

```bash
df -h
```

---

## Why Do We Run It?

Check:

```
Is storage mounted?
```

Very common with:

- EBS
- NFS
- Persistent Volumes

---

## Output

```
/dev/sdb on /data type ext4
```

---

## Meaning

```
Disk = /dev/sdb

Mounted at

/data
```

---

## Real Issue

Application says:

```
No such file or directory
```

Check:

```bash
mount
```

Expected:

```
/data
```

Not found.

Meaning:

```
Storage not mounted
```

---

## Fix

```bash
mount /dev/sdb /data
```

---

# 4. Swap

## Command

```bash
swapon --show
```

---

## Why?

Before installing Kubernetes.

---

## Output

```
NAME      TYPE SIZE USED
/swapfile file 2G 500M
```

---

## Meaning

Swap enabled.

---

## Kubernetes Problem

Run:

```bash
kubeadm init
```

Error:

```
swap is enabled
```

---

## Fix

Temporary:

```bash
swapoff -a
```

Permanent:

Edit:

```bash
/etc/fstab
```

Comment swap entry.

---

# 5. Systemd

## Command

```bash
systemctl status kubelet
```

---

## Why?

Most common Kubernetes troubleshooting command.

---

## Healthy Output

```
Active: active (running)
```

Good.

---

## Problem Output

```
Active: failed
```

or

```
Active: inactive
```

---

## Meaning

Kubelet not running.

Node won't join cluster.

---

## Fix

Start:

```bash
systemctl start kubelet
```

Enable:

```bash
systemctl enable kubelet
```

---

# 6. journalctl

## Command

```bash
journalctl -u kubelet
```

---

## Why?

See kubelet logs.

---

## Real Issue

Node status:

```
NotReady
```

---

## Run

```bash
journalctl -u kubelet
```

Output:

```
failed to connect to container runtime
```

---

## Meaning

Kubelet cannot talk to containerd.

---

## Next Step

Check:

```bash
systemctl status containerd
```

---

## Another Example

Output:

```
certificate expired
```

Meaning:

```
Node certificate expired
```

Renew certificates.

---

# 7. Syslog

## Command

```bash
tail -f /var/log/syslog
```

or

```bash
cat /var/log/messages
```

---

## Why?

Observe system events live.

---

## Output

```
disk pressure detected
```

---

## Meaning

Disk nearly full.

---

## Solution

```bash
df -h
```

Then cleanup.

---

# 8. SELinux

## Command

```bash
getenforce
```

---

## Output

```
Enforcing
```

---

## Meaning

SELinux active.

---

## Real Issue

Application cannot access file.

Even though:

```
chmod 777
```

is applied.

Still fails.

---

## Check

```bash
getenforce
```

Output:

```
Enforcing
```

---

## Verify

```bash
setenforce 0
```

Application works.

Root cause:

```
SELinux policy blocking access
```

---

## Fix

Don't disable permanently.

Instead update policy.

---

# 9. AppArmor

## Command

```bash
aa-status
```

---

## Output

```
10 profiles loaded
5 profiles enforced
```

---

## Meaning

AppArmor active.

---

## Real Kubernetes Issue

Pod starts.

Immediately crashes.

---

## Logs

```
Permission denied
```

---

## Check

```bash
aa-status
```

AppArmor profile blocking action.

---

## Fix

Modify profile.

Not disable AppArmor.

---

# Most Common Kubernetes Troubleshooting Flow

Suppose interviewer asks:

> "A node suddenly becomes NotReady. What commands would you run?"
> 

Answer:

| Step | Command | Purpose |
| --- | --- | --- |
| 1 | `kubectl get nodes` | Verify status |
| 2 | `systemctl status kubelet` | Is kubelet running? |
| 3 | `journalctl -u kubelet` | Check kubelet errors |
| 4 | `systemctl status containerd` | Runtime healthy? |
| 5 | `df -h` | Disk full? |
| 6 | `free -m` | Memory issue? |
| 7 | `swapon --show` | Swap enabled? |
| 8 | `mount` | Storage mounted? |
| 9 | `iptables -L` | Network blocked? |
| 10 | `getenforce` | SELinux blocking? |

This troubleshooting chain is much more valuable in interviews than simply memorizing what IPTables, Systemd, or SELinux are. Interviewers often want to know **how you would use these tools to diagnose a real production problem**.