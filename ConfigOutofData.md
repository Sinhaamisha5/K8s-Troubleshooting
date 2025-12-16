# ConfigMap/Secret Changes Don't Auto-Update Pods: The Immutability Gotcha

## The Problem: The Immutability Principle

### What Happens

You update a ConfigMap or Secret, but pods using it continue with **old values**. Changes don't automatically reflect in running pods.

```
Timeline:
09:00 - Pod starts, reads ConfigMap "message: hello world"
09:05 - You edit ConfigMap: "message: hello KodeKloud"
09:06 - Pod still sees: "message: hello world"
       (Environment variables unchanged!)
09:10 - You restart pod
09:11 - Pod starts, reads new ConfigMap: "message: hello KodeKloud"
```

### Why This Happens: Immutability in Kubernetes

```
Kubernetes Principle: Resources don't change in place
Instead: Create new resource to replace old one

Example - Changing Image:
OLD: Image: v1.0
CHANGE: Kubernetes doesn't update in place
INSTEAD: Creates new pod with Image: v2.0
         Deletes old pod with Image: v1.0

Same with ConfigMap/Secret in Environment Variables:
OLD: ENV VAR = old value (read when pod started)
CHANGE: ConfigMap updated
KUBERNETES DOESN'T: Update environment variable in running pod
INSTEAD: Pod must restart to read new ConfigMap

🔹 Which Kubernetes resource continuously watches for changes?
✅ Controllers (Control Plane components)

Kubernetes controllers continuously watch the API Server for state changes.

Examples:
Deployment Controller
ReplicaSet Controller
StatefulSet Controller
DaemonSet Controller
Node Controller
HPA Controller

👉 Controllers use a watch mechanism to:

Observe desired state vs actual state

Take corrective action automatically

📌 Pods themselves do NOT watch for changes — they are passive.
```
```
Definition
A ConfigMap is a Kubernetes object used to store non-sensitive configuration data in key-value format.
Why ConfigMap Is Used

  Externalizes configuration from application code
  Avoids rebuilding container images for config changes
  Enables environment-specific configs (dev, qa, prod)
  Improves portability and maintainability
```
### Critical Understanding

```
Pod reads ConfigMap/Secret at STARTUP
NOT continuously watching for changes

Pod Memory:
┌─────────────────────────────────┐
│ Environment Variables           │
│ (set at pod start time)         │
│                                 │
│ MESSAGE=hello world             │ ← Snapshot from ConfigMap
│ DB_HOST=mysql-svc               │ ← Snapshot from Secret
│ DB_PASS=mysecretpass            │ ← Snapshot from Secret
└─────────────────────────────────┘
     ↑
     Locked in at startup
     Never re-read while pod running!
```

---

## Real Example 1: ConfigMap Changes

### The Setup

**ConfigMap Definition:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-message
  namespace: production
data:
  message: "hello world"
```

**Deployment Using ConfigMap:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: web
        image: web-app:1.0
        env:
        - name: MESSAGE           # ← Environment variable name
          valueFrom:
            configMapKeyRef:
              name: web-message   # ← References ConfigMap
              key: message        # ← References key in ConfigMap
```

### Step 1: Pod Starts and Reads ConfigMap

```bash
# Pod starts
kubectl get pods -n production
# STATUS: Running

# Access web app
curl localhost:3000
# Response: "hello world"

# Check environment variable inside pod
kubectl exec -it web-app-pod-xyz -- env | grep MESSAGE
# Output: MESSAGE=hello world
```

### Step 2: Edit ConfigMap (The Change)

```bash
# Edit ConfigMap
kubectl edit configmap web-message -n production
```

Change:
```yaml
# BEFORE
data:
  message: "hello world"

# AFTER
data:
  message: "hello KodeKloud"
```

Save and exit.

### Step 3: Pod DOESN'T See the Change

```bash
# Access web app again
curl localhost:3000
# Response: "hello world"  ← STILL OLD!

# Check environment variable
kubectl exec -it web-app-pod-xyz -- env | grep MESSAGE
# Output: MESSAGE=hello world  ← STILL OLD!
```

**Why?** Pod read ConfigMap at startup. Still has old value in memory. ConfigMap is just external data source—pod doesn't watch it.

### Step 4: Restart Pod to Apply Change

