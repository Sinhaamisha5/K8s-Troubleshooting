# StatefulSet Storage Portability Case Study: AWS vs Other Cloud Providers

## The Scenario

### The JIRA Ticket
Developer reports: "StatefulSet works perfectly on AWS but fails on minikube, AKS, and GKE. We get 'PersistentVolume not found' errors when deploying the same YAML manifest."

### Key Details from Developer
- Same Kubernetes YAML manifest used everywhere
- Works fine on AWS (EBS)
- Fails on: minikube, AKS (Azure), GKE (Google Cloud)
- PVC claims created but PVs are never provisioned
- Error: `kubectl describe pod` shows PV not found

---

## Understanding StatefulSet + Storage Architecture

### How StatefulSets Work with Storage

A StatefulSet with PersistentVolume is different from regular Deployments:

```
StatefulSet with 3 replicas, each needs 1GB storage:

1. Create StatefulSet
   ↓
2. First pod created: database-0
   ├─ Create PVC: data-database-0 (1GB)
   ├─ Wait for PVC to bind to a PV
   └─ Pod mounts the volume
   
3. Only after pod database-0 is FULLY READY:
   ├─ Create second pod: database-1
   ├─ Create PVC: data-database-1 (1GB)
   ├─ Wait for PVC to bind to a PV
   └─ Pod mounts the volume
   
4. Only after pod database-1 is FULLY READY:
   ├─ Create third pod: database-2
   ├─ Create PVC: data-database-2 (1GB)
   ├─ Wait for PVC to bind to a PV
   └─ Pod mounts the volume

KEY: Sequential pod creation, NOT parallel
     (unlike regular Deployments which create all replicas at once)
```

### The Storage Provisioning Flow

When you create a StatefulSet with `volumeClaimTemplates`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: database-service
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:13
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql
  
  # THIS is the critical part
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: ebs        # <-- PROBLEM: AWS-specific!
      resources:
        requests:
          storage: 1Gi
```

### The Flow:

```
1. StatefulSet sees volumeClaimTemplates section
2. For EACH pod replica, it creates a PVC:
   - Pod database-0 → Creates PVC: postgres-data-database-0
   - Pod database-1 → Creates PVC: postgres-data-database-1
   - Pod database-2 → Creates PVC: postgres-data-database-2

3. Each PVC references storageClassName: "ebs"
   
4. Kubernetes looks for StorageClass named "ebs":
   ```
   kubectl get storageclass
   ```
   
   On AWS: StorageClass "ebs" exists (AWS default)
   On GKE: No "ebs" StorageClass! (Has "standard", "premium-rwo")
   On AKS: No "ebs" StorageClass! (Has "default", "managed-premium")
   On minikube: No "ebs" StorageClass! (Has "standard")

5. If StorageClass not found:
   ├─ PVC stays in "Pending" state
   ├─ Provisioner can't run
   ├─ PV is never created
   └─ Pod can't mount volume → Pod stays Pending
```

---

## The Root Cause

The developer hardcoded AWS-specific StorageClass name:

```yaml
storageClassName: ebs    # ❌ Only exists on AWS
```

When deployed elsewhere:
- AKS: No "ebs" class → PVC Pending → Pod fails
- GKE: No "ebs" class → PVC Pending → Pod fails
- minikube: No "ebs" class → PVC Pending → Pod fails

---

## Reproducing the Issue

### Step 1: Check Available StorageClasses

On AWS:
```bash
kubectl get storageclass
```
Output:
```
NAME            PROVISIONER             RECLAIM POLICY   VOLUME BINDING MODE
ebs             ebs.csi.aws.com         Delete           WaitForFirstConsumer
gp2 (default)   kubernetes.io/aws-ebs   Delete           WaitForFirstConsumer
```

On GKE:
```bash
kubectl get storageclass
```
Output:
```
NAME                       PROVISIONER             RECLAIM POLICY   VOLUME BINDING MODE
standard (default)         kubernetes.io/gce-pd    Delete           Immediate
standard-rwo               pd.csi.storage.gke.io   Delete           WaitForFirstConsumer
premium-rwo                pd.csi.storage.gke.io   Delete           WaitForFirstConsumer
```

On AKS:
```bash
kubectl get storageclass
```
Output:
```
NAME                PROVISIONER                     RECLAIM POLICY   VOLUME BINDING MODE
default (default)   kubernetes.io/azure-disk        Delete           Immediate
managed-premium     kubernetes.io/azure-disk        Delete           Immediate
azurefile           kubernetes.io/azure-file        Delete           Immediate
azurefile-premium   kubernetes.io/azure-file        Delete           Immediate
```

On minikube:
```bash
kubectl get storageclass
```
Output:
```
NAME                 PROVISIONER                RECLAIM POLICY   VOLUME BINDING MODE
standard (default)   k8s.io/minikube-hostpath   Delete           Immediate
```

**They're ALL different!**

### Step 2: Deploy StatefulSet on GKE with "ebs" StorageClass

```bash
kubectl apply -f statefulset.yaml -n production
```

Check pod status:
```bash
kubectl get pods -n production -l app=database
```

Output:
```
NAME         READY   STATUS    RESTARTS   AGE
database-0   0/1     Pending   0          5m
```

Pod stuck in Pending. Check PVC:

```bash
kubectl get pvc -n production
```

Output:
```
NAME                   STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
postgres-data-database-0   Pending   <none>   <none>     RWO            ebs            5m
```

PVC is Pending because StorageClass "ebs" doesn't exist on GKE.

### Step 3: Check PVC Events

```bash
kubectl describe pvc postgres-data-database-0 -n production
```

Output:
```
Name:          postgres-data-database-0
Namespace:     production
StorageClass:  ebs
Status:        Pending
Volume:        <unbound>

