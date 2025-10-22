# SRE Case Study: Network Connectivity Between Services

## The Incident

### Initial Alert
It's Friday afternoon at 2 PM. The customer-service team reports that checkout transactions are failing with timeout errors. Your monitoring shows that the `checkout-api` service is up and responding, but the `payment-service` it depends on seems unreachable.

### Key Symptoms
```
ERROR: payment-service connection timeout after 30s
Stack trace shows: java.net.ConnectException: Connection refused
Affected: All requests from checkout-api to payment-service
Not affected: payment-service responding to direct health checks
Status: ~15% of transactions failing, cascading to other services
```

The confusing part: both services are running, both have pods, and nothing was deployed recently. This is a network issue, not an application crash.

---

## Phase 1: Initial Triage

### Step 1: Verify Both Services Are Actually Running

```bash
# Check if payment-service deployment is running
kubectl get deployments -n production
```

**Output:**
```
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
checkout-api            3/3     3            3           45d
payment-service         3/3     3            3           120d
notification-service    2/2     2            2           30d
```

Both services look healthy. Let's dig deeper.

### Step 2: Check If Pods Are Actually Ready

```bash
kubectl get pods -n production -l app=payment-service
```

**Output:**
```
NAME                              READY   STATUS    RESTARTS   AGE
payment-service-7c8b9d2e-abc12    1/1     Running   0          2h
payment-service-7c8b9d2e-def34    1/1     Running   0          3h
payment-service-7c8b9d2e-ghi56    0/1     Running   8          25m
```

Interesting: 2 pods are ready, 1 is stuck "Running" but not "Ready". That's a clue, but let's continue with the general troubleshooting.

### Step 3: Check Service and Endpoints

This is critical—in Kubernetes, a Service is just a frontend. It needs actual Endpoints (IPs of healthy pods) to route to.

```bash
kubectl get svc -n production payment-service
```

**Output:**
```
NAME              TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
payment-service   ClusterIP   10.0.45.123   <none>        8080/TCP   120d
```

Service exists. Now check its endpoints:

```bash
kubectl get endpoints -n production payment-service
```

**Output:**
```
NAME              ENDPOINTS                               AGE
payment-service   10.244.1.15:8080,10.244.2.14:8080     120d
```

Wait—there are only 2 endpoints, but 3 pods are listed as "Running". The third pod (ghi56) is not in the endpoints list. This is because it's not passing health checks.

### Step 4: Describe the Service for More Details

```bash
kubectl describe svc payment-service -n production
```

**Output:**
```
Name:              payment-service
Namespace:         production
Labels:            app=payment-service
Annotations:       <none>
Selector:          app=payment-service
Type:              ClusterIP
IP:                10.0.45.123
Port:              http 8080/TCP
TargetPort:        8080/TCP
Endpoints:         10.244.1.15:8080,10.244.2.14:8080
Session Affinity:  None
Events:            <none>
```

The selector is `app=payment-service`, which should match all 3 pods. Let's check why the third isn't included.

---

## Phase 2: Deep Dive - Why Can't checkout-api Reach payment-service?

### Step 5: Test from Inside a Pod

Let's actually try making a connection from checkout-api to payment-service:

```bash
# Get a checkout-api pod
kubectl get pods -n production -l app=checkout-api
```

**Output:**
```
NAME                              READY   STATUS    RESTARTS   AGE
checkout-api-5d4e3c2f-abc12       1/1     Running   0          1h
checkout-api-5d4e3c2f-def34       1/1     Running   0          2h
checkout-api-5d4e3c2f-ghi56       1/1     Running   2          30m
```

Now exec into one and test:

```bash
kubectl exec -it checkout-api-5d4e3c2f-abc12 -n production -- /bin/bash
```

Inside the pod, test DNS resolution first:

```bash
# Resolve the service name
nslookup payment-service.production.svc.cluster.local
```

**Output:**
```
Server:         10.0.0.10
Address:        10.0.0.10#53

Name:   payment-service.production.svc.cluster.local
Address: 10.0.45.123
```

DNS is working fine. Now test connectivity to the service IP:

```bash
# Try to connect
curl -v http://payment-service.production.svc.cluster.local:8080/health
```

