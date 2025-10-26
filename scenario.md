# Three Real-Time Production Kubernetes Challenges

## Challenge 1: Resource Sharing & Isolation - From Namespace Bleed to Resource Quota to Pod Limits

### The Problem

You have a shared Kubernetes cluster with multiple environments/teams:
- **dev namespace**: Team A's microservices
- **qa namespace**: Team B's microservices  
- **prod namespace**: Team C's microservices

All running on the same 3-node cluster (48GB RAM total).

### Initial Setup (The Naive Approach)

Just throw services into namespaces without resource limits:

```yaml
# Team A deploys their payment-service to dev namespace
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: dev
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: payment-service
        image: payment-service:1.0
        # No resource limits or requests!
```

### The Disaster

**Timeline:**
```
09:00 - Everything fine. Cluster healthy.
        dev:    10GB used
        qa:     5GB used
        prod:   8GB used
        Total:  23GB used, 25GB available

10:00 - Team A's payment-service has a memory leak
        Pods start consuming more and more memory
        dev:    20GB used (leak consuming memory)
        qa:     5GB used
        prod:   8GB used
        Total:  33GB used, 15GB available

11:00 - Node memory pressure! Kubelet triggers eviction
        BUT: It doesn't know which namespace is the culprit
        It just kills ANY pods when memory runs out
        
        Eviction victims (random):
        - qa-api-pod (in QA namespace) → CrashLoopBackOff (OOMKilled)
        - prod-database-pod (in prod namespace) → CrashLoopBackOff (OOMKilled)
        - dev-payment-service-pod (the actual culprit) → Still running!
        
        Result: QA and Prod are down because of Team A's memory leak!
```

### Why This Happened

Namespaces provide logical isolation, not resource isolation:

```
Namespaces = Organization / Multi-tenancy
NOT resource boundaries!

Think of it like:
- Namespaces = Different departments in a bank
- Resource Quota = Budget per department
- Resource Limits = Spending limit per employee

Without Resource Quota/Limits:
One department's budget overrun affects other departments!
```

### Solution 1: Implement Resource Quota

Add ResourceQuota to each namespace:

```yaml
# For dev namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.memory: "10Gi"      # Total RAM that dev namespace can REQUEST
    requests.cpu: "8"             # Total CPU cores that dev can REQUEST
    limits.memory: "12Gi"         # Total RAM limit (hard cap)
    limits.cpu: "10"              # Total CPU limit (hard cap)
    pods: "50"                    # Max pods in this namespace
    requests.storage: "100Gi"
---
# For qa namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: qa-quota
  namespace: qa
spec:
  hard:
    requests.memory: "8Gi"
    requests.cpu: "6"
    limits.memory: "10Gi"
    limits.cpu: "8"
    pods: "30"
    requests.storage: "50Gi"
---
# For prod namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: prod-quota
  namespace: prod
spec:
  hard:
    requests.memory: "20Gi"       # Prod gets most resources
    requests.cpu: "16"
    limits.memory: "25Gi"
    limits.cpu: "20"
    pods: "100"
    requests.storage: "200Gi"
```

Apply them:

```bash
kubectl apply -f resource-quota.yaml
```

Verify:

```bash
kubectl describe resourcequota dev-quota -n dev
```

Output:
```
Name:            dev-quota
Namespace:       dev
Resource         Used  Hard
--------         ----  ----
limits.cpu       2     10
limits.memory    4Gi   12Gi
pods             8     50
requests.cpu     1     8
requests.memory  3Gi   10Gi
requests.storage 0     100Gi
```

### What Resource Quota Does

Now when Team A's payment-service tries to use more than 10Gi (the dev namespace limit):

```bash
# Developer tries to scale payment-service
kubectl scale deployment payment-service -n dev --replicas 5
```

The new pods will fail to schedule with error:

```
Error from server (Forbidden): error when creating "pod": Pod "payment-service-abc12" is forbidden: 
exceeded quota: dev-quota, requested: memory=2Gi, used: 10Gi, limited: 10Gi
```

**Blast radius is limited!** Team A can't consume more than their quota, so QA and Prod are protected.

### Challenge with Resource Quota

```
New Issue: payment-service pods in dev are CrashLooping with OOM

Why?
Dev namespace limit is 10Gi
3 payment-service replicas, each requesting 2Gi = 6Gi
That should fit, but pods are still crashing...

Let me check...
```

Actually, the memory limit needs to be set ON THE POD, not just the namespace quota.

