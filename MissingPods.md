# Missing Pods: Troubleshooting Guide & Interview Case Study

## What Are Missing Pods?

### Definition

Missing pods is a scenario where:
```
You deploy a Deployment with X replicas specified
BUT
Fewer pods than X actually get created
AND
No pods are stuck in Pending status (so it's not a scheduling issue)
```

This is confusing because:
- Deployment shows "Desired: 5, Ready: 2" (not Pending, just missing!)
- No error messages in deployment events
- Pods quietly fail to be created
- The pod silently doesn't exist

```
Expected State:
Deployment (5 replicas desired)
├─ Pod 1: Running
├─ Pod 2: Running
├─ Pod 3: Running
├─ Pod 4: Running
└─ Pod 5: Running

Actual State:
Deployment (5 replicas desired)
├─ Pod 1: Running
├─ Pod 2: Running
└─ [3 pods missing, no error visible!]
```

---

## Root Cause 1: Namespace Resource Quota Exceeded

### The Scenario

You have a ResourceQuota on a namespace that limits the total number of pods. When that limit is reached, new pods silently fail to create.

```
Namespace: staging

ResourceQuota:
pods: 5 (hard limit)

Existing pods: data-processor (3 replicas) = 3 pods used

New Deployment: api (5 replicas wanted)
├─ Pods 1-2: Created successfully (now 5 pods total, hitting quota)
├─ Pods 3-5: FAILED to create (would exceed quota)
└─ Reason: Quota exceeded, but no visible error in Deployment!
```

### Investigation

**Step 1: Describe the deployment**

```bash
kubectl describe deployment api -n staging
```

Output:
```
Name:           api
Namespace:      staging
Replicas:       5 desired | 2 updated | 2 total
...

Conditions:
  Type             Status  Reason
  ----             ------  ------
  Available        False   MinimumReplicasUnavailable
  Progressing      True    NewReplicaSetAvailable

Status:
  Desired Replicas: 5
  Available Replicas: 2
  Unavailable Replicas: 3
```

**Observation:** Only 2 pods out of 5 are created, but they're not in Pending state. The deployment shows "MinimumReplicasUnavailable" but doesn't give root cause.

**Step 2: Check deployment events**

```bash
kubectl describe deployment api -n staging | grep -A 20 "Events:"
```

Output (might be vague):
```
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  2m    deployment-controller  Scaled up replica set api-abc123 to 5
```

Not helpful. The scaling event shows it tried to scale to 5, but doesn't say why it failed.

**Step 3: Check namespace-level events (this is the key!)**

```bash
kubectl get events -n staging --sort-by='.lastTimestamp'
```

Output (last events):
```
NAMESPACE   LAST SEEN   TYPE      REASON                OBJECT                   MESSAGE
staging     2m          Normal    Pulling               pod/api-def456           Pulling image "api:1.0"
staging     1m          Normal    Pulled                pod/api-def456           Image pulled successfully
staging     1m          Normal    Created               pod/api-def456           Created container api
staging     1m          Warning   FailedCreatePod       pod/api-ghi789           Error creating pod:
                                                                                  forbidden: exceeded quota:
                                                                                  pod-quota, requested: pods=1,
                                                                                  used: pods=5, limited: pods=5
staging     1m          Warning   FailedCreatePod       pod/api-jkl012           Error creating pod:
                                                                                  forbidden: exceeded quota:
                                                                                  pod-quota, requested: pods=1,
                                                                                  used: pods=5, limited: pods=5
```

**AHA!** There's the error: **"exceeded quota: pod-quota"**

The first 2 pods were created, hitting the quota limit (5 pods total). The remaining 3 pods couldn't be created because the quota was exceeded.

**Step 4: Check the ResourceQuota on namespace**

```bash
kubectl describe namespace staging
```

Output:
```
Name:         staging
Labels:       <none>
Annotations:  <none>
Status:       Active

Resource Quotas:
  Name:       pod-quota
  Resource    Used   Hard
  --------    ----   ----
  pods        5      5

No resource limits.
```

**Confirmed:** The namespace has a ResourceQuota limiting pods to 5 total. Since data-processor has 3 pods and api created 2 pods, the quota is exhausted. No more pods can be created.

### Solution: Increase ResourceQuota

**Option 1: Edit the ResourceQuota**