**Output:**
```
*   Trying 10.0.45.123:8080...
* TCP_CONNECT_TIMEOUT
* Failed to connect to payment-service.production.svc.cluster.local port 8080: Connection timed out
curl: (7) Failed to connect to port 8080: Connection timed out
```

So DNS works, but the connection times out. This narrows it down to:
1. Network policy blocking
2. iptables rules on the node
3. Issue on the payment-service pods themselves
4. Network plugin issue

### Step 6: Check Network Policies

```bash
kubectl get networkpolicies -n production
```

**Output:**
```
NAME                   POD-SELECTOR           AGE
allow-internal         app=payment-service    2d
checkout-api-to-pay    app=checkout-api       15m
payment-svc-ingress    app=payment-service    2d
```

There's a `checkout-api-to-pay` policy that was created 15 minutes ago. That's around when the issues started. Let's examine it:

```bash
kubectl describe networkpolicy checkout-api-to-pay -n production
```

**Output:**
```
Name:         checkout-api-to-pay
Namespace:    production
Pod Selector: app=checkout-api
Events:       <none>

Policy Types:
  Ingress
  Egress

Ingress Rules:
  <none>

Egress Rules:
  To Port: 8080/TCP
  To:
    Pod Selector: app=payment-service

Allow traffic on port 8080 to pods with app=payment-service
```

The policy looks correct—it allows checkout-api pods to egress on port 8080 to payment-service pods. Let's check the payment-service ingress policy:

```bash
kubectl describe networkpolicy payment-svc-ingress -n production
```

**Output:**
```
Name:         payment-svc-ingress
Namespace:    production
Pod Selector: app=payment-service

Policy Types:
  Ingress

Ingress Rules:
  From:
    Pod Selector: app=notification-service
  To:
    Ports:
    - Protocol: TCP
      Port: 8080

  From:
    Pod Selector: app=checkout-api
  To:
    Ports:
    - Protocol: TCP
      Port: 8080
```

**AHA! Here's the problem:** The ingress policy only allows traffic FROM `notification-service` and `checkout-api`. But let me check the label on checkout-api pods...

```bash
kubectl get pods -n production checkout-api-5d4e3c2f-abc12 --show-labels
```

**Output:**
```
NAME                        READY   STATUS    RESTARTS   AGE   LABELS
checkout-api-5d4e3c2f-abc12 1/1     Running   0          1h    app=checkout-api,version=v2,service=checkout
```

The label is `app=checkout-api`. That should match... unless...

Let's check what labels are on the checkout-api pod when traffic is actually being sent. You realize you need to check: **are the labels on the pods that the network policy is matching the same pods that are actually trying to make the connection?**

### Step 7: Trace the Actual Network Traffic

Time to bring out the heavy artillery. If you have Cilium installed (Cilium CNI):

```bash
# Check if Cilium is installed
kubectl get pods -n kube-system -l k8s-app=cilium
```

If Cilium is installed, use Hubble to trace traffic:

```bash
# Install Hubble CLI if not already installed
cilium hubble install

# Enable Hubble UI
cilium hubble ui

# Or use command line
hubble observe --pod=production/checkout-api-5d4e3c2f-abc12 \
               --to-pod=production/payment-service-7c8b9d2e-abc12
```

**Hubble output might show:**
```
TIMESTAMP                     SOURCE                                    DESTINATION                              TYPE            STATE
Oct 22 14:32:12.123456 UTC   production/checkout-api-5d4e3c2f-abc12   production/payment-service-7c8b9d2e-abc12  DROPPED         POLICY_DENIED
Oct 22 14:32:12.234567 UTC   production/checkout-api-5d4e3c2f-abc12   production/payment-service-7c8b9d2e-def34  DROPPED         POLICY_DENIED
```

It's showing DROPPED due to policy. But that contradicts what we saw in the network policy... unless there's another policy being applied somewhere.

### Step 8: Check for Default Deny Policies

```bash
kubectl get networkpolicies -n production -o yaml
```

Scan through and look for a default-deny policy:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

**There it is!** A `default-deny-all` policy that denies all traffic unless explicitly allowed. This is a security best practice, but it means every connection must be explicitly allowed.