### Solution 2: Add Resource Limits on Individual Pods

Add requests and limits to each pod specification:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: dev
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: payment-service
        image: payment-service:1.0
        
        # NEW: Specify resource requirements
        resources:
          requests:
            memory: "512Mi"    # Minimum memory needed
            cpu: "250m"       # Minimum CPU cores needed
          limits:
            memory: "1Gi"     # Maximum memory allowed
            cpu: "500m"       # Maximum CPU cores allowed
```

Apply it:

```bash
kubectl apply -f payment-service.yaml
```

### What This Does

**requests:** Tells Kubernetes: "I need at least this much to run properly"
- Used for pod scheduling (scheduler checks if node has this much available)
- Used for resource quota calculation

**limits:** Tells Kubernetes: "Don't let me use more than this"
- Hard cap on memory (will be OOMKilled if exceeded)
- Soft cap on CPU (gets throttled if exceeded)

### Memory Leak Still Happening

```
New issue: payment-service pod still crashes with OOMKilled

Logs show:
ERROR: java.lang.OutOfMemoryError: Java heap space
```

The pod has a memory limit of 1Gi. The memory leak causes it to eventually hit 1Gi, then gets OOMKilled.

**Now the blast radius is small:** Only the payment-service pod dies, not the whole QA environment.

But we still need to find and fix the leak.

### Solution 3: Diagnose the Memory Leak

Get a shell into the pod:

```bash
kubectl exec -it payment-service-abc12 -n dev -- /bin/bash
```

Inside the pod, check process memory:

```bash
# Check memory usage
ps aux

# For Java apps, check heap
jps -v | grep payment-service
```

Take a heap dump:

```bash
# Generate heap dump
jmap -dump:live,format=b,file=heap.bin <pid>

# Copy it out of pod
kubectl cp dev/payment-service-abc12:/path/to/heap.bin ./heap.bin
```

Take a thread dump:

```bash
# Generate thread dump
jstack <pid> > thread_dump.txt

# Copy it out
kubectl cp dev/payment-service-abc12:/path/to/thread_dump.txt ./thread_dump.txt
```

Share with development team. They analyze and find:

```
From heap dump analysis:
- Cache object is growing unbounded
- Class com.example.PaymentMethodCache has 500MB of objects
- No eviction policy, objects added but never removed
- Threads blocked trying to allocate memory
```

Developer fixes it:

```java
// BEFORE (Memory leak)
private static final Map<String, PaymentMethod> cache = new HashMap<>();

void cachePaymentMethod(String id, PaymentMethod method) {
    cache.put(id, method);  // Never evicted!
}

// AFTER (Fixed)
private final Cache<String, PaymentMethod> cache = CacheBuilder.newBuilder()
    .maximumSize(10000)           // Max 10k entries
    .expireAfterWrite(1, TimeUnit.HOURS)  // Auto evict after 1 hour
    .build(new CacheLoader<String, PaymentMethod>() {
        public PaymentMethod load(String id) {
            return fetchPaymentMethod(id);
        }
    });
```

Deploy fixed version:

```bash
# Build and push new image
docker build -t payment-service:1.1 .
docker push payment-service:1.1

# Update deployment
kubectl set image deployment/payment-service \
  payment-service=payment-service:1.1 \
  -n dev
```

Watch it stabilize:

```bash
kubectl get pods payment-service-* -n dev --watch
```

### Full Solution Architecture

```
Resource Sharing in Production Cluster:

Cluster Level (48GB RAM)
├── Namespace: dev (ResourceQuota: 10Gi limit)
│   ├── payment-service (Pod limit: 1Gi)
│   ├── auth-service (Pod limit: 512Mi)
│   └── user-service (Pod limit: 512Mi)
│
├── Namespace: qa (ResourceQuota: 8Gi limit)
│   ├── payment-service (Pod limit: 1Gi)
│   ├── auth-service (Pod limit: 512Mi)
│   └── user-service (Pod limit: 512Mi)
│
└── Namespace: prod (ResourceQuota: 20Gi limit)
    ├── payment-service (Pod limit: 2Gi)
    ├── auth-service (Pod limit: 1Gi)
    ├── database (Pod limit: 5Gi)
    └── cache (Pod limit: 2Gi)