```bash
# Option 1: Restart single pod
kubectl delete pod web-app-pod-xyz -n production
# Deployment creates new pod, which reads new ConfigMap

# Option 2: Restart entire deployment (better)
kubectl rollout restart deployment web-app -n production
# Deletes all pods, deployment creates new ones
```

Pod restart process:
```
Old Pod Terminating...
  ↓
New Pod Starting...
  ↓
New Pod reads ConfigMap
  ↓
New Pod sees: MESSAGE=hello KodeKloud
  ↓
Access web app
```

### After Restart

```bash
# Access web app after restart
curl localhost:3000
# Response: "hello KodeKloud"  ← NOW UPDATED!

# Check environment variable
kubectl exec -it web-app-pod-new -- env | grep MESSAGE
# Output: MESSAGE=hello KodeKloud  ← NOW UPDATED!
```

---

## Real Example 2: Secret Changes (More Critical)

### The Scenario

A Secret contains a database password. Password is updated, but pods still have old password cached.

```
Deployed App:
└─ Two-tier app (uses MySQL database)
   ├─ Pod sees: DB_PASS from Secret = "oldpass123"
   ├─ Database updated: password changed to "newpass456"
   └─ Pod still tries: "oldpass123" → Connection FAILS!
```

### The Setup

**Secret Definition:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
data:
  mysql-host: bXlzcWwsc3ZjCg==       # base64: mysql.svc
  mysql-root-password: bW5zd3Jvb3Q=  # base64: mnswroot (TYPO!)
```

**Deployment Using Secret:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: two-tier-app
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: app
        image: two-tier-app:1.0
        env:
        - name: MYSQL_HOST
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: mysql-host
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: mysql-root-password
        
        # Readiness probe checks if can connect to MySQL
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - mysql -h $MYSQL_HOST -p$MYSQL_PASSWORD -e "SELECT 1"
          initialDelaySeconds: 10
          periodSeconds: 5
```

### Investigation: Why Pod Not Ready?

```bash
kubectl get pods -n production two-tier-app-xyz
# STATUS: Running, READY: 0/1
# Readiness probe is failing!

kubectl describe pod two-tier-app-xyz -n production
# Events show: Readiness probe failed
# Reason: Error: "Unknown server host 'mnswroot'"

# That's weird, let me check the secret
```

### Checking the Secret

```bash
# View secret
kubectl get secret app-secrets -n production -o yaml
```

Output:
```yaml
data:
  mysql-host: bXlzcWwsc3ZjCg==
  mysql-root-password: bW5zd3Jvb3Q=
```

Decode the password:
```bash
echo "bW5zd3Jvb3Q=" | base64 -d
# Output: mnswroot  ← TYPO!

# Should be: mypassword
```

### Someone Fixed the Secret

```bash
# Cluster admin noticed the typo and fixed it:
kubectl edit secret app-secrets -n production
```

Changed:
```yaml
# BEFORE (typo)
mysql-root-password: bW5zd3Jvb3Q=  # base64: mnswroot

# AFTER (fixed)
mysql-root-password: bXlwYXNzd29yZA==  # base64: mypassword
```

### But Pod Still Fails!

```bash
# Check pod again
kubectl get pods -n production two-tier-app-xyz
# STATUS: Running, READY: 0/1  ← STILL NOT READY!

# Exec into pod and check environment variable
kubectl exec -it two-tier-app-xyz -n production -- sh
```

Inside pod:
```bash
# Check what password the pod sees
echo $MYSQL_PASSWORD
# Output: mnswroot  ← STILL OLD VALUE!

# This is wrong! Secret was fixed but pod wasn't restarted!
```

### Why Pod Didn't Pick Up Fix

Pod started at 09:00 with `MYSQL_PASSWORD=mnswroot` (from old secret).
Secret was updated at 09:30.
Pod still has `MYSQL_PASSWORD=mnswroot` in its environment.
Pod readiness probe keeps failing because password is wrong.

### Solution: Restart Pod

```bash
# Restart the deployment to pick up new secret
kubectl rollout restart deployment two-tier-app -n production

# Wait for pod to restart
kubectl get pods -n production -l app=two-tier-app --watch
```

Pod restart process:
```
Old Pod Terminating...
  ↓
New Pod Starting...
  ↓
New Pod reads Secret (gets fixed password)
  ↓
New Pod: MYSQL_PASSWORD=mypassword
  ↓
Readiness probe: Can now connect to MySQL
  ↓
Pod transitions to Ready
```