The issue: The `payment-svc-ingress` policy was created 2 days ago, but the `checkout-api-to-pay` policy was created 15 minutes ago. There was a window where connections worked, then someone created a new policy that conflicted.

### Step 9: Check Node-Level iptables (Alternative Investigation)

If you didn't have Cilium, you'd check iptables on the node:

```bash
# Find which node the payment-service pod is on
kubectl get pods -n production payment-service-7c8b9d2e-abc12 -o wide
```

**Output:**
```
NAME                              READY   STATUS    RESTARTS   AGE   IP            NODE
payment-service-7c8b9d2e-abc12    1/1     Running   0          2h    10.244.1.15   node-1
```

Then SSH to that node:

```bash
ssh node-1

# Check iptables rules for the service IP
sudo iptables -L -n -v | grep 10.0.45.123

# Or use iptables-save to see all rules
sudo iptables-save | grep 10.0.45.123
```

You'd look for REJECT or DROP rules blocking the traffic.

### Step 10: Check CoreDNS (DNS Investigation)

Let's also verify DNS is working properly on the CoreDNS side:

```bash
# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns
```

**CoreDNS logs might show:**
```
[INFO] 10.244.1.1:54332 - 54332 "A IN payment-service.production.svc.cluster.local. udp 57 false 512" NOERROR qr,aa,rd 90 0.001234s
```

This shows the query was resolved successfully. No DNS issues.

---

## Phase 3: Investigating Why One Pod Isn't in Endpoints

### Step 11: Check the "Not Ready" Pod

We noticed earlier that one pod wasn't in the endpoints list:

```bash
kubectl get pods -n production -l app=payment-service -o wide
```

**Output:**
```
NAME                              READY   STATUS    RESTARTS   AGE   IP            NODE
payment-service-7c8b9d2e-abc12    1/1     Running   0          2h    10.244.1.15   node-1
payment-service-7c8b9d2e-def34    1/1     Running   0          3h    10.244.2.14   node-2
payment-service-7c8b9d2e-ghi56    0/1     Running   8          25m   10.244.3.20   node-3
```

The ghi56 pod shows "0/1", meaning it's not ready. Let's check the readiness probe:

```bash
kubectl describe pod payment-service-7c8b9d2e-ghi56 -n production
```

**Output:**
```
Name:         payment-service-7c8b9d2e-ghi56
Namespace:    production
...
Containers:
  payment-service:
    Container ID:   containerd://xyz789...
    Image:          payment-service:2.1.0
    Ready:          False
    Restart Count:  8
    State:
      Running (started 12 minutes ago)
    Last State:
      Terminated (reason:CrashLoopBackOff, exit code:1)
    Readiness probe:     http-get http://:8080/health delay=5s timeout=3s period=10s #success=1 #failure=3
    Ready Condition Status: False

Events:
  Type     Reason     Age                  From             Message
  ----     ------     ----                 ------           ----
  Warning  Unhealthy  5m3s (x3 over 5m43s) kubelet          Readiness probe failed (exit code: 1)
```

The readiness probe is failing. Let's check what's wrong:

```bash
kubectl logs -n production payment-service-7c8b9d2e-ghi56
```

**Output:**
```
2024-10-22 14:05:23 [INFO] Starting payment-service version 2.1.0
2024-10-22 14:05:24 [INFO] Connecting to database...
2024-10-22 14:05:25 [ERROR] Failed to connect to payment-db: timeout
2024-10-22 14:05:26 [ERROR] Health check failed: database unavailable
2024-10-22 14:05:27 [FATAL] Pod will continue running but is not ready
```

This pod is failing its readiness probe because it can't reach the payment database. It's not a network connectivity issue with checkout-api → payment-service; it's a separate issue with payment-service → payment-db.

---

## Phase 4: Root Cause Analysis

You now have multiple issues:

### Issue 1: Network Policy Conflict (Primary)
The `default-deny-all` policy is blocking traffic that isn't explicitly allowed in the positive rules.

### Issue 2: Missing Database Connectivity (Secondary)
One payment-service pod can't reach the database, causing it to be unhealthy.

### Issue 3: Label/Selector Mismatch? (Investigated & Cleared)
After checking, labels are correct and match the policies.

---

## Phase 5: The Fixes