```

With this setup:
- **ResourceQuota** prevents one namespace from starving others
- **Pod limits** prevent one pod from taking down the whole namespace
- **Memory leaks** only affect the single pod, not the namespace or cluster
- **Monitoring & debugging** with heap dumps helps find root causes

---

## Challenge 2: CrashLoopBackOff Due to Memory Leak - Diagnosis & Fix via Change Management

### The Scenario

Monday morning, prod alert fires:

```
CRITICAL: 5 pods in payment-service restarting continuously
Error: CrashLoopBackOff with OOMKilled
```

You've already implemented Resource Quota and Pod Limits from Challenge 1. So the situation is:
- Payment-service pods have 2Gi memory limit
- They keep hitting the limit and getting OOMKilled
- Restarts every 30 seconds (CrashLoopBackOff)
- This happened after the last deployment

### Investigation

Check pod logs:

```bash
kubectl logs -f payment-service-abc12 -n prod
```

Output:
```
2024-10-22T09:00:00Z [INFO] Application starting...
2024-10-22T09:00:02Z [INFO] Loading cache...
2024-10-22T09:00:03Z [INFO] Initializing connection pool...
2024-10-22T09:00:05Z [WARN] Memory usage: 500MB
2024-10-22T09:00:10Z [WARN] Memory usage: 1GB
2024-10-22T09:00:15Z [WARN] Memory usage: 1.5GB
2024-10-22T09:00:20Z [ERROR] Memory allocation failed
2024-10-22T09:00:20Z [FATAL] OutOfMemoryError: Java heap space
[Pod killed by kubelet due to OOMKilled]
```

Check previous crashes:

```bash
kubectl logs payment-service-abc12 -n prod --previous
```

Same pattern. Memory grows over 20 seconds, then crash.

### Diagnosis: Heap Dump

Get into the pod before it crashes:

```bash
# Get a shell
kubectl exec -it payment-service-abc12 -n prod -- bash
```

Inside pod, while it's running (and memory is climbing), take a heap dump:

```bash
# Find Java process
ps aux | grep java

# Output:
# root    123  45.2  40.0 2000000 800000 ?  Ssl  09:00  0:05 java -Xmx2g ...

# Take heap dump
jmap -dump:live,format=b,file=/tmp/heap.bin 123

# Wait for it to complete
# File size should be around 1GB+
```

Copy it from pod to local:

```bash
kubectl cp prod/payment-service-abc12:/tmp/heap.bin ./heap.bin
```

Analyze the heap dump locally using Java tools:

```bash
# Using jhat (Java Heap Analysis Tool)
jhat -J-Xmx4g heap.bin

# This starts a web server on http://localhost:7000
# Open browser and navigate through object graphs
```

Or use Eclipse MAT (Memory Analyzer Tool) for better visualization.

### What the Heap Dump Shows

After analyzing, you find:

```
Top Memory Consumers:
1. HashMap in PaymentTransactionCache: 1.2GB
   - Contains 10 million transaction objects
   - Never gets cleared!

2. Connection pool: 600MB
   - 500 database connections open
   - Should only be 50!

3. Request queue: 200MB
   - Queue has 100k pending requests
   - Something is not processing them
```

### Thread Dump Analysis

While pod is still running, take a thread dump:

```bash
# Inside pod
jstack 123 > /tmp/thread_dump.txt

# Copy out
kubectl cp prod/payment-service-abc12:/tmp/thread_dump.txt ./thread_dump.txt
```

Analyze the thread dump:

```
"http-nio-8080-exec-1" #25 daemon prio=5 os_prio=0 tid=0x00007f8b2c0e3000 nid=0x2345 
waiting on condition [0x00007f8b27b7e000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000007bdf02000> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:834)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireQueued(AbstractQueuedSynchronizer.java:867)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquire(AbstractQueuedSynchronizer.java:1199)
        at java.util.concurrent.locks.ReentrantLock.lock(ReentrantLock.java:213)
        at com.example.PaymentTransactionCache.getTransaction(PaymentTransactionCache.java:45)
```

Threads are blocked on a lock in the payment cache, causing the request queue to pile up!

### Root Cause Analysis

Development team analyzes findings:

```
Problem 1: PaymentTransactionCache memory leak
- Application caches all transactions (millions of them)
- No cleanup logic
- Cache grows until OOM

Problem 2: Database connection pool exhaustion
- 500 connections in pool (should be 50)
- Threads blocked waiting for connections
- This is because threads are stuck in lock contention

Problem 3: Lock contention in cache
- Multiple threads accessing cache with synchronized methods
- Threads pile up waiting for lock
- Queue grows -> memory grows -> OOM
```

### Solution via Change Management

This needs to go through change management. Create a ticket:

```
CHANGE REQUEST: CR-2024-1022-001
Title: Fix Memory Leak in Payment-Service Cache