### After Restart

```bash
# Verify pod is ready
kubectl get pods -n production two-tier-app-xyz
# STATUS: Running, READY: 1/1  ← NOW READY!

# Verify environment variable
kubectl exec -it two-tier-app-xyz -n production -- sh
echo $MYSQL_PASSWORD
# Output: mypassword  ← NOW CORRECT!

# Verify readiness probe passing
kubectl describe pod two-tier-app-xyz -n production
# Events: Readiness probe passed
```

---

## Understanding Environment Variables vs Mounted Files

### Environment Variables (DON'T AUTO-UPDATE)

```yaml
env:
- name: MY_VAR
  valueFrom:
    configMapKeyRef:
      name: my-config
      key: mykey
```

Behavior:
- Read at pod startup
- Stored in pod memory
- Never updated while pod running
- **Requires pod restart to update**

### Mounted ConfigMap/Secret (AUTO-UPDATES!)

```yaml
volumeMounts:
- name: config
  mountPath: /etc/config

volumes:
- name: config
  configMap:
    name: my-config
```

Behavior:
- File appears in container filesystem
- Kubernetes syncs updates in background
- Pod can read new file content immediately
- **NO pod restart needed!**

**Example:**
```bash
# ConfigMap mounted as volume
kubectl get pod myapp-xyz -- cat /etc/config/mykey
# Output: original value

# Edit ConfigMap
kubectl edit configmap my-config
# Change value

# Check pod sees new value immediately!
kubectl get pod myapp-xyz -- cat /etc/config/mykey
# Output: new value  ← Updated without restart!
```

---

## The Real-World Problem

### Small Scale (Easy to Manage)

```
2 deployments
3 ConfigMaps
2 Secrets

You change 1 ConfigMap
Easy to remember: "Oh, only web-app uses this"
Restart: kubectl rollout restart deployment web-app
```

### Large Scale (Difficult to Manage)

```
100+ deployments
500+ ConfigMaps
200+ Secrets

You change 1 ConfigMap
Question: Which deployments use this ConfigMap?
Answer: ???

Deployment1 uses it? Yes
Deployment2 uses it? Maybe
Deployment3 uses it? Unknown
...

Now you have to:
1. Search for which deployments reference it
2. Restart all of them
3. Hope you didn't miss any

Risk: Forgotten deployment still has old config!
```

### The Tracking Problem

```
ConfigMap: payment-api-config

Who uses it?
- payment-api deployment (confirmed)
- notification-service deployment (confirmed)
- reporting-service deployment (possibly?)
- legacy-app deployment (unknown)

If you only restart first 2, legacy-app still has old config!
Months later: Legacy app uses wrong payment endpoint
               But you forgot it uses payment-api-config
               PRODUCTION BUG!
```

---

## Troubleshooting Checklist: Config/Secret Not Updating

```bash
# 1. Check pod is using ConfigMap/Secret via environment variable
kubectl get pod <pod> -o yaml | grep -A 5 "valueFrom:"

# 2. Check ConfigMap/Secret has been updated
kubectl get configmap <cm-name> -o yaml
kubectl get secret <secret-name> -o yaml

# 3. Verify pod environment variable still has OLD value
kubectl exec <pod> -- env | grep <VAR_NAME>

# 4. Root cause confirmed: Pod not restarted after config change

# 5. Restart pod/deployment
kubectl rollout restart deployment <deployment>

# 6. Verify pod now has NEW value
kubectl exec <pod> -- env | grep <VAR_NAME>

# 7. Verify application working with new config
# (e.g., readiness probe passing, service responding correctly)
```

---

## Interview Story: ConfigMap/Secret Auto-Update Gotcha

**Setup:**
"Early in my Kubernetes career, I made a mistake that taught me an important lesson about how Kubernetes handles configuration changes."

**The Mistake:**
"We had a web app using a ConfigMap for its greeting message. I changed the ConfigMap, expecting the pod to automatically pick up the change. But when I accessed the service, it still returned the old message. I thought there was a bug in the app or a caching issue."

**Investigation:**
"I exec'd into the pod and checked the environment variable. It still had the old value. That's when I realized: the pod reads the ConfigMap at startup and caches the value in memory. Just changing the ConfigMap doesn't tell the pod to re-read it."

