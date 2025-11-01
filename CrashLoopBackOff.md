# CrashLoopBackOff: Troubleshooting Guide & Interview Case Study

## What is CrashLoopBackOff?

### Definition

CrashLoopBackOff is **not the error itself**—it's a **symptom/state** indicating that a container is crashing and restarting repeatedly.

```
CrashLoopBackOff State:
Container starts → crashes → restarts → crashes → restarts...

Key characteristic: Restart count > 0
```

### The Restart Loop Explained

```
Timeline:
09:00:00 - Container starts
09:00:05 - Container crashes → kubelet detects failure
09:00:05 - Backoff period begins (1 second)
09:00:06 - kubelet tries to restart → Container starts again
09:00:10 - Container crashes → kubelet detects failure
09:00:10 - Backoff period increases (2 seconds, exponential backoff)
09:00:12 - kubelet tries to restart → Container starts again
09:00:15 - Container crashes
09:00:15 - Backoff period increases (4 seconds)
09:00:19 - kubelet tries to restart
...
Max backoff: 5 minutes between restarts
```

### RestartPolicy Determines Behavior

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  restartPolicy: Always  # Default
  containers:
  - name: app
    image: my-app:1.0
```

**Three RestartPolicy options:**

1. **Always** (Default)
   - Restarts whether container exits successfully or fails
   - Used for: Most applications (web servers, APIs, workers)
   - Example: Exit code 0 or 1, both trigger restart

2. **OnFailure**
   - Only restarts if container exits with non-zero exit code
   - Used for: Jobs, batch processes
   - Example: Exit code 0 (success) = no restart, Exit code 1 (error) = restart

3. **Never**
   - Never restarts container
   - Used for: Jobs that should run once
   - Pod stays in completed/failed state

---

## Exit Codes: The First Clue

Exit codes tell you what went wrong. Common ones:

### Exit Code 0
```
Meaning: Successful exit
CrashLoopBackOff: Only if RestartPolicy=Always (will restart anyway)
Solution: May not be an error at all
```

### Exit Code 1
```
Meaning: General application error
CrashLoopBackOff: YES
Debug: Check application logs immediately
Cause: Could be misconfiguration, missing dependencies, bugs, etc.
```

### Exit Code 137
```
Meaning: External SIGKILL (signal 137 = 128 + 9, where 9 is SIGKILL)
CrashLoopBackOff: YES
Debug: This is NOT an application crash, something KILLED the container
Causes:
- Liveness probe failed (kubelet killed container)
- Memory limit exceeded (OOMKilled)
- Node eviction
- Admin manually killed
Solution: Check events, liveness probe, resource limits
```

### Other Common Exit Codes
- **137**: SIGKILL (external)
- **139**: SIGSEGV (segmentation fault, memory corruption)
- **124**: SIGTERM timeout
- **255**: Abnormal exit

---

## Real Examples from Production

### Example 1: Missing Environment Variables (Exit Code 1)

**Scenario**: MySQL pod in CrashLoopBackOff

**Investigation**:
```bash
kubectl describe pod mysql-pod
# Shows: Exit Code 1 (application error)
```

**Logs reveal**:
```
ERROR: MySQL requires MYSQL_ROOT_PASSWORD environment variable
ERROR: MySQL requires MYSQL_DATABASE environment variable
ERROR: Application startup failed
```

**Root Cause**: Environment variables not provided

**Pod Definition (Broken)**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  template:
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        # Missing environment variables!
```

**Fixed Pod Definition**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  template:
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        - name: MYSQL_DATABASE
          value: myapp
```

**Solution Steps**:
```bash
# 1. Create secret with password
kubectl create secret generic mysql-secret \
  --from-literal=root-password=MySecurePassword123

# 2. Apply fixed deployment
kubectl apply -f mysql-deployment.yaml

# 3. Verify pod is running
kubectl get pods
# mysql-0 should now be Running/Ready
```

**Key Learning**: Exit code 1 = check application startup requirements, check logs

---

### Example 2: File Permission Issue (Exit Code 1)

**Scenario**: Orders API pod in CrashLoopBackOff

**Investigation**:
```bash
kubectl describe pod orders-api-pod
# Shows: script.sh: Permission denied
# Exit Code: 1
```

**Root Cause**: Startup script doesn't have execute permissions

**Pod Definition (Broken)**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
spec:
  template:
    spec:
      containers:
      - name: orders-api
        image: orders-api:1.0
        command: ["/app/script.sh"]  # This file is not executable
```

