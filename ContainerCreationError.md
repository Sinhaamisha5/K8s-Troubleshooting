# Container Creation Errors: Complete Troubleshooting Guide

## Container Lifecycle in Kubernetes

Understanding when errors occur requires understanding the container creation process:

```
Step 1: Pull Image
  ↓
Step 2: Generate Container Configuration
  ↓
Step 3: Create Container
  ↓
Step 4: Start Container (Run Process)
  ↓
Step 5: Monitor & Report Status
```

Each step can fail with different errors.

---

## The Four Container States

### 1. ImagePull (Image pulling)
```
Status: ImagePullBackOff, ImagePullError
Happens: During Step 1 - Image pull
Cause: Image doesn't exist, wrong registry, auth issues
Not the focus of this guide (covered elsewhere)
```

### 2. CreateContainerConfigError
```
Status: CreateContainerConfigError
Happens: During Step 2 - Generate configuration
Cause: Missing Secrets, ConfigMaps, volumes, or invalid config
Example: Secret referenced in env var doesn't exist
FIX: Create missing resources, NO pod restart needed
```

### 3. CreateContainerError
```
Status: CreateContainerError
Happens: During Step 3 - Create container
Cause: No command specified, invalid security context, etc.
Example: Image has no entrypoint, pod doesn't specify command
FIX: Specify command in pod spec, NO pod restart needed
```

### 4. RunContainerError (or CrashLoopBackOff)
```
Status: RunContainerError (briefly), then CrashLoopBackOff
Happens: During Step 4 - Start container
Cause: Invalid command, process fails to start
Example: Command specified doesn't exist or is invalid
FIX: Fix command, pod will auto-restart (RestartPolicy)
```

### 5. Application Running Successfully
```
Status: Running
Happens: After all steps succeed
Condition: Container process is executing normally
```

---

## Error 1: CreateContainerConfigError

### What It Means

```
Pod passed steps 1-2 but failed at step 2.5 (configuration generation).

The configuration couldn't be generated because:
- Referenced Secret doesn't exist
- Referenced ConfigMap doesn't exist
- Referenced key in Secret/ConfigMap doesn't exist
- Invalid volume mount
```

### Lifecycle View

```
Step 1: Pull Image ✓ SUCCESS
Step 2: Generate Configuration ✗ FAILED (missing resource)
Step 3: Create Container ⊘ SKIPPED
Step 4: Start Container ⊘ SKIPPED
Result: CreateContainerConfigError
```

### Real Example: Missing Secret

**Pod Definition (References Secret)**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    - name: DB_PASSWORD           # Environment variable name
      valueFrom:
        secretKeyRef:
          name: db-secret         # ← Referenced secret
          key: password           # ← Expected key in secret
```

**Problem:**
```bash
kubectl get secrets
# OUTPUT: No "db-secret" found!
```

The pod references `db-secret` but it doesn't exist.

**Investigation:**

```bash
kubectl describe pod app-pod
```

Output:
```
Name:            app-pod
Namespace:       default
...
Containers:
  app:
    Image:       myapp:1.0
    Environment:
      DB_PASSWORD:  <set to the key 'password' of secret 'db-secret'>
    Mounts:      <none>

Events:
  Type     Reason                   Age   From             Message
  ----     ------                   ----  ----             -------
  Warning  FailedCreateContainer    1m    kubelet          Error: couldn't find key password in Secret default/db-secret
  Warning  CreateContainerConfigError 1m   kubelet          Secret "db-secret" not found
```

**Root Cause:** Secret `db-secret` doesn't exist in the `default` namespace.

### Solution: Create the Missing Secret

```bash
# Create the secret
kubectl create secret generic db-secret \
  --from-literal=password=mysecretpassword

# Verify it exists
kubectl get secrets
# OUTPUT:
# NAME        TYPE      DATA   AGE
# db-secret   Opaque    1      5s
```

**Key Point:** No pod restart needed! Pod will automatically start once secret exists.

```bash
# Pod automatically transitions to Running
kubectl get pods app-pod
# STATUS: Running
```

### Other CreateContainerConfigError Scenarios

**Scenario 2: Missing ConfigMap Key**

```yaml
env:
- name: APP_CONFIG
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: config.yaml          # ← This key might not exist
```

```bash
# Check what keys exist in ConfigMap
kubectl get configmap app-config -o yaml

# If key doesn't exist, add it:
kubectl create configmap app-config \
  --from-file=config.yaml=./config.yaml \
  --dry-run=client -o yaml | kubectl apply -f -