**The Real Example (More Critical):**
"Later, we had a more serious issue with Secrets. A database password had a typo that was fixed in the Secret. But the pod that connects to the database kept failing its readiness probe with 'connection refused'. The password was fixed in the Secret, but the pod was using the old (incorrect) password from when it started."

**The Solution:**
"I restarted the pod with `kubectl rollout restart deployment`. The new pod read the updated Secret and immediately connected to the database successfully."

**Key Insight:**
"The critical lesson: Kubernetes doesn't dynamically update environment variables. Pods read ConfigMap/Secret values at startup and cache them. If you change external configuration, you MUST restart the pods that use it. This is by design—Kubernetes follows an immutability principle where resources don't change in place; instead, you create new ones."

**The Gotcha in Production:**
"This becomes a real problem at scale. You change one ConfigMap used by 20 deployments, but forget to restart half of them. Those deployments continue with stale configuration for hours or days until someone notices. This is why mounted volumes (which auto-update) are often better than environment variables for configuration that changes frequently."

---

## Best Practices to Avoid This Issue

### 1. Use Mounted Volumes for Frequently-Changing Config

Instead of environment variables:
```yaml
# DON'T: Environment variables (don't auto-update)
env:
- name: CONFIG
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: settings

# DO: Mounted volume (auto-updates)
volumeMounts:
- name: config
  mountPath: /etc/config
volumes:
- name: config
  configMap:
    name: app-config
```

### 2. Document ConfigMap/Secret Dependencies

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: payment-api-config
  namespace: production
  annotations:
    description: "Configuration for payment API"
    used-by-deployments: |
      - payment-api (v1.0+)
      - notification-service (v2.1+)
      - reporting-service (v3.0+)
    update-note: "Restart these deployments after any change"
```

### 3. Automate Pod Restart on Config Change

Use tools like:
- **Reloader** - Watches ConfigMaps/Secrets, auto-restarts pods on change
- **Sealed Secrets** - Automated secret management
- **Flux/ArgoCD** - GitOps tools that manage rollouts

```bash
# Reloader annotation (auto-restarts pod when ConfigMap changes)
annotations:
  reloader.stakater.com/match: "true"
```

### 4. Use Config as Volume Mounts for Dynamic Content

```yaml
# Application reads from file, not environment variable
# File updates automatically when ConfigMap changes
containers:
- name: app
  image: myapp:1.0
  volumeMounts:
  - name: config
    mountPath: /etc/config
volumes:
- name: config
  configMap:
    name: app-config
```

### 5. Include Hash/Checksum in Pod Spec

This forces pod restart when ConfigMap changes:

```yaml
annotations:
  config-checksum: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

When ConfigMap changes, checksum changes, which triggers pod restart.

---

## Comparison: Environment Variables vs Mounted Volumes

| Aspect | Environment Variables | Mounted Volumes |
|--------|----------------------|-----------------|
| **Read timing** | At pod startup only | Anytime from filesystem |
| **Updates** | ❌ Never | ✅ Automatically |
| **Pod restart needed?** | ✅ Yes | ❌ No |
| **Use case** | Static config, don't change | Dynamic config, changes often |
| **Performance** | Faster (in memory) | Slightly slower (file reads) |
| **Best for** | Secrets, passwords | Configuration files, settings |

---

## Key Takeaways

1. **Environment variables are immutable** → Read at pod startup, never change

2. **ConfigMap/Secret updates don't auto-reflect** → Pod must restart to see changes

3. **Pod reads config at startup** → Value cached in memory forever

4. **Must restart pod after config change** → `kubectl rollout restart deployment <name>`

5. **Mounted volumes are better for dynamic config** → Auto-update without restart

6. **At scale, tracking becomes difficult** → Easy to forget which pods use which config

7. **Always verify after changing config** → Check pod environment and test functionality

8. **Consider automation** → Tools like Reloader auto-restart pods on config change

9. **Document dependencies** → Which deployments use which ConfigMaps/Secrets

10. **Use volumes for frequently-changing config** → Environment variables for static values

---

## Real-World Scenarios

**Scenario 1: Expired JWT Token**
```
Secret contains JWT token that expires
Token updated in Secret
But 50 deployments still using old token
Half of them can't authenticate
Time to find and restart all of them: hours

Better: Store token in volume, app re-reads periodically
```