```bash
kubectl edit resourcequota pod-quota -n staging
```

Change:
```yaml
# BEFORE
spec:
  hard:
    pods: "5"

# AFTER
spec:
  hard:
    pods: "10"    # Increased quota
```

Save and exit.

**Option 2: Patch the ResourceQuota**

```bash
kubectl patch resourcequota pod-quota -n staging \
  -p '{"spec":{"hard":{"pods":"10"}}}'
```

**Step 5: Verify the quota increase**

```bash
kubectl describe resourcequota pod-quota -n staging
```

Output:
```
Resource    Used   Hard
--------    ----   ----
pods        5      10    ← Now increased!
```

### Step 6: Restart deployment to trigger recreation

```bash
# Trigger rolling restart
kubectl rollout restart deployment api -n staging

# Watch pods being created
kubectl get pods -n staging -l app=api --watch
```

Output:
```
NAME                  READY   STATUS              RESTARTS   AGE
api-abc123-abc12      1/1     Running             0          0s
api-abc123-def34      1/1     Running             0          0s
api-abc123-ghi56      0/1     ContainerCreating   0          2s
api-abc123-jkl78      0/1     ContainerCreating   0          2s
api-abc123-mno90      0/1     ContainerCreating   0          2s
```

Wait for all 5 to be running:

```bash
# Verify all replicas are now created
kubectl get pods -n staging -l app=api
```

Output:
```
NAME                  READY   STATUS    RESTARTS   AGE
api-abc123-abc12      1/1     Running   0          5m
api-abc123-def34      1/1     Running   0          5m
api-abc123-ghi56      1/1     Running   0          3m
api-abc123-jkl78      1/1     Running   0          3m
api-abc123-mno90      1/1     Running   0          3m
```

All 5 pods now running!

### Why This Happens

ResourceQuotas are often applied by cluster admins to:
- Limit usage in dev/staging environments
- Prevent one team from consuming all cluster resources
- Ensure fair sharing of resources

But sometimes they're set too low for actual workload requirements.

### Best Practice: Set Reasonable Quotas

When creating a namespace, set quotas that accommodate expected workloads:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: staging-quota
  namespace: staging
spec:
  hard:
    pods: "50"              # Reasonable limit for staging
    requests.cpu: "20"
    requests.memory: "20Gi"
    limits.cpu: "40"
    limits.memory: "40Gi"
```

---

## Root Cause 2: Missing ServiceAccount

### The Scenario

A pod references a ServiceAccount that doesn't exist in the namespace. The pod fails to create because it can't find the account.

```
Deployment spec:
serviceAccountName: analytics-sa

Existing ServiceAccounts in namespace:
- default

Result: Pod can't be created (analytics-sa doesn't exist!)
```

### Investigation

**Step 1: Describe the deployment (looks fine)**

```bash
kubectl describe deployment analytics -n staging
```

Output:
```
Name:           analytics
Replicas:       1 desired | 0 updated | 0 total
...

Conditions:
  Type             Status  Reason
  ----             ------  ------
  Available        False   MinimumReplicasUnavailable
  Progressing      False   ProgressDeadlineExceeded

Status:
  Desired Replicas: 1
  Available Replicas: 0
  Updated Replicas: 0
```

**Observation:** No pods created, deployment shows "ProgressDeadlineExceeded" but no clear reason why.

**Step 2: Check deployment events (not helpful)**

```bash
kubectl describe deployment analytics -n staging | grep -A 10 "Events:"
```

Output:
```
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  1m    deployment-controller  Scaled up replica set analytics-abc123 to 1
```

Still no root cause shown.

**Step 3: Check namespace events (this reveals it!)**

```bash
kubectl get events -n staging --sort-by='.lastTimestamp' | tail -20
```

Output:
```
NAMESPACE   LAST SEEN   TYPE      REASON                OBJECT                      MESSAGE
staging     1m          Warning   FailedCreatePod       pod/analytics-abc123-def56  Error creating pod:
                                                                                    error looking up
                                                                                    service account
                                                                                    staging/analytics-sa:
                                                                                    serviceaccount
                                                                                    "analytics-sa"
                                                                                    not found