Events:
  Type     Reason                Age               From                         Message
  ----     ------                ----              ----                         -------
  Warning  ProvisioningFailed    2m (x5 over 3m)  persistentvolume-controller  Failed to provision volume with StorageClass "ebs": storageclass.storage.k8s.io "ebs" not found
```

**Clear error:** StorageClass "ebs" not found!

### Step 4: Check Pod Events

```bash
kubectl describe pod database-0 -n production
```

Output:
```
Name:       database-0
...
Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  3m (x10 over 3m)  default-scheduler  0/3 nodes available: 3 node(s) didn't match pod affinity rules, 3 Insufficient PersistentVolumes.
```

Pod can't be scheduled because there's no PV for it to mount.

---

## The Solution

### Option 1: Use Storage Class Name from Local Provider

Check which StorageClass is available and use that:

**For GKE:**
```yaml
volumeClaimTemplates:
- metadata:
    name: postgres-data
  spec:
    accessModes: [ "ReadWriteOnce" ]
    storageClassName: standard-rwo    # ✅ GKE specific
    resources:
      requests:
        storage: 1Gi
```

**For AKS:**
```yaml
volumeClaimTemplates:
- metadata:
    name: postgres-data
  spec:
    accessModes: [ "ReadWriteOnce" ]
    storageClassName: managed-premium  # ✅ AKS specific
    resources:
      requests:
        storage: 1Gi
```

**For minikube:**
```yaml
volumeClaimTemplates:
- metadata:
    name: postgres-data
  spec:
    accessModes: [ "ReadWriteOnce" ]
    storageClassName: standard         # ✅ minikube specific
    resources:
      requests:
        storage: 1Gi
```

**For AWS:**
```yaml
volumeClaimTemplates:
- metadata:
    name: postgres-data
  spec:
    accessModes: [ "ReadWriteOnce" ]
    storageClassName: gp2             # ✅ AWS specific
    resources:
      requests:
        storage: 1Gi
```

### Option 2: Use Default StorageClass (Best Practice)

Most clusters have a default StorageClass. Just don't specify one:

```yaml
volumeClaimTemplates:
- metadata:
    name: postgres-data
  spec:
    accessModes: [ "ReadWriteOnce" ]
    # Don't specify storageClassName - it will use the default!
    resources:
      requests:
        storage: 1Gi
