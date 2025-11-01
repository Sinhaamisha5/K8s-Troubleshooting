# Pending Pods: Troubleshooting Guide & Interview Case Study

## What is a Pending Pod?

### Definition

A Pending pod means:
```
Kubernetes received the request to run the pod
BUT
It cannot schedule the pod on any of the cluster's nodes
```

The scheduler has evaluated the pod against all available nodes and found no suitable node to run it on.

```
Pod Lifecycle:
Pending → ContainerCreating → Running (goal)
         ↑
      Pod stuck here = Cannot find suitable node
```

### Why Pods Get Stuck in Pending

There are **three main reasons**:

1. **Insufficient Resources** (CPU, Memory)
2. **Node Selector / Label Mismatch**
3. **Taints & Tolerations Mismatch**

Each requires different debugging and solutions.

---

## Problem 1: Insufficient Resources

### The Scenario

You want to schedule a pod that needs 2 CPUs, but your nodes only have 2 CPUs total and already have pods running on them.

```
Cluster Nodes:
┌─────────────────────────────┐
│ Node 1: 2 CPU total         │
│ ├─ Pod A: 0.5 CPU used      │
│ ├─ Pod B: 0.5 CPU used      │
│ ├─ Pod C: 0.5 CPU used      │
│ └─ Available: 0.5 CPU       │ ← Not enough for new 2 CPU pod!
└─────────────────────────────┘

┌─────────────────────────────┐
│ Node 2 (control plane):     │
│ ├─ kube-apiserver           │
│ └─ Can't schedule user pods  │
└─────────────────────────────┘

New Pod Request: 2 CPU needed
Result: PENDING (no node has 2 CPU available)
```

### Investigation

**Step 1: Check pod status and events**

```bash
kubectl get pods
```

Output:
```
NAME               READY   STATUS    RESTARTS   AGE
data-processor     0/1     Pending   0          5m
ml-api             0/1     Pending   0          5m
web-app            0/1     Pending   0          5m
```

**Step 2: Describe the pod to see why it's pending**

```bash
kubectl describe pod data-processor
```

Output (events section):
```
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  5m    default-scheduler  0/2 nodes available:
                                     - 1 node(s) had insufficient CPU
                                     - 1 node(s) didn't match pod affinity rules
```

**Step 3: Check the pod's resource requests**

```bash
kubectl describe pod data-processor | grep -A 5 "Requests"
```

Output:
```
Requests:
  cpu:      2
  memory:   512Mi
```

The pod is requesting **2 CPUs**.

**Step 4: Check node capacity and usage**

```bash
kubectl get nodes
```

Output:
```
NAME                 STATUS   ROLES           AGE   VERSION
node-1               Ready    <none>          10d   v1.27.3
node-2               Ready    control-plane   10d   v1.27.3
```

Detailed node info:

```bash
kubectl describe node node-1
```

Output (capacity section):
```
Capacity:
  cpu:                2
  memory:             8Gi
  ...

Allocatable:
  cpu:                2
  memory:             8Gi
  ...
```

Node 1 has 2 CPUs total. Now check what's allocated:

```bash
kubectl describe node node-1 | grep -A 20 "Allocated resources"
```

Output:
```
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests     Limits
  --------           --------     ------
  cpu                1500m (75%)  2000m (100%)
  memory             2Gi (25%)    4Gi (50%)
```

**Analysis:**
- Node has 2000m (2 CPUs) allocatable
- Already have 1500m (1.5 CPUs) requested
- Available: 500m (0.5 CPUs)
- Pod requesting: 2000m (2 CPUs)
- **Result: Not enough!**

### Why Only Requests Matter (Not Limits)

**Key insight:**

