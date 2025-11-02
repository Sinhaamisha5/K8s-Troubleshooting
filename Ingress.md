# Ingress Controller Troubleshooting: Path Rewriting and Nginx Configuration

## The Problem: Ingress Returning 404 While Services Work

### The Scenario

You have an Ingress that routes traffic to services, but all requests return 404, even though the services work fine when accessed directly.

```
Direct access (port-forward): ✅ Works perfectly
Access through Ingress: ❌ Returns 404 from application

The services are working, the Ingress controller is working,
but something is wrong between them.
```

### The Setup

**Services:**
- `/techie` → `techie-service:80`
- `/k8s` → `k8s-service:80`
- `/useless` → `useless-service:80`

**Ingress Definition:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /techie
        pathType: Prefix
        backend:
          service:
            name: techie-service
            port:
              number: 80
      - path: /k8s
        pathType: Prefix
        backend:
          service:
            name: k8s-service
            port:
              number: 80
      - path: /useless
        pathType: Prefix
        backend:
          service:
            name: useless-service
            port:
              number: 80
```

### Step 1: Test Direct Access (Works)

```bash
# Port-forward to service
kubectl port-forward service/techie-service 4444:80

# In another terminal
curl localhost:4444
# Output:
# "All we have to do is hash the capacity adapter"
# ✅ Service works!
```

### Step 2: Test Through Ingress (Fails)

```bash
# Get Ingress IP
INGRESS_IP=$(kubectl get ingress app-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Try to access
curl -v $INGRESS_IP/techie
# Output:
# < HTTP/1.1 404 Not Found
# "Cannot GET /techie"
# ❌ Ingress returns 404!
```

### Step 3: Check Ingress Controller Logs

```bash
kubectl logs -n ingress-nginx deployment/nginx-ingress-controller

# Output:
# GET /techie HTTP/1.1" 404
# GET /k8s HTTP/1.1" 404
# GET /useless HTTP/1.1" 404
```

The Ingress controller received requests, forwarded them to services, and got 404 responses back.

### Step 4: Understand the Problem

The issue: **The path is being passed to the application!**

```
Request: GET /techie
  ↓
Ingress Controller routes to techie-service
  ↓
Request passed to service: GET /techie  ← WRONG!
  ↓
Application receives: GET /techie
  ↓
Application only knows: GET /
  ↓
Response: 404 Cannot GET /techie
```

The application doesn't know about the `/techie` path—that's only for the Ingress to differentiate traffic!

---

## Understanding Path Rewriting

### The Concept

Ingress paths are for **routing decisions**, not application paths.

```
Ingress path = Routing rule (tells Ingress where to send traffic)
Application path = What the app actually sees

They are NOT the same!

Example:
Ingress path: /techie  (used to route to techie-service)
Application path: /    (what techie-service expects)
```

### How Nginx Ingress Controller Works

```
1. Request comes: GET /techie/resource
2. Nginx matches: path /techie → techie-service
3. Nginx forwards: GET /techie/resource to service
4. Application sees: GET /techie/resource
5. Application wants: GET /resource (just the resource part!)
6. Result: 404 (application doesn't have /techie route)
```

### The Solution: rewrite-target Annotation

The `rewrite-target` annotation tells Nginx to rewrite the path before sending to the backend.

```
rewrite-target = What to pass to the backend

Example:
rewrite-target: /

Transforms: GET /techie/resource → GET /resource
```

---

## The rewrite-target Annotation

### Basic Syntax

```yaml
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /
```

### How It Works with Regex Capture Groups

#### Example 1: Simple Rewrite

```yaml
path: /techie(/|$)(.*)
rewrite-target: /$2
```

**Explanation:**
- `path` regex: `/techie(/|$)(.*)`
  - `$1` (first capture group): `(/|$)` - slash or end of string
  - `$2` (second capture group): `(.*)` - everything after

- `rewrite-target: /$2` - Pass only the second capture group

**Flow:**
```
Request: GET /techie/hello/world
  ↓
Regex matches:
  - $1 = / (the slash)
  - $2 = hello/world (everything after slash)
  ↓
Rewrite target: /$2 = /hello/world
  ↓
Application receives: GET /hello/world
```

#### Example 2: Multiple Services

```yaml
- path: /techie(/|$)(.*)
  rewrite-target: /$2
  backend:
    service:
      name: techie-service
      
- path: /k8s(/|$)(.*)
  rewrite-target: /$2
  backend:
    service:
      name: k8s-service
      
- path: /useless(/|$)(.*)
  rewrite-target: /$2
  backend:
    service:
      name: useless-service
```

Each service gets the path rewritten independently.

#### Example 3: Simple Case (What We Need)

```yaml
path: /
rewrite-target: /
```

This says: "Forward the exact path without modification."

But actually, for our case, we just need:

```yaml
rewrite-target: /
```

This rewrites ALL paths to `/`, stripping the prefix.

### Common Patterns

#### Pattern 1: Strip Prefix

```yaml
# User requests: GET /api/resource
# Path regex: /api/(.*)
# rewrite-target: /$1
# Service sees: GET /resource
```

#### Pattern 2: Preserve Sub-Path

```yaml
# User requests: GET /v1/users/123
# Path regex: /v1/(.*)
# rewrite-target: /api/$1
# Service sees: GET /api/users/123
```

#### Pattern 3: No Rewrite

```yaml
# User requests: GET /resource
# Path regex: /
# rewrite-target: (none, or /)
# Service sees: GET /resource (unchanged)
```

---

## Real Example: Fixing Our Ingress

### The Broken Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /techie
        pathType: Prefix
        backend:
          service:
            name: techie-service
            port:
              number: 80
      # More paths...
```

**Problem:** No `rewrite-target`, so `/techie` is passed to the application.

### The Fix

Add `rewrite-target` annotation:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /techie
        pathType: Prefix
        backend:
          service:
            name: techie-service
            port:
              number: 80
      - path: /k8s
        pathType: Prefix
        backend:
          service:
            name: k8s-service
            port:
              number: 80
      - path: /useless
        pathType: Prefix
        backend:
          service:
            name: useless-service
            port:
              number: 80
```

### Apply and Test

```bash
kubectl apply -f ingress-fixed.yaml

# Test again
curl $INGRESS_IP/techie
# Output:
# "All we have to do is hash the capacity adapter"
# ✅ Works!

curl $INGRESS_IP/k8s
# Output:
# "Life is the ultimate gift"
# ✅ Works!

curl $INGRESS_IP/useless
# Output:
# "Obsession is the most popular boat name"
# ✅ Works!
```

---

## Understanding Nginx Configuration

### Inside the Ingress Controller Pod

The Ingress controller generates an Nginx config file based on your Ingress definition:

```bash
# Find Nginx pod
kubectl get pods -n ingress-nginx

# Exec into it
kubectl exec -it <nginx-pod> -n ingress-nginx -- /bin/sh

# Find Nginx config
find / -name "nginx.conf" 2>/dev/null

# View it
cat /etc/nginx/nginx.conf

# Or view specific server config
cat /etc/nginx/conf.d/default.conf
```

### What the Nginx Config Looks Like

```nginx
server {
    listen 80;
    server_name _;

    # Location blocks (one per Ingress path)
    location ~ ^/techie {
        rewrite ^/techie/(.*)$ /$1 break;  # ← From rewrite-target
        proxy_pass http://techie-service;
    }

    location ~ ^/k8s {
        rewrite ^/k8s/(.*)$ /$1 break;
        proxy_pass http://k8s-service;
    }

    location ~ ^/useless {
        rewrite ^/useless/(.*)$ /$1 break;
        proxy_pass http://useless-service;
    }
}
```

Key components:
- `location` blocks: Match incoming paths
- `rewrite`: Transforms the path
- `proxy_pass`: Forwards to backend service

### Checking Timeouts in Nginx Config

```bash
# Search for timeout settings
kubectl exec -it <nginx-pod> -n ingress-nginx -- grep -i timeout /etc/nginx/nginx.conf

# Typical output:
# proxy_connect_timeout 60s;
# proxy_send_timeout 60s;
# proxy_read_timeout 60s;
```

These are configured via Ingress annotations:
```yaml
annotations:
  nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
  nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
```

---

## Troubleshooting Checklist

```bash
# 1. Verify Ingress exists and has correct paths
kubectl get ingress
kubectl describe ingress <name>

# 2. Check Ingress has endpoints (backend services exist)
kubectl get endpoints

# 3. Verify services are running
kubectl get svc

# 4. Test service directly (port-forward)
kubectl port-forward service/<name> 8080:80

# 5. Check Ingress controller logs
kubectl logs -n ingress-nginx deployment/nginx-ingress-controller

# 6. Verify rewrite-target annotation exists
kubectl get ingress <name> -o yaml | grep rewrite-target

# 7. Verify Ingress IP is set
kubectl get ingress -o wide

# 8. Test Ingress access
curl <INGRESS_IP>/<path>

# 9. Check Nginx config inside pod
kubectl exec -it <nginx-pod> -n ingress-nginx -- cat /etc/nginx/nginx.conf

# 10. Verify path regex correctness
# Ingress path regex must match what you're requesting
```

---

## Common Ingress Issues

### Issue 1: Missing rewrite-target

**Symptom:** Service works with port-forward, returns 404 through Ingress

**Cause:** Path is being forwarded to application unchanged

**Solution:** Add `rewrite-target` annotation

### Issue 2: Incorrect Path Type

```yaml
# WRONG
pathType: Exact
path: /api
# This only matches /api, not /api/users

# CORRECT
pathType: Prefix
path: /api
# This matches /api, /api/users, /api/anything
```

### Issue 3: Service Not Found

```yaml
# WRONG
backend:
  service:
    name: nonexistent-service
    port:
      number: 80

# Result: 503 Service Unavailable
```

### Issue 4: Wrong Port

```yaml
# WRONG
backend:
  service:
    name: my-service
    port:
      number: 8080  # Service is actually on 80

# Result: Connection Refused
```

### Issue 5: Wrong Ingress Class

```yaml
# WRONG
ingressClassName: traefik  # You're using nginx!

# Result: Ingress not processed
```

### Issue 6: Regex Path Not Matching

```yaml
# If you request /api/users
# But path is /v1(/|$)(.*)
# They don't match! Request returns 404
```

---

## Interview Story: The 404 Mystery

**The Problem:**
"We deployed a service with an Ingress, but accessing it through the Ingress returned 404 errors. The same service worked fine when accessed directly with port-forward. This was really confusing—why would it work one way but not the other?"

**Investigation:**
"I checked the Ingress controller logs and saw it was successfully routing requests to the services, but getting 404 responses back. I also tested the services directly and confirmed they worked. So the services were fine, the Ingress was routing correctly, but something was wrong with what the services were receiving."

**Debugging:**
"I realized the Ingress was passing the full path to the services. When someone requested `/techie/resource`, the service received `GET /techie/resource`. But the service only knew about `GET /resource` (without the `/techie` prefix). The `/techie` path was just for the Ingress to route traffic—the application didn't need it!"

**The Solution:**
"I added the `rewrite-target: /` annotation to the Ingress. This tells Nginx to strip the prefix and pass just `/resource` to the service. After that, everything worked."

**Key Insight:**
"The important lesson: Ingress paths are for routing decisions, not application paths. You need to rewrite the path before forwarding to the backend. Understanding how Nginx processes requests—location blocks, rewrite rules, proxy_pass—is crucial for debugging Ingress issues."

---

## Best Practices

### 1. Always Consider Path Rewriting

```yaml
# If your paths are for routing (not app-aware):
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /

# If your paths are app-aware:
# (No rewrite needed)
```

### 2. Document Path Strategy

```yaml
metadata:
  annotations:
    description: "Ingress paths are routing prefixes, not app paths"
    routing: "/techie → techie-service"
    rewriting: "Strips /techie prefix before forwarding"
```

### 3. Test Both Ways

```bash
# 1. Direct service access (port-forward)
kubectl port-forward service/techie-service 8080:80
curl localhost:8080

# 2. Through Ingress
curl $INGRESS_IP/techie

# Both should work!
```

### 4. Check Nginx Config During Troubleshooting

```bash
# Don't guess what Nginx is doing—check the actual config
kubectl exec -it <nginx-pod> -n ingress-nginx -- cat /etc/nginx/nginx.conf

# This is the source of truth
```

### 5. Use Clear Path Structures

```yaml
# GOOD: Clear separation of concerns
paths:
- path: /api/v1/users(/|$)(.*)
  rewrite-target: /$2
  backend: users-service
  
- path: /api/v1/orders(/|$)(.*)
  rewrite-target: /$2
  backend: orders-service

# BAD: Ambiguous
paths:
- path: /something
  backend: someservice
```

---

## Key Takeaways

1. **Ingress paths ≠ Application paths** → They're for routing, not app logic

2. **rewrite-target is essential** → Strips prefixes before forwarding

3. **Nginx config is the source of truth** → Check it when troubleshooting

4. **Test both direct and through Ingress** → Identifies where problem is

5. **Understand Nginx location blocks** → How paths are evaluated

6. **Use capture groups for complex rewrites** → $1, $2, etc.

7. **Service must be reachable** → Check endpoints

8. **Port numbers must match** → Service port vs container port

9. **Ingress class must be correct** → nginx vs traefik vs others

10. **Document path strategies** → Help future troubleshooters

---

## Nginx Ingress Controller Key Annotations

```yaml
# Path rewriting
nginx.ingress.kubernetes.io/rewrite-target: /

# Timeouts
nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
nginx.ingress.kubernetes.io/proxy-read-timeout: "60"

# Buffering
nginx.ingress.kubernetes.io/proxy-buffering: "off"
nginx.ingress.kubernetes.io/proxy-buffer-size: "4k"

# Headers
nginx.ingress.kubernetes.io/enable-cors: "true"
nginx.ingress.kubernetes.io/cors-allow-origin: "*"

# Rate limiting
nginx.ingress.kubernetes.io/limit-rps: "10"

# SSL
nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

All of these compile into the Nginx config and affect behavior!