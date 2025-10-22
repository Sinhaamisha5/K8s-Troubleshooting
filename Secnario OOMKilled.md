# SRE Case Study: Pod Crash Loop with Subtle Resource Issues

## The Incident

### Initial Alert
It's 3 AM on a Tuesday. Your alerting system fires: "Pod restart rate elevated on payment-service cluster." You see that pods in the `payment-processor` deployment are restarting every 2-5 minutes, but the app is still serving traffic (though with degraded performance).

### Key Observation
This is interesting because:
- It's not a complete outage (Kubernetes keeps restarting the pods)
- It's intermittent, not consistent (some pods crash, others don't)
- The error pattern suggests it's not a configuration issue that would affect all pods equally
- The timing suggests it might be load-related

---

## Phase 1: Initial Investigation

### Step 1: Check Pod Status and Events
```bash
kubectl get pods -n payments -l app=payment-processor
```

**Output shows:**
```
NAME                                    READY   STATUS    RESTARTS   AGE
payment-processor-7d8f9c4b2f-abc12      0/1     CrashLoopBackOff   47       5h
payment-processor-7d8f9c4b2f-def34      1/1     Running            2        3h
payment-processor-7d8f9c4b2f-ghi56      0/1     CrashLoopBackOff   45       5h
payment-processor-7d8f9c4b2f-jkl78      1/1     Running            1        2h
```

Notice the restart counts are high and inconsistent across pods.

### Step 2: Deep Dive with kubectl describe
```bash
kubectl describe pod payment-processor-7d8f9c4b2f-abc12 -n payments
```

**Key findings in the output:**
```
Name:         payment-processor-7d8f9c4b2f-abc12
Namespace:    payments
...
Containers:
  payment-processor:
    Container ID:   containerd://a1b2c3d4...
    Image:          payment-processor:1.2.3
    Image ID:       sha256:abc123...
    Port:           8080/TCP
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
      Started:      2024-10-22T08:15:32Z
      Finished:     2024-10-22T08:16:45Z
    Ready:          False
    Restart Count:  47
    
    Limits:
      memory:  512Mi
    Requests:
      memory:  256Mi
    
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-token (ro)
```

### The First Clue: OOMKilled with Exit Code 137

This is critical. Exit code 137 = SIGKILL, which is what Kubernetes sends when it's out of memory. The pod is being killed due to OOM (Out of Memory), not a crash in the application logic.

---

## Phase 2: Understanding the Resource Picture

### Step 3: Check Node Capacity and Pressure
```bash
kubectl describe nodes | grep -A 5 "Allocated resources"
```

You discover that the nodes hosting payment-processor have:
- **Memory**: 16 GB total, 14.2 GB already allocated to other pods
- **Only 1.8 GB available** across the node

But your pod has a limit of 512Mi. That should fit, right? Not quite...

### Step 4: Check Resource Metrics
```bash
kubectl top pods -n payments -l app=payment-processor
```

**Output:**
```
NAME                                    CPU(cores)   MEMORY(Mi)
payment-processor-7d8f9c4b2f-abc12      234m         487Mi
payment-processor-7d8f9c4b2f-def34      198m         512Mi
payment-processor-7d8f9c4b2f-ghi56      267m         501Mi
```

**The Aha Moment:** The pods are using **500+ Mi of RAM**, but the limit is only **512Mi**. They're right at the edge. Any spike in memory usage will trigger an OOM kill.

---

## Phase 3: Root Cause Analysis

### Step 5: Examine Application Logs
```bash
kubectl logs -n payments payment-processor-7d8f9c4b2f-abc12 --previous
```

**Application logs show:**
```
2024-10-22 08:16:44 [INFO] Processing batch of 1000 transactions...
2024-10-22 08:16:45 [INFO] Building in-memory cache for payment methods...
2024-10-22 08:16:46 [ERROR] Memory allocation failed
2024-10-22 08:16:47 [FATAL] Application terminated due to memory pressure
```

### Step 6: Investigate Memory Growth Pattern
You enable more detailed monitoring and observe:

```bash
# SSH to the node and check memory usage over time
kubectl debug node/<node-name> -it --image=ubuntu
```

Inside the node debug pod, you check:
```bash
# Check kernel messages for OOM events
dmesg | tail -20
```