### Fix 1: Audit and Correct Network Policies

First, let's understand what policies currently exist and their interactions:

```bash
kubectl get networkpolicies -n production -o yaml > netpol-audit.yaml
```

Examine the policies and create the proper rule set. The issue was that `checkout-api-to-pay` was created after `default-deny-all`, creating a deny-by-default situation for ingress on payment-service.

**The corrected payment-service ingress policy:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-service-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
  - Ingress
  - Egress
  
  # Allow ingress from checkout-api
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: checkout-api
      namespaceSelector:
        matchLabels:
          name: production
    ports:
    - protocol: TCP
      port: 8080
  
  # Allow ingress from notification-service
  - from:
    - podSelector:
        matchLabels:
          app: notification-service
      namespaceSelector:
        matchLabels:
          name: production
    ports:
    - protocol: TCP
      port: 8080
  
  # Allow DNS queries (critical!)
  - from:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  
  # Allow egress to payment-db
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: payment-db
      namespaceSelector:
        matchLabels:
          name: production
    ports:
    - protocol: TCP
      port: 5432
  
  # Allow DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

Apply it:

```bash
kubectl apply -f payment-service-netpol.yaml -n production
```

**Key improvements:**
- Explicitly allows ingress from checkout-api and notification-service
- Explicitly allows egress to payment-db
- **Allows DNS queries** (this is often forgotten!)
- Uses namespace selectors to be explicit about cross-namespace policies

### Fix 2: Database Connectivity for Payment-Service

For the payment-service pod that's failing readiness checks:

```bash
# Check if payment-db is accessible from the node hosting ghi56
kubectl exec -it payment-service-7c8b9d2e-ghi56 -n production -- /bin/bash

# Inside the pod, test connectivity to the database
nc -zv payment-db.production.svc.cluster.local 5432
```

If it times out, check for a network policy blocking egress to the database:

```bash
# Check all network policies affecting payment-db
kubectl get networkpolicies -n production -o yaml | grep -A 10 "payment-db"
```

Update the policy to allow payment-service to reach payment-db. Once the policy is fixed:

```bash
# Delete and recreate the payment-service pod to clear the unhealthy state
kubectl delete pod payment-service-7c8b9d2e-ghi56 -n production
```

Kubernetes will create a new pod, which should now have access to the database.

### Fix 3: Implement Cilium Network Policy Debugging

For ongoing troubleshooting, install Cilium if not already present:

```bash
# Install Cilium
helm repo add cilium https://helm.cilium.io
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true
```

Then you can trace traffic in real-time:

```bash
# Forward Hubble UI to localhost
kubectl port-forward -n kube-system svc/hubble-ui 12000:80

# Or use CLI to check denied traffic
hubble observe --verdict=DROPPED --pod=production/checkout-api-5d4e3c2f-abc12
```

---

## Phase 6: Prevention and Monitoring

### Add Network Policy Validation

Create a pre-deployment check:

```bash
#!/bin/bash
# validate-netpol.sh - Check for common network policy issues

echo "Checking for default-deny policies..."
kubectl get networkpolicies --all-namespaces -o json | \
  jq '.items[] | select(.spec.policyTypes[] == "Ingress" and (.spec.ingress | length) == 0)'

echo "Checking for pods not covered by any policy..."
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name)"' | \
  while read pod; do
    ns=$(echo $pod | cut -d/ -f1)
    name=$(echo $pod | cut -d/ -f2)
    labels=$(kubectl get pod $name -n $ns -o json | jq -r '.metadata.labels | to_entries | map("\(.key)=\(.value)") | join(",")')
    
    # Check if any network policy in the namespace matches
    kubectl get networkpolicies -n $ns -o json | \
      jq -e ".items[] | select(.spec.podSelector.matchLabels | to_entries | map(\"\(.key)=\(.value)\") | join(\",\") | . == \"$labels\")" > /dev/null
    
    if [ $? -ne 0 ]; then
      echo "WARNING: Pod $pod ($labels) not matched by any policy!"
    fi
  done
```

### Create Alert for Dropped Connections

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: network-connectivity-alerts
  namespace: production