```

**Found it!** ServiceAccount "analytics-sa" not found!

**Step 4: Check what ServiceAccounts exist**

```bash
kubectl get serviceaccounts -n staging
```

Output:
```
NAME      SECRETS   AGE
default   1         30d
```

Only the default ServiceAccount exists. The "analytics-sa" doesn't exist.

**Step 5: Check deployment spec to see what account it's looking for**

```bash
kubectl get deployment analytics -n staging -o yaml | grep -A 5 "serviceAccount"
```

Output:
```
spec:
  template:
    spec:
      serviceAccountName: analytics-sa
```

The deployment explicitly requests "analytics-sa".

### Solution: Create the Missing ServiceAccount

**Option 1: Create using kubectl**

```bash
kubectl create serviceaccount analytics-sa -n staging
```

Verify it was created:

```bash
kubectl get serviceaccounts -n staging
```

Output:
```
NAME            SECRETS   AGE
analytics-sa    1         5s
default         1         30d
```

**Option 2: Create using YAML manifest**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: analytics-sa
  namespace: staging
```

Apply it:

```bash
kubectl apply -f serviceaccount.yaml
```

### Step 6: Restart deployment

```bash
kubectl rollout restart deployment analytics -n staging

# Watch pod being created
kubectl get pods -n staging -l app=analytics --watch
```

Output:
```
NAME                      READY   STATUS              RESTARTS   AGE
analytics-abc123-def56    0/1     ContainerCreating   0          2s
analytics-abc123-def56    1/1     Running             0          5s
```

Pod now running!

### ServiceAccount Purpose

ServiceAccounts are used for:
- Authenticating pods to Kubernetes API
- Providing credentials for RBAC (Role-Based Access Control)
- Allowing pods to access cluster resources

Every pod needs a ServiceAccount (uses default if not specified).

### Common ServiceAccount Issues

**Scenario 1: Pod needs to read secrets**

```yaml
# Pod definition references service account with permissions
apiVersion: v1
kind: Pod
metadata:
  name: secret-reader
spec:
  serviceAccountName: secret-reader-sa
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/secrets
  volumes:
  - name: secret-vol
    secret:
      secretName: my-secret

---
# But if secret-reader-sa doesn't exist:
# Error: serviceaccount "secret-reader-sa" not found
# Fix: Create the serviceaccount
kubectl create serviceaccount secret-reader-sa
```

**Scenario 2: Pod needs cluster-admin permissions**

```yaml
# Create service account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-sa
  namespace: default

---
# Bind it to cluster-admin role
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-sa-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-sa
  namespace: default
```

---

## Root Cause 3: Missing Dependencies (Secrets, ConfigMaps, etc.)

Similar to ServiceAccount, pods might reference other resources that don't exist:

### Investigation for Missing Dependencies

```bash
# Check events for clues
kubectl get events -n staging --sort-by='.lastTimestamp'

# Look for errors like:
# - secret "my-secret" not found
# - configmap "my-config" not found
# - persistentvolumeclaim "my-pvc" not found
```

### Common Missing Dependencies

**Missing Secret:**
```yaml
spec:
  containers:
  - name: app
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret      # ← This secret might not exist!
          key: password
```

Fix:
```bash
kubectl create secret generic db-secret \
  --from-literal=password=mysecretpassword \
  -n staging
```

**Missing ConfigMap:**
```yaml
spec:
  containers:
  - name: app
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: app-config        # ← This configmap might not exist!
```

Fix:
```bash
kubectl create configmap app-config \
  --from-file=config.yaml=./config.yaml \
  -n staging
```

**Missing PersistentVolumeClaim:**
```yaml
spec:
  containers:
  - name: app
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data-pvc     # ← This PVC might not exist!
```

Fix:
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
  namespace: staging
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
EOF
```

---

## Troubleshooting Checklist for Missing Pods

```bash
# 1. Check if pods are missing (not pending, just missing)
kubectl get deployment <deployment-name>
# Look for: Replicas: X desired | Y ready
# If Y < X and pods aren't Pending, they're missing

# 2. Describe deployment (might not show root cause)
kubectl describe deployment <deployment-name>

# 3. Check deployment events (still might not be helpful)
kubectl describe deployment <deployment-name> | grep -A 20 "Events:"

# 4. CHECK NAMESPACE EVENTS (this is where errors are!)
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
# Look for: FailedCreatePod, exceeded quota, not found

# 5. If quota error:
kubectl describe namespace <namespace>
# Check ResourceQuota section

