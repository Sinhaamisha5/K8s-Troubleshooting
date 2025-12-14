# Kubernetes Networking - Complete Reference Guide

## Table of Contents
1. [Pod Networking Fundamentals](#pod-networking-fundamentals)
2. [Cluster Networking Requirements](#cluster-networking-requirements)
3. [Container Network Interface (CNI)](#container-network-interface-cni)
4. [CNI Plugins Deep Dive](#cni-plugins-deep-dive)
5. [IP Address Management (IPAM)](#ip-address-management-ipam)
6. [Troubleshooting Commands](#troubleshooting-commands)

---

## Pod Networking Fundamentals

### Core Requirements
Kubernetes pod networking must satisfy three fundamental requirements:

1. **Unique IP per Pod**: Each pod gets its own IP address
2. **Node-local Communication**: Pods on the same node can reach each other using IPs
3. **Cross-node Communication**: Pods across different nodes communicate without NAT

### How It Works (Step by Step)

#### Single Node Setup
```bash
# 1. Create bridge network on the node
ip link add v-net-0 type bridge
ip link set dev v-net-0 up
ip addr add 10.244.1.1/24 dev v-net-0

# 2. When a pod is created, create veth pair
ip link add veth-pod1 type veth peer name veth-pod1-br

# 3. Attach one end to pod's namespace
ip link set veth-pod1 netns <pod-namespace>

# 4. Attach other end to bridge
ip link set veth-pod1-br master v-net-0

# 5. Assign IP to pod interface
ip -n <pod-namespace> addr add 10.244.1.2/24 dev veth-pod1

# 6. Add default route in pod
ip -n <pod-namespace> route add default via 10.244.1.1

# 7. Bring up interface
ip -n <pod-namespace> link set veth-pod1 up
```

#### Multi-Node Architecture
```
Node 1 (192.168.1.11)
├── Bridge: 10.244.1.0/24
├── Pod A: 10.244.1.2
└── Pod B: 10.244.1.3

Node 2 (192.168.1.12)
├── Bridge: 10.244.2.0/24
├── Pod C: 10.244.2.2
└── Pod D: 10.244.2.3

Node 3 (192.168.1.13)
├── Bridge: 10.244.3.0/24
├── Pod E: 10.244.3.2
└── Pod F: 10.244.3.3
```

### Cross-Node Communication Setup

**Problem**: Pod A (10.244.1.2) on Node 1 cannot reach Pod C (10.244.2.2) on Node 2

**Solution**: Add routes on each node

```bash
# On Node 1 - tell it how to reach other subnets
ip route add 10.244.2.0/24 via 192.168.1.12  # Node 2's subnet via Node 2 IP
ip route add 10.244.3.0/24 via 192.168.1.13  # Node 3's subnet via Node 3 IP

# On Node 2
ip route add 10.244.1.0/24 via 192.168.1.11  # Node 1's subnet
ip route add 10.244.3.0/24 via 192.168.1.13  # Node 3's subnet

# On Node 3
ip route add 10.244.1.0/24 via 192.168.1.11  # Node 1's subnet
ip route add 10.244.2.0/24 via 192.168.1.12  # Node 2's subnet
```

**Better Solution**: Use a router to centrally manage routes (this is what CNI plugins do!)

### Traffic Flow Example

**Scenario**: Pod A (10.244.1.2 on Node 1) pings Pod C (10.244.2.2 on Node 2)

```
Pod A (10.244.1.2)
  → veth pair
  → Bridge (10.244.1.1) on Node 1
  → Node 1 routing table sees: "10.244.2.0/24 via 192.168.1.12"
  → Packet sent to Node 2 (192.168.1.12) via eth0
  → Node 2 receives packet on eth0
  → Node 2 routing sees: "10.244.2.0/24 is local"
  → Bridge (10.244.2.1) on Node 2
  → veth pair
  → Pod C (10.244.2.2)
```

---

## Cluster Networking Requirements

### Required Open Ports

| Port(s) | Component | Node Type | Purpose |
|---------|-----------|-----------|---------|
| 6443 | kube-apiserver | Master | API access for all components |
| 10250 | kubelet | Master & Worker | Node management, pod logs/exec |
| 10259 | kube-scheduler | Master | Scheduling operations |
| 10257 | kube-controller-manager | Master | Controller operations |
| 2379 | etcd | Master | Client communication |
| 2380 | etcd | Master | Server-to-server (multi-master) |
| 30000-32767 | NodePort Services | Worker | External service access |

### Network Verification Commands

```bash
# View network interfaces
ip link

# View IP addresses
ip addr

# Add IP to interface
ip addr add 192.168.1.10/24 dev eth0

# View routing table
ip route

# Add route
ip route add 10.244.0.0/16 via 192.168.1.1

# Check IP forwarding (must be 1)
cat /proc/sys/net/ipv4/ip_forward
# Enable if needed:
echo 1 > /proc/sys/net/ipv4/ip_forward

# View ARP table
arp

# View listening ports and services
netstat -plnt
```

### Basic Node Requirements

Each node must have:
- ✅ At least one network interface with IP address
- ✅ Unique hostname
- ✅ Unique MAC address
- ✅ IP forwarding enabled
- ✅ Required ports open in firewall

---

## Container Network Interface (CNI)

### What is CNI?

CNI is a **standard specification** that defines:
- How container runtimes should configure networking
- What network plugins must implement
- How plugins are invoked and configured

**Analogy**: CNI is like a universal USB standard - any CNI-compliant plugin works with any CNI-compliant runtime.

### CNI Standards Define:

1. **Container Runtime Responsibilities**:
   - Create network namespace for container
   - Identify which network container should join
   - Invoke CNI plugin when container is created (ADD)
   - Invoke CNI plugin when container is deleted (DEL)

2. **Plugin Responsibilities**:
   - Support ADD and DEL operations
   - Assign IP address to container
   - Set up routes within container
   - Connect container to network

### CNI Directory Structure

```bash
# Plugin executables location
/opt/cni/bin/
├── bridge          # Bridge plugin
├── dhcp            # DHCP plugin
├── flannel         # Flannel plugin
├── host-local      # Host-local IPAM
├── weave-net       # Weave plugin
├── calico          # Calico plugin
└── ...

# Configuration files location
/etc/cni/net.d/
├── 10-bridge.conf      # Bridge config
├── 10-flannel.conflist # Flannel config
└── 10-calico.conflist  # Calico config
```

**Note**: Runtime uses the **first file alphabetically** (10- prefix controls ordering)

### CNI Configuration File Example

```json
{
  "cniVersion": "0.2.0",
  "name": "mynet",
  "type": "bridge",
  "bridge": "cni0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "subnet": "10.244.0.0/16",
    "routes": [
      { "dst": "0.0.0.0/0" }
    ]
  }
}
```

**Configuration Fields**:
- `cniVersion`: CNI specification version
- `name`: Network name
- `type`: Plugin to use (bridge, flannel, calico, etc.)
- `bridge`: Bridge interface name
- `isGateway`: Bridge acts as gateway (gets IP)
- `ipMasq`: Enable NAT/masquerading
- `ipam`: IP Address Management configuration
  - `type`: IPAM plugin (host-local, dhcp, etc.)
  - `subnet`: IP range for pods
  - `routes`: Default routing rules

### How Kubernetes Uses CNI

#### When a Pod is Created:

1. **Kubelet** receives pod creation request
2. **Container Runtime** (containerd/CRI-O) creates container
3. **Runtime creates network namespace** for container
4. **Runtime reads CNI config** from `/etc/cni/net.d/`
5. **Runtime executes CNI plugin** from `/opt/cni/bin/`
   ```bash
   /opt/cni/bin/bridge add <container-id> <namespace>
   ```
6. **Plugin performs**:
   - Creates veth pair
   - Attaches to bridge
   - Assigns IP via IPAM
   - Configures routes
   - Brings up interface

#### When a Pod is Deleted:

```bash
/opt/cni/bin/bridge del <container-id> <namespace>
```

Plugin performs cleanup:
- Deletes veth pair
- Frees IP address
- Removes routes

---

## CNI Plugins Deep Dive

### 1. Flannel CNI

**Type**: Overlay network (encapsulation-based)

**How it works**:
- Creates a **flat network** across all nodes
- Uses **VXLAN** or **host-gw** for packet encapsulation
- Each node gets a subnet from a larger cluster CIDR

**Architecture**:
```
Node 1 (10.244.1.0/24)
  ↓ flannel.1 interface (VXLAN tunnel)
Node 2 (10.244.2.0/24)
  ↓ flannel.1 interface (VXLAN tunnel)
Node 3 (10.244.3.0/24)
```

**Pros**:
- ✅ Simple to set up
- ✅ Works in most environments
- ✅ Good for learning/testing

**Cons**:
- ❌ No NetworkPolicy support (no firewall rules)
- ❌ Encapsulation overhead
- ❌ Limited features

**Deploy Flannel**:
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

---

### 2. Weave Net CNI

**Type**: Overlay network with mesh architecture

**How it works**:
- Deploys **Weave agent** (pod) on every node via DaemonSet
- Agents form a **peer-to-peer mesh network**
- Automatic topology discovery
- Encapsulates packets between nodes

**Architecture**:
```
        Weave Agent (Node 1)
       /                    \
Weave Agent (Node 2) ←→ Weave Agent (Node 3)
```

Each agent knows about:
- All other nodes in cluster
- All pod IPs and locations
- Optimal routes between nodes

**How Packet Flow Works**:
```
Pod A (10.32.0.2 on Node 1) → Pod B (10.32.1.5 on Node 2)

1. Pod A sends packet to 10.32.1.5
2. Weave agent on Node 1 intercepts packet
3. Agent encapsulates packet:
   - Original: src=10.32.0.2, dst=10.32.1.5
   - Encapsulated: src=Node1-IP, dst=Node2-IP (inner packet preserved)
4. Packet sent over physical network to Node 2
5. Weave agent on Node 2 decapsulates packet
6. Original packet delivered to Pod B
```

**IP Range**:
- Default: **10.32.0.0/12** (approximately 1 million IPs)
- Range: 10.32.0.1 to 10.47.255.254
- Automatically divided among nodes

**Pros**:
- ✅ NetworkPolicy support (firewall rules work!)
- ✅ Simple deployment
- ✅ Automatic network discovery
- ✅ Encryption support

**Cons**:
- ❌ Encapsulation overhead
- ❌ More resource usage than Flannel

**Deploy Weave**:
```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

**Verify Weave**:
```bash
# Check Weave pods
kubectl get pods -n kube-system | grep weave

# Check pod route (should go through Weave)
kubectl exec <pod-name> -- ip route
# Output shows: default via 10.32.x.1 dev eth0
```

---

### 3. Calico CNI

**Type**: Layer 3 network (BGP-based, no encapsulation)

**How it works**:
- Uses **BGP** (Border Gateway Protocol) for routing
- **No packet encapsulation** - native IP routing
- **iptables or eBPF** for NetworkPolicy enforcement
- Each node is a virtual router

**Architecture**:
```
Node 1 (10.244.1.0/24)
  ↓ BGP announces routes
Router (BGP speaker)
  ↓ BGP propagates routes
Node 2 (10.244.2.0/24)
```

**Key Components**:
- **Calico Node**: DaemonSet on each node (manages routing)
- **Felix**: Configures routes and iptables
- **BIRD**: BGP client (exchanges routes)
- **Calico CNI Plugin**: Invoked by container runtime

**Pros**:
- ✅ **NetworkPolicy support** (advanced firewall rules)
- ✅ No encapsulation overhead (better performance)
- ✅ Native IP routing
- ✅ Scales to large clusters
- ✅ eBPF support (best performance)

**Cons**:
- ❌ More complex setup
- ❌ May require BGP knowledge for troubleshooting

**Deploy Calico**:
```bash
# Install operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml

# Download custom resources
curl https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml -o custom-resources.yaml

# Edit CIDR (change to your cluster's pod CIDR)
vi custom-resources.yaml
# Change: cidr: 172.17.0.0/16

# Apply
kubectl create -f custom-resources.yaml
```

---

### CNI Comparison Table

| Feature | Flannel | Weave | Calico |
|---------|---------|-------|--------|
| **NetworkPolicy** | ❌ No | ✅ Yes | ✅ Yes |
| **Encapsulation** | VXLAN | VXLAN | None (BGP) |
| **Performance** | Good | Good | Best |
| **Complexity** | Low | Medium | High |
| **Encryption** | No | Yes | Yes |
| **IP Range** | Configurable | 10.32.0.0/12 | Configurable |
| **Best For** | Simple/Learning | Medium clusters | Production/Security |

---

## IP Address Management (IPAM)

### How IPs are Assigned

#### Manual IPAM (Conceptual)
```bash
# Simple file-based approach
# Store assigned IPs in /tmp/ips.txt

# When pod created:
ip=$(get_free_ip_from_file)
ip -n <namespace> addr add $ip/24 dev eth0
ip -n <namespace> route add default via <gateway>

# When pod deleted:
free_ip_to_file $ip
```

**Problems with manual approach**:
- ❌ Race conditions (two pods get same IP)
- ❌ Not scalable
- ❌ No coordination across nodes

#### CNI IPAM Plugins

**host-local Plugin** (most common):
- Maintains IP allocation state **locally on each node**
- Stores data in `/var/lib/cni/networks/<network-name>/`
- Each node independently assigns IPs from its subnet

**DHCP Plugin**:
- Uses external DHCP server
- Less common in Kubernetes

**Configuration Example**:
```json
{
  "ipam": {
    "type": "host-local",
    "subnet": "10.244.0.0/16",
    "rangeStart": "10.244.1.2",
    "rangeEnd": "10.244.1.254",
    "gateway": "10.244.1.1",
    "routes": [
      { "dst": "0.0.0.0/0" }
    ]
  }
}
```

### IP Ranges by CNI

#### Flannel
```
Cluster CIDR: 10.244.0.0/16
├── Node 1: 10.244.1.0/24 (254 pods)
├── Node 2: 10.244.2.0/24 (254 pods)
└── Node 3: 10.244.3.0/24 (254 pods)
```

#### Weave
```
Default Range: 10.32.0.0/12
├── Approximately 1 million IPs total
├── Automatically divided among nodes
└── Configurable during deployment
```

#### Calico
```
Cluster CIDR: User-defined (e.g., 172.17.0.0/16)
├── Node 1: 172.17.1.0/26 (62 pods)
├── Node 2: 172.17.1.64/26 (62 pods)
└── Dynamic block allocation
```

### Checking Pod IPs

```bash
# Get pod IP
kubectl get pod <pod-name> -o wide

# Check IP from inside pod
kubectl exec <pod-name> -- ip addr show eth0

# View IPAM state (on node)
cat /var/lib/cni/networks/<network-name>/last_reserved_ip
ls /var/lib/cni/networks/<network-name>/
```

---

## Troubleshooting Commands

### CNI Configuration

```bash
# Check which CNI is being used
ls /etc/cni/net.d/

# View CNI config
cat /etc/cni/net.d/10-<plugin>.conf

# Check CNI binaries
ls /opt/cni/bin/

# View kubelet CNI config
ps aux | grep kubelet | grep cni
```

### Network Verification

```bash
# Check node networking
ip addr
ip route
ip link

# Check bridge interfaces
ip link show type bridge
brctl show  # or: bridge link show

# Check for Flannel interface
ip link show flannel.1

# Check for Weave interface
ip link show weave

# View iptables rules (for NetworkPolicies)
iptables -L -n -v
iptables -t nat -L -n -v
```

### Pod Networking

```bash
# Get pod details including IP
kubectl get pods -o wide

# Check pod network namespace (on node)
# First find container ID
crictl ps | grep <pod-name>

# Find network namespace
crictl inspect <container-id> | grep networkNamespace

# Inspect namespace
ip netns exec <namespace> ip addr
ip netns exec <namespace> ip route

# Test connectivity from pod
kubectl exec <pod-name> -- ping <target-ip>
kubectl exec <pod-name> -- curl <target-service>
kubectl exec <pod-name> -- nslookup kubernetes.default
```

### CNI Plugin Logs

```bash
# Flannel logs
kubectl logs -n kube-flannel <flannel-pod>

# Weave logs
kubectl logs -n kube-system <weave-pod>

# Calico logs
kubectl logs -n calico-system <calico-node-pod>
kubectl logs -n calico-system <calico-kube-controllers-pod>

# Check CNI plugin execution logs (on node)
journalctl -u kubelet | grep CNI
```

### Common Issues

#### Issue: Pods not getting IPs
```bash
# Check CNI config exists
ls /etc/cni/net.d/

# Check CNI plugin exists
ls /opt/cni/bin/

# Check kubelet logs
journalctl -u kubelet -f

# Verify IP pool not exhausted
kubectl get ippool -o yaml  # For Calico
```

#### Issue: Pods can't communicate
```bash
# Check routes on node
ip route

# Check if IP forwarding enabled
cat /proc/sys/net/ipv4/ip_forward  # Should be 1

# Test from pod
kubectl exec <pod> -- ping <target-pod-ip>

# Check NetworkPolicies
kubectl get networkpolicy -A
```

#### Issue: NetworkPolicies not working
```bash
# Verify CNI supports NetworkPolicy
kubectl get pods -n kube-system | grep -E 'calico|weave'

# Flannel does NOT support NetworkPolicies!

# Check iptables rules are created
iptables -L -n | grep <pod-ip>
```

---

## Quick Reference

### CNI Plugin Files Locations
```
/opt/cni/bin/          → Plugin binaries
/etc/cni/net.d/        → Plugin configs
/var/lib/cni/          → IPAM state
```

### Key Commands
```bash
# View CNI config
cat /etc/cni/net.d/*.conf

# Check IP ranges
kubectl get ippool -o yaml  # Calico
kubectl describe ds weave-net -n kube-system  # Weave

# Test pod networking
kubectl run test --image=busybox --command -- sleep 3600
kubectl exec test -- ping <ip>

# View routes
ip route
kubectl exec <pod> -- ip route
```

### Network Policy Test
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-policy
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
```

---

---

## Service Networking

### What are Services?

Services are **virtual constructs** (not actual network interfaces like pods) that provide:
- Stable IP address for accessing pods
- DNS name for service discovery
- Load balancing across multiple pod replicas
- Abstraction layer between clients and pods

**Key Difference from Pods**:
- **Pods**: Have actual network interfaces, namespaces, veth pairs
- **Services**: Virtual IPs managed by kube-proxy using forwarding rules

### Why Services?

**Problem**: Pods are ephemeral (can be created/destroyed)
- Pod IPs change when pods restart
- Hard to track which pods to connect to
- Need load balancing across replicas

**Solution**: Services provide stable endpoint
```
Blue Pod → db-service (10.103.132.104)
                ↓ (kube-proxy forwards)
          Orange Pod (10.244.1.2)
```

---

### Service Types

#### 1. ClusterIP (Default)
- **Internal only** - accessible within cluster
- Every pod can reach it regardless of node
- Gets IP from service cluster IP range

**Use case**: Internal services (databases, APIs, microservices)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-service
spec:
  type: ClusterIP  # Default, can be omitted
  selector:
    app: database
  ports:
  - port: 3306
    targetPort: 3306
```

**Example**:
```bash
kubectl get service
NAME         TYPE        CLUSTER-IP       PORT(S)    AGE
db-service   ClusterIP   10.103.132.104   3306/TCP   12h
```

**Traffic Flow**:
```
Any Pod (any node)
  → Service IP (10.103.132.104:3306)
  → kube-proxy DNAT rule
  → Backend Pod (10.244.1.2:3306)
```

---

#### 2. NodePort
- **External access** - accessible from outside cluster
- Still gets ClusterIP for internal access
- Exposes service on **same port on every node** (30000-32767 range)

**Use case**: External access without load balancer (testing, on-prem)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
  - port: 80          # ClusterIP port
    targetPort: 8080  # Pod port
    nodePort: 30080   # External port (optional, auto-assigned if omitted)
```

**Access Methods**:
```bash
# Internal access (from any pod)
curl http://web-service:80

# External access (from outside cluster)
curl http://<any-node-ip>:30080
curl http://192.168.1.11:30080
curl http://192.168.1.12:30080
curl http://192.168.1.13:30080  # Any node works!
```

**Traffic Flow**:
```
External Client
  → Node IP:30080 (any node)
  → kube-proxy iptables rule
  → Service ClusterIP
  → Backend Pod (on any node)
```

---

### How Service Networking Works

#### IP Range Planning

Three separate IP ranges must **not overlap**:

| Range Type | Example | Typical CIDR | Purpose |
|------------|---------|--------------|---------|
| **Node IPs** | 192.168.1.11-13 | 192.168.1.0/24 | Physical/VM network |
| **Pod IPs** | 10.244.0.0-255.255 | 10.244.0.0/16 | Pod network (CNI managed) |
| **Service IPs** | 10.96.0.0 - 10.111.255.255 | 10.96.0.0/12 | Service virtual IPs |

**Why they can't overlap**:
- Routing confusion (where should packets go?)
- Conflicts in iptables rules
- Service discovery failures

---

#### Service IP Assignment Process

**Step 1: Service Creation**
```bash
kubectl create service clusterip db-service --tcp=3306:3306
```

**Step 2: API Server Assigns IP**
- Reads `--service-cluster-ip-range` flag (e.g., 10.96.0.0/12)
- Assigns next available IP (e.g., 10.103.132.104)
- Stores in etcd

**Step 3: kube-proxy Creates Forwarding Rules**
- Watches API server for service changes
- Creates iptables/IPVS rules on **every node**
- Rules forward traffic: Service IP → Pod IP

```
Service Created
     ↓
API Server assigns IP (10.103.132.104)
     ↓
kube-proxy on Node 1 creates iptables rules
kube-proxy on Node 2 creates iptables rules
kube-proxy on Node 3 creates iptables rules
     ↓
Service accessible cluster-wide
```

---

### kube-proxy: The Traffic Forwarder

**Role**: Creates and maintains forwarding rules for services

**Key Points**:
- Runs on **every node** (deployed as DaemonSet)
- Watches API server for service/endpoint changes
- Does NOT actually proxy traffic (despite the name!)
- Creates rules, kernel handles actual forwarding

#### kube-proxy Modes

**1. iptables Mode (Default)**
- Uses iptables DNAT rules
- Kernel-level forwarding (fast)
- Randomly selects pod for load balancing

**2. IPVS Mode**
- Uses Linux IPVS (IP Virtual Server)
- Better performance at scale
- More load balancing algorithms

**3. userspace Mode (Legacy)**
- kube-proxy actually proxies traffic
- Slow (userspace overhead)
- Rarely used

**Configure mode**:
```bash
kube-proxy --proxy-mode=iptables  # Default
kube-proxy --proxy-mode=ipvs
```

---

### iptables Rules Example

**Scenario**: Service `db-service` (10.103.132.104:3306) → Pod (10.244.1.2:3306)

**View rules**:
```bash
iptables -L -t nat | grep db-service
```

**Output**:
```
KUBE-SVC-XA5OGUC7YRHOS3PU  tcp  --  anywhere  10.103.132.104  /* default/db-service: cluster IP */ tcp dpt:3306
DNAT                      tcp  --  anywhere  anywhere        /* default/db-service: */ tcp to:10.244.1.2:3306
KUBE-SEP-JBWCWHHQM57V2WN7  all  --  anywhere  anywhere        /* default/db-service: */
```

**What these rules do**:
1. **KUBE-SVC-*** : Matches traffic to service IP:port
2. **DNAT**: Destination NAT - changes destination from service IP to pod IP
3. **KUBE-SEP-***: Service endpoint (actual pod)

**Traffic transformation**:
```
Before DNAT:
  src: 10.244.2.5 (client pod)
  dst: 10.103.132.104:3306 (service)

After DNAT (by iptables):
  src: 10.244.2.5 (client pod)
  dst: 10.244.1.2:3306 (actual pod)
```

---

### Service with Multiple Pods (Load Balancing)

**Scenario**: Service backed by 3 pods

```bash
kubectl get pods -o wide
NAME      IP           NODE
web-1     10.244.1.2   node-1
web-2     10.244.2.3   node-2
web-3     10.244.3.4   node-3

kubectl get service
NAME          CLUSTER-IP      PORT(S)
web-service   10.103.132.105  80/TCP
```

**iptables creates rules for each pod**:
```bash
iptables -t nat -L KUBE-SVC-WEB | grep KUBE-SEP
```

**Output**:
```
KUBE-SEP-AAA  33.33% probability  → 10.244.1.2:80
KUBE-SEP-BBB  50.00% probability  → 10.244.2.3:80
KUBE-SEP-CCC  100%   probability  → 10.244.3.4:80
```

**Load balancing**: iptables randomly selects one pod using probability rules

---

### Complete Traffic Flow Examples

#### Example 1: ClusterIP (Pod-to-Pod via Service)

```
Client Pod (10.244.3.5 on Node 3)
  ↓ sends packet to db-service (10.103.132.104:3306)
Node 3's network stack
  ↓ iptables DNAT: 10.103.132.104 → 10.244.1.2
Check routing table
  ↓ "10.244.1.0/24 via 192.168.1.11" (Node 1)
Exit via eth0 to Node 1
  ↓ arrives at Node 1 (192.168.1.11)
Node 1's network stack
  ↓ routes to local pod subnet 10.244.1.0/24
Bridge network (cni0)
  ↓ veth pair
Database Pod (10.244.1.2:3306)
```

#### Example 2: NodePort (External Access)

```
External User
  ↓ curl http://192.168.1.12:30080
Node 2's eth0 (192.168.1.12:30080)
  ↓ iptables: NodePort 30080 → Service ClusterIP
  ↓ iptables DNAT: ClusterIP → Pod IP (10.244.3.4:8080)
Routing: Pod on Node 3
  ↓ packet sent to Node 3 via physical network
Node 3 (192.168.1.13)
  ↓ routes to local pod subnet
Bridge + veth pair
  ↓
Web Pod (10.244.3.4:8080)
```

---

### Verifying Service Networking

#### Check Service IP Range

```bash
# View kube-apiserver config
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep service-cluster-ip-range

# Output example:
--service-cluster-ip-range=10.96.0.0/12
```

**IP Range Calculation**:
- 10.96.0.0/12 means:
  - Start: 10.96.0.0
  - End: 10.111.255.255
  - Total: ~1 million IPs

#### Check Pod IP Range

```bash
# For Weave
kubectl logs <weave-pod> -n kube-system | grep ipalloc-range

# For Calico
kubectl get ippool -o yaml | grep cidr

# For Flannel
kubectl get cm kube-flannel-cfg -n kube-flannel -o yaml | grep Network
```

#### Check Node IP Range

```bash
# Get node IPs
kubectl get nodes -o wide

# View interface
ip addr show eth0

# Example:
# eth0: 192.168.1.11/24
# Therefore node range: 192.168.1.0/24
```

#### Verify kube-proxy

```bash
# Check kube-proxy pods (should be one per node)
kubectl get pods -n kube-system | grep kube-proxy

# Check proxy mode
kubectl logs kube-proxy-xxxxx -n kube-system | grep "Using"
# Output: "Using iptables proxy"

# Verify DaemonSet
kubectl get daemonset -n kube-system
NAME         DESIRED   CURRENT   READY
kube-proxy   3         3         3
```

#### View Service Endpoints

```bash
# Services
kubectl get services

# Endpoints (actual pod IPs behind service)
kubectl get endpoints
NAME         ENDPOINTS
db-service   10.244.1.2:3306
web-service  10.244.1.3:80,10.244.2.4:80,10.244.3.5:80

# Describe service (shows endpoints)
kubectl describe service web-service
```

#### Test Service Connectivity

```bash
# From within a pod
kubectl exec <pod-name> -- curl http://web-service:80

# From within cluster (using service DNS)
kubectl run test --image=busybox --rm -it -- sh
/ # nslookup web-service
/ # wget -O- http://web-service:80

# From outside cluster (NodePort)
curl http://<any-node-ip>:<nodePort>
```

---

### Troubleshooting Service Issues

#### Issue: Service not accessible

```bash
# 1. Check service exists and has endpoints
kubectl get service <service-name>
kubectl get endpoints <service-name>

# If no endpoints, check:
# - Pod selector matches pod labels
kubectl get pods --show-labels
kubectl describe service <service-name>  # Check selector

# - Pods are running
kubectl get pods

# 2. Check kube-proxy is running
kubectl get pods -n kube-system | grep kube-proxy

# 3. Verify iptables rules exist
iptables -t nat -L | grep <service-name>

# 4. Check DNS resolution (if using service name)
kubectl exec <pod> -- nslookup <service-name>
```

#### Issue: NodePort not accessible externally

```bash
# 1. Check service type is NodePort
kubectl get service <service-name>

# 2. Verify NodePort is in valid range (30000-32767)
kubectl describe service <service-name>

# 3. Check firewall rules on nodes
# Cloud providers: Check security groups
# On-prem: Check iptables/firewall

# 4. Test from node itself first
curl http://localhost:<nodePort>

# 5. Check kube-proxy logs
kubectl logs kube-proxy-xxxxx -n kube-system
```

#### Issue: Service load balancing not working

```bash
# 1. Check multiple endpoints exist
kubectl get endpoints <service-name>

# Should show multiple IPs:
# ENDPOINTS: 10.244.1.2:80,10.244.2.3:80,10.244.3.4:80

# 2. Verify iptables rules for all endpoints
iptables -t nat -L KUBE-SVC-<hash> -n

# 3. Test multiple times to see different pods respond
for i in {1..10}; do
  kubectl exec test-pod -- curl -s http://web-service | grep hostname
done
```

---

### Service Networking Commands Reference

```bash
# View all services
kubectl get services -A
kubectl get svc -A

# Create ClusterIP service
kubectl create service clusterip my-service --tcp=80:8080

# Create NodePort service
kubectl create service nodeport my-service --tcp=80:8080 --node-port=30080

# Expose deployment as service
kubectl expose deployment my-app --port=80 --target-port=8080 --type=NodePort

# View service details
kubectl describe service <service-name>

# View endpoints
kubectl get endpoints <service-name>

# Delete service
kubectl delete service <service-name>

# Edit service
kubectl edit service <service-name>

# Get service YAML
kubectl get service <service-name> -o yaml

# Check kube-proxy config
kubectl get configmap kube-proxy -n kube-system -o yaml

# View iptables rules
iptables -t nat -L -n | grep <service-name>
iptables -t nat -L KUBE-SERVICES -n

# Check service IP range
ps aux | grep kube-apiserver | grep service-cluster-ip-range
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep service-cluster-ip-range
```

---

### Service Networking Architecture Summary

```
                    Service (Virtual - No interface)
                    IP: 10.103.132.104 (from service range)
                    DNS: db-service.default.svc.cluster.local
                              ↓
                    kube-proxy (on every node)
                    Creates iptables/IPVS rules
                    ↙          ↓          ↘
            Pod 1           Pod 2         Pod 3
        10.244.1.2      10.244.2.3    10.244.3.4
         (Node 1)        (Node 2)      (Node 3)
            ↓               ↓             ↓
        veth pairs     veth pairs    veth pairs
            ↓               ↓             ↓
        Bridges        Bridges       Bridges
         (CNI)          (CNI)         (CNI)
```

### Key Service Networking Facts

1. **Services are virtual** - no actual network interface, just iptables rules
2. **kube-proxy runs on every node** - ensures services work from any pod/node
3. **Three IP ranges**: nodes, pods, services - must not overlap
4. **Service IP assigned by API server** - from configured service CIDR
5. **Traffic forwarding by kernel** - iptables/IPVS, not actual proxying
6. **Load balancing automatic** - when multiple pods match service selector
7. **ClusterIP = internal only** - pods can access, external cannot
8. **NodePort = external access** - any node IP + port reaches service
9. **Services span nodes** - pod can access service regardless of location
10. **DNS names automatic** - `<service>.<namespace>.svc.cluster.local`

---

## Summary

**Kubernetes Networking Stack**:
```
Application (Pod)
    ↓
Container Runtime (containerd/CRI-O)
    ↓
CNI Plugin (Flannel/Weave/Calico)
    ↓
Linux Networking (bridge/veth/routes/iptables)
    ↓
kube-proxy (Service forwarding rules)
    ↓
Physical Network
```

**Complete Networking Flow**:
```
External User
    ↓ NodePort (30080)
Node's eth0 (192.168.1.11)
    ↓ iptables: NodePort → ClusterIP
Service (10.103.132.104)
    ↓ iptables DNAT: ClusterIP → Pod IP
Pod subnet routing
    ↓ CNI bridge + veth
Pod (10.244.2.5)
```

**Key Takeaways**:
1. Each pod gets unique IP from CNI-managed pool
2. CNI plugins handle pod networking automatically
3. NetworkPolicies require CNI with policy support
4. Services provide stable IPs via kube-proxy rules
5. Three separate IP ranges: nodes, pods, services
6. kube-proxy creates iptables rules on every node
7. Services work cluster-wide regardless of pod location

Use this guide as your reference for understanding and troubleshooting Kubernetes networking! 🚀