spec:
  groups:
  - name: network.rules
    rules:
    
    # Alert on high rate of dropped packets
    - alert: HighNetworkPolicyDropRate
      expr: |
        rate(cilium_dropped_packets_total[5m]) > 100
      for: 5m
      annotations:
        summary: "High rate of dropped packets, possible network policy issue"
        
    # Alert when service has no endpoints
    - alert: ServiceNoHealthyEndpoints
      expr: |
        count(kube_endpoint_address_available{namespace="production"}) by (endpoint) == 0
      for: 2m
      annotations:
        summary: "Service {{ $labels.endpoint }} has no healthy endpoints"
        
    # Alert on pod readiness probe failures
    - alert: PodReadinessProbeFailing
      expr: |
        rate(kube_pod_container_status_ready{namespace="production",condition="false"}[5m]) > 0
      for: 5m
      annotations:
        summary: "Pod readiness probe failing for {{ $labels.pod }}"
```

### Create Network Policy Documentation

Document in your runbook:

```markdown
# Network Policy Troubleshooting Guide

## Common Issues

### 1. Service Unreachable Despite Pods Running
**Check:**
- DNS: `nslookup service-name.namespace.svc.cluster.local`
- Endpoints: `kubectl get endpoints service-name`
- Network Policies: `kubectl get networkpolicies -n namespace`
- Pod Logs: `kubectl logs pod-name`

### 2. Connection Timed Out
**Likely Causes:**
- Network policy blocking ingress
- Pod failing readiness probe (not in endpoints)
- CNI plugin issue
- Firewall rule on node

**Debug:**
- Check: `kubectl describe svc service-name` for endpoints
- Check: `kubect describe pod` for readiness probe status
- Trace: `hubble observe --pod=source --to-pod=dest`

### 3. DNS Resolution Fails
**Likely Causes:**
- CoreDNS pod down
- Network policy blocking DNS traffic to port 53
- Kubernetes DNS service misconfigured