# 6. If serviceaccount error:
kubectl get serviceaccounts -n <namespace>
# Verify required account exists

# 7. If secret/configmap error:
kubectl get secrets,configmaps -n <namespace>
# Verify required resources exist

# 8. If PVC error:
kubectl get pvc -n <namespace>
# Verify required PVCs exist

# 9. Get detailed pod events (for pods that did create)
kubectl describe pod <pod-name>
```

---

## Real Debugging Example: Missing Pods Scenario

```bash
# Alert: "Deployment has fewer pods than expected"

# Step 1: Check deployment
kubectl get deployment api-service -n staging
# OUTPUT: Replicas: 5 desired | 2 updated | 2 total
# Only 2 out of 5 pods created!

# Step 2: Check if pods are pending
kubectl get pods -n staging -l app=api-service
# OUTPUT: All pods are Running or Pending
# No, we only see 2 pods. 3 are completely missing!

# Step 3: Check deployment events (vague)
kubectl describe deployment api-service -n staging | tail -20
# No helpful info about why pods aren't being created

# Step 4: Check namespace events (THIS IS KEY!)
kubectl get events -n staging --sort-by='.lastTimestamp'
# OUTPUT:
# FailedCreatePod: exceeded quota: pod-quota,
#                  requested: pods=1, used: pods=5, limited: pods=5

# Root cause found: ResourceQuota exceeded!

# Step 5: Check quota
kubectl describe namespace staging
# OUTPUT: ResourceQuota pod-quota with hard limit of 5 pods

# Step 6: Increase quota
kubectl edit resourcequota pod-quota -n staging
# Change: pods: "5" → pods: "10"

# Step 7: Restart deployment
kubectl rollout restart deployment api-service -n staging

# Step 8: Verify all pods now exist
kubectl get pods -n staging -l app=api-service
# OUTPUT: Now showing all 5 pods!
```

---

## Interview Story: Missing Pods

**Setup:**
"We had a deployment that should have created 5 replicas, but only 2 pods were actually created. The confusing part was there were no error messages visible in the deployment itself."

**Investigation Process:**

1. **First Instinct (Wrong):** 
"I initially thought it was a scheduling issue and checked if pods were in Pending state. But there were no Pending pods—just 2 running and 3 completely missing. That was weird."

2. **Checked Deployment Events:**
"I ran `kubectl describe deployment` but the events only showed it tried to scale to 5, without explaining why it failed. No obvious errors."

3. **Key Insight:**
"I realized to check NAMESPACE-level events instead of just deployment events. That's where I found the real error: 'exceeded quota: pod-quota'. The namespace had a ResourceQuota limiting total pods to 5. Since another deployment already had 3 pods, there was only room for 2 more."

4. **Fixed the Quota:**
"I increased the ResourceQuota from 5 to 10 pods, restarted the deployment, and all 5 pods were created immediately."

5. **Second Example - Missing ServiceAccount:**
"Then we had another deployment that couldn't create any pods. Checking namespace events showed: 'serviceaccount not found'. The deployment was trying to use a ServiceAccount that didn't exist. I created the missing ServiceAccount and restarted the deployment."

**Key Insight:**
"The critical lesson: when pods are missing (not Pending, just not existing), always check namespace-level events with `kubectl get events`. That's where the real error messages are. The deployment events are too vague. Also, check for ResourceQuotas, missing ServiceAccounts, and missing dependencies like Secrets and ConfigMaps."

---

## Key Differences: Missing vs Pending

| Condition | Missing Pods | Pending Pods |
|-----------|------------|--------------|
| **What it means** | Pod object not created at all | Pod created but can't be scheduled |
| **Visible in `kubectl get pods`?** | No, doesn't show in pod list | Yes, shows as "Pending" |
| **Error Location** | Namespace events (not deployment events) | Pod events or deployment events |
| **Common Causes** | ResourceQuota, missing ServiceAccount, missing dependencies | Insufficient resources, node labels, taints |
| **Fix Approach** | Increase quota or create missing resources | Reduce requests, add labels, add tolerations |

---

## Common Mistakes When Debugging Missing Pods

❌ **Mistake 1:** Only checking deployment events
✅ **Fix:** Also check namespace events with `kubectl get events`

❌ **Mistake 2:** Assuming it's a scheduling issue
✅ **Fix:** Verify pods actually exist (or don't) with `kubectl get pods`

❌ **Mistake 3:** Not reading the full error message
✅ **Fix:** Look for "exceeded quota" or "not found" keywords in events

❌ **Mistake 4:** Not checking ResourceQuota or ServiceAccount
✅ **Fix:** Always verify these prerequisites exist

❌ **Mistake 5:** Not restarting deployment after fixing quota/dependencies
✅ **Fix:** Even after fixing, need to `kubectl rollout restart` for pods to be created

❌ **Mistake 6:** Assuming pods are in Pending state
✅ **Fix:** Check with `kubectl get pods` - if nothing shows, they're missing (different issue!)

---

## Quick Reference: Debugging Missing Pods

| Error Message | Root Cause | Solution |
|---------------|-----------|----------|
| "exceeded quota" | ResourceQuota limit reached | Increase quota or decrease replicas |
| "service account not found" | ServiceAccount doesn't exist | Create the ServiceAccount |
| "secret not found" | Referenced secret missing | Create the secret |
| "configmap not found" | Referenced ConfigMap missing | Create the ConfigMap |
| "persistentvolumeclaim not found" | Referenced PVC missing | Create the PVC |
| "no nodes available" | All nodes tainted/unavailable | Add nodes or remove taints |

---

## Key Takeaways

1. **Missing pods ≠ Pending pods** → Different issues requiring different solutions

2. **Check namespace events** → This is where real error messages appear
   ```bash
   kubectl get events -n <namespace> --sort-by='.lastTimestamp'
   ```

3. **ResourceQuota limits** → Most common cause of missing pods in shared clusters

4. **Missing dependencies** → ServiceAccount, Secrets, ConfigMaps, PVCs must exist

5. **Deployment events are vague** → They show "ScalingReplicaSet" but not why it failed

6. **Restart deployment after fixing** → Pod creation is retried after restart
   ```bash
   kubectl rollout restart deployment <name>
   ```

7. **Always verify prerequisites exist** → Before blaming Kubernetes, check your resources

---

## Real-World Best Practices

**1. Set Reasonable ResourceQuotas**

```yaml
# staging namespace - allow reasonable workload
apiVersion: v1
kind: ResourceQuota
metadata:
  name: staging-quota
  namespace: staging