```
Scheduler Uses: REQUESTS (for scheduling decisions)
Kubelet Uses: LIMITS (for enforcement on node)

Example Pod:
resources:
  requests:
    cpu: 500m      ← Scheduler checks this
    memory: 256Mi
  limits:
    cpu: 1000m     ← Limits don't affect scheduling
    memory: 512Mi

Scheduler logic:
"Does this node have 500m CPU available? Yes → Schedule it"
(Even though limit is 1000m, which might not fit!)

This is why:
- Set accurate REQUESTS (for fair scheduling)
- Set appropriate LIMITS (to prevent runaway usage)
```

### Solution 1: Reduce Resource Requests

Only reduce if your app doesn't actually need that much:

**Before (Broken)**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-processor
spec:
  template:
    spec:
      containers:
      - name: processor
        image: data-processor:1.0
        resources:
          requests:
            cpu: 2000m        # Too much!
            memory: 512Mi
          limits:
            cpu: 2000m
            memory: 512Mi
```

**After (Fixed)**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-processor
spec:
  template:
    spec:
      containers:
      - name: processor
        image: data-processor:1.0
        resources:
          requests:
            cpu: 500m         # Reduced to fit on node
            memory: 512Mi
          limits:
            cpu: 1000m        # Allow some headroom beyond request
            memory: 512Mi
```

**Deploy**:
```bash
kubectl apply -f data-processor.yaml
kubectl get pods  # data-processor should now be Running
```

### Solution 2: Add More Nodes

If the app genuinely needs 2 CPUs:

**On cloud providers:**
```bash
# AWS - Add node to cluster
eksctl create nodegroup --cluster=my-cluster --name=new-nodes --node-type=t3.large

# GKE - Add nodes
gcloud container node-pools create new-pool \
  --cluster=my-cluster \
  --machine-type=n1-standard-2

# AKS - Add nodes
az aks nodepool add \
  --resource-group=myResourceGroup \
  --cluster-name=myCluster \
  --name=newnodepool \
  --node-vm-size=Standard_D2s_v3
```

**On bare metal/on-premises:**
```bash
# Add node hardware, join to cluster
kubeadm join <control-plane-ip>:6443 --token=<token> --discovery-token-ca-cert-hash sha256:<hash>
```

**Verify cluster now has capacity:**
```bash
kubectl get nodes
kubectl describe node node-3
# Should show available CPU

kubectl get pods  # data-processor should now be Running
```

### How to Determine Right Resource Requests

**Method 1: Monitor actual usage in staging**

```bash
# Deploy to staging without limits
kubectl set resources deployment/data-processor \
  --requests=cpu=100m,memory=128Mi \
  --limits=cpu=2000m,memory=2Gi

# Run production-like load test
# Monitor actual usage with metrics-server
kubectl top pod data-processor --containers --watch

# Wait for peak traffic
# Note the peak CPU and memory usage
```

**Method 2: Use Vertical Pod Autoscaler to recommend**

```bash
# Install VPA
helm install vpa autoscaler/vertical-pod-autoscaler

# Create VPA policy
kubectl apply -f - <<EOF
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: data-processor-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: data-processor
  updatePolicy:
    updateMode: "off"  # Just recommend, don't auto-update
EOF

# Check recommendations
kubectl describe vpa data-processor-vpa
```

**Method 3: Use load test results**

```
Load Test Results:
- Peak CPU: 750m
- Peak Memory: 384Mi

Set requests to ~80% of peak for safety:
- CPU requests: 600m (750m * 0.8)
- Memory requests: 300Mi (384Mi * 0.8)

Set limits to peak or slightly higher:
- CPU limits: 900m (750m * 1.2)
- Memory limits: 450Mi (384Mi * 1.2)
```

---

## Problem 2: Node Selector / Label Mismatch

### The Scenario

Your pod has a node selector requiring specific node labels, but nodes don't have those labels.

```
Pod Spec:
nodeSelector:
  type: gpu          ← Pod wants node with label: type=gpu

Nodes:
Node 1: labels: [kubernetes.io/hostname=node-1]  ← No type=gpu label!
Node 2: labels: [node-role.kubernetes.io/control-plane]  ← No type=gpu label!

Result: PENDING (no node has type=gpu label)
```