**Debug:**
- Check CoreDNS: `kubectl get pods -n kube-system -l k8s-app=kube-dns`
- Check logs: `kubectl logs -n kube-system -l k8s-app=kube-dns`
- Test DNS: `nslookup` from inside pod
```

---

## Phase 7: Timeline and Resolution

```
14:05 - Initial alert: checkout transactions timing out
14:10 - Verified both services are running
14:15 - Discovered payment-service endpoint missing one pod
14:20 - Network policy investigation began
14:25 - Found default-deny-all policy + egress rule conflict
14:30 - Checked CoreDNS (working fine)
14:35 - Used Hubble to trace DROPPED packets from network policy
14:40 - Identified missing ingress rule for checkout-api
14:45 - Applied corrected network policy
14:50 - Tested connectivity: working!
14:55 - Investigated failing readiness probe on ghi56 pod
15:00 - Found ghi56 can't reach payment-db (same network policy issue)
15:05 - Applied egress rule for payment-db
15:10 - Deleted unhealthy pod, new pod became healthy
15:15 - All services showing 100% connectivity
15:30 - Implemented Hubble monitoring and alerts
16:00 - Updated runbook with findings
```

---

## How to Tell This Story in an Interview

**Opening:**
"We had a production incident where checkout-api couldn't reach payment-service, even though both services were running and their endpoints looked correct. This was an interesting network troubleshooting scenario because multiple issues were layered on top of each other."

**Investigation Process:**
"I started systematically by checking if the services were actually running and had endpoints. Then I tested connectivity from inside a pod using `kubectl exec` and `curl` to see if DNS worked—it did—but the actual TCP connection was timing out. That told me it wasn't DNS; it was something blocking the traffic after the resolution."

**Network Policy Deep Dive:**
"I checked network policies and found a `default-deny-all` policy that required explicit allow rules. The payment-service ingress policy existed, but it was missing the rules to allow traffic from checkout-api. I used Cilium Hubble to trace the actual packets and saw them being DROPPED by the network policy, which confirmed the issue."

**Secondary Issue:**
"While fixing the first issue, I noticed one payment-service pod wasn't in the endpoints list because its readiness probe was failing. It was failing because that pod couldn't reach the payment database—another network policy issue. I fixed both by updating the network policies to explicitly allow the necessary traffic and DNS queries."

**Tools & Techniques:**
"The key tools were `kubectl describe` for policy inspection, `kubectl exec` + `curl` for testing, Cilium Hubble for packet-level tracing, and CoreDNS logs to rule out DNS issues. This approach of checking each layer—service definition, endpoints, policies, DNS, actual packets—is systematic and catches network issues quickly."

**Preventive Measures:**
"After fixing it, I set up Prometheus alerts for services with no healthy endpoints, implemented Hubble monitoring for dropped traffic, and created a validation script that runs on every deployment to catch missing network policies early. I also documented the common network policy issues in the runbook."

---

## Interview Tips

**What This Demonstrates:**
✅ Systematic debugging methodology (top to bottom layer analysis)  
✅ Deep understanding of Kubernetes networking  
✅ Knowledge of network policies and their interaction  
✅ Ability to use advanced tools (Cilium Hubble, tcpdump, iptables)  
✅ Understanding of DNS, service discovery, readiness probes  
✅ Practical debugging techniques (exec, curl, packet tracing)  
✅ Thinking about prevention and monitoring  

**Follow-Up Questions You Might Get:**

**Q: "What if Cilium wasn't installed?"**
A: "I'd use tcpdump on the node hosting the pods to capture packets and see where they were being dropped. Or I'd check iptables rules directly since kube-proxy manages the service routing through iptables."

**Q: "How do you handle policies in multi-namespace environments?"**
A: "You use namespace selectors in addition to pod selectors. The key is being explicit—specify both the namespace and the pod labels you want to allow traffic from/to. It's verbose but much clearer than implicit rules."

**Q: "What about policies for egress traffic?"**
A: "Egress policies are often forgotten but just as important. You need to allow pods to make outbound connections to services they depend on, plus allow DNS queries to port 53. I always audit egress rules as part of network policy troubleshooting."

**Q: "How do you test network policies safely?"**
A: "I use a separate test namespace with similar workloads, apply the policies there first, and verify behavior before applying to production. Or I use Cilium's network policy editor which shows the impact before applying. Always have a rollback plan ready."

**Q: "Have you seen network policies cause performance issues?"**
A: "Yes, overly complex policies or recursive label matching can cause performance degradation. I recommend keeping policies simple, using consistent naming conventions, and monitoring the kube-proxy performance metrics."

---

## Common Pitfalls to Avoid in Your Story

❌ **Don't say:** "We just deleted all the network policies and it worked"  
✅ **Do say:** "We carefully reviewed each policy to understand their intent before fixing the allow rules"

❌ **Don't skip:** The DNS investigation step  
✅ **Do include:** How you verified DNS worked before investigating lower layers

❌ **Don't forget:** That readiness probes affect endpoints  
✅ **Do mention:** How unhealthy pods are automatically removed from endpoints

❌ **Don't just fix the symptom:** By allowing all traffic  
✅ **Do implement:** Proper, minimal-privilege policies that allow only what's needed

❌ **Don't end with:** "And then we fixed it"  
✅ **Do end with:** "And I implemented monitoring and validation to prevent it happening again"









# Simple Network Connectivity Case Study

## The Incident

### What Happened
You're on-call and get an alert: "API Gateway can't reach backend service." Developers say their requests are timing out. You have 5 minutes to figure it out.

---

## Investigation (The Simple Way)

### Step 1: Check If Services Exist

```bash
kubectl get svc -n production
```

Output:
```
NAME                TYPE        CLUSTER-IP      PORT(S)
api-gateway         ClusterIP   10.0.10.50      8080/TCP
backend-service     ClusterIP   10.0.10.100     3000/TCP
```

Both services exist. Good.

### Step 2: Check If Pods Are Running

```bash
kubectl get pods -n production -l app=backend-service
```

Output:
```
NAME                          READY   STATUS    RESTARTS
backend-service-abc123        1/1     Running   0
backend-service-def456        1/1     Running   0
backend-service-ghi789        1/1     Running   0
```

All pods are running and ready (1/1 means ready). Good.

### Step 3: Check Service Endpoints (The Critical Check)

```bash
kubectl get endpoints backend-service -n production
```

Output:
```
NAME              ENDPOINTS
backend-service   10.244.1.5:3000,10.244.1.6:3000,10.244.1.7:3000
```

Endpoints exist and match the running pods. This means the service has healthy pods to route traffic to.

### Step 4: Test From Inside a Pod

Now let's actually try to connect:

```bash
# Get an api-gateway pod
kubectl get pods -n production -l app=api-gateway
```

Output:
```
NAME                      READY   STATUS
api-gateway-pod-001       1/1     Running
```

Now exec into it and test:

```bash
kubectl exec -it api-gateway-pod-001 -n production -- sh
```

Inside the pod, try to reach the backend service:

```bash
# Test DNS first
nslookup backend-service