Description:
- PaymentTransactionCache has memory leak, causing OOMKilled
- Cache grows unbounded with no eviction policy
- Affects prod payment-service, causes intermittent outages

Root Cause:
- No TTL or size limits on cache
- No background cleanup thread
- Lock contention on cache causing thread pile-up

Solution:
1. Implement bounded cache with max size 100k transactions
2. Add TTL of 1 hour for cache entries
3. Use ConcurrentHashMap with read-write locks (better scaling)
4. Add metrics to monitor cache size and hit rate

Changes:
- com/example/PaymentTransactionCache.java
- application.properties

Rollback Plan:
- Revert to previous version: payment-service:3.2.1
- Clear cache: App restart will lose cached transactions
- No data loss as transactions are read from database

Testing:
- Load test with 1000 concurrent users
- Verify cache size stays under 500MB
- Verify no thread stalls
- Monitor for 24 hours

Risk: Low (cache is optimization only, database is source of truth)
```

### Code Changes

Developer creates the fix:

```java
// BEFORE (Problematic)
public class PaymentTransactionCache {
    private static final Map<String, PaymentTransaction> cache = Collections.synchronizedMap(new HashMap<>());
    
    public PaymentTransaction getTransaction(String id) {
        // Always miss on first call, adds to cache
        return cache.computeIfAbsent(id, k -> fetchFromDatabase(k));
        // Never evicts! Grows forever
    }
}

// AFTER (Fixed)
public class PaymentTransactionCache {
    // Bounded cache with 1 hour TTL
    private final Cache<String, PaymentTransaction> cache = CacheBuilder.newBuilder()
        .maximumSize(100_000)                    // Max 100k entries
        .expireAfterWrite(1, TimeUnit.HOURS)     // Auto evict after 1 hour
        .recordStats()                            // Track hits/misses
        .build(new CacheLoader<String, PaymentTransaction>() {
            @Override
            public PaymentTransaction load(String key) throws Exception {
                return fetchFromDatabase(key);
            }
        });
    
    public PaymentTransaction getTransaction(String id) {
        return cache.getUnchecked(id);
    }
    
    // Expose metrics
    public CacheStats getStats() {
        return cache.stats();
    }
}
```

Also fix connection pool:

```properties
# BEFORE
spring.datasource.hikari.maximum-pool-size=500

# AFTER (For 50 concurrent connections)
spring.datasource.hikari.maximum-pool-size=50
spring.datasource.hikari.minimum-idle=10
```

### Testing Before Deployment

Run load tests:

```bash
# Build new version
docker build -t payment-service:3.3.0 .
docker push payment-service:3.3.0

# Test in staging
kubectl set image deployment/payment-service \
  payment-service=payment-service:3.3.0 \
  -n staging

# Run load test
./load-test.sh --users 1000 --duration 10m

# Monitor memory
kubectl top pod -n staging -l app=payment-service --containers

# Expected output:
# NAME                          CPU(cores)   MEMORY(Mi)
# payment-service-abc12         250m         450Mi    ← Stable, not growing!
# payment-service-def34         200m         420Mi
# payment-service-ghi56         220m         410Mi

# Check cache metrics
kubectl logs payment-service-abc12 -n staging | grep "CacheStats"
# Output: Hit rate: 85%, Cache size: 95000 entries, Evictions: 12000
```

Looks good! Proceed with production deployment.

### Production Deployment

Use blue-green deployment for safety:

```bash
# Current version (blue)
kubectl get deployment payment-service -n prod
# Replicas: 3, Version: 3.2.1

# Create new deployment (green)
kubectl apply -f payment-service-v3.3.0.yaml -n prod

# Both versions running side by side
kubectl get pods -n prod -l app=payment-service
# payment-service-v3.2.1-abc12
# payment-service-v3.2.1-def34
# payment-service-v3.3.0-ghi56
# payment-service-v3.3.0-jkl78
# payment-service-v3.3.0-mno90

# Traffic initially goes to old version via service selector
# New pods are "warming up"

# Gradually shift traffic (using service mesh or DNS weighting)
# Monitor metrics

# If all looks good, scale down old version
kubectl scale deployment payment-service-blue -n prod --replicas 0

# If issues, quick rollback:
kubectl scale deployment payment-service-green -n prod --replicas 0
kubectl scale deployment payment-service-blue -n prod --replicas 3
```

### Verification

After deployment, verify fix worked:

```bash
# Watch pods - should NOT restart
kubectl get pods -l app=payment-service -n prod --watch