**Diagnosis**:
```bash
# Can't exec into pod because it's crashing
# But we can test the image locally

# Run the docker image locally
docker run -it orders-api:1.0 /bin/bash

# Inside container, check permissions
ls -la /app/script.sh
# Output: -rw-r--r-- (NOT executable, needs x bit)

# Check what we need
chmod +x /app/script.sh
ls -la /app/script.sh
# Output: -rwxr-xr-x (now executable!)
```

**Fix**: Update Dockerfile to make script executable

**Dockerfile (Before)**:
```dockerfile
FROM ubuntu:20.04
COPY script.sh /app/script.sh
# Missing chmod!
ENTRYPOINT ["/app/script.sh"]
```

**Dockerfile (After)**:
```dockerfile
FROM ubuntu:20.04
COPY script.sh /app/script.sh
RUN chmod +x /app/script.sh  # Add this!
ENTRYPOINT ["/app/script.sh"]
```

**Rebuild and deploy**:
```bash
docker build -t orders-api:1.1 .
docker push orders-api:1.1
kubectl set image deployment/orders-api orders-api=orders-api:1.1
kubectl get pods  # Pod now running!
```

**Key Learning**: Check file permissions in Docker images, use `docker run` to test images

---

### Example 3: Missing Volume Mount (Exit Code 1)

**Scenario**: Search/Nginx pod in CrashLoopBackOff

**Investigation**:
```bash
kubectl describe pod search-pod
# Shows: nginx.conf: No such file or directory
# Exit Code: 1
```

**Root Cause**: Volume is defined but NOT mounted to container

**Pod Definition (Broken)**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: search
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        # MISSING: volumeMounts section!
      
      volumes:
      - name: nginx-conf  # This volume exists
        configMap:
          name: nginx-config
          # But container doesn't mount it!
```

**Logs show**:
```
nginx: [emerg] open() "/etc/nginx/nginx.conf" failed (2: No such file or directory)
```

**Fix: Add volumeMounts**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: search
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        
        # NEW: Mount the volume
        volumeMounts:
        - name: nginx-conf
          mountPath: /etc/nginx  # Where to mount
      
      volumes:
      - name: nginx-conf
        configMap:
          name: nginx-config
```

**Verify ConfigMap exists**:
```bash
kubectl get configmap nginx-config
# Should show the config map

kubectl get configmap nginx-config -o yaml
# Verify it contains valid nginx configuration
```

**Deploy**:
```bash
kubectl apply -f search-deployment.yaml
kubectl get pods  # Pod now running!
```

**Key Learning**: Define volumes AND volumeMounts. Volumes alone don't help—container must mount them

---

### Example 4: Memory Limit Exceeded (Exit Code 137)

**Scenario**: Shipping API pod in CrashLoopBackOff

**Investigation**:
```bash
kubectl describe pod shipping-api-pod
# Shows: Exit Code: 137 (SIGKILL)
# Message: OOMKilled - memory limit exceeded
```

**Root Cause**: Pod is trying to use more memory than its limit

**Pod Definition (Broken)**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shipping-api
spec:
  template:
    spec:
      containers:
      - name: shipping
        image: shipping-api:1.0
        resources:
          limits:
            memory: "128Mi"  # Too low for this app!
          requests:
            memory: "64Mi"
```

**How to determine right limit**:

**Option 1: Monitor actual usage**:
```bash
# If metrics-server is running
kubectl top pod shipping-api-pod --containers

# Output:
# NAME                CPU(cores)   MEMORY(Mi)
# shipping-api-pod    150m         200Mi
# The app is using 200Mi!
```

**Option 2: Load test the application**:
```bash
# Run load test
# Monitor memory consumption during peak traffic

# Use the peak usage as your limit
# Add 20-30% buffer for safety
```

**Fix: Increase memory limit**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shipping-api
spec:
  template:
    spec:
      containers:
      - name: shipping
        image: shipping-api:1.0
        resources:
          limits:
            memory: "256Mi"  # Increased based on monitoring
          requests:
            memory: "200Mi"
```

**Deploy and verify**:
```bash
kubectl apply -f shipping-deployment.yaml
kubectl get pods  # Pod now running!
kubectl top pod -l app=shipping-api --containers  # Monitor memory
```

**Key Learning**: Exit code 137 often means OOMKilled. Find actual usage via metrics-server and set appropriate limit

---

### Example 5: Wrong Liveness Probe Path (Exit Code 137)

**Scenario**: Notifications pod in CrashLoopBackOff

**Investigation**:
```bash
kubectl describe pod notifications-pod
# Shows: Exit Code: 137 (external kill)
# Events show: Liveness probe failed - HTTP 404
```

