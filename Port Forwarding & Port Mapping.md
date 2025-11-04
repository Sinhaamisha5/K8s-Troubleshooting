# Port Forwarding & Port Mapping: Complete Troubleshooting Guide

## The Problem: Application Unreachable

### The Scenario

You deploy an application with a Service and Deployment. You try to access it locally using `kubectl port-forward`, but connection fails.

```
$ kubectl port-forward service/neoncat 3000:80
Forwarding from 127.0.0.1:3000 -> 8000  ← Error here!
error: error forwarding port 8000 to pod: unable to do port forwarding
```

The error says port 8000 is unreachable, but you only specified port 80. What's going on?

---

## The Three Port Layers

Understanding port mapping requires understanding three distinct ports:

### Layer 1: Local Port (Your Machine)

```
Your Machine:
Port 3000 (the port YOU choose for your machine)
```

This is the port you use when accessing from your local machine:
```bash
curl localhost:3000
```

### Layer 2: Service Port (Service Frontend)

```
Kubernetes Service:
Port 80 (the port SERVICE listens on)
```

This is exposed by the Service resource. External traffic enters here.

```yaml
service:
  spec:
    ports:
    - port: 80           # ← Service port (frontend)
      targetPort: 8000   # ← Container port (backend)
```

### Layer 3: Container Port (Pod Backend)

```
Container/Pod:
Port 8000 (where the application actually listens)
```

This is where your application is actually running inside the container.

```yaml
deployment:
  spec:
    template:
      spec:
        containers:
        - containerPort: 8000  # ← Where app listens
```

### Complete Flow

```
Your Machine (Port 3000)
           ↓
       kubectl port-forward
           ↓
Service Port (Port 80)
           ↓
Service Routes to Container Port
           ↓
Container Port (Port 8000)
           ↓
Application receives request
```

---

## Real Example: The Port Mismatch

### The Setup

**Service Definition:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: neoncat
spec:
  selector:
    app: neoncat
  ports:
  - port: 80           # ← Service listens on 80
    targetPort: 8000   # ← Forwards to 8000
  type: ClusterIP
```

**Deployment Definition:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: neoncat
spec:
  template:
    spec:
      containers:
      - name: neoncat
        image: neoncat:1.0
        ports:
        - containerPort: 80  # ← Application runs on 80!
```

### The Problem

```
Service says: "Forward traffic to port 8000"
Container says: "I'm listening on port 80"

Result: Port mismatch!
Service tries to route to 8000, but container listening on 80
Connection refused!
```

### Step 1: Try Port Forward

```bash
kubectl port-forward service/neoncat 3000:80
```

Output:
```
Forwarding from 127.0.0.1:3000 -> 8000
error: error forwarding port 8000 to pod: unable to do port forwarding
```

**Key observation:** It says "Forwarding from 3000 to 8000", but we only specified port 80!

### Step 2: Understand the Error Message

The port-forward output tells you the complete path:
```
Forwarding from 127.0.0.1:3000 -> 8000
                   ↑ Local port    ↑ Target port (where it tries to connect)
```