**Scenario 2: Database Connection String**
```
Database URL changes
Updated in ConfigMap
But 30 deployments still connect to old database
Connections fail silently
Takes hours to debug

Better: Use mounted volume, app re-reads on reconnect
```

**Scenario 3: Feature Flag**
```
Feature flag needs to be toggled off immediately
Updated in ConfigMap
But 100 deployments still have feature enabled
Feature keeps causing issues
Must manually restart all 100 deployments

Better: App reads feature flag from mounted file each request
```

---

## Prevention Checklist Before Changing Config

```bash
# BEFORE changing ConfigMap/Secret:

# 1. Document which deployments use it
grep -r "name: my-config" k8s-manifests/

# 2. Verify which pods are currently running
kubectl get pods -o wide

# 3. Make the ConfigMap/Secret change
kubectl edit configmap my-config

# 4. Immediately restart ALL affected deployments
kubectl rollout restart deployment deployment1
kubectl rollout restart deployment deployment2
kubectl rollout restart deployment deployment3

# 5. Verify all pods transitioned to new ones
kubectl get pods -w

# 6. Verify new pods see updated config
kubectl exec <new-pod> -- env | grep <VAR>

# 7. Verify application working
# (health checks, functionality tests, etc.)
```



# Reloader: Automatic Pod Restart on ConfigMap/Secret Changes

## The Problem Reloader Solves

As we learned in the previous guide: ConfigMap and Secret changes don't automatically update pods. You must manually restart them.

```
OLD WORKFLOW (Without Reloader):
1. Change ConfigMap
2. Remember which deployments use it
3. Manually restart each deployment
4. Risk forgetting some deployments
5. Production bugs occur weeks later

NEW WORKFLOW (With Reloader):
1. Change ConfigMap
2. Reloader automatically detects change
3. Reloader auto-restarts affected deployments
4. No manual intervention needed
5. Guaranteed all pods get new config
```

---

## What is Reloader?

Reloader is a Kubernetes controller that:
- Watches ConfigMaps and Secrets
- Detects when they change
- Automatically performs rolling restart of affected Deployments, DaemonSets, and StatefulSets
- Requires just one annotation on your workload

**GitHub:** https://github.com/stakater/Reloader

---

## How Reloader Works

### The Annotation

Add one annotation to your Deployment/StatefulSet/DaemonSet:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  annotations:
    reloader.stakater.com/match: "true"  # ← Add this
spec:
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:1.0
        env:
        - name: CONFIG_MESSAGE
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: message
```

### How It Works Behind the Scenes

```
1. Reloader controller starts
2. Reads all Deployments with annotation: reloader.stakater.com/match: "true"
3. For each deployment, identifies all ConfigMaps and Secrets it uses
4. Watches those ConfigMaps and Secrets for changes
5. When change detected:
   ├─ Perform rolling restart (kubectl rollout restart)
   ├─ Delete old pods
   ├─ Create new pods (which read new config)
   └─ No downtime if replicas > 1
```

---

## Installation

### Option 1: Using Kubectl (Vanilla Manifests)

```bash
# Clone the Reloader repository
git clone https://github.com/stakater/Reloader.git
cd Reloader

# Deploy using manifests
kubectl apply -f deployments/kubernetes/

# Verify deployment
kubectl get deployment -n default reloader
# Or check in kube-system namespace depending on manifests
```

### Option 2: Using Helm (Recommended)

```bash
# Add Helm repo
helm repo add stakater https://stakater.github.io/stakater-helm-charts
helm repo update

# Install Reloader
helm install reloader stakater/reloader \
  --namespace default \
  --create-namespace

# Verify installation
kubectl get deployment reloader
kubectl get pods -l app.kubernetes.io/name=reloader
```

### Option 3: Using Helm with Custom Values

```bash
# Install with custom namespace and configuration
helm install reloader stakater/reloader \
  --namespace reloader-system \
  --create-namespace \
  --set reloader.deployment.env.logFormat=json \
  --set reloader.deployment.replicas=2
```

### Verify Installation

```bash
# Check Reloader pod is running
kubectl get pods --all-namespaces | grep reloader

# Check logs
kubectl logs -l app.kubernetes.io/name=reloader -n default

# Should see log entries like:
# "Watching ConfigMaps and Secrets for changes..."
```

---

## Real Example: Using Reloader

### Setup: Web App with ConfigMap

**Step 1: Create ConfigMap**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-message
  namespace: production
data:
  message: "hello world"
```