**Root Cause**: Liveness probe is checking wrong endpoint

**Pod Definition (Broken)**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notifications
spec:
  template:
    spec:
      containers:
      - name: notifications
        image: notifications:1.0
        
        livenessProbe:
          httpGet:
            path: /healthz      # Wrong path!
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 10
```

**What's happening**:
```
Timeline:
09:00:03 - Container starts (after 3 second delay)
09:00:03 - Kubelet sends GET /healthz to port 8080
09:00:03 - App responds with 404 (path doesn't exist)
09:00:03 - Liveness probe FAILS
09:00:03 - Kubelet kills pod (SIGKILL = exit code 137)
09:00:03 - Pod restarts (restart loop!)
```

**Events show**:
```bash
kubectl describe pod notifications-pod
# Events:
# - Liveness probe failed (status: 404)
# - Container killed by kubelet (exit code 137)
```

**Check what endpoints are available**:
```bash
# Maybe it's /health not /healthz
# Check application documentation or code
```

**Fix: Use correct endpoint**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notifications
spec:
  template:
    spec:
      containers:
      - name: notifications
        image: notifications:1.0
        
        livenessProbe:
          httpGet:
            path: /health        # Correct path
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 10
```

**Deploy and verify**:
```bash
kubectl apply -f notifications-deployment.yaml
kubectl get pods  # Pod now running!
kubectl describe pod notifications-pod | grep -A 10 Events
# No more liveness probe failures
```

**Key Learning**: Exit code 137 + liveness probe failure = wrong endpoint or server not ready

---

### Example 6: Liveness Probe Timeout (Exit Code 137)

**Scenario**: Analytics pod in CrashLoopBackOff

**Investigation**:
```bash
kubectl describe pod analytics-pod
# Shows: Exit Code: 137
# Events show: Liveness probe failed - connection refused
```

**Root Cause**: Liveness probe starts too early, before app is ready

**Pod Definition (Broken)**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: analytics
spec:
  template:
    spec:
      containers:
      - name: analytics
        image: analytics:1.0
        
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 1    # TOO SHORT! App needs time to load
          periodSeconds: 1          # Too frequent, hammering the app
          timeoutSeconds: 1
```

**What's happening**:
```
Timeline:
09:00:00 - Container starts
09:00:01 - Kubelet sends GET /health (after 1 second delay)
         - But app is still loading libraries/initializing!
         - Server not listening yet
         - Connection refused!
09:00:01 - Liveness probe FAILS
09:00:01 - Kubelet kills pod (SIGKILL = exit code 137)
09:00:01 - Pod restarts
Loop continues because problem persists...
```

**Application startup requirements**:
- Analytics app loads data from disk (slow)
- Initializes ML models
- Takes ~15-20 seconds to be ready
- Then HTTP server starts listening

**Fix: Increase initialDelaySeconds**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: analytics
spec:
  template:
    spec:
      containers:
      - name: analytics
        image: analytics:1.0
        
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 20   # Wait for app to load
          periodSeconds: 10         # Check every 10 seconds (not every 1)
          timeoutSeconds: 5
```

**New timeline**:
```
09:00:00 - Container starts
09:00:20 - Kubelet sends first GET /health
         - App is now loaded and listening
         - Returns 200 OK
09:00:20 - Liveness probe PASSES
09:00:30 - Kubelet sends second GET /health (10 seconds later)
         - Returns 200 OK
...continues running successfully
```

**Deploy and verify**:
```bash
kubectl apply -f analytics-deployment.yaml
kubectl get pods  # Pod now running!
kubectl describe pod analytics-pod
# Check events - liveness probe should now be passing
```

**Probe tuning considerations**:
```
initialDelaySeconds:
- Small apps (nginx): 5-10 seconds
- Medium apps (Node.js): 10-15 seconds
- Large apps (Java, ML models): 20-60 seconds

periodSeconds (how often to check):
- Aggressive monitoring: 5-10 seconds
- Normal monitoring: 10-30 seconds
- Don't set too low (< 5) as it stresses the app

timeoutSeconds:
- Normal services: 1-3 seconds
- Slow services: 5-10 seconds
- Should be less than periodSeconds
```

**Key Learning**: Application-specific startup time matters for probe configuration

---

## Troubleshooting Checklist

When you see CrashLoopBackOff:

```bash
# 1. Check restart count and exit code
kubectl describe pod <pod-name>
# Look for: Restart Count, Exit Code, Last State

# 2. Check logs (current)
kubectl logs <pod-name>
# Shows current startup output

# 3. Check previous logs (from crash)
kubectl logs <pod-name> --previous
# Shows what happened before crash

# 4. Check events
kubectl describe pod <pod-name>
# Look for: OOMKilled, Liveness probe failed, etc.

# 5. If exit code 1 (application error)
# → Check logs for missing config/env vars
# → Check if dependencies are available
# → Check if files have correct permissions

# 6. If exit code 137 (external kill)
# → Check if it's OOMKilled → increase memory limit
# → Check if liveness probe failing → fix endpoint or delay
# → Check if liveness probe timing out → increase initialDelaySeconds

# 7. Check for volume/mount issues
kubectl describe pod <pod-name> | grep -A 5 "Mounts:"
kubectl describe pod <pod-name> | grep -A 5 "Volumes:"

# 8. Check resource limits vs actual usage
kubectl top pod <pod-name> --containers
# Compare with limits defined in pod spec

# 9. If unsure, test the image locally
docker run -it <image>:<tag> /bin/bash
# Check permissions, test startup command
```

---

## Interview Story

**Setup:**
"In production, we had multiple pods in CrashLoopBackOff state. They weren't all failing for the same reason, which is why understanding the exit codes and events was critical."

**Investigation Process:**
"I used `kubectl describe pod` to look at three things: the exit code (which tells you the type of failure), the events section (which shows what kubelet observed), and the logs (which show application output).

Exit code 1 meant application error—I'd check logs for missing environment variables or configuration.

Exit code 137 meant external SIGKILL—I'd check if the liveness probe was failing or if memory was exceeded."

**Examples:**
1. "MySQL pod had exit code 1 because MYSQL_ROOT_PASSWORD env var wasn't set. Fixed by creating a Secret and mounting it.

2. "Orders API had script permission issues—couldn't execute the startup script. Tested the image locally with Docker to find this.

3. "Search/Nginx pod had volume defined but wasn't mounted to the container path. Common mistake: creating volumes but forgetting volumeMounts.

4. "Shipping API had exit code 137 (OOMKilled). Used metrics-server to find actual memory usage, then increased the memory limit.

5. "Notifications pod had liveness probe pointing to wrong endpoint (/healthz instead of /health). The 404 error in events was the giveaway.

6. "Analytics pod had liveness probe failing because initialDelaySeconds was too low. The app needed 20 seconds to load, but probe started checking after 1 second. Increased the delay and it stabilized."

**Key Insight:**
"The critical thing is understanding that CrashLoopBackOff is a symptom, not the error. The exit code and events tell you what actually happened. Exit code 1 means look at logs. Exit code 137 means look at why something killed the process—usually liveness probe or OOMKilled."

---

## Common Mistakes in Probe Configuration

### Mistake 1: initialDelaySeconds Too Low

```yaml
# WRONG - checks too early
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 1  # App not ready yet!
```

### Mistake 2: periodSeconds Too Low

```yaml
# WRONG - hammers the app with checks
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  periodSeconds: 1  # Checks every 1 second - stresses app
```

### Mistake 3: Wrong Endpoint

```yaml
# WRONG - app doesn't have this endpoint
livenessProbe:
  httpGet:
    path: /healthz  # Should be /health
    port: 8080
```

### Mistake 4: No readinessProbe, only livenessProbe

```yaml
# WRONG - pod gets traffic before it's ready
containers:
- name: app
  image: myapp:1.0
  livenessProbe:
    httpGet:
      path: /health
      port: 8080
  # Missing readinessProbe!
```

**Correct approach**:
```yaml
containers:
- name: app
  image: myapp:1.0
  
  # Readiness: Is app ready to accept traffic?
  readinessProbe:
    httpGet:
      path: /health
      port: 8080
    initialDelaySeconds: 10
    periodSeconds: 5
  
  # Liveness: Is app still alive?
  livenessProbe:
    httpGet:
      path: /health
      port: 8080
    initialDelaySeconds: 30
    periodSeconds: 10
```

---

## Key Takeaways

1. **Exit Code 1** = Application error → Check logs for why app is failing to start

2. **Exit Code 137** = External SIGKILL → Check if OOMKilled or liveness probe failed

3. **CrashLoopBackOff is a symptom** → Find the root cause using exit codes, events, and logs

4. **Liveness probe configuration is app-specific** → Tune based on actual startup time

5. **Volume + VolumeMount are both needed** → One without the other doesn't work

6. **Always have adequate resource limits** → Too tight causes OOMKilled

7. **Test images locally with Docker** → Faster than debugging in pod

8. **Know your application** → Understand startup requirements and timing