```

**Scenario 3: Missing Volume**

```yaml
spec:
  containers:
  - name: app
    volumeMounts:
    - name: config          # ← This volume name
      mountPath: /etc/config
  volumes:
  - name: config            # ← Must match!
    configMap:
      name: missing-config  # ← This ConfigMap doesn't exist!
```

Fix: Create the missing ConfigMap or correct the reference.

**Scenario 4: Invalid Secret Path**

```yaml
env:
- name: API_KEY
  valueFrom:
    secretKeyRef:
      name: api-secrets
      key: api_key          # ← This exact key must exist

# If secret has key "apiKey" instead of "api_key"
# You get CreateContainerConfigError!
```

Fix: Match the exact key name.

---

## Error 2: CreateContainerError

### What It Means

```
Pod passed steps 1-2 but failed at step 3.

The container runtime couldn't create the container because:
- No command specified (no entrypoint in image, no command in pod spec)
- Invalid security context
- Invalid volume mount destination
- Other container runtime issues
```

### Lifecycle View

```
Step 1: Pull Image ✓ SUCCESS
Step 2: Generate Configuration ✓ SUCCESS
Step 3: Create Container ✗ FAILED (runtime issue)
Step 4: Start Container ⊘ SKIPPED
Result: CreateContainerError
```

### Real Example: No Command Specified

**Image Definition:**
```dockerfile
FROM ubuntu:20.04
# No ENTRYPOINT or CMD specified!
```

**Pod Definition:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-command-pod
spec:
  containers:
  - name: app
    image: ubuntu:20.04
    # No command or args specified!
```

**Problem:**

The image doesn't specify an entrypoint, and the pod spec doesn't specify a command. The container runtime doesn't know what process to run.

**Investigation:**

```bash
kubectl describe pod no-command-pod
```

Output:
```
Events:
  Type     Reason              Age   From      Message
  ----     ------              ----  ----      -------
  Warning  FailedCreateContainer  1m kubelet  Error: failed to create container:
                                              failed to generate container spec:
                                              no command specified
```

**Root Cause:** No command specified - neither in image nor in pod spec.

### Solution: Specify a Command

**Option 1: Specify command in pod spec**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-command-pod
spec:
  containers:
  - name: app
    image: ubuntu:20.04
    command: ["/bin/sh"]        # ← Add command
    args: ["-c", "sleep 3600"]  # ← Add arguments
```

**Option 2: Use image with entrypoint**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: web
    image: nginx:latest         # Already has entrypoint
```

**Option 3: Deployment with command**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        image: my-app:1.0
        command: ["/app/start.sh"]  # ← Specify entrypoint
        args: ["--port", "8080"]     # ← Specify arguments
```

**Key Point:** No restart needed! Pod will create after spec is updated.

```bash
kubectl apply -f pod-with-command.yaml
# Pod immediately transitions to Running
```

### Other CreateContainerError Scenarios

**Scenario 2: Invalid Security Context**

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    securityContext:
      runAsUser: 1000
      runAsGroup: 1000
      fsGroup: invalid-value  # ← Invalid!
```

Fix: Use valid numeric UID/GID values.

**Scenario 3: Invalid Volume Mount**

```yaml
spec:
  containers:
  - name: app
    volumeMounts:
    - name: data
      mountPath: /invalid/mount/path  # ← May be invalid
  volumes:
  - name: data
    emptyDir: {}
```

Fix: Ensure mount paths are valid for the container filesystem.

---

## Error 3: RunContainerError (then CrashLoopBackOff)

### What It Means

```
Pod passed steps 1-3 but failed at step 4.

The container was created successfully but failed when starting:
- Command/entrypoint doesn't exist
- Invalid command syntax
- Process exits immediately
- Permission denied executing command
```

### Lifecycle View

```
Step 1: Pull Image ✓ SUCCESS
Step 2: Generate Configuration ✓ SUCCESS
Step 3: Create Container ✓ SUCCESS
Step 4: Start Container ✗ FAILED (process fails to run)
Result: RunContainerError → CrashLoopBackOff (due to restart)
```

### Real Example: Invalid Command

**Pod Definition:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: invalid-command-pod
spec:
  containers:
  - name: app
    image: ubuntu:20.04
    command: ["/bin/gibberish"]  # ← Command doesn't exist!
    args: ["--invalid", "args"]