```

Check what the default is:

```bash
kubectl get storageclass
```

The `(default)` label shows which one is the default:

```
NAME                 PROVISIONER                     RECLAIM POLICY   VOLUME BINDING MODE
standard (default)   k8s.io/gce-pd                   Delete           Immediate
premium-rwo          pd.csi.storage.gke.io           Delete           WaitForFirstConsumer
```

### Option 3: Create Custom StorageClass Named "ebs" on Each Provider (Most Portable)

Create a StorageClass named "ebs" that works with each provider's native storage:

**On GKE:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs  # Custom name matching the StatefulSet manifest
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-standard
  replication-type: regional-pd
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

**On AKS:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs  # Custom name matching the StatefulSet manifest
provisioner: kubernetes.io/azure-disk
parameters:
  storageaccounttype: Standard_LRS
  kind: Managed
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

**On minikube:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs  # Custom name matching the StatefulSet manifest
provisioner: k8s.io/minikube-hostpath
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

Then the original YAML will work everywhere!

### Option 4: Use Helm with Values Override (For Enterprise)

Create a values file per environment:

**values-aws.yaml:**
```yaml
storageClass: ebs
```

**values-gke.yaml:**
```yaml
storageClass: standard-rwo
```

**values-aks.yaml:**
```yaml
storageClass: managed-premium
```

**StatefulSet Helm template:**
```yaml
volumeClaimTemplates:
- metadata:
    name: postgres-data
  spec:
    accessModes: [ "ReadWriteOnce" ]
    storageClassName: {{ .Values.storageClass }}
    resources:
      requests:
        storage: 1Gi
```

Deploy with:
```bash
# AWS
helm install database ./chart -f values-aws.yaml

# GKE
helm install database ./chart -f values-gke.yaml

# AKS
helm install database ./chart -f values-aks.yaml
```

---

## Important StatefulSet + Storage Concepts

### Concept 1: Sequential Pod Provisioning

```bash
# Create StatefulSet with 3 replicas
kubectl apply -f statefulset.yaml

# Watch pods come up one at a time
kubectl get pods --watch
```

Output:
```
NAME         READY   STATUS    RESTARTS   AGE
database-0   0/1     Pending   0          0s
database-0   0/1     Pending   0          2s
database-0   1/1     Running   0          15s    ← database-0 fully ready
database-1   0/1     Pending   0          0s     ← Then database-1 starts
database-1   0/1     Pending   0          2s
database-1   1/1     Running   0          12s    ← database-1 fully ready
database-2   0/1     Pending   0          0s     ← Then database-2 starts
database-2   0/1     Pending   0          2s
database-2   1/1     Running   0          10s    ← database-2 fully ready
```

**Key insight:** Pods start sequentially, waiting for readiness. This is important for databases like PostgreSQL where you need the primary to be ready before replicas join.

### Concept 2: Persistent Storage Persists

Unlike regular Deployments where pods are ephemeral, StatefulSet pods have:
- Stable network identity: `database-0.database-service`, `database-1.database-service`, etc.
- Stable storage: Each pod gets its own PVC that persists across restarts

```
Pod restarted? 
→ database-0 comes back up
→ Mounts the same postgres-data-database-0 PVC
→ Data is still there (persistent!)

Pod deleted from StatefulSet?
→ PVC is NOT automatically deleted
→ You must manually delete it: kubectl delete pvc postgres-data-database-0
```

### Concept 3: PVC Not Auto-Deleted

Unlike Deployments, StatefulSet PVCs are NOT deleted when the StatefulSet is deleted:

```bash
# Delete the StatefulSet
kubectl delete statefulset database

# PVCs still exist!
kubectl get pvc
```

Output:
```
NAME                          STATUS   VOLUME                                     CAPACITY   ACCESSMODES
postgres-data-database-0      Bound    pvc-abc123                                 1Gi        RWO
postgres-data-database-1      Bound    pvc-def456                                 1Gi        RWO
postgres-data-database-2      Bound    pvc-ghi789                                 1Gi        RWO
```

This is intentional—to prevent accidental data loss. You must manually delete:

```bash
# Delete the PVCs explicitly
kubectl delete pvc postgres-data-database-0 postgres-data-database-1 postgres-data-database-2

# Or delete all matching PVCs
kubectl delete pvc -l app=database
```

### Concept 4: Third-Party Storage (Ceph, etc.)

If you want to use Ceph instead of cloud provider storage:

```bash
# Install Ceph CSI driver via Helm
helm repo add ceph-csi https://ceph.github.io/csi-charts
helm install ceph-csi ceph-csi/ceph-csi-rbd \
  -n ceph-csi-rbd \
  --create-namespace
```

Then create StorageClass for Ceph:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd
provisioner: rbd.csi.ceph.io
parameters:
  clusterID: ceph-cluster-id
  pool: kubernetes-pool
  imageFeatures: layering
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

Then use it in StatefulSet:

```yaml
volumeClaimTemplates:
- metadata:
    name: postgres-data
  spec:
    accessModes: [ "ReadWriteOnce" ]
    storageClassName: ceph-rbd  # ✅ Works everywhere with Ceph installed
    resources:
      requests:
        storage: 1Gi