**Output shows:**
```
[1234.567] oom-kill: constraint=CONSTRAINT_NONE, nodemask=... 
[1234.567] Out of memory: Kill process 5678 (java) score 567 or sacrifice child
[1234.567] Killed process 5678 (java) total-vm:524288kB, anon-rss:512000kB...
```

This confirms the kernel is killing the process due to OOM.

### Step 7: Analyze the Memory Leak Hypothesis

You request a heap dump and analyze it:
```bash
# In production-like environment, enable JVM profiling
kubectl set env deployment/payment-processor -n payments \
  JAVA_OPTS="-Xmx450m -Xms256m -XX:+PrintGCDetails"
```

The heap dumps reveal a classic memory leak: **the payment method cache is never evicted**. Every transaction caches payment method info, but the cache has no TTL or size limit. Over 2-5 hours of production traffic, it grows until it hits the limit.

---

## Phase 4: The Fix - Multi-Layered Approach

### Step 4a: Immediate Short-Term Fix (Increase Memory)

While you investigate the leak, increase the memory allocation to buy time:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-processor
  namespace: payments
spec:
  template:
    spec:
      containers:
      - name: payment-processor
        image: payment-processor:1.2.3
        resources:
          requests:
            memory: "512Mi"    # Increased from 256Mi
            cpu: "250m"
          limits:
            memory: "1Gi"      # Increased from 512Mi
            cpu: "500m"
```

**Apply the fix:**
```bash
kubectl apply -f deployment-fix.yaml -n payments
```

Watch the rollout:
```bash
kubectl rollout status deployment/payment-processor -n payments
```

**Results:** Pods stop crashing immediately.

### Step 4b: Medium-Term Fix (Fix the Memory Leak)

You submit a code fix to the development team:

```java
// BEFORE (Problematic)
private static final Map<String, PaymentMethod> cache = new HashMap<>();

void cachePaymentMethod(String id, PaymentMethod method) {
    cache.put(id, method);  // Never removes old entries!
}

// AFTER (Fixed)
private final Map<String, PaymentMethod> cache = CacheBuilder.newBuilder()
    .maximumSize(10000)           // Limit to 10k entries
    .expireAfterWrite(2, TimeUnit.HOURS)  // Auto-evict after 2 hours
    .build(new CacheLoader<String, PaymentMethod>() {
        public PaymentMethod load(String id) {
            return fetchPaymentMethod(id);
        }
    });
```

Deploy the patched version:
```bash
kubectl set image deployment/payment-processor \
  payment-processor=payment-processor:1.2.4 \
  -n payments
```

### Step 4c: Long-Term Fix (Vertical Pod Autoscaling)

Implement Vertical Pod Autoscaler (VPA) to automatically right-size resources:

```bash
# Install VPA if not already installed
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install vpa autoscaler/vertical-pod-autoscaler \
  --namespace kube-system \
  --set serviceAccount.create=true
```

Create a VPA policy for the deployment:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: payment-processor-vpa
  namespace: payments
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-processor
  updatePolicy:
    updateMode: "Auto"  # Automatically update pod resources
  resourcePolicy:
    containerPolicies:
    - containerName: payment-processor
      minAllowed:
        memory: "256Mi"
        cpu: "100m"
      maxAllowed:
        memory: "2Gi"
        cpu: "1"
      # Prevent rapid scaling up/down
      controlledValues: ["requests"]
```

Apply it:
```bash
kubectl apply -f vpa-policy.yaml -n payments
```

Now VPA will watch actual resource usage and recommend (or automatically adjust) the resource requests.

---

## Phase 5: Monitoring and Prevention

### Implement Proactive Alerting

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pod-resource-alerts
  namespace: payments
spec:
  groups:
  - name: pod.rules
    rules:
    # Alert when memory usage is approaching limit (80%+)
    - alert: PodMemoryNearLimit
      expr: |
        (container_memory_usage_bytes / container_spec_memory_limit_bytes) > 0.8
      for: 5m
      annotations:
        summary: "Pod {{ $labels.pod }} memory approaching limit"
        
    # Alert on high restart count
    - alert: HighPodRestartRate
      expr: |
        rate(kube_pod_container_status_restarts_total[15m]) > 0.1
      for: 5m
      annotations:
        summary: "Pod {{ $labels.pod }} restarting frequently"
        
    # Alert on OOM kills
    - alert: PodOOMKilled
      expr: kube_pod_container_last_terminated_reason{reason="OOMKilled"} == 1
      annotations:
        summary: "Pod {{ $labels.pod }} was OOM killed"