### Investigation

**Step 1: Describe the pod and look for pending reason**

```bash
kubectl describe pod ml-api
```

Output (events):
```
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  5m    default-scheduler  0/2 nodes available:
                                     - 1 node(s) didn't match pod affinity rules
                                     - 1 node(s) had untolerated taints
```

**Step 2: Check pod's node selector**

```bash
kubectl get pod ml-api -o yaml | grep -A 5 nodeSelector
```

Output:
```
nodeSelector:
  type: gpu
```

The pod requires a node with label `type: gpu`.

**Step 3: Check node labels**

```bash
kubectl get nodes --show-labels
```

Output:
```
NAME    STATUS   ROLES           AGE   VERSION   LABELS
node-1  Ready    <none>          10d   v1.27.3   kubernetes.io/hostname=node-1,...
node-2  Ready    control-plane   10d   v1.27.3   node-role.kubernetes.io/control-plane,...
```

Neither node has `type: gpu` label.

**Step 4: Verify node details**

```bash
kubectl describe node node-1 | grep -A 10 "Labels:"
```

Output:
```
Labels:
  beta.kubernetes.io/arch=amd64
  beta.kubernetes.io/os=linux
  kubernetes.io/arch=amd64
  kubernetes.io/hostname=node-1
  kubernetes.io/os=linux
```

Confirmed: `type: gpu` label is missing.

### Solution: Add Label to Node

If you have GPU hardware on the node:

**Add the label:**
```bash
kubectl label node node-1 type=gpu
```

**Verify label was added:**
```bash
kubectl get nodes --show-labels | grep node-1
```

Output:
```
node-1  Ready   <none>  10d  v1.27.3  ...,type=gpu
```

**Pod should now schedule:**
```bash
kubectl get pods
# ml-api should now be Running or ContainerCreating
```

**If pod still pending, check events:**
```bash
kubectl describe pod ml-api | tail -20
# Should show pod was scheduled now
```

### Pod Definition Using Node Selector

**Basic Node Selector:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-api
spec:
  template:
    spec:
      nodeSelector:
        type: gpu        # Pod will only schedule on nodes with this label
      containers:
      - name: ml-api
        image: ml-api:1.0
```

### Advanced: Node Affinity (More Flexible)

Node selectors are simple but limited. Node affinity is more powerful:

**Required affinity (must match)**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-api
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: type
                operator: In
                values: ["gpu", "tpu"]  # Can specify multiple acceptable values
      containers:
      - name: ml-api
        image: ml-api:1.0
```

**Preferred affinity (nice to have)**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-api
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100  # Higher weight = preferred
            nodeSelector:
              matchExpressions:
              - key: type
                operator: In
                values: ["gpu"]  # Prefer GPU if available
      containers:
      - name: ml-api
        image: ml-api:1.0
```

### Common Node Selector Labels

```bash
# See all labels across cluster
kubectl get nodes --show-labels

# Common labels you might set:
- workload-type: gpu, cpu, memory-intensive
- availability-zone: us-east-1a, us-east-1b, us-east-1c
- node-type: compute, storage, database
- tier: premium, standard, spot
- region: us-east, us-west, eu-west

# Kubernetes default labels:
- kubernetes.io/hostname: node-1
- kubernetes.io/os: linux / windows
- kubernetes.io/arch: amd64 / arm64
- node-role.kubernetes.io/master: true
- node-role.kubernetes.io/control-plane: true
```

---

## Problem 3: Taints & Tolerations Mismatch

### What Are Taints?

A **taint** is a node-level mechanism to **repel pods** from being scheduled unless they have a toleration.

```
Taint: "Node doesn't want pods unless they tolerate me"
Toleration: "Pod says: I can tolerate that taint, schedule me"