# Monitor memory
kubectl top pods -n prod -l app=payment-service

# Expected: Stable memory, no CrashLoopBackOff

# Check cache metrics
kubectl logs payment-service-abc12 -n prod | grep "CacheStats"

# Alert checks
# - CrashLoopBackOff should go to 0
# - Memory usage should stabilize
# - No OOMKilled events
```

### Post-Incident

Update runbook:

```markdown
## Memory Leak Diagnosis & Resolution

### If you see CrashLoopBackOff with OOMKilled:

1. **Get logs**
   kubectl logs -f <pod> --previous

2. **Take heap dump**
   kubectl exec -it <pod> -- bash
   jmap -dump:live,format=b,file=/tmp/heap.bin <pid>
   kubectl cp <pod>:/tmp/heap.bin ./heap.bin

3. **Analyze heap dump**
   Use Eclipse MAT or jhat

4. **Take thread dump**
   jstack <pid> > thread_dump.txt
   kubectl cp <pod>:/thread_dump.txt ./thread_dump.txt

5. **Share with development team**
   Include: Heap dump, thread dump, pod logs, metrics

6. **Create change request**
   Document root cause and fix

7. **Test in staging** before prod deployment

### Common Causes:
- Unbounded caches
- Thread pools not releasing threads
- Collecting all logs in memory
- Connection pool leaks
- Circular object references
```

---

## Challenge 3: Kubernetes Upgrade - From Manual Process to Documented Procedure

### The Challenge

Your cluster runs Kubernetes 1.27.3. New version 1.28.5 is available with security patches and features you need. But upgrades can break things:

```
Real risks:
- API versions change (v1beta1 becomes deprecated)
- kubelet upgrade can disconnect nodes
- Pod evictions can cause downtime
- Worker node upgrade can cause pod restarts
- Rollback is complex

This requires careful planning and documentation.
```

### Pre-Upgrade Assessment

Create an upgrade runbook:

```markdown
# Kubernetes Upgrade Guide: 1.27.3 → 1.28.5

## Step 1: Pre-Upgrade Checklist

### 1.1 Backup Everything
```bash
# Backup etcd (the database)
sudo systemctl stop kubelet

# Take snapshot
sudo /usr/local/bin/etcdctl snapshot save backup-1.27.3.db

# Verify backup
sudo /usr/local/bin/etcdctl snapshot status backup-1.27.3.db

# Store backup in safe location
sudo cp backup-1.27.3.db /backup/etcd/

# Start kubelet again
sudo systemctl start kubelet
```

### 1.2 Check API Deprecations

```bash
# Download release notes
curl -L https://github.com/kubernetes/kubernetes/releases/tag/v1.28.5/ReleaseNotes.md

# Check for deprecated APIs
kubectl api-resources --verbs=list

# Common deprecations to check:
# - policy/v1beta1 PodDisruptionBudget (now v1)
# - rbac.authorization.k8s.io/v1beta1 (now v1)
# - storage.k8s.io/v1beta1 StorageClass (now v1)
```

### 1.3 Test All APIs We Use

Create a script to check:

```bash
#!/bin/bash
# check-apis.sh

echo "Checking API usage in cluster..."

# Get all resources using deprecated versions
kubectl get all -A -o json | \
  jq '.items[] | select(.apiVersion | contains("beta")) | {apiVersion, kind, name, namespace}'

# Output might show:
# "apiVersion": "policy/v1beta1"
# "apiVersion": "rbac.authorization.k8s.io/v1beta1"
```

Fix any deprecated APIs before upgrade.

### 1.4 Check Addon Versions

Some addons might need updating:

```bash
# Check CoreDNS version
kubectl get deployment -n kube-system coredns -o jsonpath='{.spec.template.spec.containers[0].image}'
# Output: coredns:1.9.3 (might need update to 1.10.x for k8s 1.28)

# Check kube-proxy version
kubectl get daemonset -n kube-system kube-proxy -o jsonpath='{.spec.template.spec.containers[0].image}'

# Check metrics-server
kubectl get deployment -n kube-system metrics-server -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### 1.5 Document Current State

```bash
# Save current cluster state
kubectl cluster-info dump > cluster-state-before-upgrade.yaml

# List all nodes
kubectl get nodes -o wide > nodes-before.txt

# List all pods
kubectl get pods -A -o wide > pods-before.txt