Breaking it down:
- Local: 3000 (your machine)
- Service port: 80 (from your command: `3000:80`)
- Container port: 8000 (from Service's targetPort)

Result: 3000 → 80 → 8000

But the container is actually listening on port 80, not 8000!

### Step 3: Check Service Configuration

```bash
kubectl get service neoncat -o yaml
```

Output:
```yaml
spec:
  ports:
  - port: 80
    targetPort: 8000      # ← WRONG! Should be 80
```

### Step 4: Check Container Port

```bash
kubectl get deployment neoncat -o yaml
```

Output:
```yaml
spec:
  template:
    spec:
      containers:
      - name: neoncat
        ports:
        - containerPort: 80   # ← Container is on port 80
```

**Mismatch found!**
- Service targetPort: 8000
- Container port: 80

### Step 5: Fix the Service

```bash
kubectl edit service neoncat
```

Change:
```yaml
# BEFORE (wrong)
ports:
- port: 80
  targetPort: 8000

# AFTER (correct)
ports:
- port: 80
  targetPort: 80
```

Save and exit.

### Step 6: Try Port Forward Again

```bash
kubectl port-forward service/neoncat 3000:80
```

Output:
```
Forwarding from 127.0.0.1:3000 -> 80
Forwarding from [::1]:3000 -> 80
```

✅ **Now it shows "-> 80" which matches the container port!**

### Step 7: Test Access

```bash
curl localhost:3000
```

Output:
```html
<html>
  <body>
    <h1>NeonCat App</h1>
  </body>
</html>
```

✅ **Success! Application is now accessible!**

---

## Port Mapping Explained

### The Three Scenarios

#### Scenario 1: All Ports the Same (Simplest)

```yaml
# Service
ports:
- port: 8080           # Service port
  targetPort: 8080     # Container port

# Deployment
containerPort: 8080    # Application port

# Port Forward Command
kubectl port-forward service/myapp 8080:8080

# Flow:
Your Machine (8080)
  ↓
Service (8080)
  ↓
Container (8080)
  ↓
Application (listening on 8080) ✓
```

**Easiest:** Everything on same port.

#### Scenario 2: Port Forwarding Maps Local to Service

```yaml
# Service (unchanged)
ports:
- port: 8080
  targetPort: 8080

# Deployment (unchanged)
containerPort: 8080

# Port Forward Command
kubectl port-forward service/myapp 3000:8080
                                   ↑ Local  ↑ Service port

# Flow:
Your Machine (3000) [what YOU type in curl/browser]
  ↓
Service (8080) [from port-forward: port parameter]
  ↓
Container (8080) [from Service: targetPort]
  ↓
Application (listening on 8080) ✓

# To access:
curl localhost:3000  # NOT localhost:8080!
```

#### Scenario 3: Service Maps to Different Container Port

```yaml
# Service
ports:
- port: 80           # Outside port
  targetPort: 8080   # Container port (different!)

# Deployment
containerPort: 8080

# Port Forward Command (use Service port: 80)
kubectl port-forward service/myapp 3000:80

# Flow:
Your Machine (3000) [what YOU type]
  ↓
Service (80) [from port-forward: port parameter]
  ↓
Container (8080) [from Service: targetPort]
  ↓
Application (listening on 8080) ✓

# To access:
curl localhost:3000  # NOT localhost:8080!
```

**Key:** Port-forward always uses Service port, Service's targetPort must match container port.

---

## Port Mapping Terminology

### Service Port

```yaml
ports:
- port: 80  # ← This is the Service Port (frontend)
  targetPort: 8000
```

- Exposed by the Service
- What external traffic hits first
- Used in kubectl port-forward command
- Used by Service discovery (DNS)

### Target Port

```yaml
ports:
- port: 80
  targetPort: 8000  # ← This is the Target Port (backend)
```

- Where Service forwards traffic to
- Must match container port
- Can be port number or port name
- What kubelet actually connects to

### Container Port

```yaml
containers:
- name: app
  image: myapp:1.0
  ports:
  - containerPort: 8000  # ← This is Container Port
```

- Where application actually listens
- Must match Service targetPort
- What's inside the container
- Not technically required (but good practice)

---

## The Complete Port Mapping Decision Tree

```
1. Application runs on port X
   ↓
2. Container exposes port X
   └─ containerPort: X
   ↓
3. Service routes to port X
   └─ targetPort: X
   ↓
4. Service listens on port Y (usually Y=X, can be different)
   └─ port: Y
   ↓
5. kubectl port-forward uses Service port
   └─ kubectl port-forward service/name LOCALPORT:Y
   ↓
6. Access via localhost:LOCALPORT
   └─ curl localhost:LOCALPORT
```

---

## Common Port Mapping Mistakes

### Mistake 1: Mismatched Service and Container Ports

```yaml
# WRONG
Service:
  targetPort: 8000

Deployment:
  containerPort: 80  # ← MISMATCH!
```

**Result:** Service tries to connect to 8000, but container listening on 80 → Connection refused

**Fix:** Make them match
```yaml
Service:
  targetPort: 80  # ← Now matches

Deployment:
  containerPort: 80
```

### Mistake 2: Using Container Port in Port Forward

```bash
# WRONG
kubectl port-forward service/myapp 3000:8000
# ↑ Using container port directly

# Correct
kubectl port-forward service/myapp 3000:80
# ↑ Using Service port (which routes to 8000)
```

### Mistake 3: Not Understanding Port Forward Mapping

```bash
# CONFUSING
kubectl port-forward service/myapp 3000:8080

# Question: Does this mean:
# A) Local 3000 → Container 8080?
# B) Local 3000 → Service 8080 → Container 8080?

# Answer: B! First parameter is LOCAL port, second is SERVICE port
# Service then forwards to container based on targetPort
```

### Mistake 4: Using Container Port Number in Service Port

```yaml
# DON'T (confusing):
ports:
- port: 8000           # Service port same as container?
  targetPort: 8000     # Happens to work but confusing

# DO (clear):
ports:
- port: 80             # Standard HTTP port for Service
  targetPort: 8000     # Application runs on 8000
```

### Mistake 5: Forgetting Port Forward is Temporary

```bash
kubectl port-forward service/myapp 3000:80

# This only works:
# - While the port-forward is running
# - From YOUR local machine
# - On YOUR local port 3000

# It does NOT:
# - Make app accessible to others
# - Make app accessible outside localhost
# - Persist after port-forward is closed
```

---

## Troubleshooting Guide: Application Not Accessible

### Step 1: Check Service Exists

```bash
kubectl get svc myapp
```

If not found: Create the Service first!

### Step 2: Check Service Port Configuration

```bash
kubectl get svc myapp -o yaml
```

Look for:
```yaml
ports:
- port: 80           # ← Service port
  targetPort: 8000   # ← Should match container port
```

### Step 3: Check Deployment/Pod Container Port

```bash
kubectl get deployment myapp -o yaml
```

Look for:
```yaml
containers:
- containerPort: 8000  # ← Should match targetPort
```

### Step 4: Check Service Has Endpoints

```bash
kubectl get endpoints myapp
```

Output should show pod IPs, e.g.:
```
ENDPOINTS
10.244.1.5:8000,10.244.1.6:8000
```

If empty: Pods not matching service selector or not ready!

### Step 5: Verify Ports Match

- Container port (from deployment spec) 
- Should equal Service targetPort
- Should NOT equal kubectl port-forward's second parameter!

```yaml
# Correct Matching:
Service targetPort: 8000  ← Must match...
Deployment containerPort: 8000  ← ...this!

# Port Forward (uses Service port, not container port):
kubectl port-forward service/myapp 3000:80  ← 80 is Service port
```

### Step 6: Try Direct Connection to Pod

```bash
# Get pod name
kubectl get pods

# Try direct port-forward to pod (bypasses service)
kubectl port-forward pod/myapp-abc123 3000:8000

# If this works but service doesn't: Service misconfiguration
# If this fails: Container or pod issue
```

### Step 7: Check Pod Logs

```bash
kubectl logs myapp-abc123

# Look for:
# - "Listening on port X" messages
# - "Address already in use" errors
# - Connection refused errors
```

### Step 8: Exec Into Pod and Check

```bash
kubectl exec -it myapp-abc123 -- /bin/sh

# Inside pod:
# Check what port app is actually listening on
netstat -tlnp | grep LISTEN

# Should see port 8000 or whatever containerPort is
```

---

## Kubectl Port Forward Usage

### Basic Syntax

```bash
kubectl port-forward <resource/name> [LOCAL_PORT:]SERVICE_PORT
```

### Examples

#### Forward to Service

```bash
# Format: SERVICE_PORT is the Service's port (NOT container port!)
kubectl port-forward service/myapp 8080:80
# Local 8080 → Service port 80 → Container port (from targetPort)

# Different local port
kubectl port-forward service/myapp 3000:80
# Local 3000 → Service port 80

# Same port everywhere
kubectl port-forward service/myapp 8080:8080
```

#### Forward to Pod

```bash
# Format: CONTAINER_PORT (bypasses service)
kubectl port-forward pod/myapp-abc123 8080:8080
# Local 8080 → Pod container port 8080

# Bypass service to debug
kubectl port-forward pod/myapp-abc123 3000:8000
```

#### Forward to Deployment

```bash
# Forwards to first pod in deployment
kubectl port-forward deployment/myapp 8080:8080
```

### Common Flags

```bash
# Listen on all interfaces (not just localhost)
kubectl port-forward service/myapp 8080:8080 --address 0.0.0.0

# Listen on specific interface
kubectl port-forward service/myapp 8080:8080 --address 10.0.0.1

# In background
kubectl port-forward service/myapp 8080:8080 &
```

---

## Interview Story: The Port Mapping Mystery

**The Problem:**
"I deployed an application but couldn't access it locally using kubectl port-forward. The error said 'error forwarding port 8000 to pod', but I only specified port 80. I was confused."

**Investigation:**
"I checked the error message more carefully. It said 'Forwarding from 127.0.0.1:3000 -> 8000'. That told me port 8000 was unreachable. I didn't specify 8000 anywhere in my command, so where did it come from?"

**Root Cause Discovery:**
"I checked the Service definition and found targetPort: 8000. So the flow was: Local 3000 → Service port 80 → Container port 8000. But when I checked the Deployment, the container was listening on port 80, not 8000! That's the mismatch."

**The Fix:**
"I updated the Service's targetPort from 8000 to 80 to match the container port. Then port-forward worked immediately."

**Key Insight:**
"The important lesson: understand the three port layers. Service targetPort MUST match the container port. kubectl port-forward uses the Service port, not the container port. The error message showed me the complete path, which made it clear where the mismatch was."

---

## Best Practices

### 1. Match Service and Container Ports

```yaml
# Always ensure these match:
Service:
  targetPort: 8080

Deployment:
  containerPort: 8080
```

### 2. Use Standard Ports When Possible

```yaml
# For HTTP: port 80
# For HTTPS: port 443
# For your app: consistent internal port

# Good:
port: 80
targetPort: 8080  # Internal application port

# Avoid:
port: 7392
targetPort: 4831  # Confusing!
```

### 3. Use Named Ports for Clarity

```yaml
# Service
ports:
- name: http
  port: 80
  targetPort: http-backend

# Deployment
containers:
- ports:
  - name: http-backend
    containerPort: 8080

# Now it's clear what connects where!
```

### 4. Document Port Mapping

```yaml
# In Service annotations
apiVersion: v1
kind: Service
metadata:
  name: myapp
  annotations:
    ports: "Service port 80 → Container port 8080"
spec:
  ports:
  - port: 80
    targetPort: 8080
```

### 5. Test Locally Before Production

```bash
# Always test port-forward locally first:
kubectl port-forward service/myapp 3000:80
curl localhost:3000

# Verify it works before deploying to prod
```

---

## Quick Reference: Port Mapping Cheatsheet

| Component | What It Is | Where It's Configured | Typical Value |
|-----------|-----------|----------------------|---------------|
| **Local Port** | Port on YOUR machine | kubectl port-forward cmd | 3000, 8080, etc |
| **Service Port** | Frontend (external entry) | Service spec.ports.port | 80, 443, 8080 |
| **Target Port** | Backend (container entry) | Service spec.ports.targetPort | 8080, 8000 |
| **Container Port** | Where app listens | Deployment containerPort | 8080, 3000 |

**Golden Rule:**
```
Service targetPort == Container port
Both must match for traffic to flow!

kubectl port-forward uses Service port
Not container port!
```

---

## Key Takeaways

1. **Three port layers exist** → Local, Service, Container

2. **Service targetPort must match Container port** → Critical!

3. **kubectl port-forward uses Service port** → Not container port

4. **Port mismatch causes connection refused** → Check all three layers

5. **Error message shows complete path** → "Forwarding from X -> Y" shows where problem is

6. **Port-forward is temporary** → Only while running, localhost only

7. **Pod port is not the same as Service port** → They're different concepts

8. **Named ports reduce confusion** → Use them in production

9. **Bypass Service with direct pod forward** → For debugging

10. **Document port mappings** → Help future troubleshooters