spec:
  hard:
    pods: "100"
    requests.cpu: "50"
    requests.memory: "50Gi"
```

**2. Document Required ServiceAccounts**

```yaml
# In deployment comments or docs:
# This deployment requires:
# - ServiceAccount: my-app-sa
# - Secret: my-app-credentials
# - ConfigMap: my-app-config
```

**3. Use Init Containers to Verify Dependencies**

```yaml
spec:
  initContainers:
  - name: check-secrets
    image: busybox:latest
    command: ['sh', '-c', 'test -f /etc/secrets/key && echo "Secrets found"']
    volumeMounts:
    - name: secrets
      mountPath: /etc/secrets
  containers:
  - name: app
    image: myapp:1.0
```

**4. Monitor and Alert on Missing Pods**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: missing-pods-alert
spec:
  groups:
  - name: pod.rules
    rules:
    - alert: UndesiredReplicasUnavailable
      expr: |
        kube_deployment_status_replicas_unavailable{namespace="staging"} > 0
      for: 5m
      annotations:
        summary: "Deployment {{ $labels.deployment }} has unavailable replicas"
```

---

## Prevention Checklist Before Deploying

Before applying a new deployment, verify:

```bash
# 1. Namespace exists
kubectl get namespace <namespace>

# 2. ResourceQuota allows new pods
kubectl describe resourcequota -n <namespace>

# 3. Required ServiceAccounts exist
kubectl get serviceaccounts -n <namespace>

# 4. Required Secrets exist
kubectl get secrets -n <namespace>

# 5. Required ConfigMaps exist
kubectl get configmaps -n <namespace>

# 6. Required PVCs exist (if using storage)
kubectl get pvc -n <namespace>

# 7. Node capacity is available
kubectl top nodes
kubectl describe nodes | grep -A 10 "Allocated resources"

# 8. No node taints blocking pod scheduling
kubectl describe nodes | grep Taints:

# Only after verifying all above:
kubectl apply -f deployment.yaml
```