```

---

## Real Solution for the Developer

### Step 1: Update StatefulSet YAML

Remove hardcoded "ebs" storage class. Instead use default OR make it configurable:

**Option A - Use Default (Simplest):**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
  namespace: production
spec:
  serviceName: database-service
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:13
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql
  
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      # Don't specify storageClassName - use cluster default
      resources:
        requests:
          storage: 1Gi
```

**Option B - Make It Configurable (Best for Enterprise):**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
  namespace: production
spec:
  serviceName: database-service
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:13
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql
  
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: ${STORAGE_CLASS}  # Variable for each environment
      resources:
        requests:
          storage: 1Gi
```

Deploy with:
```bash
# AWS
STORAGE_CLASS=gp2 envsubst < statefulset.yaml | kubectl apply -f -

# GKE
STORAGE_CLASS=standard-rwo envsubst < statefulset.yaml | kubectl apply -f -

# AKS
STORAGE_CLASS=managed-premium envsubst < statefulset.yaml | kubectl apply -f -

# minikube
STORAGE_CLASS=standard envsubst < statefulset.yaml | kubectl apply -f -
```

### Step 2: Verify It Works

```bash
# Check pods start sequentially
kubectl get pods -l app=database --watch

# Check PVCs get created
kubectl get pvc

# Check all pods become ready
kubectl get statefulset database
```

---

## How to Explain This in the Interview

**The Problem:**
"The developer hardcoded the StorageClass name as 'ebs', which only exists on AWS. When they deployed the same manifest to GKE, AKS, and minikube, the PVCs couldn't find the StorageClass, so they remained Pending. With no PV to mount, the StatefulSet pods couldn't be scheduled."

**Understanding StatefulSets:**
"StatefulSets are different from regular Deployments in that they provision storage sequentially. Each pod gets its own PVC created from the volumeClaimTemplates. The StatefulSet waits for pod-0 to be fully ready, then provisions pod-1, then pod-2—not all at once. This is important for databases."

**The Root Cause:**
"I ran `kubectl get storageclass` on each cluster and saw they were all different. AWS has 'ebs' and 'gp2', GKE has 'standard-rwo' and 'premium-rwo', AKS has 'managed-premium', and minikube has 'standard'. The developer didn't account for this difference."

**The Solution:**
"There are several approaches: First, use the cluster's default StorageClass by not specifying one. Second, create a custom StorageClass named 'ebs' on each provider that maps to the provider's native storage. Third, make the StorageClass name configurable via environment variables or Helm values, so you have one manifest that works everywhere."

**Key Insight:**
"This is a common issue when moving workloads between cloud providers. StorageClasses are provider-specific. You need to either abstract away the provider name, use defaults, or have environment-specific configuration."

---

## Critical Points About StatefulSet + Storage

1. **PVCs are created per pod replica** from the volumeClaimTemplates
2. **Pods provision sequentially**, not in parallel (unlike Deployments)
3. **PVCs are NOT auto-deleted** when StatefulSet is deleted (data safety)
4. **StorageClass names are provider-specific** - AWS, GKE, AKS all have different defaults
5. **If StorageClass doesn't exist, PVC stays Pending** → Pod can't mount → Pod can't start
6. **Third-party storage** (Ceph, etc.) requires CSI driver installation + StorageClass configuration
7. **StatefulSet hostname is stable** - database-0 is always database-0 (good for clustering)
8. **Network identity is stable** - database-0.database-service always resolves to the same pod

---

## Checklist for Debugging StatefulSet Storage Issues

```bash
# 1. Check if StatefulSet exists
kubectl get statefulset

# 2. Check pod status (should see 0, 1, 2, etc. sequential naming)
kubectl get pods -l app=database

# 3. Check if PVCs were created
kubectl get pvc -l app=database

# 4. Check if StorageClass exists
kubectl get storageclass

# 5. Check PVC status (should show Bound if everything works)
kubectl describe pvc postgres-data-database-0

# 6. Check if PV was created (should exist if PVC is Bound)
kubectl get pv

# 7. If PVC is Pending, check why:
kubectl describe pvc postgres-data-database-0
# Look for "Failed to provision" or "not found" errors

# 8. Check provisioner logs:
kubectl logs -n kube-system -l app=csi-provisioner

# 9. If pod is Pending, check why:
kubectl describe pod database-0
# Look for "Insufficient PersistentVolumes" or storage-related errors

# 10. Test basic pod readiness (for databases)
kubectl exec database-0 -- pg_isready
```