Apply it:
```bash
kubectl apply -f configmap.yaml
```

**Step 2: Create Deployment WITH Reloader Annotation**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
  annotations:
    reloader.stakater.com/match: "true"  # ← KEY: Enable Reloader
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web
        image: web-app:1.0
        env:
        - name: MESSAGE
          valueFrom:
            configMapKeyRef:
              name: web-message
              key: message
        ports:
        - containerPort: 3000
```

Apply it:
```bash
kubectl apply -f deployment.yaml
```

**Step 3: Verify Initial State**

```bash
# Check pod
kubectl get pods -n production
# NAME                      READY   STATUS    RESTARTS   AGE
# web-app-abc123-abc12      1/1     Running   0          2m
# web-app-abc123-def34      1/1     Running   0          2m

# Check message
curl localhost:3000
# Response: "hello world"

# Check environment variable inside pod
kubectl exec -it web-app-abc123-abc12 -n production -- env | grep MESSAGE
# MESSAGE=hello world
```

**Step 4: Edit ConfigMap (The Change)**

```bash
kubectl edit configmap web-message -n production
```

Change:
```yaml
# BEFORE
data:
  message: "hello world"

# AFTER
data:
  message: "hello reloader"
```

**Step 5: Watch Reloader Do Its Magic**

Watch pods in real-time:
```bash
kubectl get pods -n production --watch
```

Output:
```
NAME                      READY   STATUS        RESTARTS   AGE
web-app-abc123-abc12      1/1     Running       0          5m
web-app-abc123-def34      1/1     Running       0          5m

# ConfigMap changed, Reloader detected it!
# Rolling restart begins...

web-app-abc123-abc12      1/1     Terminating   0          5m
web-app-abc123-ghi56      0/1     Pending       0          1s

web-app-abc123-def34      1/1     Terminating   0          5m
web-app-abc123-ghi56      1/1     Running       0          5s
web-app-abc123-jkl78      1/1     Running       0          3s

# All old pods terminated, new ones running!
```

**Step 6: Verify New Config**

```bash
# Check message (should be updated!)
curl localhost:3000
# Response: "hello reloader"  ← UPDATED!

# Check environment variable
kubectl exec -it web-app-ghi56-n production -- env | grep MESSAGE
# MESSAGE=hello reloader  ← UPDATED!
```

**No manual restart needed!** Reloader handled everything.

---

## Reloader Annotation Options

### Option 1: Auto-Detect All ConfigMaps/Secrets (Simplest)

```yaml
annotations:
  reloader.stakater.com/match: "true"
```

Effect: Restarts deployment when ANY ConfigMap or Secret it uses changes.

### Option 2: Watch Specific ConfigMap Only

```yaml
annotations:
  reloader.stakater.com/search: "true"
  reloader.stakater.com/match: "configmap=web-message"
```

Effect: Only restarts if `web-message` ConfigMap changes. Ignores other ConfigMaps/Secrets.

### Option 3: Watch Specific Secret Only

```yaml
annotations:
  reloader.stakater.com/search: "true"
  reloader.stakater.com/match: "secret=db-credentials"
```

Effect: Only restarts if `db-credentials` Secret changes.

### Option 4: Watch Multiple Specific Resources

```yaml
annotations:
  reloader.stakater.com/search: "true"
  reloader.stakater.com/match: "configmap=web-message|secret=db-credentials"
```

Effect: Restarts if either resource changes.

### Option 5: Exclude Certain Resources

```yaml
annotations:
  reloader.stakater.com/match: "true"
  reloader.stakater.com/ignore: "configmap=legacy-config|secret=old-secret"
```

Effect: Restart on any ConfigMap/Secret change EXCEPT the ones listed.

---

## Use Cases for Reloader

### Use Case 1: Configuration Updates

```yaml
# Update feature flags, settings, etc.
apiVersion: v1
kind: ConfigMap
metadata:
  name: feature-flags
data:
  enable_new_feature: "true"
  
---
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    reloader.stakater.com/match: "true"
spec:
  template:
    spec:
      containers:
      - env:
        - name: FEATURE_FLAGS
          valueFrom:
            configMapKeyRef:
              name: feature-flags
              key: enable_new_feature
```

When feature flag updates: **Pods automatically restart with new flag**

### Use Case 2: Secret Rotation

```yaml
# Rotate API keys, passwords, tokens
apiVersion: v1
kind: Secret
metadata:
  name: api-keys
