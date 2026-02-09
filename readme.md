# Building Your Own Container Runtime from Scratch
## A Complete Hands-On Lab to Understand Docker Internals

**Author:** Your 30+ Year Linux Admin Guide  
**Target Audience:** Teachers, Students, Engineers who want to understand containers deeply  
**Hardware Required:** Ubuntu 22.04/24.04, 8GB RAM, 2 vCPU (you're good!)  
**Time Required:** 3-4 hours for complete walkthrough  

---

## Table of Contents

1. [Lab Overview](#lab-overview)
2. [Prerequisites & Setup](#prerequisites--setup)
3. [Part 1: Understanding Namespaces](#part-1-understanding-namespaces)
4. [Part 2: Resource Control with cgroups](#part-2-resource-control-with-cgroups)
5. [Part 3: Filesystem Isolation with chroot](#part-3-filesystem-isolation-with-chroot)
6. [Part 4: Network Isolation](#part-4-network-isolation)
7. [Part 5: Building a Container Runtime Script](#part-5-building-a-container-runtime-script)
8. [Part 6: Running Node.js Application](#part-6-running-nodejs-application)
9. [Part 7: Running Nginx in Another Container](#part-7-running-nginx-in-another-container)
10. [Part 8: Networking Containers Together](#part-8-networking-containers-together)
11. [Bonus: Comparison with Real Docker](#bonus-comparison-with-real-docker)

---

## Lab Overview

### What We'll Build

By the end of this lab, you will have built:

```
┌─────────────────────────────────────────────────────────┐
│                    HOST MACHINE (Ubuntu)                │
│                                                         │
│  ┌────────────────┐              ┌────────────────┐     │
│  │  Container 1   │              │  Container 2   │     │
│  │                │              │                │     │
│  │  Node.js App   │◄────────────►│  Nginx Proxy   │     │ 
│  │  Port: 3000    │   veth pair  │  Port: 80      │     │
│  │                │              │                │     │
│  │  • PID NS      │              │  • PID NS      │     │
│  │  • NET NS      │              │  • NET NS      │     │
│  │  • MNT NS      │              │  • MNT NS      │     │
│  │  • UTS NS      │              │  • UTS NS      │     │
│  │  • cgroups     │              │  • cgroups     │     │
│  └────────────────┘              └────────────────┘     │
│         │                                │              │
│         └────────┬───────────────────────┘              │
│                  │                                      │
│           ┌──────▼──────┐                               │
│           │  mybr0      │ (Our custom bridge)           │
│           │  10.0.0.1   │                               │
│           └─────────────┘                               │
│                  │                                      │
│           ┌──────▼──────┐                               │
│           │  iptables   │ (NAT rules)                   │
│           └─────────────┘                               │
│                  │                                      │
│              Internet                                   │
└─────────────────────────────────────────────────────────┘
```

### Learning Objectives

After completing this lab, you'll understand:

- ✅ How namespaces create process, network, and filesystem isolation
- ✅ How cgroups limit CPU, memory, and I/O resources
- ✅ How overlay filesystems work (we'll use simple chroot)
- ✅ How container networking works (bridges, veth pairs, iptables)
- ✅ How to create a minimal container runtime
- ✅ Why Docker is just automation around Linux primitives

---

## Prerequisites & Setup

### Step 1: Verify Your System

```bash
# Check Ubuntu version
lsb_release -a
# Should show Ubuntu 22.04 or 24.04

# Check kernel version (need 4.4+)
uname -r
# Should be 5.x or 6.x

# Check if you have necessary kernel features
ls -la /proc/self/ns/
# You should see: cgroup, ipc, mnt, net, pid, user, uts
```

### Step 2: Install Required Tools

```bash
# Update package list
sudo apt update

# Install essential tools
sudo apt install -y \
    debootstrap \
    bridge-utils \
    iproute2 \
    iptables \
    net-tools \
    curl \
    tree \
    htop \
    cgroup-tools \
    libcgroup-dev

# Verify installations
which unshare   # Should show /usr/bin/unshare
which nsenter   # Should show /usr/bin/nsenter
which cgcreate  # Should show /usr/sbin/cgcreate
which ip        # Should show /usr/sbin/ip
```

### Step 3: Create Working Directory

```bash
# Create our lab directory
mkdir -p ~/container-lab
cd ~/container-lab

# Create subdirectories for different components
mkdir -p {rootfs,containers,scripts,logs}

# Verify structure
tree -L 1
```

**Expected Output:**
```
.
├── containers  # Will hold container instances
├── logs        # Container logs
├── rootfs      # Base filesystem for containers
└── scripts     # Our container runtime scripts
```

---

## Part 1: Understanding Namespaces

### What Are Namespaces?

Namespaces are kernel features that partition kernel resources so that one set of processes sees one set of resources while another set of processes sees a different set.

### 1.1 PID Namespace (Process Isolation)

**Theory:** Each PID namespace has its own process tree. PID 1 inside the namespace is actually a different process on the host.

**Hands-On:**

```bash
# Terminal 1: Check current PIDs
ps aux | head -n 20
# Note your bash PID (let's say it's 1234)

# Create a new PID namespace
sudo unshare --pid --fork --mount-proc /bin/bash

# Inside the new namespace
ps aux
# You'll see ONLY processes in this namespace!
# The first process is PID 1

# Check parent namespace from inside
cat /proc/$$/status | grep NSpid
# Shows both: PID in namespace and on host

# Exit namespace
exit
```

**Teaching Point:** This is how containers can't see each other's processes! Each container lives in its own PID namespace.

### 1.2 Network Namespace (Network Isolation)

**Theory:** Each network namespace has its own network stack: interfaces, routing tables, firewall rules.

**Hands-On:**

```bash
# Check current network interfaces
ip addr show
# You'll see eth0, lo, etc.

# Create a new network namespace
sudo ip netns add demo-ns

# List network namespaces
ip netns list
# Shows: demo-ns

# Execute command inside the namespace
sudo ip netns exec demo-ns ip addr show
# ONLY shows 'lo' (loopback) - no other interfaces!

# Compare with host
ip addr show
# Shows all interfaces

# Cleanup
sudo ip netns del demo-ns
```

**Teaching Point:** This is why containers have their own IP addresses!

### 1.3 Mount Namespace (Filesystem Isolation)

**Theory:** Mount namespace isolates the set of filesystem mount points seen by a group of processes.

**Hands-On:**

```bash
# Create directories for demonstration
mkdir -p /tmp/demo-mount
mkdir -p /tmp/demo-mount/host-view
mkdir -p /tmp/demo-mount/container-view

# Create a new mount namespace
sudo unshare --mount /bin/bash

# Inside namespace: mount a tmpfs
mount -t tmpfs tmpfs /tmp/demo-mount/container-view
echo "I am inside container" > /tmp/demo-mount/container-view/test.txt

# Check from inside
ls /tmp/demo-mount/container-view/
cat /tmp/demo-mount/container-view/test.txt

# Open another terminal (Terminal 2)
# Check from host
ls /tmp/demo-mount/container-view/
# Directory exists but is EMPTY!

# Exit namespace (Terminal 1)
exit

# Check again from host (Terminal 2)
ls /tmp/demo-mount/container-view/
# Still empty - mount was isolated!
```

**Teaching Point:** Containers can mount filesystems without affecting the host!

### 1.4 UTS Namespace (Hostname Isolation)

**Theory:** UTS namespace isolates hostname and domain name.

**Hands-On:**

```bash
# Check current hostname
hostname
# Shows your actual hostname

# Create namespace with isolated hostname
sudo unshare --uts /bin/bash

# Inside namespace: change hostname
hostname my-container
hostname
# Shows: my-container

# Open Terminal 2 - check host
hostname
# Still shows original hostname!

# Exit namespace
exit
```

**Teaching Point:** This is why each container can have its own hostname!

### 1.5 Combining Multiple Namespaces (The Container Pattern)

**Now let's create something that FEELS like a container:**

```bash
# Create a namespace with PID, NET, MNT, UTS, and IPC isolation
sudo unshare --pid --net --mount --uts --ipc --fork /bin/bash

# Inside the "container":
hostname mycontainer
hostname
# Shows: mycontainer

ps aux
# Shows only processes in this namespace

ip addr show
# Shows only loopback (no network yet)

mount -t proc proc /proc
ps aux
# Now ps works properly!

# Exit
exit
```

**Teaching Point:** Docker combines ALL these namespaces to create isolation!

---

## Part 2: Resource Control with cgroups

### What Are cgroups?

Control Groups (cgroups) limit and account for resource usage of processes. While namespaces isolate WHAT you can SEE, cgroups limit WHAT you can USE.

### 2.1 Understanding the cgroup Filesystem

```bash
# Check if cgroup v2 is mounted
mount | grep cgroup
# On modern Ubuntu, you'll see cgroup2 on /sys/fs/cgroup

# Explore the cgroup hierarchy
ls -la /sys/fs/cgroup/

# Key controllers we'll use:
# - cpu.max        (CPU limits)
# - memory.max     (Memory limits)
# - pids.max       (Process count limits)
# - io.max         (I/O limits)
```

### 2.2 CPU Limiting (Hands-On)

**Create a CPU-intensive test script:**

```bash
cat > ~/container-lab/scripts/cpu-burn.sh << 'EOF'
#!/bin/bash
# Infinite loop to consume CPU
while true; do
    echo "Burning CPU..." > /dev/null
done
EOF

chmod +x ~/container-lab/scripts/cpu-burn.sh
```

**Limit CPU without cgroups (baseline):**

```bash
# Terminal 1: Run CPU burner
~/container-lab/scripts/cpu-burn.sh &
CPU_PID=$!

# Terminal 2: Watch CPU usage
top -p $CPU_PID
# Should show ~100% CPU usage

# Kill it
kill $CPU_PID
```

**Now with cgroup CPU limits:**

```bash
# Create a cgroup for our container
sudo mkdir -p /sys/fs/cgroup/mycontainer

# Limit CPU to 20% (20000 microseconds per 100000 microsecond period)
echo "20000 100000" | sudo tee /sys/fs/cgroup/mycontainer/cpu.max

# Verify
cat /sys/fs/cgroup/mycontainer/cpu.max
# Shows: 20000 100000

# Run CPU burner in this cgroup
~/container-lab/scripts/cpu-burn.sh &
CPU_PID=$!

# Add process to cgroup
echo $CPU_PID | sudo tee /sys/fs/cgroup/mycontainer/cgroup.procs

# Watch CPU usage in Terminal 2
top -p $CPU_PID
# Now shows ~20% CPU (cgroup limit enforced!)

# Cleanup
kill $CPU_PID
sudo rmdir /sys/fs/cgroup/mycontainer
```

**Teaching Point:** Docker's `--cpus=0.2` flag does EXACTLY this!

### 2.3 Memory Limiting (Hands-On)

**Create memory hog script:**

```bash
cat > ~/container-lab/scripts/mem-hog.sh << 'EOF'
#!/bin/bash
# Allocate 500MB of memory
python3 -c "
import time
# Allocate ~500MB
data = 'x' * (500 * 1024 * 1024)
print(f'Allocated {len(data) / 1024 / 1024:.0f} MB')
time.sleep(60)
"
EOF

chmod +x ~/container-lab/scripts/mem-hog.sh
```

**Limit memory:**

```bash
# Create cgroup
sudo mkdir -p /sys/fs/cgroup/memtest

# Set memory limit to 100MB
echo "104857600" | sudo tee /sys/fs/cgroup/memtest/memory.max
# 104857600 bytes = 100 MB

# Try to allocate 500MB (will fail!)
sudo cgexec -g memory:memtest ~/container-lab/scripts/mem-hog.sh
# Process will be killed by OOM (Out Of Memory) killer!

# Check events
cat /sys/fs/cgroup/memtest/memory.events | grep oom_kill
# Shows OOM kill count

# Cleanup
sudo rmdir /sys/fs/cgroup/memtest
```

**Teaching Point:** Docker's `--memory=100m` does this!

### 2.4 Creating a Complete cgroup Profile

**This is what Docker does for EVERY container:**

```bash
# Create our container cgroup
sudo mkdir -p /sys/fs/cgroup/container1

# CPU: 50% limit
echo "50000 100000" | sudo tee /sys/fs/cgroup/container1/cpu.max

# Memory: 512MB limit
echo "536870912" | sudo tee /sys/fs/cgroup/container1/memory.max

# PIDs: Maximum 100 processes
echo "100" | sudo tee /sys/fs/cgroup/container1/pids.max

# Verify all limits
echo "=== CPU Limit ==="
cat /sys/fs/cgroup/container1/cpu.max

echo "=== Memory Limit ==="
cat /sys/fs/cgroup/container1/memory.max

echo "=== PID Limit ==="
cat /sys/fs/cgroup/container1/pids.max

# We'll use this cgroup in Part 5!
```

---

## Part 3: Filesystem Isolation with chroot

### What Is chroot?

`chroot` changes the apparent root directory for a process. Docker uses overlay filesystems, but for simplicity, we'll use chroot to demonstrate the concept.

### 3.1 Creating a Minimal Root Filesystem

**We'll use debootstrap to create a minimal Ubuntu filesystem:**

```bash
cd ~/container-lab

# Create Ubuntu 22.04 root filesystem (takes ~5 minutes)
sudo debootstrap --variant=minbase jammy rootfs/ubuntu-base http://archive.ubuntu.com/ubuntu/

# This downloads and installs a minimal Ubuntu system
# Size: ~300MB

# Verify the filesystem
ls rootfs/ubuntu-base/
# You should see: bin, boot, dev, etc, home, lib, media, mnt, opt, proc, root, run, sbin, srv, sys, tmp, usr, var

# Check size
du -sh rootfs/ubuntu-base/
```

### 3.2 Testing chroot

```bash
# Enter the chroot environment
sudo chroot rootfs/ubuntu-base /bin/bash

# Inside chroot:
pwd
# Shows: /

ls /
# Shows the minimal filesystem

# This is isolated!
ps aux
# ERROR: /proc is not mounted

# Exit
exit
```

### 3.3 Setting Up a Proper Chroot Environment

**We need to mount /proc, /sys, and /dev:**

```bash
# Create mount points
sudo mkdir -p rootfs/ubuntu-base/{proc,sys,dev,dev/pts}

# Mount special filesystems
sudo mount -t proc proc rootfs/ubuntu-base/proc
sudo mount -t sysfs sys rootfs/ubuntu-base/sys
sudo mount --bind /dev rootfs/ubuntu-base/dev
sudo mount --bind /dev/pts rootfs/ubuntu-base/dev/pts

# Enter chroot again
sudo chroot rootfs/ubuntu-base /bin/bash

# Now ps works!
ps aux

# Network works (using host's network namespace for now)
ping -c 2 8.8.8.8

# Exit
exit

# Cleanup mounts
sudo umount rootfs/ubuntu-base/dev/pts
sudo umount rootfs/ubuntu-base/dev
sudo umount rootfs/ubuntu-base/sys
sudo umount rootfs/ubuntu-base/proc
```

**Teaching Point:** Docker does this automatically with overlay2 and proper namespace mounting!

---

## Part 4: Network Isolation

### 4.1 Creating a Custom Bridge (Like docker0)

```bash
# Create our bridge interface
sudo ip link add name mybr0 type bridge

# Assign IP to bridge
sudo ip addr add 10.0.0.1/24 dev mybr0

# Bring bridge up
sudo ip link set mybr0 up

# Verify
ip addr show mybr0
# Shows: mybr0 with IP 10.0.0.1
```

### 4.2 Creating veth Pairs (Virtual Ethernet)

**veth pairs are like virtual network cables - one end in container, one in host:**

```bash
# Create network namespace for our container
sudo ip netns add container1

# Create veth pair (veth1-host <---> veth1-container)
sudo ip link add veth1-host type veth peer name veth1-cont

# Move container end into namespace
sudo ip link set veth1-cont netns container1

# Attach host end to bridge
sudo ip link set veth1-host master mybr0

# Bring up host end
sudo ip link set veth1-host up

# Configure inside namespace
sudo ip netns exec container1 ip addr add 10.0.0.2/24 dev veth1-cont
sudo ip netns exec container1 ip link set veth1-cont up
sudo ip netns exec container1 ip link set lo up

# Set default route inside container
sudo ip netns exec container1 ip route add default via 10.0.0.1

# Test connectivity
sudo ip netns exec container1 ping -c 2 10.0.0.1
# Should work! Container can reach bridge!
```

### 4.3 Enable NAT for Internet Access

```bash
# Enable IP forwarding on host
sudo sysctl -w net.ipv4.ip_forward=1

# Add iptables SNAT rule (like Docker's MASQUERADE)
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 ! -o mybr0 -j MASQUERADE

# Test internet from container
sudo ip netns exec container1 ping -c 2 8.8.8.8
# Should work! Container can reach internet!
```

### 4.4 Port Forwarding (Like docker -p 8080:80)

```bash
# Forward host port 8080 to container 10.0.0.2:80
sudo iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 10.0.0.2:80

# Allow forwarding
sudo iptables -A FORWARD -p tcp -d 10.0.0.2 --dport 80 -j ACCEPT
```

**Teaching Point:** This is EXACTLY how Docker's `-p` flag works!

---

## Part 5: Building a Container Runtime Script

Now let's put it ALL together into a reusable script!

```bash
cat > ~/container-lab/scripts/mycontainer.sh << 'EOF'
#!/bin/bash
set -e

# Configuration
CONTAINER_NAME="$1"
CONTAINER_IP="$2"
COMMAND="${3:-/bin/bash}"

if [ -z "$CONTAINER_NAME" ] || [ -z "$CONTAINER_IP" ]; then
    echo "Usage: $0 <container-name> <container-ip> [command]"
    echo "Example: $0 web1 10.0.0.2 /bin/bash"
    exit 1
fi

# Base paths
LAB_DIR="$HOME/container-lab"
ROOTFS="$LAB_DIR/rootfs/ubuntu-base"
CONTAINER_DIR="$LAB_DIR/containers/$CONTAINER_NAME"

echo "[*] Creating container: $CONTAINER_NAME with IP $CONTAINER_IP"

# Step 1: Create cgroup for resource limits
echo "[*] Setting up cgroups..."
sudo mkdir -p /sys/fs/cgroup/$CONTAINER_NAME
echo "50000 100000" | sudo tee /sys/fs/cgroup/$CONTAINER_NAME/cpu.max > /dev/null
echo "536870912" | sudo tee /sys/fs/cgroup/$CONTAINER_NAME/memory.max > /dev/null
echo "100" | sudo tee /sys/fs/cgroup/$CONTAINER_NAME/pids.max > /dev/null

# Step 2: Create container-specific rootfs
echo "[*] Preparing filesystem..."
mkdir -p $CONTAINER_DIR/rootfs
sudo mount --bind $ROOTFS $CONTAINER_DIR/rootfs

# Mount special filesystems
sudo mount -t proc proc $CONTAINER_DIR/rootfs/proc 2>/dev/null || true
sudo mount -t sysfs sys $CONTAINER_DIR/rootfs/sys 2>/dev/null || true
sudo mount --bind /dev $CONTAINER_DIR/rootfs/dev 2>/dev/null || true

# Step 3: Create network namespace
echo "[*] Setting up networking..."
sudo ip netns add $CONTAINER_NAME

# Create veth pair
VETH_HOST="veth-$CONTAINER_NAME"
VETH_CONT="veth-$CONTAINER_NAME-c"

sudo ip link add $VETH_HOST type veth peer name $VETH_CONT
sudo ip link set $VETH_CONT netns $CONTAINER_NAME
sudo ip link set $VETH_HOST master mybr0
sudo ip link set $VETH_HOST up

# Configure network inside namespace
sudo ip netns exec $CONTAINER_NAME ip addr add $CONTAINER_IP/24 dev $VETH_CONT
sudo ip netns exec $CONTAINER_NAME ip link set $VETH_CONT up
sudo ip netns exec $CONTAINER_NAME ip link set lo up
sudo ip netns exec $CONTAINER_NAME ip route add default via 10.0.0.1

# Step 4: Launch container with all namespaces
echo "[*] Starting container process..."

# Add current shell to cgroup
echo $$ | sudo tee /sys/fs/cgroup/$CONTAINER_NAME/cgroup.procs > /dev/null

# Execute in container with full isolation
sudo ip netns exec $CONTAINER_NAME \
    unshare --pid --mount --uts --ipc --fork \
    chroot $CONTAINER_DIR/rootfs \
    /bin/bash -c "
        hostname $CONTAINER_NAME
        export PS1='[$CONTAINER_NAME] \w# '
        $COMMAND
    "

echo "[*] Container $CONTAINER_NAME stopped"

# Cleanup
echo "[*] Cleaning up..."
sudo umount $CONTAINER_DIR/rootfs/dev 2>/dev/null || true
sudo umount $CONTAINER_DIR/rootfs/sys 2>/dev/null || true
sudo umount $CONTAINER_DIR/rootfs/proc 2>/dev/null || true
sudo umount $CONTAINER_DIR/rootfs 2>/dev/null || true
sudo ip netns del $CONTAINER_NAME 2>/dev/null || true
sudo rmdir /sys/fs/cgroup/$CONTAINER_NAME 2>/dev/null || true
rm -rf $CONTAINER_DIR

echo "[*] Done!"
EOF

chmod +x ~/container-lab/scripts/mycontainer.sh
```

**Test the script:**

```bash
cd ~/container-lab
sudo ./scripts/mycontainer.sh testcontainer 10.0.0.2

# Inside container:
# Check hostname
hostname
# Shows: testcontainer

# Check IP
ip addr show
# Shows: 10.0.0.2

# Check processes
ps aux
# Shows isolated process tree

# Test internet
ping -c 2 8.8.8.8
# Works!

# Exit
exit
```

**BOOM! You just created a container from scratch! 🎉**

---

## Part 6: Running Node.js Application

### 6.1 Install Node.js in Our Base Image

```bash
# Enter the base rootfs
sudo chroot ~/container-lab/rootfs/ubuntu-base /bin/bash

# Inside chroot:
apt update
apt install -y curl
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs

# Verify
node --version
npm --version

exit
```

### 6.2 Create a Simple Node.js App

```bash
# Create app directory in base rootfs
sudo mkdir -p ~/container-lab/rootfs/ubuntu-base/app

# Create Node.js app
sudo tee ~/container-lab/rootfs/ubuntu-base/app/server.js > /dev/null << 'EOF'
const http = require('http');
const os = require('os');

const PORT = 3000;

const server = http.createServer((req, res) => {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
        message: 'Hello from custom container!',
        hostname: os.hostname(),
        pid: process.pid,
        uptime: process.uptime(),
        timestamp: new Date().toISOString()
    }, null, 2));
});

server.listen(PORT, '0.0.0.0', () => {
    console.log(`Server running on port ${PORT}`);
    console.log(`Hostname: ${os.hostname()}`);
    console.log(`PID: ${process.pid}`);
});
EOF
```

### 6.3 Run Node.js Container

**Create a wrapper script:**

```bash
cat > ~/container-lab/scripts/run-nodejs.sh << 'EOF'
#!/bin/bash
cd ~/container-lab
sudo ./scripts/mycontainer.sh nodejs-app 10.0.0.2 "node /app/server.js"
EOF

chmod +x ~/container-lab/scripts/run-nodejs.sh
```

**Run it:**

```bash
# Terminal 1: Start container
~/container-lab/scripts/run-nodejs.sh

# You'll see:
# Server running on port 3000
# Hostname: nodejs-app
# PID: 1

# Terminal 2: Test from host
curl http://10.0.0.2:3000

# Should see JSON response!
```

---

## Part 7: Running Nginx in Another Container

### 7.1 Install Nginx in Base Image

```bash
sudo chroot ~/container-lab/rootfs/ubuntu-base /bin/bash

# Inside:
apt update
apt install -y nginx

# Verify
which nginx

exit
```

### 7.2 Configure Nginx as Reverse Proxy

```bash
# Create nginx config
sudo tee ~/container-lab/rootfs/ubuntu-base/etc/nginx/sites-available/default > /dev/null << 'EOF'
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;

    location / {
        proxy_pass http://10.0.0.2:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /health {
        return 200 "Nginx is healthy\n";
        add_header Content-Type text/plain;
    }
}
EOF
```

### 7.3 Run Nginx Container

```bash
cat > ~/container-lab/scripts/run-nginx.sh << 'EOF'
#!/bin/bash
cd ~/container-lab
sudo ./scripts/mycontainer.sh nginx-proxy 10.0.0.3 "/usr/sbin/nginx -g 'daemon off;'"
EOF

chmod +x ~/container-lab/scripts/run-nginx.sh
```

---

## Part 8: Networking Containers Together

### 8.1 Full Architecture

Now let's run BOTH containers and test the full flow:

```
Internet → Host:8080 → iptables DNAT → 10.0.0.3:80 (nginx) → 10.0.0.2:3000 (node)
```

### 8.2 Setup Port Forwarding

```bash
# Forward host port 8080 to nginx container
sudo iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 10.0.0.3:80
sudo iptables -A FORWARD -p tcp -d 10.0.0.3 --dport 80 -j ACCEPT
```

### 8.3 Run Complete Stack

```bash
# Terminal 1: Run Node.js container
~/container-lab/scripts/run-nodejs.sh

# Terminal 2: Run Nginx container
~/container-lab/scripts/run-nginx.sh

# Terminal 3: Test the flow
# Test Node.js directly
curl http://10.0.0.2:3000

# Test Nginx directly
curl http://10.0.0.3/health

# Test through Nginx proxy
curl http://10.0.0.3/

# Test from internet (via iptables NAT)
curl http://localhost:8080/
```

**YOU DID IT! Full container orchestration from scratch! 🔥**

---

## Bonus: Comparison with Real Docker

### What We Built vs Docker

| Feature | Our Implementation | Docker Implementation |
|---------|-------------------|----------------------|
| **Isolation** | `unshare` (namespaces) | `runc` (namespaces) |
| **Resources** | cgroups v2 manually | cgroups via containerd |
| **Filesystem** | chroot + bind mount | overlay2 (layered) |
| **Networking** | veth + bridge manually | docker0 bridge + veth |
| **NAT** | iptables manually | iptables via dockerd |
| **Image** | debootstrap rootfs | OCI image layers |
| **Runtime** | Bash script | dockerd → containerd → runc |

### Key Differences

1. **Layered Filesystem:** Docker uses overlay2 for copy-on-write. We used simple bind mount.
2. **Image Distribution:** Docker has registries and pull mechanism. We built locally.
3. **Orchestration:** Docker has docker-compose, Swarm. We managed manually.
4. **API:** Docker has REST API. We used shell scripts.

### Advantages of Docker

- **Image caching:** Layers are shared across containers
- **Distribution:** Push/pull images from registries
- **Orchestration:** Built-in networking, volumes, secrets
- **Ecosystem:** Massive community and tooling

### What We Learned

**Docker is NOT magic!** It's just:
- Namespaces (for isolation)
- cgroups (for resources)
- Filesystem layers (for efficiency)
- Network bridges (for connectivity)
- iptables (for NAT/routing)

---

## Summary & Teaching Points

### For Your Students

**Key Concepts to Emphasize:**

1. **Containers are NOT VMs**
   - They share the kernel
   - No boot process
   - Just isolated processes

2. **The 7 Namespaces**
   - PID: Process isolation
   - NET: Network isolation
   - MNT: Filesystem isolation
   - UTS: Hostname isolation
   - IPC: Inter-process communication isolation
   - USER: User ID isolation
   - CGROUP: Resource view isolation

3. **cgroups = Resource Limits**
   - CPU throttling
   - Memory limits
   - I/O limits
   - Process count limits

4. **Container Networking**
   - Bridge = virtual switch
   - veth = virtual cable
   - iptables = router/firewall

### Common Student Questions & Answers

**Q: Why are containers faster than VMs?**
A: No kernel boot! Just process creation (~100ms vs 30-60s for VMs).

**Q: Are containers less secure than VMs?**
A: Yes, shared kernel = larger attack surface. But with proper hardening (namespaces, cgroups, seccomp, AppArmor), they're secure enough for most use cases.

**Q: Can I run Windows containers on Linux?**
A: No! Containers share the kernel. Windows containers need Windows kernel.

**Q: What happens if I delete a running container?**
A: All processes in its PID namespace are killed. All data in its filesystem is lost (unless in volumes).

---

## Cleanup

When you're done with the lab:

```bash
# Remove all iptables rules
sudo iptables -t nat -F
sudo iptables -F

# Delete bridge
sudo ip link del mybr0

# Remove rootfs (optional - takes time to rebuild)
# sudo rm -rf ~/container-lab/rootfs

# Remove containers directory
sudo rm -rf ~/container-lab/containers

echo "Lab cleanup complete!"
```

---

## Next Steps

### Advanced Topics to Explore

1. **Overlay Filesystems:** Replace chroot with overlay2
2. **Image Layers:** Build multi-layer images
3. **Container Registry:** Build simple image push/pull
4. **Init System:** Add proper PID 1 (tini or similar)
5. **Security:** Add seccomp, AppArmor profiles
6. **Service Mesh:** Multiple containers with service discovery

### Recommended Reading

- Linux Kernel Documentation: namespaces, cgroups
- OCI Runtime Specification
- containerd architecture docs
- runc source code (Go, but very readable!)

---

**Congratulations!** You've built a container runtime from scratch using nothing but Linux primitives. Now when you use Docker, you understand EXACTLY what's happening under the hood.

**This knowledge makes you dangerous.** 😎

---

**Created by:** A Linux Admin with 30+ years battle scars  
**License:** Do whatever you want - teach freely!  
**Feedback:** May your containers be lightweight and your kernels panic-free! 🐧