# Save all RBAC rules
kubectl get clusterroles,clusterrolebindings,roles,rolebindings -A -o yaml > rbac-before.yaml
```

---

## Step 2: Control Plane Upgrade (Master Nodes)

For high-availability cluster, master nodes are upgraded one at a time.

### 2.1 Upgrade First Master Node

Choose one master node to start:

```bash
MASTER_NODE="k8s-master-1"

# Step 1: Drain the node (evict all pods gracefully)
# This is safe because control plane pods are on OTHER masters
kubectl drain $MASTER_NODE --ignore-daemonsets --delete-emptydir-data

# Step 2: Mark node as unschedulable (taints)
kubectl taint nodes $MASTER_NODE node-role.kubernetes.io/master=:NoSchedule

# Step 3: SSH to the master node
ssh $MASTER_NODE

# Inside master node, upgrade components:
sudo kubeadm upgrade plan

# Output shows:
# Components that will be upgraded:
# COMPONENT                 CURRENT   TARGET
# kube-apiserver            v1.27.3   v1.28.5
# kube-controller-manager   v1.27.3   v1.28.5
# kube-scheduler            v1.27.3   v1.28.5
# kube-proxy                v1.27.3   v1.28.5
# CoreDNS                   1.9.3     1.10.1
# etcd                      3.5.7     3.5.9

# Upgrade control plane
sudo kubeadm upgrade apply v1.28.5

# This will:
# 1. Backup etcd automatically
# 2. Upgrade all control plane components
# 3. Replace static pod manifests
# 4. Restart control plane pods

# Wait for control plane to become healthy
until kubectl get nodes | grep Ready > /dev/null 2>&1; do
  echo "Waiting for control plane..."
  sleep 5
done

# Upgrade kubelet and kubectl on this master node
sudo apt-get update
sudo apt-get install -y kubelet=1.28.5-00 kubectl=1.28.5-00

# Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Wait for kubelet to be ready
until kubectl get node $MASTER_NODE | grep Ready > /dev/null 2>&1; do
  echo "Waiting for kubelet..."
  sleep 5
done

# Remove taints and allow scheduling again
kubectl taint nodes $MASTER_NODE node-role.kubernetes.io/master-
kubectl uncordon $MASTER_NODE

# Verify this master node is ready
kubectl get nodes -o wide | grep $MASTER_NODE
# Should show: Ready, v1.28.5
```

### 2.2 Verify Control Plane Health

```bash
# Check all control plane pods are running
kubectl get pods -n kube-system -l component=kube-apiserver,component=kube-controller-manager,component=kube-scheduler

# All should be Running

# Check apiserver logs
kubectl logs -n kube-system -l component=kube-apiserver -f

# Should not show errors

# Verify cluster is still working
kubectl get nodes
kubectl get pods -A
```

### 2.3 Upgrade Remaining Master Nodes

Repeat the process for each remaining master node:

```bash
for MASTER in k8s-master-2 k8s-master-3; do
  echo "Upgrading $MASTER..."
  
  # Drain
  kubectl drain $MASTER --ignore-daemonsets --delete-emptydir-data
  
  # SSH and upgrade
  ssh $MASTER
  
  # Inside node:
  sudo kubeadm upgrade node
  sudo apt-get install -y kubelet=1.28.5-00 kubectl=1.28.5-00
  sudo systemctl daemon-reload
  sudo systemctl restart kubelet
  
  # Uncordon
  kubectl uncordon $MASTER
  
  # Verify Ready
  kubectl get node $MASTER
  
  sleep 2m  # Wait a bit before next master
done
```

---

## Step 3: Worker Node Upgrade

This is critical—worker nodes host your applications. Upgrade one at a time to maintain availability.

### 3.1 Upgrade First Worker Node

```bash
WORKER_NODE="k8s-worker-1"

# Step 1: Drain all pods from this node gracefully
# This evicts pods which are rescheduled to other worker nodes
kubectl drain $WORKER_NODE \
  --ignore-daemonsets \         # Keep daemonsets (they should run on all nodes)
  --delete-emptydir-data \       # Delete pods with emptydir volumes
  --grace-period=120 \           # 2 minutes to gracefully shutdown
  --timeout=5m

# During drain:
# - Pods receive SIGTERM
# - 2 minutes to clean up
# - Then SIGKILL
# - Pods restart on other nodes

# Step 2: SSH to worker node
ssh $WORKER_NODE

# Inside worker node:
# Upgrade kubelet and kubectl (NO kubeadm upgrade on worker)
sudo apt-get update
sudo apt-get install -y kubelet=1.28.5-00 kubectl=1.28.5-00