```

**Problem:**

The command `/bin/gibberish` doesn't exist in the image. The container runtime can CREATE the container, but when trying to RUN the process, it fails.

**Investigation:**

```bash
kubectl get pods
```

Output:
```
NAME                     READY   STATUS              RESTARTS   AGE
invalid-command-pod      0/1     RunContainerError   0          2s
# Then quickly transitions to:
invalid-command-pod      0/1     CrashLoopBackOff    1          5s
```

Describe the pod:

```bash
kubectl describe pod invalid-command-pod
```

Output:
```
Last State:     Terminated
  Reason:       ContainerCannotRun
  Message:      OCI runtime error: exec format error
  
Or:

Last State:     Terminated
  Reason:       StartError
  Message:      executable file not found in $PATH
```

Check logs:

```bash
kubectl logs invalid-command-pod
# Shows error output from failed command
```

**Root Cause:** Command `/bin/gibberish` doesn't exist in the ubuntu image.

### Solution: Use Valid Command

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: invalid-command-pod
spec:
  containers:
  - name: app
    image: ubuntu:20.04
    command: ["/bin/sleep"]      # ← Valid command
    args: ["3600"]               # ← Valid argument
```

Apply:

```bash
kubectl apply -f valid-command-pod.yaml
```

Pod transitions to Running:

```bash
kubectl get pods
# STATUS: Running
```

### Other RunContainerError Scenarios

**Scenario 2: Command Syntax Error**

```yaml
command: ["/bin/bash", "-c", "invalid syntax here!!!"]
# Shell will fail to parse
```

Fix: Test command locally first.

**Scenario 3: Permission Denied**

```yaml
command: ["/root/scripts/app.sh"]  # Not executable!
```

In Dockerfile:
```dockerfile
RUN chmod +x /root/scripts/app.sh  # Make executable
```

**Scenario 4: Wrong Path to Binary**

```yaml
command: ["/usr/bin/java"]  # Binary doesn't exist
```

Fix: Find correct path:
```bash
docker run myapp:1.0 which java
# Or check in Dockerfile where binary is installed
```

---

## Decision Tree: Which Container Error?

```
Pod has error state?

├─ ImagePullBackOff / ImagePullError
│  └─ Image doesn't exist or can't be pulled
│
├─ CreateContainerConfigError
│  └─ Problem generating configuration (step 2)
│  └─ Missing Secret, ConfigMap, or invalid volume reference
│  └─ FIX: Create missing resources
│  └─ No restart needed
│
├─ CreateContainerError
│  └─ Problem creating container (step 3)
│  └─ No command specified or invalid security context
│  └─ FIX: Add command or fix security context
│  └─ No restart needed
│
├─ RunContainerError (briefly) → CrashLoopBackOff
│  └─ Problem starting container (step 4)
│  └─ Invalid command or process fails
│  └─ FIX: Fix command
│  └─ Will auto-restart due to RestartPolicy
│
└─ Running
   └─ Everything succeeded!
```

---

## Troubleshooting Checklist

```bash
# 1. Check pod status
kubectl get pods <pod-name>
# Note the STATUS column

# 2. Describe pod for detailed error
kubectl describe pod <pod-name>
# Look under "Last State" and "Events" sections

# 3. If CreateContainerConfigError:
# Check for missing Secrets
kubectl get secrets -n <namespace>

# Check for missing ConfigMaps
kubectl get configmaps -n <namespace>

# Verify secret keys
kubectl get secret <secret-name> -o yaml

# 4. If CreateContainerError:
# Check if image has entrypoint
docker image inspect <image> | grep -i "entrypoint\|cmd"

# Or check pod spec for command
kubectl get pod <pod-name> -o yaml | grep -A 5 "command:"

# 5. If RunContainerError / CrashLoopBackOff:
# Check logs
kubectl logs <pod-name>

# Check if command exists
kubectl exec <pod-name> -- which <command>

# 6. Re-apply with fixed config
kubectl apply -f fixed-pod.yaml

# 7. Verify pod now running
kubectl get pods <pod-name>
```

---

## Real Debugging Example: CreateContainerConfigError

```bash
# Alert: "Pod in CreateContainerConfigError state"

# Step 1: Describe pod
kubectl describe pod app-pod
# Output: Secret "db-secret" not found

# Step 2: Verify secret exists
kubectl get secrets
# Output: No db-secret

# Step 3: Create the secret
kubectl create secret generic db-secret \
  --from-literal=password=secretvalue

# Step 4: Verify secret created
kubectl get secrets
# db-secret is now listed

# Step 5: Pod automatically transitions
kubectl get pods app-pod
# STATUS: Running ✓
```

