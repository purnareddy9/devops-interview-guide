---
layout: default
title: "🐧 Linux — DevOps Interview Guide"
render_with_liquid: false
---

# 🐧 Linux — DevOps Interview Guide

> Back to [Main Index](./README.md) | Next: [Networking →](./Networking.md)

---

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [File System](#file-system)
3. [Process Management](#process-management)
4. [User & Permission Management](#user--permission-management)
5. [Networking Commands](#networking-commands)
6. [Shell Scripting](#shell-scripting)
7. [Performance & Troubleshooting](#performance--troubleshooting)
8. [Interview Questions](#interview-questions)
9. [Scenario-Based Questions](#scenario-based-questions)
10. [Hands-On Labs](#hands-on-labs)
11. [Command Reference](#command-reference)

---

## Core Concepts

### Linux Architecture

```
┌─────────────────────────────────┐
│         User Applications        │
├─────────────────────────────────┤
│          Shell / CLI             │
├─────────────────────────────────┤
│        System Libraries          │
├─────────────────────────────────┤
│           Linux Kernel           │
│  (Process, Memory, I/O, FS Mgmt) │
├─────────────────────────────────┤
│           Hardware               │
└─────────────────────────────────┘
```

### Key Distributions for DevOps

| Distro | Use Case | Package Manager |
|--------|----------|-----------------|
| Ubuntu/Debian | General, containers | `apt` |
| CentOS/RHEL | Enterprise servers | `yum` / `dnf` |
| Alpine | Docker images (tiny) | `apk` |
| Amazon Linux | AWS environments | `yum` / `dnf` |
| Arch Linux | Bleeding edge | `pacman` |

---

## File System

### Linux Directory Structure

```
/
├── bin/      → Essential user binaries (ls, cp, mv)
├── sbin/     → System binaries (iptables, fdisk)
├── etc/      → Configuration files
├── home/     → User home directories
├── var/      → Variable data (logs, spool, databases)
├── tmp/      → Temporary files (cleared on reboot)
├── opt/      → Optional software packages
├── usr/      → User programs and data
│   ├── bin/  → Non-essential binaries
│   ├── lib/  → Libraries
│   └── local/→ Locally compiled software
├── proc/     → Virtual FS for process/kernel info
├── sys/      → Virtual FS for device/driver info
├── dev/      → Device files
└── mnt/      → Mount points
```

### Essential File Commands

```bash
# Navigation
ls -lah                    # List with human-readable sizes
ls -lt                     # Sort by modification time
find / -name "*.conf" -type f 2>/dev/null   # Find files
find /var/log -mtime -7    # Files modified in last 7 days
locate nginx.conf          # Fast file search (uses DB)

# File Operations
cp -rp /src /dest          # Copy preserving permissions
mv /old /new               # Move/rename
rm -rf /path               # Force recursive delete (CAUTION)
ln -s /target /link        # Symbolic link
ln /target /link           # Hard link

# Viewing Files
cat file.txt               # Print file
less file.txt              # Paginate (q to quit)
head -20 file.txt          # First 20 lines
tail -f /var/log/syslog    # Follow live logs
grep -r "error" /var/log/  # Recursive search

# File info
stat file.txt              # Detailed file metadata
file /bin/bash             # Determine file type
wc -l file.txt             # Count lines
du -sh /var/log/           # Disk usage of directory
df -h                      # Disk free (all filesystems)
```

---

## Process Management

### Understanding Processes

```bash
# List processes
ps aux                     # All processes
ps aux | grep nginx        # Filter by name
pstree                     # Process tree
top                        # Real-time process monitor
htop                       # Enhanced top (interactive)

# Process control
kill -9 <PID>              # Force kill
kill -15 <PID>             # Graceful termination (SIGTERM)
killall nginx              # Kill by name
pkill -f "python script"   # Kill by pattern

# Background/foreground
./long_script.sh &         # Run in background
jobs                       # List background jobs
fg %1                      # Bring job 1 to foreground
bg %1                      # Resume job 1 in background
nohup ./script.sh &        # Run immune to hangup

# Process priority
nice -n 10 ./script.sh     # Start with lower priority
renice -n -5 -p <PID>      # Change priority of running proc
```

### Process States

| State | Symbol | Description |
|-------|--------|-------------|
| Running | R | Actively using CPU |
| Sleeping | S | Waiting for event |
| Uninterruptible Sleep | D | Waiting for I/O |
| Zombie | Z | Finished but not reaped |
| Stopped | T | Paused (Ctrl+Z) |

---

## User & Permission Management

### File Permissions

```
-rwxr-xr--  1  owner  group  size  date  filename
│││││││││└─ other: read only
││││││└──── group: read + execute
│││└─────── owner: read + write + execute
││└──────── special bits
│└───────── type: - file, d dir, l symlink
```

```bash
# Permission commands
chmod 755 file.sh          # rwxr-xr-x
chmod +x script.sh         # Add execute
chmod -R 644 /var/www/     # Recursive
chown user:group file      # Change owner
chown -R www-data /var/www # Recursive ownership

# Special permissions
chmod 4755 binary          # SetUID (runs as owner)
chmod 2755 directory       # SetGID (inherits group)
chmod 1777 /tmp            # Sticky bit (only owner deletes)

# ACLs (more granular)
setfacl -m u:bob:rw file   # Grant bob read/write
getfacl file               # View ACLs
```

### User Management

```bash
# Users
useradd -m -s /bin/bash bob  # Create user with home + shell
usermod -aG docker bob        # Add to docker group
passwd bob                     # Set password
userdel -r bob                 # Delete user + home dir

# Groups
groupadd devops
groupmod -n newname devops
groups bob                     # Show user's groups

# Sudo
visudo                         # Edit sudoers safely
# Add: bob ALL=(ALL) NOPASSWD: ALL

# Switch user
su - bob                       # Switch to bob
sudo -u bob command            # Run command as bob
```

---

## Networking Commands

```bash
# Interface info
ip addr show                   # IP addresses
ip link show                   # Interface status
ifconfig -a                    # Legacy (deprecated)

# Routing
ip route show                  # Routing table
ip route add 10.0.0.0/8 via 192.168.1.1
traceroute google.com          # Trace hops
tracepath google.com           # Similar, no root needed

# Connectivity
ping -c 4 google.com           # ICMP test
curl -I https://example.com    # HTTP headers
wget -O /dev/null http://test  # Download test
telnet host 22                 # TCP connectivity test
nc -zv host 443                # Netcat port check

# DNS
dig google.com                 # DNS lookup
dig +short google.com          # Short output
nslookup google.com            # Legacy DNS lookup
host google.com                # Simple DNS

# Ports & connections
ss -tulnp                      # Active sockets (preferred)
netstat -tulnp                 # Legacy equivalent
lsof -i :80                    # Who is using port 80
fuser 80/tcp                   # PID using port

# Firewall
iptables -L -n                 # List rules
ufw status                     # UFW status (Ubuntu)
firewall-cmd --list-all        # Firewalld (RHEL/CentOS)
```

---

## Shell Scripting

### Script Fundamentals

```bash
#!/bin/bash
# Script: deploy.sh
set -euo pipefail              # Exit on error, unset var, pipe fail
IFS=$'\n\t'                    # Safe word splitting

# Variables
APP_NAME="myapp"
VERSION="${1:-latest}"         # Default to 'latest' if no arg
readonly MAX_RETRIES=3

# Arrays
SERVERS=("web01" "web02" "web03")
for server in "${SERVERS[@]}"; do
    echo "Deploying to $server"
done

# Functions
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

check_command() {
    command -v "$1" >/dev/null 2>&1 || {
        log "ERROR: $1 is not installed"
        exit 1
    }
}

# Conditionals
if [[ -f "/etc/nginx/nginx.conf" ]]; then
    log "Nginx config found"
elif [[ -d "/etc/apache2" ]]; then
    log "Apache found"
else
    log "No web server config found"
fi

# String operations
str="hello world"
echo "${str^^}"               # HELLO WORLD (uppercase)
echo "${str:0:5}"             # hello (substring)
echo "${str/world/DevOps}"    # hello DevOps (replace)

# Arithmetic
count=0
((count++))
result=$((10 * 3 + 2))

# Error handling
trap 'log "Error on line $LINENO"' ERR
trap 'cleanup' EXIT

cleanup() {
    rm -f /tmp/deploy.lock
}

# Read input
read -r -p "Enter environment: " ENV
read -r -s -p "Enter password: " PASS   # Silent input

# Here document
cat <<EOF > /tmp/config.txt
server=${APP_NAME}
version=${VERSION}
EOF
```

### Useful One-Liners

```bash
# Find and replace in multiple files
find . -name "*.conf" -exec sed -i 's/old/new/g' {} \;

# Count occurrences
grep -c "ERROR" /var/log/app.log

# Sort and unique
sort file.txt | uniq -c | sort -rn

# Extract columns (awk)
awk '{print $1, $3}' access.log
awk -F: '{print $1}' /etc/passwd | head

# Process CSV
awk -F',' 'NR>1 {sum+=$3} END {print "Total:", sum}' data.csv

# Check last N lines matching pattern
tail -n 1000 /var/log/app.log | grep -i "error"

# Extract IP addresses
grep -Eo '([0-9]{1,3}\.){3}[0-9]{1,3}' /var/log/nginx/access.log

# Monitor file changes
watch -n 2 'tail -20 /var/log/app.log'

# Parallel execution
echo "server1 server2 server3" | xargs -P3 -n1 ssh -n user@{} uptime
```

---

## Performance & Troubleshooting

### System Performance Analysis

```bash
# CPU
top -b -n1                     # Snapshot of top
mpstat -P ALL 1                # Per-CPU stats
vmstat 1 10                    # CPU, memory, I/O overview
sar -u 1 5                     # CPU utilization history
lscpu                          # CPU architecture info

# Memory
free -h                        # Memory usage
cat /proc/meminfo              # Detailed memory info
vmstat -s                      # Memory stats
slabtop                        # Kernel slab cache

# Disk I/O
iostat -xz 1                   # Disk I/O statistics
iotop                          # Per-process disk I/O
lsblk                          # Block devices
blkid                          # Block device attributes
fdisk -l                       # Partition table

# Network
iftop                          # Network bandwidth per connection
nethogs                        # Per-process network usage
sar -n DEV 1                   # Network device stats
tcpdump -i eth0 port 80        # Packet capture
ss -s                          # Socket statistics summary

# System load
uptime                         # Load average (1, 5, 15 min)
w                              # Who is logged in + load
dmesg | tail -20               # Kernel ring buffer
journalctl -xe                 # Systemd journal errors
```

### Load Average Explained

```
Load Average: 0.5, 1.2, 0.8
              │    │    └── 15-minute average
              │    └─────── 5-minute average
              └──────────── 1-minute average

Rule: Load avg should be < number of CPU cores
4-core system: load avg of 4.0 means 100% utilized
```

### Troubleshooting High CPU

```bash
# Step 1: Identify high CPU process
top -b -n1 -o %CPU | head -20

# Step 2: Get PID details
ps -p <PID> -o pid,ppid,cmd,%cpu,%mem,etime

# Step 3: Check what it's doing
strace -p <PID> -c          # Syscall summary
lsof -p <PID>               # Open files
cat /proc/<PID>/status       # Process status
cat /proc/<PID>/cmdline      # Full command

# Step 4: Check threads
ps -p <PID> -L -o pid,tid,%cpu,comm
```

### Troubleshooting Disk Full

```bash
# Find what's using space
du -sh /* 2>/dev/null | sort -hr | head -20
du -sh /var/* | sort -hr
find / -size +100M -type f 2>/dev/null

# Check for deleted but open files
lsof | grep deleted | awk '{print $7, $1, $2}' | sort -hr

# Truncate log safely (don't delete open files)
> /var/log/huge.log           # Truncate to zero
truncate -s 0 /var/log/huge.log

# Inode exhaustion (no space but df shows space)
df -i                          # Check inode usage
find /tmp -type f | wc -l     # Count files in /tmp
```

---

## Interview Questions

### 🟢 Beginner

**Q1: What is the difference between a process and a thread?**

> A **process** is an independent program in execution with its own memory space, file handles, and resources. A **thread** is a lightweight unit of execution within a process that shares the same memory space. Processes are isolated (crash in one doesn't affect another); threads within the same process share memory, making communication easier but crashes more dangerous.

**Q2: Explain the Linux boot process.**

> 1. **BIOS/UEFI** — firmware runs POST, finds bootable device
> 2. **Bootloader (GRUB2)** — loads kernel into memory
> 3. **Kernel** — initializes hardware, mounts initramfs
> 4. **initramfs** — temporary root FS, loads modules needed to mount real root
> 5. **init/systemd (PID 1)** — starts all user-space services
> 6. **Runlevel/Target** — reaches target (multi-user, graphical)
> 7. **Login prompt**

**Q3: What does `chmod 755` mean?**

> - **7 (owner)**: rwx = 4+2+1 = read, write, execute
> - **5 (group)**: r-x = 4+0+1 = read, execute
> - **5 (others)**: r-x = read, execute
> Typical for executable scripts and directories.

**Q4: What is the difference between `>` and `>>`?**

> `>` **redirects** output, overwriting the file. `>>` **appends** output to the file. Example: `echo "log" >> /var/log/myapp.log` adds a line without losing existing content.

**Q5: Explain hard links vs symbolic links.**

> A **hard link** is a second directory entry pointing to the same inode (same data blocks). Deleting the original doesn't affect the hard link. Limited to same filesystem, cannot link directories.
> A **symbolic (soft) link** is a pointer (shortcut) to another file by path. Can cross filesystems, can link directories, but breaks if the target is deleted.

---

### 🟡 Intermediate

**Q6: How does the Linux kernel handle memory management?**

> The kernel uses **virtual memory** with paging. Each process gets its own virtual address space mapped to physical RAM via page tables. The kernel handles:
> - **Demand paging**: loads pages only when accessed
> - **Swap**: moves inactive pages to disk when RAM is full
> - **OOM killer**: kills processes when memory is critically low
> - **Huge pages**: 2MB pages for performance-critical workloads
> - **Buddy system + slab allocator**: for kernel memory management

**Q7: What is the difference between `ps aux` and `ps -ef`?**

> Both show all processes. `ps aux` (BSD syntax) shows CPU/mem % and the full command. `ps -ef` (Unix syntax) shows PPID, start time, and CMD. In practice for DevOps, `ps aux | grep <name>` is most common. `ps -ef --forest` shows parent-child relationships.

**Q8: Explain `inode` in Linux.**

> An **inode** is a data structure that stores file metadata: permissions, owner, size, timestamps, and pointers to data blocks. **Not** the filename (directory entries map names → inodes). Each filesystem has a fixed number of inodes. You can run out of inodes even with free disk space (`df -i` to check). Hard links share the same inode.

**Q9: What is `systemd` and how does it differ from `init`?**

> `systemd` is a modern init system (PID 1) that:
> - Starts services in **parallel** (faster boot vs sequential init)
> - Uses **unit files** (`.service`, `.socket`, `.timer`) instead of shell scripts
> - Provides `journalctl` for **centralized logging**
> - Manages **dependencies** between services declaratively
> - Supports **cgroups** for resource control per service
> - Has socket activation (start service on first connection)

**Q10: How would you find all files modified in the last 24 hours?**

```bash
find / -mtime -1 -type f 2>/dev/null
# -mtime -1: less than 1 day ago
# For minutes: -mmin -60 (last 60 min)
# For access time: -atime
# For change time (metadata): -ctime
```

---

### 🔴 Advanced

**Q11: Explain Linux namespaces and cgroups (foundation of containers).**

> **Namespaces** provide **isolation** — each namespace gives processes a different view of the system:
> - `pid`: isolated process IDs (PID 1 inside container)
> - `net`: isolated network stack (interfaces, routes, iptables)
> - `mnt`: isolated mount points (filesystem view)
> - `uts`: isolated hostname/domain name
> - `ipc`: isolated inter-process communication
> - `user`: isolated user/group IDs (UID mapping)
> - `cgroup`: isolated cgroup hierarchy
>
> **cgroups (Control Groups)** provide **resource limits**:
> - Limit CPU, memory, disk I/O, network bandwidth per process group
> - Used by Docker to enforce container resource limits (`--memory`, `--cpus`)
> - `systemd` uses cgroups for service resource management

**Q12: What is the difference between TCP and UDP at the OS level?**

> At the socket level: `SOCK_STREAM` (TCP) vs `SOCK_DGRAM` (UDP). TCP uses `connect()`, `accept()`, and maintains state in the kernel's TCP state machine (SYN_SENT, ESTABLISHED, TIME_WAIT, etc.). The kernel buffers data, handles retransmission, and manages the sliding window. UDP just sends datagrams with no connection state — lower overhead, no delivery guarantee.

**Q13: How would you diagnose and fix a system with high iowait?**

```bash
# Diagnose
iostat -xz 1 10              # Look for high %await, %util
iotop -a                     # Find which process is doing I/O
pidstat -d 1                 # Per-process disk stats
cat /proc/<PID>/io           # Raw I/O for specific PID
blktrace -d /dev/sda         # Deep block device tracing

# Common causes & fixes
# 1. Swap thrashing → add RAM, reduce swappiness
echo 10 > /proc/sys/vm/swappiness

# 2. Slow disk → check for failing drive
smartctl -a /dev/sda

# 3. Database checkpoint storms → tune DB flush settings

# 4. Log rotation → compress logs, archive to slower storage
```

---

## Scenario-Based Questions

### 🔵 Scenario 1: Server is Unreachable

*"Your production server suddenly becomes unreachable. How do you investigate?"*

```
1. Check monitoring/alerting dashboard
2. Try ping from multiple locations (rule out network issue)
3. Check cloud console — is the VM running?
4. Try alternate access: serial console, VNC, AWS SSM
5. If accessible via console:
   - Check: dmesg | tail -50 (kernel panics, OOM)
   - Check: systemctl status (failed services)
   - Check: netstat -tulnp | grep sshd (SSH running?)
   - Check: iptables -L (firewall blocking SSH?)
   - Check: df -h (disk full? — can prevent SSH login)
6. Review cloud security group / ACL rules
7. Check recent deployments or cron jobs
```

### 🔵 Scenario 2: Application OOM Kill

*"Your Java application keeps getting OOM killed. What do you do?"*

```bash
# 1. Confirm OOM kill
dmesg | grep -i "out of memory"
journalctl -xe | grep -i oom
grep "Killed process" /var/log/syslog

# 2. See what was killed and what triggered it
dmesg | grep "oom_kill_process" -A 20

# 3. Analyze memory usage trend
free -h; vmstat 1 10

# 4. For JVM: enable GC logging and heap dump
# Add to JVM args:
-Xmx2g -XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heapdump.hprof

# 5. Solutions:
# - Increase container/VM memory limits
# - Fix memory leak (analyze heap dump with MAT/VisualVM)
# - Add swap as buffer (not recommended for production)
# - Tune JVM heap size appropriately
```

### 🔵 Scenario 3: Disk Space Alert

*"You receive a disk space alert at 95% on /var. How do you respond?"*

```bash
# Step 1: Quantify
df -h /var

# Step 2: Find large consumers
du -sh /var/* | sort -hr | head -10
du -sh /var/log/* | sort -hr

# Step 3: Check for deleted-but-open files (common in logs)
lsof +L1 | grep /var    # +L1 = link count < 1 (deleted)

# Step 4: Quick wins
# - Rotate logs
logrotate -f /etc/logrotate.conf
# - Clear old journal logs
journalctl --vacuum-size=500M
# - Clean package cache
apt-get clean   # or yum clean all
# - Remove old Docker images
docker system prune -af

# Step 5: Long-term
# - Add log rotation policy
# - Move /var/log to separate partition
# - Set up monitoring with threshold alerts at 80%
```

---

## Hands-On Labs

### Lab 1: Process Investigation Script

```bash
#!/bin/bash
# Script to investigate high resource usage

echo "=== TOP 5 CPU CONSUMERS ==="
ps aux --sort=-%cpu | head -6

echo -e "\n=== TOP 5 MEMORY CONSUMERS ==="
ps aux --sort=-%mem | head -6

echo -e "\n=== DISK USAGE ==="
df -h | grep -v tmpfs

echo -e "\n=== DISK I/O ==="
iostat -x 1 3 | tail -15

echo -e "\n=== NETWORK CONNECTIONS ==="
ss -s
```

### Lab 2: User and Permissions Setup

```bash
# Create a web deployment user
sudo useradd -r -s /bin/false -d /var/www/myapp deploy_user
sudo mkdir -p /var/www/myapp
sudo chown deploy_user:deploy_user /var/www/myapp
sudo chmod 750 /var/www/myapp

# Allow deploy_user to restart nginx only
echo "deploy_user ALL=(ALL) NOPASSWD: /bin/systemctl restart nginx" \
  | sudo tee /etc/sudoers.d/deploy_user

# Verify
sudo -u deploy_user sudo systemctl restart nginx
```

### Lab 3: Log Analysis

```bash
# Parse nginx access log for top IPs and status codes
awk '{print $1}' /var/log/nginx/access.log | \
    sort | uniq -c | sort -rn | head -20

# Count HTTP status codes
awk '{print $9}' /var/log/nginx/access.log | \
    sort | uniq -c | sort -rn

# Find slowest requests (if response time logged)
awk '$NF > 1.0 {print $NF, $7}' /var/log/nginx/access.log | \
    sort -rn | head -10

# Live error monitoring
tail -f /var/log/nginx/error.log | grep -E "error|crit|alert|emerg"
```

---

## Command Reference

### Essential Commands Quick Table

| Category | Command | Purpose |
|----------|---------|---------|
| Files | `find / -name "*.log" -mtime -1` | Find recent logs |
| Files | `rsync -avz src/ dst/` | Sync directories |
| Process | `kill -l` | List all signals |
| Process | `pgrep -a nginx` | Find PID by name |
| Network | `ss -tulnp` | Active listeners |
| Network | `tcpdump -i any -w dump.pcap` | Capture traffic |
| Disk | `lsblk -f` | Filesystems on devices |
| Disk | `mount -t ext4 /dev/sdb1 /mnt` | Mount partition |
| System | `sysctl -a` | Kernel parameters |
| System | `ulimit -a` | User limits |
| Logs | `journalctl -u nginx -f` | Follow service logs |
| Logs | `tail -f /var/log/syslog` | Follow syslog |

### Sed & Awk Quick Reference

```bash
# sed — stream editor
sed -n '10,20p' file.txt          # Print lines 10-20
sed -i 's/foo/bar/g' file.txt     # In-place replace
sed '/^#/d' file.conf             # Delete comment lines
sed 's/[[:space:]]*$//' file.txt  # Remove trailing whitespace

# awk — pattern scanning
awk 'NR==5' file.txt              # Print line 5
awk '$3 > 100' file.txt           # Lines where field 3 > 100
awk '{sum+=$1} END{print sum}'    # Sum column 1
awk '!seen[$0]++'                 # Remove duplicate lines
```

---

> **Cross-links:** [Networking →](./Networking.md) | [Docker (uses Linux namespaces) →](./Docker.md) | [Security →](./Security.md) | [Monitoring →](./Monitoring.md)