type: Opaque
data:
  api_key: YWJjZGVmZ2hpams=
  
---
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    reloader.stakater.com/match: "true"
spec:
  template:
    spec:
      containers:
      - env:
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: api-keys
              key: api_key
```

When secret updates: **Pods automatically restart with new key**

### Use Case 3: Certificate Updates

```yaml
# Update TLS certificates
apiVersion: v1
kind: Secret
metadata:
  name: tls-cert
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi...
  tls.key: LS0tLS1CRUdJTi...
  
---
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    reloader.stakater.com/match: "true"
spec:
  template:
    spec:
      volumes:
      - name: certs
        secret:
          secretName: tls-cert
```

When certificate updates: **Pods automatically restart with new cert**

---

## Comparison: Before and After Reloader

### Without Reloader

```bash
# Step 1: Update ConfigMap
kubectl edit configmap app-config

# Step 2: Oh no, which deployments use this?
# Let me search...
grep -r "app-config" k8s-manifests/
# Found: payment-api, notification-service, reporting-engine

# Step 3: Manually restart each one
kubectl rollout restart deployment payment-api
kubectl rollout restart deployment notification-service
kubectl rollout restart deployment reporting-engine

# Step 4: What if I missed one?
# Let me check:
grep -r "app-config" k8s-manifests/ | grep -v payment-api | grep -v notification | grep -v reporting
# Oh no! legacy-system also uses it!

kubectl rollout restart deployment legacy-system

# Step 5: Verify each one
kubectl get deployments
# Check status of each restart...

# Time spent: 15-30 minutes
# Risk: Human error, forgot deployments, mistakes
```

### With Reloader

```bash
# Step 1: Update ConfigMap
kubectl edit configmap app-config

# That's it! Reloader does everything else automatically.
# Time spent: < 1 minute
# Risk: Zero (automatic)
```

---

## Best Practices with Reloader

### 1. Use Match: "true" for Generic Deployments

```yaml
annotations:
  reloader.stakater.com/match: "true"
```

Safest option: Auto-discovers all ConfigMaps and Secrets.

### 2. Be Specific for Critical Deployments

```yaml
annotations:
  reloader.stakater.com/match: "true"
  reloader.stakater.com/ignore: "configmap=debug-config"
```

For production deployments, ignore debug/non-critical configs.

### 3. Use on All Workload Types

```yaml
# Deployments
annotations:
  reloader.stakater.com/match: "true"

# StatefulSets
annotations:
  reloader.stakater.com/match: "true"

# DaemonSets
annotations:
  reloader.stakater.com/match: "true"
```

Reloader works on all resource types.

### 4. Don't Mix Environment Variables and Volumes

Best practice: If using Reloader, stick to ONE method:

```yaml
# GOOD: Only environment variables + Reloader
env:
- name: CONFIG
  valueFrom:
    configMapKeyRef:
      name: my-config
      key: data

# GOOD: Only mounted volumes (auto-update, no restart needed)
volumeMounts:
- name: config
  mountPath: /etc/config

# AVOID: Mixing both (confusing!)
```

### 5. Monitor Reloader Logs

```bash
# Check if Reloader is working
kubectl logs -f -l app.kubernetes.io/name=reloader

# Look for messages like:
# "Restarting deployment/payment-api due to ConfigMap change"
# "Detected change in secret/db-credentials"
```

---

## Troubleshooting Reloader

### Issue 1: Reloader Installed But Pods Not Restarting

**Check 1: Annotation Present?**
```bash
kubectl get deployment myapp -o yaml | grep reloader
# Should show: reloader.stakater.com/match: "true"
```

**Check 2: Reloader Pod Running?**
```bash
kubectl get pods -l app.kubernetes.io/name=reloader
# Should show: Running status
```

**Check 3: Reloader Has RBAC Permissions?**
```bash
kubectl get clusterrole reloader
kubectl get clusterrolebinding reloader
# Should exist
```

### Issue 2: Reloader Doesn't Detect ConfigMap Changes

**Check 1: ConfigMap in Same Namespace?**
```bash
# ConfigMap and Deployment must be in same namespace
kubectl get configmap -n production
kubectl get deployment -n production
```

**Check 2: ConfigMap Referenced Correctly?**
```bash
kubectl get deployment myapp -o yaml | grep configMapKeyRef -A 3
# Verify name matches actual ConfigMap name
```

**Check 3: Reloader Logs**
```bash
kubectl logs -l app.kubernetes.io/name=reloader --tail=50
# Look for errors or "watching" messages
```

---

## Real-World Scenario: Secret Rotation

```bash
# Scenario: Database password rotation every 90 days