# Should show something like:
# Name:   backend-service
# Address: 10.0.10.100
```

DNS works. Now try to actually connect:

```bash
# Use curl or wget
curl http://backend-service:3000/health

# Or if curl isn't available, use telnet/nc
nc -zv backend-service 3000
```

**If you get a timeout here**, you've found the issue. The problem is between the pod and the backend service.

---

## Most Common Causes (And Quick Fixes)

### Cause 1: Network Policy Blocking Traffic

**How to check:**
```bash
kubectl get networkpolicies -n production
```

If there are policies listed, one of them might be blocking the traffic.

**Quick fix:**
Check if there's a rule explicitly allowing traffic from api-gateway to backend-service. If not, add it:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend-service
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api-gateway
    ports:
    - protocol: TCP
      port: 3000
```

Apply it:
```bash
kubectl apply -f network-policy.yaml
```

Test again:
```bash
# From inside api-gateway pod
curl http://backend-service:3000/health
```

Should work now.

---

### Cause 2: Service Port Mismatch

**How to check:**
```bash
kubectl describe svc backend-service -n production
```

Output:
```
Port:              http  3000/TCP
TargetPort:        8080/TCP
```

**The Issue:** The service listens on port 3000, but pods inside are listening on port 8080. The traffic gets routed to 3000, but nothing is listening there.

**Quick fix:**
Either fix the service definition:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend-service
  ports:
  - port: 3000
    targetPort: 8080      # Make sure this matches what the pod is listening on
```

Or check what port the pod is actually listening on:

```bash
# From inside the pod
netstat -tlnp | grep LISTEN

# Should show something like:
# Proto Recv-Q Send-Q Local Address    State
# tcp        0      0 0.0.0.0:8080     LISTEN
```

---

### Cause 3: Pod Not Healthy (Readiness Probe Failing)

```bash
kubectl get pods -n production -l app=backend-service
```

If you see something like:
```
NAME                          READY   STATUS
backend-service-abc123        0/1     Running
```

The 0/1 means the pod is running but not ready. Check why:

```bash
kubectl describe pod backend-service-abc123 -n production
```

Look for "Readiness probe" section:
```
Readiness probe:     http-get http://:8080/health
Readiness:           False
```

Check the logs:
```bash
kubectl logs backend-service-abc123 -n production
```

Logs might show:
```
ERROR: Can't connect to database
Health check failed
```

**Fix:** Fix the underlying issue (database connectivity, etc.), and the pod will become healthy and rejoin the service.

---

### Cause 4: Wrong Service Name

```bash
# Inside api-gateway pod
curl http://backend-service:3000/health
```

If you get "Name or service not known" error, the DNS lookup failed. The service name might be wrong.

```bash
# Check what the correct name is
kubectl get svc -n production
```

Common mistakes:
- Using wrong namespace (if backend-service is in a different namespace, use: `backend-service.other-namespace.svc.cluster.local`)
- Typo in service name
- Service doesn't exist

---

## Quick Checklist (Do This First)

When someone says "I can't reach service X", do this in order:

```bash
# 1. Service exists?
kubectl get svc -n production | grep service-name

# 2. Pods running?
kubectl get pods -n production -l app=service-name

# 3. Pods healthy? (Look for READY column to be 1/1)
kubectl get pods -n production -l app=service-name

# 4. Pods have endpoints?
kubectl get endpoints service-name -n production

# 5. Test from source pod
kubectl exec -it <source-pod> -n production -- sh
# Inside the pod:
curl http://service-name:port/health

