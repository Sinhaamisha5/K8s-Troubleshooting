# Kubernetes DNS Study Notes

## Table of Contents
1. [DNS Basics in Kubernetes](#dns-basics-in-kubernetes)
2. [DNS Server Solution](#dns-server-solution)
3. [Service DNS Records](#service-dns-records)
4. [Pod DNS Records](#pod-dns-records)
5. [CoreDNS Configuration](#coredns-configuration)
6. [DNS Resolution Process](#dns-resolution-process)
7. [Practical Examples](#practical-examples)
8. [Common Issues and Solutions](#common-issues-and-solutions)

---

## DNS Basics in Kubernetes

### What is DNS in Kubernetes?
DNS (Domain Name System) in Kubernetes allows pods and services to communicate with each other using names instead of IP addresses. Every Kubernetes cluster has a built-in DNS server that automatically manages DNS records.

### Key Concepts
- **Cluster DNS**: A centralized DNS server deployed in every Kubernetes cluster
- **Default DNS**: CoreDNS (since Kubernetes v1.12), previously Kube-DNS
- **Automatic Registration**: Services and pods get DNS records automatically
- **Namespace Awareness**: DNS names include namespace information

---

## DNS Server Solution

### CoreDNS Deployment

CoreDNS runs as pods in the `kube-system` namespace:

```bash
# View CoreDNS pods
kubectl get pods -n kube-system

# Typical output shows 2 replicas for high availability
NAME                                   READY   STATUS    RESTARTS   AGE
coredns-74ff55c5b-91cxv               1/1     Running   0          14m
coredns-74ff55c5b-kgtwz               1/1     Running   0          14m
```

### CoreDNS Service

The DNS service is named `kube-dns` with a ClusterIP (typically `10.96.0.10`):

```bash
# View DNS service
kubectl get svc -n kube-system

NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                 AGE
kube-dns     ClusterIP   10.96.0.10     <none>        53/UDP,53/TCP,9153/TCP   16m
```

**Important**: This IP address (`10.96.0.10`) is automatically configured in all pods' `/etc/resolv.conf` file.

---

## Service DNS Records

### DNS Naming Structure for Services

Services get DNS records automatically when created. The naming follows this hierarchy:

```
<service-name>.<namespace>.svc.<cluster-domain>
```

**Default values**:
- Cluster domain: `cluster.local`
- Default namespace: `default`

### Example Service DNS Record

If you have a service named `web-service` in the `default` namespace:

**Full DNS name (FQDN)**:
```
web-service.default.svc.cluster.local
```

### Accessing Services - Multiple Valid Forms

You can access a service using any of these names (from the same namespace):

```bash
# Short name (same namespace only)
curl http://web-service

# With namespace
curl http://web-service.default

# With service subdomain
curl http://web-service.default.svc

# Full FQDN
curl http://web-service.default.svc.cluster.local
```

### Cross-Namespace Access

To access a service in a **different namespace**, you MUST include the namespace:

```bash
# Service in 'apps' namespace
curl http://web-service.apps

# Or use full FQDN
curl http://web-service.apps.svc.cluster.local
```

### DNS Hierarchy Breakdown

```
web-service.apps.svc.cluster.local
    │       │    │       │
    │       │    │       └── Root domain (cluster-wide)
    │       │    └────────── Service subdomain (all services)
    │       └─────────────── Namespace subdomain
    └─────────────────────── Service name
```

---

## Pod DNS Records

### Default Behavior
**Important**: DNS records for pods are **NOT created by default**. This must be explicitly enabled.

### Pod DNS Naming Structure

When enabled, pod DNS records follow this format:

```
<pod-ip-with-dashes>.<namespace>.pod.<cluster-domain>
```

### IP Address Conversion Rule

Pod IP addresses are converted to DNS names by **replacing dots (.) with dashes (-)**.

**Example**:
- Pod IP: `10.244.2.5`
- Pod DNS name: `10-244-2-5.default.pod.cluster.local`

### Accessing Pods via DNS

```bash
# Must use FULL FQDN for pod DNS
curl http://10-244-2-5.default.pod.cluster.local

# Short names DO NOT WORK for pods
curl http://10-244-2-5  # ❌ This will FAIL
```

### Key Differences: Services vs Pods

| Aspect | Services | Pods |
|--------|----------|------|
| **DNS Creation** | Automatic | Must be enabled |
| **Short Names** | ✅ Work within namespace | ❌ Don't work |
| **FQDN Required** | Optional | ✅ Always required |
| **Subdomain** | `.svc` | `.pod` |

---

## CoreDNS Configuration

### Corefile Location

CoreDNS configuration is stored in a file called **Corefile** at:
```
/etc/coredns/Corefile
```

### ConfigMap Storage

The Corefile is managed through a Kubernetes ConfigMap:

```bash
# View CoreDNS ConfigMap
kubectl get configmap -n kube-system

NAME      DATA   AGE
coredns   1      168d

# View ConfigMap contents
kubectl describe configmap coredns -n kube-system
```

### Sample Corefile Configuration

```
.:53 {
    errors                    # Log errors
    health                    # Health check endpoint
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure         # Enable pod DNS records
        upstream              # Use upstream DNS servers
        fallthrough in-addr.arpa ip6.arpa
    }
    prometheus :9153          # Metrics for monitoring
    proxy . /etc/resolv.conf  # Forward external queries
    cache 30                  # Cache DNS responses
    reload                    # Auto-reload on changes
}
```

### Configuration Breakdown

1. **errors**: Logs DNS errors for troubleshooting
2. **health**: Provides health check endpoints for Kubernetes
3. **kubernetes plugin**: Main plugin that integrates with Kubernetes
   - Sets domain to `cluster.local`
   - Enables pod DNS with `pods insecure`
   - Converts pod IPs to dashed format
4. **prometheus**: Exposes metrics on port 9153
5. **proxy**: Forwards external DNS queries (like `www.google.com`) to upstream DNS
6. **cache**: Caches DNS responses for 30 seconds
7. **reload**: Automatically reloads configuration when ConfigMap changes

### Root Domain Configuration

The default root domain is **`cluster.local`** as specified in the Corefile:

```
kubernetes cluster.local in-addr.arpa ip6.arpa {
    ...
}
```

---

## DNS Resolution Process

### Pod DNS Configuration

Every pod gets an automatically configured `/etc/resolv.conf` file:

```bash
# View pod's DNS configuration
cat /etc/resolv.conf

nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
```

### Configuration Breakdown

- **nameserver**: IP of the CoreDNS service (`kube-dns`)
- **search domains**: Automatically appended to short names for resolution

### How Kubelet Configures DNS

The Kubelet automatically configures DNS settings based on its config file:

```yaml
# /var/lib/kubelet/config.yaml
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
```

### DNS Resolution Flow

1. **Pod makes DNS query** → "web-service"
2. **Checks /etc/resolv.conf** → Uses nameserver `10.96.0.10`
3. **Query sent to CoreDNS** → Running in kube-system namespace
4. **CoreDNS checks Kubernetes API** → Looks up service records
5. **Returns IP address** → e.g., `10.107.37.188`
6. **Pod connects to service** → Using the resolved IP

### Search Domain Behavior

When you use a short name like `web-service`, the resolver tries appending search domains:

1. `web-service.default.svc.cluster.local` ✅ (Found!)
2. `web-service.svc.cluster.local`
3. `web-service.cluster.local`

This is why short names work for services in the same namespace.

### Why Pod Names Need FQDN

Search domains only work for the `.svc` subdomain, NOT for `.pod`:

```bash
# Service resolution with short name
host web-service
# Result: web-service.default.svc.cluster.local has address 10.107.37.188

# Pod resolution with short name FAILS
host 10-244-2-5
# Result: Host 10-244-2-5 not found: 3(NXDOMAIN)

# Pod resolution with FQDN WORKS
host 10-244-2-5.default.pod.cluster.local
# Result: 10-244-2-5.default.pod.cluster.local has address 10.244.2.5
```

---

## Practical Examples

### Example 1: Service Communication in Same Namespace

**Scenario**: 
- Test pod in `default` namespace
- Web service in `default` namespace

```bash
# All these work from test pod:
curl http://web-service
curl http://web-service.default
curl http://web-service.default.svc
curl http://web-service.default.svc.cluster.local
```

### Example 2: Cross-Namespace Service Access

**Scenario**:
- Test pod in `default` namespace
- Web service in `payroll` namespace

```bash
# Short name FAILS (wrong namespace)
curl http://web-service  # ❌ NXDOMAIN

# Include namespace (WORKS)
curl http://web-service.payroll  # ✅

# Full FQDN (WORKS)
curl http://web-service.payroll.svc.cluster.local  # ✅
```

### Example 3: Application with Database Connection

**Problem**: Web app in `default` namespace needs MySQL in `payroll` namespace

```yaml
# ❌ WRONG - Won't resolve
env:
  - name: DB_Host
    value: mysql

# ✅ CORRECT - Includes namespace
env:
  - name: DB_Host
    value: mysql.payroll
```

**Fix the deployment**:
```bash
# Edit deployment
kubectl edit deploy webapp

# Update DB_Host value to: mysql.payroll
# Save and Kubernetes will recreate pods
```

### Example 4: DNS Resolution Testing with nslookup

```bash
# From HR pod, test MySQL service resolution

# Wrong - missing namespace
kubectl exec hr -- nslookup mysql
# Result: server can't find mysql: NXDOMAIN

# Correct - includes namespace
kubectl exec hr -- nslookup mysql.payroll
# Result:
# Server:         10.96.0.10
# Address:        10.96.0.10#53
# Name:   mysql.payroll.svc.cluster.local
# Address: 10.98.149.84

# Save output to file
kubectl exec hr -- nslookup mysql.payroll > /root/nslookup.out
```

---

## Common Issues and Solutions

### Issue 1: "Connection Refused" or "Name Not Found"

**Problem**: Cannot reach a service using short name

**Solutions**:
1. Check if service and pod are in same namespace
2. If different namespaces, use format: `service-name.namespace`
3. Verify service exists: `kubectl get svc -n <namespace>`

### Issue 2: Pod DNS Not Resolving

**Problem**: Cannot access pod using DNS name

**Solutions**:
1. Remember: Pod DNS requires FULL FQDN
2. Use format: `<ip-with-dashes>.namespace.pod.cluster.local`
3. Check if pod DNS is enabled in CoreDNS configuration

### Issue 3: External DNS Not Working

**Problem**: Cannot resolve external domains like `www.google.com`

**Solutions**:
1. Check CoreDNS Corefile has `proxy . /etc/resolv.conf`
2. Verify CoreDNS pods are running: `kubectl get pods -n kube-system`
3. Check CoreDNS logs: `kubectl logs -n kube-system coredns-xxx`

### Issue 4: DNS Queries Timeout

**Problem**: DNS resolution is slow or times out

**Solutions**:
1. Check CoreDNS pod resources and limits
2. Verify kube-dns service is accessible: `kubectl get svc -n kube-system`
3. Check network policies aren't blocking DNS traffic (port 53)
4. Increase cache TTL in Corefile if needed

---

## Quick Reference Commands

```bash
# Alias for convenience
alias k='kubectl'

# Check CoreDNS pods
k get pods -n kube-system | grep coredns

# Check DNS service
k get svc -n kube-system kube-dns

# View CoreDNS ConfigMap
k get configmap coredns -n kube-system -o yaml

# Test DNS from a pod
k exec <pod-name> -- nslookup <service-name>
k exec <pod-name> -- cat /etc/resolv.conf

# Check service endpoints
k describe svc <service-name> -n <namespace>

# View service DNS name
k get svc <service-name> -n <namespace>

# Edit CoreDNS configuration
k edit configmap coredns -n kube-system

# Check pod labels (for service selector matching)
k get pods --show-labels
```

---

## DNS Naming Cheat Sheet

### Services
```
# Same namespace
<service-name>
<service-name>.<namespace>
<service-name>.<namespace>.svc
<service-name>.<namespace>.svc.cluster.local

# Example
web-service
web-service.default
web-service.default.svc
web-service.default.svc.cluster.local
```

### Pods
```
# Always requires FQDN
<pod-ip-with-dashes>.<namespace>.pod.cluster.local

# Example
10-244-2-5.default.pod.cluster.local
```

### Headless Services
```
# Same as regular services but no ClusterIP
<service-name>.<namespace>.svc.cluster.local
```

---

## Why Use Services Instead of Pod DNS?

### The Critical Problem with Pod DNS

**Pod IPs are ephemeral (temporary)** - they change when pods are recreated!

```
Pod Lifecycle:
1. Pod created → IP: 10.244.2.5 → DNS: 10-244-2-5.default.pod.cluster.local
2. Pod deleted (crash, update, scaling down)
3. New pod created → IP: 10.244.3.8 → DNS: 10-244-3-8.default.pod.cluster.local
   ❌ OLD DNS NAME NO LONGER WORKS!
```

**Problem**: If other pods communicate using pod DNS, they break when the target pod recreates!

### The Solution: Services Provide Stable DNS Names

Services act as a **stable abstraction layer** in front of pods:

```
Service Architecture:
┌─────────────────────────────────────────┐
│  Service: web-service                   │
│  DNS: web-service.default.svc.cluster.local │
│  IP: 10.107.37.188 (ClusterIP - STABLE)│
└────────────┬────────────────────────────┘
             │ (routes traffic to)
             ├──────────┬──────────┐
             ▼          ▼          ▼
          Pod 1      Pod 2      Pod 3
       10.244.1.5  10.244.2.8  10.244.3.2
       (can die)   (can die)   (can die)
```

### How Services Maintain Communication

**Step-by-Step Example:**

1. **Initial Setup:**
   ```bash
   # Create a deployment with 3 nginx pods
   kubectl create deployment web --image=nginx --replicas=3
   
   # Pods get IPs: 10.244.1.5, 10.244.2.8, 10.244.3.2
   ```

2. **Create a Service:**
   ```bash
   # Service points to pods with label "app=web"
   kubectl expose deployment web --port=80 --name=web-service
   
   # Service gets stable DNS: web-service.default.svc.cluster.local
   # Service gets stable ClusterIP: 10.107.37.188
   ```

3. **Other Pods Connect to Service:**
   ```bash
   # Test pod connects to web-service (NOT to pod IPs directly)
   curl http://web-service  # Uses service DNS - STABLE!
   ```

4. **Pod Dies and Recreates:**
   ```
   Before:
   - Pod 1: 10.244.1.5 ✅
   - Pod 2: 10.244.2.8 ✅
   - Pod 3: 10.244.3.2 ✅
   
   Pod 2 crashes and recreates:
   
   After:
   - Pod 1: 10.244.1.5 ✅
   - Pod 2: 10.244.4.9 ✅ (NEW IP!)
   - Pod 3: 10.244.3.2 ✅
   ```

5. **Communication Still Works:**
   ```bash
   # Test pod STILL connects successfully
   curl http://web-service  # ✅ WORKS!
   
   # Why? Service DNS didn't change!
   # Service automatically updated its endpoints to new pod IP
   ```

### Service Endpoints - The Magic Behind It

Services maintain a list of **endpoints** (pod IPs) automatically:

```bash
# View service details
kubectl get svc web-service
NAME          TYPE        CLUSTER-IP      PORT(S)   AGE
web-service   ClusterIP   10.107.37.188   80/TCP    5m

# View endpoints (actual pod IPs behind the service)
kubectl get endpoints web-service
NAME          ENDPOINTS                                    AGE
web-service   10.244.1.5:80,10.244.2.8:80,10.244.3.2:80   5m

# After pod recreates, endpoints automatically update:
kubectl get endpoints web-service
NAME          ENDPOINTS                                    AGE
web-service   10.244.1.5:80,10.244.4.9:80,10.244.3.2:80   6m
              # Notice 10.244.2.8 changed to 10.244.4.9 automatically!
```

### How Kubernetes Keeps Endpoints Updated

```
1. Pod with label "app=web" is deleted
   ↓
2. Kubernetes controller notices pod is gone
   ↓
3. Removes old pod IP from service endpoints
   ↓
4. New pod is created with label "app=web"
   ↓
5. Kubernetes controller discovers new pod
   ↓
6. Adds new pod IP to service endpoints
   ↓
7. Service DNS unchanged, traffic routes to new pod
   ✅ COMMUNICATION NEVER BREAKS!
```

### Real-World Analogy

Think of it like a business:

**Without Service (Using Pod DNS):**
```
❌ BAD: "Call John at extension 2245"
   Problem: John quits, Sarah hired
   Extension 2245 doesn't work anymore!
```

**With Service (Using Service DNS):**
```
✅ GOOD: "Call the Sales Department at extension 1000"
   Employees come and go, but extension 1000 always works
   Calls automatically route to whoever is working in sales
```

### When Would You Use Pod DNS?

**Almost never in production!** Pod DNS is mainly useful for:

1. **Debugging specific pods:**
   ```bash
   # Connect to a specific pod for troubleshooting
   curl http://10-244-2-5.default.pod.cluster.local
   ```

2. **StatefulSets with stable network identities:**
   ```
   StatefulSets create pods with predictable names:
   - database-0 → database-0.database-service.default.svc.cluster.local
   - database-1 → database-1.database-service.default.svc.cluster.local
   
   These DNS names are stable even when pods restart
   (but they still use services under the hood!)
   ```

3. **Direct pod-to-pod communication for special cases:**
   - Advanced networking scenarios
   - Service mesh implementations
   - Very specific debugging situations

### Summary Table

| Aspect | Pod DNS | Service DNS |
|--------|---------|-------------|
| **Stability** | ❌ Changes on pod restart | ✅ Never changes |
| **Use Case** | Debugging only | Production communication |
| **IP Changes** | ❌ Breaks communication | ✅ Handles automatically |
| **Load Balancing** | ❌ No | ✅ Yes (across multiple pods) |
| **Recommended** | ❌ No | ✅ Yes |

### Best Practice

**ALWAYS use Services for pod-to-pod communication:**

```yaml
# ❌ WRONG - Hardcode pod IP or pod DNS
env:
  - name: DATABASE_HOST
    value: 10-244-2-5.default.pod.cluster.local

# ✅ CORRECT - Use service DNS
env:
  - name: DATABASE_HOST
    value: mysql-service
    # or mysql-service.database (if in different namespace)
```

## Key Takeaways

1. ✅ **CoreDNS** is the default DNS solution (since k8s v1.12)
2. ✅ **Service DNS** is created automatically
3. ✅ **Pod DNS** must be explicitly enabled
4. ✅ **Short names** work for services in the same namespace only
5. ✅ **Cross-namespace** access requires including namespace in DNS name
6. ✅ **Pod DNS** always requires full FQDN
7. ✅ **Search domains** in `/etc/resolv.conf` enable short names for services
8. ✅ **CoreDNS configuration** is stored in a ConfigMap
9. ✅ **Kubelet** automatically configures pod DNS settings
10. ✅ **Default cluster domain** is `cluster.local`
11. ✅ **NEVER use pod DNS for production communication** - pods are ephemeral!
12. ✅ **ALWAYS use Services** - they provide stable DNS names and handle pod IP changes automatically

---

## Study Tips

1. **Practice DNS resolution** from different pods and namespaces
2. **Use nslookup and host** commands to verify DNS records
3. **Understand the difference** between service and pod DNS patterns
4. **Remember** that pod DNS needs FQDN while services work with short names
5. **Know the CoreDNS architecture**: pods, service, ConfigMap, Corefile
6. **Memorize the DNS hierarchy**: `name.namespace.type.cluster.local`

---

*End of Study Notes*