# Current state: Old password in Secret
kubectl get secret db-credentials -o yaml
# Shows: old_password_hash

# Pods using it are running fine
kubectl get pods | grep myapp
# 3 pods running, connected to database

# Step 1: Update the Secret with new password
kubectl edit secret db-credentials

# Change password value

# Step 2: Reloader automatically detects change
# Reloader performs rolling restart:
# - Pod 1 terminates
# - New Pod 1 starts (reads new password)
# - Pod 2 terminates
# - New Pod 2 starts (reads new password)
# - Pod 3 terminates
# - New Pod 3 starts (reads new password)

# Step 3: Verify all pods connected to database
kubectl get pods | grep myapp
# All 3 pods show: READY 1/1 (meaning readiness probe passed)

# Step 4: Verify database connection works
kubectl logs <pod> | grep "Connected to database"

# Done! Password rotated, all pods updated, zero downtime!
```

---

## Reloader vs Manual Restart

| Aspect | Manual Restart | Reloader |
|--------|----------------|----------|
| **Detection** | Manual checking | Automatic |
| **Restart triggering** | Manual command | Automatic |
| **Time to restart** | 5-30 minutes | < 1 minute |
| **Risk of forgetting** | High (human error) | Zero (automatic) |
| **Scaling** | Harder with 100+ deployments | Same effort always |
| **Maintenance** | Required | None |
| **Automation** | Manual | Fully automated |

---

## Interview Story: Reloader Saves the Day

**Problem in Production:**
"We had a critical incident where a database password expired. The operations team updated the Secret with the new password, but forgot to restart the deployments using it. For hours, the deployments couldn't connect to the database, causing customer-facing issues."

**The Lesson:**
"After that incident, we implemented Reloader across the entire cluster. Now whenever a Secret or ConfigMap is updated, Reloader automatically restarts the affected deployments. We don't have to remember which deployments use which configs."

**Implementation:**
"We added a simple annotation to all our deployments: `reloader.stakater.com/match: "true"`. For critical deployments, we use specific annotations to only watch certain ConfigMaps/Secrets. This ensures deployments get new configuration immediately without manual intervention."

**The Impact:**
"It's been a lifesaver. Feature flag updates roll out instantly. Secret rotations happen automatically. Configuration changes are applied within seconds. No more manual tracking of which deployment uses which config. And no more human errors causing incidents."

---

## Key Takeaways

1. **Reloader solves the immutability problem** → Automatic restart on config changes

2. **Simple annotation enables it** → `reloader.stakater.com/match: "true"`

3. **No manual tracking needed** → Reloader discovers all ConfigMaps/Secrets automatically

4. **Works with all workload types** → Deployments, StatefulSets, DaemonSets

5. **Flexible options available** → Match specific resources or ignore certain ones

6. **Rolling restart preserves availability** → No downtime if replicas > 1

7. **Highly recommended for production** → Saves time and prevents human error

8. **Easy to install** → Kubectl manifests or Helm

9. **Scales with infrastructure** → Same effort for 5 deployments or 500

10. **Best used with annotations** → Simple, effective, maintainable

---

## Next Steps

1. **Install Reloader in your cluster**
   ```bash
   helm install reloader stakater/reloader -n reloader-system --create-namespace
   ```

2. **Add annotation to critical deployments**
   ```yaml
   annotations:
     reloader.stakater.com/match: "true"
   ```

3. **Test it**
   - Edit a ConfigMap
   - Watch pods automatically restart
   - Verify new configuration

4. **Roll out to all deployments**
   - Add annotation to all production workloads
   - Remove all manual restart procedures
   - Focus on application development, not operations

---

## Conclusion

Reloader transforms ConfigMap/Secret updates from a manual, error-prone process into an automated, reliable one. Combined with the knowledge of immutability from the previous guide, Reloader gives you the best of both worlds: Kubernetes's immutability principle AND automatic configuration propagation.

This is why I use Reloader in most, if not all, of my production deployments. It's a lifesaver.