# Step 3: Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Wait for kubelet to report Ready
until kubectl get node $WORKER_NODE | grep Ready > /dev/null 2>&1; do
  echo "Waiting for kubelet..."
  sleep 5
done

# Step 4: Uncordon (allow pods to schedule again)
kubectl uncordon $WORKER_NODE

# Verify
kubectl get node $WORKER_NODE -o wide
# Should show: Ready, v1.28.5

# Verify pods are coming back
kubectl get pods -A -o wide | grep $WORKER_NODE
```

### 3.2 Monitor During Worker Upgrade

```bash
# Watch pods
watch kubectl get pods -A -o wide

# Watch nodes
watch kubectl get nodes -o wide

# Check node status
kubectl describe node $WORKER_NODE

# Look for:
# - Node is Ready
# - Kubelet version 1.28.5
# - No pressure conditions (MemoryPressure, DiskPressure, etc.)
```

### 3.3 Upgrade All Worker Nodes

```bash
# For each worker node:
for WORKER in k8s-worker-1 k8s-worker-2 k8s-worker-3 k8s-worker-4; do
  echo "Upgrading $WORKER..."
  
  # Drain
  kubectl drain $WORKER \
    --ignore-daemonsets \
    --delete-emptydir-data \
    --grace-period=120 \
    --timeout=10m  # Allow more time for stateful apps
  
  # SSH and upgrade
  ssh $WORKER
  
  # Inside node:
  sudo apt-get update
  sudo apt-get install -y kubelet=1.28.5-00 kubectl=1.28.5-00
  sudo systemctl daemon-reload
  sudo systemctl restart kubelet
  
  # Wait to be Ready
  exit  # Exit SSH
  
  until kubectl get node $WORKER | grep Ready > /dev/null 2>&1; do
    echo "Waiting for $WORKER..."
    sleep 10
  done
  
  # Uncordon
  kubectl uncordon $WORKER
  
  # Verify pods are running
  kubectl get pods -A -o wide | grep $WORKER
  
  echo "$WORKER upgrade complete. Waiting 5 minutes before next..."
  sleep 5m
done
```

---

## Step 4: Post-Upgrade Verification

### 4.1 Verify All Nodes

```bash
kubectl get nodes -o wide

# All should show:
# - Status: Ready
# - Version: v1.28.5
# - No pressure conditions
```

### 4.2 Verify All Pods

```bash
# Check all pods are running
kubectl get pods -A

# No CrashLoopBackOff, Pending, or ImagePullBackOff

# Specific checks:
# CoreDNS should be running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Metrics-server should be running
kubectl get deployment -n kube-system metrics-server

# All addons running
kubectl get pods -n kube-system
```

### 4.3 Test Cluster Functionality

```bash
# Create test pod
kubectl run test-pod --image=nginx

# Verify it can reach DNS
kubectl exec test-pod -- nslookup kubernetes.default

# Verify it can reach API server
kubectl exec test-pod -- curl -k https://kubernetes.default.svc.cluster.local/api/v1/namespaces

# Verify volume mounting works (if using storage)
kubectl create -f test-pvc.yaml
kubectl get pvc

# Cleanup
kubectl delete pod test-pod
kubectl delete pvc test-pvc
```

### 4.4 Compare Before/After State

```bash
# Compare node states
diff nodes-before.txt <(kubectl get nodes -o wide)

# Compare pod states  
diff pods-before.txt <(kubectl get pods -A -o wide)

# Any significant differences?
# New nodes? Missing pods? Different versions?
```

### 4.5 Check Upgrade Artifacts

```bash
# Verify etcd
sudo /usr/local/bin/etcdctl member list

# Verify API server logs
kubectl logs -n kube-system -l component=kube-apiserver -f | tail -100

# Should show successful startup messages
```

---

## Step 5: Rollback Plan (If Something Goes Wrong)

If upgrade fails and you need to rollback:

```bash
# Restore from backup (if cluster is down)
# On first master node:

sudo systemctl stop kubelet

# Restore etcd
sudo /usr/local/bin/etcdctl snapshot restore backup-1.27.3.db \
  --data-dir=/var/lib/etcd.restored

# Move corrupted data
sudo mv /var/lib/etcd /var/lib/etcd.backup
sudo mv /var/lib/etcd.restored /var/lib/etcd

# Downgrade kubelet
sudo apt-get install -y kubelet=1.27.3-00 kubectl=1.27.3-00

# Start
sudo systemctl restart kubelet