Example:
Node has taint: workload=machine-learning:NoSchedule
Pod has toleration: workload=machine-learning

Result: Pod can be scheduled on this node
```

### The Scenario

You have a node with a specific taint, and your pod doesn't have the matching toleration.

```
Node 1:
├─ Taint: workload=machine-learning:NoSchedule
└─ Only pods with matching toleration can schedule here

Pod Request:
└─ No toleration for workload=machine-learning taint

Result: PENDING (pod can't tolerate the taint)
```

### Investigation

**Step 1: Describe pod to see pending reason**

```bash
kubectl describe pod web-app
```

Output (events):
```
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  5m    default-scheduler  0/2 nodes available:
                                     - 1 node(s) had untolerated taint "workload=machine-learning:NoSchedule"
                                     - 1 node(s) didn't match pod affinity rules
```

**Step 2: Check what taints the node has**

```bash
kubectl describe node node-1 | grep -A 10 "Taints:"
```

Output:
```
Taints:
  workload=machine-learning:NoSchedule
```

The node has a taint: `workload=machine-learning:NoSchedule`

**Step 3: Check if pod has toleration**

```bash
kubectl get pod web-app -o yaml | grep -A 10 tolerations
```

Output:
```
tolerations: null
# or
# (no tolerations field)
```

The pod has no tolerations for this taint.

### Taint Components Explained

```
Taint Format: key=value:effect

Example: workload=machine-learning:NoSchedule

Components:
- key: workload
- value: machine-learning
- effect: NoSchedule (or NoExecute, PreferNoSchedule)

Effects:
- NoSchedule: Don't schedule new pods (existing pods unaffected)
- NoExecute: Don't schedule new pods AND evict existing pods
- PreferNoSchedule: Prefer not to schedule (but will if necessary)
```

### Solution: Add Toleration to Pod

The pod must have a toleration matching the taint.

**Before (Broken)**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  template:
    spec:
      containers:
      - name: web-app
        image: web-app:1.0
      # Missing tolerations!
```

**After (Fixed)**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  template:
    spec:
      tolerations:
      - key: workload
        operator: Equal
        value: machine-learning
        effect: NoSchedule
      containers:
      - name: web-app
        image: web-app:1.0
```

**Deploy**:
```bash
kubectl apply -f web-app.yaml
kubectl get pods  # web-app should now be Running
```

### Toleration Operators

```yaml
tolerations:

# 1. Exact match
- key: workload
  operator: Equal       # Value must match exactly
  value: machine-learning
  effect: NoSchedule

# 2. Exists (any value)
- key: workload
  operator: Exists      # Ignores value, just checks key exists
  effect: NoSchedule
  # Pod tolerates ANY value for the "workload" key

# 3. Tolerate all taints on a node
- operator: Exists      # No key specified
  # Tolerates all taints (use with caution!)
```

### Common Taint Scenarios

**Scenario 1: Dedicated GPU node**

```bash
# Add taint to node
kubectl taint node gpu-node workload=gpu:NoSchedule

# Only GPU workload pods should have toleration
# Prevents other pods from accidentally using expensive GPU node
```

**Scenario 2: Node under maintenance**

```bash
# Add taint to prevent scheduling
kubectl taint node node-1 maintenance=true:NoSchedule

# Existing pods continue running
# No new pods get scheduled

# Or use NoExecute to evict existing pods
kubectl taint node node-1 maintenance=true:NoExecute
```

**Scenario 3: Spot/preemptible instances on cloud**

```bash
# AWS Spot instances might be evicted
# Add toleration for uncertainty
tolerations:
- key: node.kubernetes.io/not-ready
  operator: Exists
  effect: NoExecute
  tolerationSeconds: 300  # Wait 300s before evicting
```

**Scenario 4: Control plane nodes**

```bash
# Control plane nodes already have taint
kubectl describe node control-plane | grep Taints:
# Output: Taints: node-role.kubernetes.io/master:NoSchedule

# To schedule on control plane (not recommended), add:
tolerations:
- key: node-role.kubernetes.io/master
  operator: Equal
  effect: NoSchedule
```

---

## Troubleshooting Checklist for Pending Pods

```bash
# 1. Get all pending pods
kubectl get pods | grep Pending

# 2. For each pending pod, describe it
kubectl describe pod <pod-name>
# Read the Events section carefully - tells you the reason

# 3. If "Insufficient CPU/Memory":
# Check pod requests
kubectl get pod <pod-name> -o yaml | grep -A 5 resources:

# Check node capacity
kubectl describe node <node-name> | grep -A 20 "Allocated resources"

# Solution: Reduce requests OR add nodes

# 4. If "didn't match pod affinity rules" or "didn't match node selector":
# Check pod's node selector/affinity
kubectl get pod <pod-name> -o yaml | grep -A 10 "nodeSelector\|affinity"

# Check node labels
kubectl get nodes --show-labels

# Solution: Add missing labels to nodes

# 5. If "untolerated taint":
# Check node taints
kubectl describe node <node-name> | grep Taints:

# Check pod tolerations
kubectl get pod <pod-name> -o yaml | grep -A 10 tolerations

# Solution: Add matching toleration to pod

# 6. Check available resources on all nodes
kubectl top nodes

# 7. Get more detailed scheduler information
kubectl get events --sort-by='.lastTimestamp'
```

---

## Interview Story: Pending Pods

**Setup:**
"In production, we had three pods stuck in Pending state. Each had a different root cause, which is why systematic troubleshooting was important."

**Investigation Process:**

1. **First Pod - Insufficient Resources:**
"The data-processor pod was requesting 2 CPUs, but the node only had 2 CPUs total and already had other pods running on it using 1.5 CPUs. I checked the pod's resource requests with `kubectl describe pod` and found it was way over-provisioned for what it actually needed. I reduced the CPU request from 2 to 500m, and it immediately scheduled. The lesson: always benchmark your application to understand its actual resource needs."

2. **Second Pod - Missing Node Label:**
"The ml-api pod had a nodeSelector for `type: gpu`. I checked which nodes in the cluster had that label using `kubectl get nodes --show-labels` and found none. The node with GPU hardware didn't have the label applied. I added the label with `kubectl label node node-1 type=gpu` and the pod scheduled immediately."

3. **Third Pod - Missing Toleration:**
"The web-app pod was pending because the node had a taint `workload=machine-learning:NoSchedule`. The pod didn't have a matching toleration. I added the toleration to the pod spec, redeployed, and it scheduled successfully. This taught me that taints are node-level, tolerations are pod-level—they must match for scheduling to work."

**Key Insight:**
"Pending pods mean the scheduler couldn't find a suitable node. There are three common reasons: insufficient resources (fix by reducing requests or adding nodes), missing node labels (fix by adding labels or using affinity), or missing taints toleration (fix by adding toleration). The events section of `kubectl describe pod` always tells you which category it falls into."

---

## Key Differences: Requests vs Limits

This is crucial for understanding pending pods:

```
REQUEST:
- What scheduler uses to decide where to place pod
- Minimum guaranteed resource for pod
- If node doesn't have this much available, pod won't schedule
- Must be <= Limits

LIMIT:
- Maximum resource pod can use after scheduling
- Used by kubelet to prevent overuse on node
- Does NOT affect scheduling decisions
- Can be higher than requests (but pod only gets requests)

Example:
resources:
  requests:    ← Scheduler checks this
    cpu: 500m
    memory: 256Mi
  limits:      ← Kubelet enforces this
    cpu: 1000m
    memory: 512Mi

Scheduler sees:
"Does node have 500m CPU available? Yes → Schedule"

Kubelet enforces:
"Pod can use max 1000m CPU and 512Mi memory"

So always set REQUESTS based on minimum needed,
and LIMITS based on maximum allowed
```

---

## Quick Reference: Three Pending Pod Causes

| Cause | How to Diagnose | Solution |
|-------|-----------------|----------|
| **Insufficient Resources** | `kubectl describe pod` shows "insufficient CPU/Memory" | 1. Reduce resource requests, OR 2. Add nodes, OR 3. Verify metrics are accurate |
| **Node Label Missing** | `kubectl describe pod` shows "didn't match pod affinity rules" | 1. Add label to node: `kubectl label node <node> key=value`, OR 2. Remove nodeSelector from pod |
| **Taint Not Tolerated** | `kubectl describe pod` shows "untolerated taint" | 1. Add toleration to pod spec, OR 2. Remove taint from node |

---

## Real-World Debugging Example

```bash
# You get alert: "3 pods in Pending state"

# Step 1: Identify pending pods
kubectl get pods --all-namespaces | grep Pending
# OUTPUT:
# production   data-processor   0/1   Pending   0   5m
# production   ml-api           0/1   Pending   0   5m
# production   web-app          0/1   Pending   0   5m

# Step 2: Check first pod
kubectl describe pod data-processor -n production | grep -A 20 Events:
# OUTPUT: "0/2 nodes available: 1 node had insufficient CPU"

# Solution: Reduce requests
kubectl set resources deployment data-processor \
  -n production \
  --requests=cpu=500m,memory=256Mi

# Step 3: Check second pod
kubectl describe pod ml-api -n production | grep -A 20 Events:
# OUTPUT: "didn't match pod affinity rules"

# Check what selector it needs
kubectl get pod ml-api -n production -o yaml | grep nodeSelector -A 5
# OUTPUT: type: gpu

# Add label to node
kubectl label node node-1 type=gpu

# Step 4: Check third pod
kubectl describe pod web-app -n production | grep -A 20 Events:
# OUTPUT: "had untolerated taint workload=machine-learning:NoSchedule"

# Add toleration
kubectl patch deployment web-app -n production -p '
{
  "spec": {
    "template": {
      "spec": {
        "tolerations": [
          {
            "key": "workload",
            "operator": "Equal",
            "value": "machine-learning",
            "effect": "NoSchedule"
          }
        ]
      }
    }
  }
}'

# Verify all pods now running
kubectl get pods -n production
# All should be Running now!
```

---

## Common Mistakes When Debugging Pending Pods

❌ **Mistake 1:** Looking only at pod definition, not checking node state
✅ **Fix:** Always check both `kubectl describe pod` AND node labels/taints

❌ **Mistake 2:** Confusing requests and limits
✅ **Fix:** Remember: scheduler uses REQUESTS, kubelet uses LIMITS

❌ **Mistake 3:** Assuming insufficient resources is always the problem
✅ **Fix:** Read the events message - it tells you the actual reason

❌ **Mistake 4:** Blindly increasing resource requests
✅ **Fix:** First understand actual usage via `kubectl top` or load testing

❌ **Mistake 5:** Not checking if taints were recently added
✅ **Fix:** Always check `kubectl describe node` for taints when troubleshooting

❌ **Mistake 6:** Setting node selector that doesn't exist anywhere
✅ **Fix:** Verify nodes have the label before setting nodeSelector

---

## Key Takeaways

1. **Pending = Scheduler can't find suitable node** → Check events for specific reason

2. **Three main causes:**
   - Insufficient resources → Check requests vs node capacity
   - Missing node label → Add label to node
   - Missing toleration → Add toleration to pod

3. **Scheduler uses REQUESTS** (not limits) for placement decisions

4. **Read `kubectl describe pod` events** → Always tells you the exact issue

5. **Node labels and taints go together** → nodeSelector/affinity needs labels, taints need tolerations

6. **Always benchmark your app** → Set realistic resource requests, not arbitrary values