# 6. Check network policies
kubectl get networkpolicies -n production
```

If all those checks pass but it still doesn't work, then you dig deeper with Hubble/tcpdump.

---

## Real Example: Step By Step

### Scenario
You get an alert: "Web app can't reach cache-service"

### What you do:

**30 seconds to identify:**
```bash
# Quick checks
kubectl get svc cache-service -n production
kubectl get pods -n production -l app=cache-service
kubectl get endpoints cache-service -n production
```

All look good? Then...

**1 minute to reproduce:**
```bash
# Get a web-app pod
kubectl exec -it web-app-pod-001 -n production -- sh

# Inside the pod, try to connect
curl http://cache-service:6379/ping
# Times out
```

**2 minutes to find the cause:**
```bash
# Check network policies
kubectl get networkpolicies -n production
# See if there's a rule allowing web-app → cache-service

# If no policy or policy is too restrictive, that's the issue
```

**3 minutes to fix:**
```bash
# Add or update network policy
kubectl apply -f allow-web-to-cache.yaml

# Test again
curl http://cache-service:6379/ping
# Works!
```

**Done.** Total time: 3 minutes.

---

## Things to Know

### What DNS Names Work Inside a Pod

From inside a pod in the `production` namespace:
- `cache-service` → resolves to 10.0.10.100 (if both in same namespace)
- `cache-service.production` → also works
- `cache-service.production.svc.cluster.local` → full name, always works

Use the full name if you're crossing namespaces or unsure.

### Why Endpoints Matter

A Service without Endpoints can't route traffic anywhere. If you see:

```bash
kubectl get endpoints cache-service
# Returns empty or shows: <none>
```

This means either:
1. No pods match the service selector
2. All pods are unhealthy (readiness probes failing)
3. All pods are on nodes that have a taint the pods don't tolerate

---

## 60-Second Story for Interview

**Setup:**
"We had an alert that the web app couldn't reach our cache service. Both were running, but requests were timing out."

**Investigation:**
"I did a quick checklist: service exists, pods are running, pods are marked ready, service has endpoints. All good. Then I exec'd into a web app pod and tried to actually curl the cache service, and it timed out. That told me the connection itself was failing."

**Root Cause:**
"I checked network policies and found that while we had a default policy, we hadn't explicitly allowed traffic from web-app pods to the cache-service. The fix was adding a simple NetworkPolicy that allows that specific connection."

**Fix:**
"I added a NetworkPolicy allowing the traffic, applied it, and re-tested. The curl worked immediately. The app was back online."

**Key Point:**
"The important debugging step was using `kubectl exec` and actually testing the connection from inside the pod, rather than trying to guess from the outside. That immediately showed me it was a network/connectivity issue, not an application issue."

---

## When to Move to Advanced Tools

**Stick with simple checks if:**
- Network policies are clearly the issue
- A pod is just unhealthy (readiness probe failing)
- Service configuration is wrong
- DNS resolution failed

**Move to Hubble/tcpdump if:**
- All the simple checks pass but it still doesn't work
- You need packet-level visibility
- You suspect node-level networking issues
- iptables rules might be interfering

---

## What Not to Say in Interview

❌ "I deleted the network policies and it worked"  
✅ "I identified the network policy was too restrictive and added the necessary allow rule"

❌ "I'm not sure what the issue was"  
✅ "I followed a systematic approach: checked service exists, checked endpoints, tested connectivity from a pod"

❌ "DNS must be broken"  
✅ "I tested DNS resolution first to rule it out"

---

## Common Mistakes People Make

**Mistake 1:** Forgetting to check endpoints
- Just because a service exists doesn't mean it has healthy pods to route to
- Always check: `kubectl get endpoints`

**Mistake 2:** Not actually testing from the source pod
- Check DNS, but also test actual TCP connection
- Use `curl` or `nc` to verify the path works

**Mistake 3:** Assuming it's DNS
- DNS is usually working
- The real issue is usually network policies or unhealthy pods
- Always test DNS last, after confirming service/endpoints exist

**Mistake 4:** Testing from outside the cluster
- Services only work inside the cluster by default
- Always test from within a pod using `kubectl exec`
- External access needs Ingress or LoadBalancer

**Mistake 5:** Changing too many things at once
- Apply one fix at a time
- Test after each fix
- This way you know what actually solved it