# Wait for health
kubectl get nodes
```

---

## Step 6: Documentation & Post-Mortem

### Update Upgrade Log

```markdown
# Upgrade Log: 1.27.3 → 1.28.5

**Date**: 2024-10-22  
**Duration**: 3 hours  
**Downtime**: 0 minutes  

**Timeline:**
- 08:00 - Backup etcd
- 08:15 - Check API deprecations (none found)
- 08:30 - Upgrade master-1 (25 min)
- 08:55 - Upgrade master-2 (25 min)
- 09:20 - Upgrade master-3 (25 min)
- 09:45 - Upgrade worker-1 (20 min, pods rescheduled smoothly)
- 10:05 - Upgrade worker-2 (20 min)
- 10:25 - Upgrade worker-3 (20 min)
- 10:45 - Upgrade worker-4 (20 min)
- 11:00 - Post-upgrade verification complete

**Issues Encountered**: None  
**Pods evicted & restarted**: 0 unexpected restarts  
**Rollback needed**: No  

**Metrics:**
- All nodes Ready in < 2 minutes after each upgrade
- No connection timeouts
- All applications continued to work during upgrade
- No memory pressure or CPU throttling
```

### Create Lessons Learned

```markdown
# Upgrade Lessons Learned

**What Went Well:**
- Draining one node at a time prevented any pod downtime
- Having backups gave confidence to proceed
- Checking for deprecated APIs prevented compatibility issues

**What We Could Improve:**
- Upgrade took 3 hours; could parallelize more (but needs higher env)
- Monitor metrics during upgrade (set up Prometheus scraping)
- Automate backup process with cron job

**For Next Upgrade:**
- Set up automated pre-upgrade checks
- Create CI/CD pipeline for upgrade testing in staging
- Improve rollback automation
- Schedule during low-traffic window (though this worked fine)
```

---

## How to Present These Three Challenges in Interview

### Challenge 1: Resource Sharing

> "We had a shared multi-tenant cluster with dev, qa, and prod namespaces. When Team A's service had a memory leak, it started consuming memory from the entire cluster, causing QA and Prod services to get OOMKilled and crash. This taught me that namespaces alone don't provide resource isolation.
>
> I implemented ResourceQuota at the namespace level to limit total memory and CPU each namespace could use. Then I added resource requests and limits on individual pods. This created a blast radius hierarchy: if a pod leaks memory, it only crashes itself; if a namespace exceeds its quota, new pods can't be scheduled.
>
> When we still had CrashLoopBackOff issues, I took heap dumps and thread dumps, shared them with the dev team, and they found an unbounded cache that was never evicted. They fixed it by adding a bounded cache with TTL and proper cleanup."

### Challenge 2: Diagnosis & Fix

> "For a CrashLoopBackOff caused by memory leak, I systematically checked: application logs showed growing memory, heap dump revealed an unbounded cache consuming 1.2GB, thread dump showed lock contention. I shared these findings with the dev team through change management, they fixed the cache configuration, we tested in staging with load tests, monitored memory stabilization, then did a blue-green deployment to production with quick rollback capability.
>
> The key was systematic diagnosis using heap dumps and thread dumps—not just killing the pod and restarting it. This found the actual root cause so they could fix it properly."

### Challenge 3: Upgrade Process

> "Kubernetes upgrades are complex because both control plane and worker nodes need upgrading, and mistakes can cause outages. I documented a detailed procedure:
>
> 1. Backup etcd and check for deprecated APIs
> 2. Upgrade control plane nodes one at a time (masters can coordinate, unlike workers)
> 3. Upgrade worker nodes one at a time using drain to evict pods gracefully
> 4. Each node gets thoroughly tested before the next upgrade starts
>
> The key principle is sequential upgrades with thorough verification at each step. Draining nodes ensures pods restart on healthy nodes with new version while maintaining uptime. We went from 1.27.3 to 1.28.5 with zero downtime."

---

## Key Takeaways

**Challenge 1 - Resource Sharing:**
- Namespaces = Organization, not resource boundary
- ResourceQuota = Budget per namespace
- Pod limits = Budget per pod
- Monitoring & debugging finds root causes

**Challenge 2 - Diagnosis:**
- Use heap dumps to find memory leaks
- Use thread dumps to find deadlocks
- Change management for safe deployments
- Load test before production

**Challenge 3 - Upgrades:**
- Backup everything first
- Check for deprecations
- Upgrade control plane then workers
- Drain nodes to evict pods gracefully
- Test at each step