---

## Real Debugging Example: CreateContainerError

```bash
# Alert: "Pod in CreateContainerError state"

# Step 1: Describe pod
kubectl describe pod no-cmd-pod
# Output: no command specified

# Step 2: Check pod spec
kubectl get pod no-cmd-pod -o yaml | grep -A 10 "containers:"
# No command field

# Step 3: Update pod spec
kubectl set image pod/no-cmd-pod \
  app=myapp:1.0 \
  --record

# Better: Edit deployment
kubectl edit pod no-cmd-pod
# Add: command: ["/bin/sleep", "3600"]

# Step 4: Pod transitions to Running
kubectl get pods no-cmd-pod
# STATUS: Running ✓
```

---

## Real Debugging Example: RunContainerError

```bash
# Alert: "Pod in CrashLoopBackOff state"

# Step 1: Check pod status
kubectl get pods gibberish-pod
# STATUS: CrashLoopBackOff

# Step 2: Check logs for error
kubectl logs gibberish-pod
# Output: command not found

# Step 3: Describe pod
kubectl describe pod gibberish-pod
# Output: executable not found in $PATH

# Step 4: Fix the command
kubectl edit pod gibberish-pod
# Change: command: ["/bin/gibberish"] 
# To:     command: ["/bin/sleep", "3600"]

# Or edit deployment and redeploy
kubectl apply -f fixed-pod.yaml

# Step 5: Pod transitions to Running
kubectl get pods gibberish-pod
# STATUS: Running ✓
```

---

## Interview Story: Container Creation Errors

**Setup:**
"We had three different pods failing with different errors, which helped me understand the container creation lifecycle."

**Example 1 - CreateContainerConfigError:**
"First pod failed with CreateContainerConfigError. The pod was referencing a Secret called 'db-secret' in an environment variable, but the secret didn't exist. I checked the pod spec, verified the secret was missing, created it, and the pod immediately started. No restart needed—Kubernetes automatically retried the configuration generation."

**Example 2 - CreateContainerError:**
"Second pod had CreateContainerError. The error said 'no command specified'. The image was Ubuntu with no entrypoint, and the pod spec didn't specify a command. I added `command: ['/bin/sleep', '3600']` to the pod spec, applied it, and the pod immediately transitioned to Running."

**Example 3 - RunContainerError:**
"Third pod briefly showed RunContainerError then went to CrashLoopBackOff. The pod had an invalid command `/bin/gibberish` which doesn't exist. The container was created successfully, but failed when trying to run the process. I fixed the command, reapplied the pod definition, and it stayed running."

**Key Insight:**
"Understanding the container lifecycle is crucial. CreateContainerConfigError and CreateContainerError happen BEFORE the process even starts, so you can fix them without restarting. RunContainerError happens DURING process startup, which triggers the restart loop. Each error has a specific cause, and `kubectl describe` always tells you what it is."

---

## Common Mistakes When Debugging Container Errors

❌ **Mistake 1:** Restarting pod for CreateContainerConfigError
✅ **Fix:** Create missing Secret/ConfigMap, pod restarts automatically

❌ **Mistake 2:** Not checking if image has entrypoint
✅ **Fix:** Always verify image has entrypoint or specify command in pod

❌ **Mistake 3:** Using wrong secret key name
```yaml
# Pod references "api_key" but secret has "apiKey"
# CreateContainerConfigError!
```
✅ **Fix:** Match exact key names

❌ **Mistake 4:** Specifying invalid command
```yaml
command: ["/bin/invalidcommand"]  # Doesn't exist!
```
✅ **Fix:** Test command exists in image

❌ **Mistake 5:** Not reading error messages
✅ **Fix:** Always check `kubectl describe pod` and logs

---

## Key Takeaways

1. **Container lifecycle has 4-5 steps** → Understand which step fails

2. **CreateContainerConfigError** = Step 2 failed
   - Missing Secret, ConfigMap, or invalid config
   - Fix without restarting pod

3. **CreateContainerError** = Step 3 failed
   - No command specified or invalid security context
   - Fix without restarting pod

4. **RunContainerError** = Step 4 failed
   - Invalid command or process fails
   - Will auto-restart due to RestartPolicy

5. **`kubectl describe pod`** → Always shows the exact error

6. **Test locally first** → Verify image and command work with `docker run`

7. **Security contexts and volumes** → Can cause CreateContainerError

8. **Pod spec overrides image** → Command in pod spec overrides Dockerfile CMD/ENTRYPOINT