```

### Create a Dashboard

A Grafana dashboard showing:
- Memory usage vs. limit (with warning at 80%)
- Memory trend over time (to spot memory leaks)
- Restart count and frequency
- Pod eviction events
- Node pressure conditions

---

## Phase 6: Post-Incident Documentation

### What We Learned
1. **Memory limits that are too tight are a silent killer** - the app appeared to be crashing randomly, but it was actually running out of memory
2. **Memory leaks show up as gradual degradation** - not an immediate, obvious failure
3. **Exit code 137 is a dead giveaway** - always check what exit codes mean
4. **Requests vs. Limits matter** - just because a pod has a limit doesn't mean it will stay within that limit
5. **Observability is crucial** - we needed heap dumps, kernel logs, and metrics to figure out what was happening

### Prevention for Next Time
- **Every application deployment requires memory profiling** before going to production
- **Run load tests to baseline memory usage** with realistic traffic patterns
- **Set alerts for high memory usage** (>70% of limit) so we catch issues early
- **Implement resource quotas at the namespace level** to prevent one app from starving others
- **Use Vertical Pod Autoscaler** to continuously optimize resource allocation
- **Regular heap dump analysis** for Java applications to detect potential leaks

### Timeline
```
03:00 - Alert fires
03:05 - Initial investigation shows CrashLoopBackOff
03:10 - Discover OOMKilled exit code
03:15 - Identify memory limit is too low
03:20 - Increase memory limits and deploy
03:25 - Pods stabilize, no more restarts
03:30 - Begin investigating root cause
04:00 - Identify memory leak in payment method cache
05:00 - Code fix developed and tested
05:30 - Deploy patched version
06:00 - VPA installed and configured
06:30 - Monitoring and alerting enhanced
```

---

## How to Tell This Story in an Interview

**Opening:**
"I dealt with a production incident where payment-processor pods were crashing intermittently, restarting every 2-5 minutes. This was interesting because it wasn't a complete outage—Kubernetes was successfully restarting them, so we still had degraded service."

**Investigation:**
"I started by checking pod status with `kubectl describe` and immediately noticed the 'OOMKilled' exit code with code 137. That told me it was a memory issue, not an application crash. I then checked the actual memory usage with `kubectl top pods` and found the pods were using 500+ MB of RAM against a 512 MB limit."

**Root Cause:**
"Looking at application logs and performing heap dumps, I discovered a memory leak—the payment method cache was growing unbounded with no eviction policy. Over 2-5 hours of production traffic, it would consume all available memory."

**Solution:**
"I took a three-pronged approach. First, immediately increased memory limits to stabilize the service. Second, worked with the development team to fix the leak by implementing a bounded cache with TTL. Third, I set up Vertical Pod Autoscaler so this type of resource misalignment would be automatically corrected in the future."

**Takeaway:**
"This taught me that not all crashes are due to code bugs—sometimes they're due to resource misconfiguration or gradual degradation. It's why observability—heap dumps, kernel logs, and metrics trending—is so critical to being an effective SRE."

---

## Interview Tips

**What This Demonstrates:**
✅ Systematic debugging approach  
✅ Understanding of Kubernetes resource management  
✅ Knowledge of OOM killing and kernel behavior  
✅ Ability to investigate both symptoms and root causes  
✅ Knowledge of memory profiling and leak detection  
✅ Thinking about long-term solutions (VPA, monitoring)  
✅ Post-incident learning and prevention mindset  

**Questions You Might Get:**
- "How would you prevent this in the first place?" → Load testing, profiling, alerts
- "What if it was a legitimate memory spike?" → Discuss horizontal scaling vs. vertical
- "How do you decide between quick fixes and permanent fixes?" → Discuss trade-offs, incident severity
- "Have you used VPA before?" → Yes, and here's what I've learned about its limitations...

**Avoid:**
- Don't say you just increased the memory limit and called it done
- Don't skip the root cause analysis
- Don't forget the prevention/learning aspect