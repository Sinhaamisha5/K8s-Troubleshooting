# Network Policy Troubleshooting: AND vs OR Logic and Misconfiguration

## The Scenario: Network Policy Misconfiguration

### Initial Requirements

You have two identical namespaces (QA and Staging) with the same applications that should have specific network connectivity rules:

```
Allowed Connections:
✅ frontend → api
✅ api → database

Blocked Connections:
❌ frontend → database
❌ api → frontend
❌ Cross-namespace traffic
❌ External connections should be configurable
```

### What's Actually Happening (QA Namespace)

```
❌ frontend CAN connect to database (should be blocked!)
❌ api CAN connect to frontend (should be blocked!)
✅ frontend → api (correct)
✅ api → database (correct)
```

### What's Actually Happening (Staging Namespace)

```
❌ Nothing works! (all connections blocked)
❌ frontend CANNOT connect to api (should work!)
❌ api CANNOT connect to database (should work!)
❌ External traffic blocked (default deny egress applied)
❌ Cross-namespace traffic allowed (should be blocked!)
```

---

## The Critical Concept: AND vs OR in Network Policies

This is the KEY to understanding network policy misconfigurations.

### The Two Syntax Patterns

#### Pattern 1: AND (Conjunction) - Single Rule with Multiple Selectors

```yaml
# All conditions must be met for traffic to be allowed
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: qa
    podSelector:
      matchLabels:
        app: api
```

**Logic:** `(namespace=qa) AND (pod=api)`

Traffic allowed ONLY from:
- Pods labeled `app: api`
- Inside namespace labeled `environment: qa`
- Both conditions MUST be true

#### Pattern 2: OR (Disjunction) - List of Rules

```yaml
# Any ONE condition matching allows traffic (additive)
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: qa
  - podSelector:
      matchLabels:
        app: api
```

**Logic:** `(namespace=qa) OR (pod=api)`

Traffic allowed from:
- ANY pod labeled `app: api` (in ANY namespace!)
- ANY pod (in ANY namespace) from a namespace labeled `environment: qa`
- Only ONE condition needs to be true

### Visual Difference

```
AND Pattern (single rule):
┌─────────────────────────────┐
│ Namespace: qa               │ ← BOTH required
│ AND                         │
│ Pod: api                    │
└─────────────────────────────┘
Traffic from: (qa namespace) AND (api pod)
Example: api pod in qa ns ✓, api pod in staging ✗, other pod in qa ns ✗

OR Pattern (list of rules):
┌─────────────────────────────┐
│ Namespace: qa               │ ← First rule
├─────────────────────────────┤
│ OR                          │
├─────────────────────────────┤
│ Pod: api                    │ ← Second rule
└─────────────────────────────┘
Traffic from: (qa namespace) OR (api pod)
Example: api pod in qa ns ✓, api pod in staging ✓, other pod in qa ns ✓
```

---

## Real Example 1: QA Namespace - The OR Problem

### The Problem

Frontend can connect to database (should be blocked).

### Investigation

**Test 1: Verify frontend → api works**
```bash
kubectl exec -it frontend-pod -n qa-test -- /bin/sh

# Inside pod:
curl api-service:8080
# Output: 200 OK ✓ Correct!
```

**Test 2: Verify frontend → database blocked**
```bash
# Inside frontend pod:
telnet dbservice 3306
# Output: Connection successful (should fail!)
# ✗ WRONG! Frontend shouldn't reach database!
```

### Check Network Policy

```bash
kubectl get networkpolicies -n qa-test
```

Output:
```
NAME    POD-SELECTOR   AGE
dbnp    app=db         5m
```

**Describe the database network policy:**

```bash
kubectl describe networkpolicy dbnp -n qa-test
```

Output:
```
Name:         dbnp
Namespace:    qa-test
Pod selector: app=db

Ingress allow from:
  - Namespace selector: environment=qa
  - Pod selector: app=api
```

**The Issue Found!**

The YAML shows two conditions in a LIST:
```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: qa
  - podSelector:
      matchLabels:
        app: api
```

This is OR logic! Traffic allowed from:
- Pods in namespace labeled `environment: qa` (ANY pod!)
- Any pod labeled `app: api`

**Frontend pod is in the qa-test namespace which has `environment: qa` label.**
**Result:** Frontend matches the first rule → allowed!

### The Fix: Convert OR to AND

Remove the list, combine into single rule:

```yaml
# BEFORE (OR - WRONG):
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: qa
  - podSelector:
      matchLabels:
        app: api

# AFTER (AND - CORRECT):
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: qa
    podSelector:
      matchLabels:
        app: api
```

Apply the fix:
```bash
kubectl edit networkpolicy dbnp -n qa-test
```

Remove the list separator, combine conditions.

### Verify the Fix

```bash
# Inside frontend pod:
telnet dbservice 3306
# Now times out (connection refused) ✓ Correct!
```

---

## Real Example 2: QA Namespace - The Missing Default Deny

### The Problem

API can connect to frontend (should be blocked).

### Investigation

**Check what network policies exist:**

```bash
kubectl get networkpolicies -n qa-test
```

Output:
```
NAME           POD-SELECTOR
dbnp           app=db
api-np         app=api
```

**Examine api-np:**

```bash
kubectl describe networkpolicy api-np -n qa-test
```

Output:
```
Pod selector: app=api

Ingress allow from:
  - Pod selector: app=frontend
```

This says "Only frontend pods can connect to api pods" ✓ Correct!

**But there's no policy for the frontend pod!**

### The Issue

Network policy default behavior:

```
If NO network policy matches a pod:
  → All traffic is ALLOWED (default allow)

If a network policy matches a pod:
  → Only traffic explicitly allowed is permitted
```

Since there's no network policy selecting the frontend pod, **all traffic is allowed to it by default**!

**Result:** API can connect to frontend (and anything else!)

### The Solution: Default Deny Policy

Create a default deny ingress policy:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: qa-test
spec:
  podSelector: {}  # ← Empty selector = ALL PODS
  policyTypes:
  - Ingress
  # No ingress rules = deny all ingress
```

Apply it:
```bash
kubectl apply -f default-deny-ingress.yaml -n qa-test
```

### How It Works

```
Network policies are additive:
- More specific policies override generic ones
- default-deny-ingress is generic (matches all pods)
- Specific policies (api-np, dbnp) override it

Result:
- database pod: has specific dbnp rule → uses that (allows from api)
- api pod: has specific api-np rule → uses that (allows from frontend)
- frontend pod: NO specific rule → uses default-deny-ingress (denies all)
```

### Verify the Fix

```bash
# Inside api pod:
curl frontend-service:3000
# Now times out ✓ Correct!
```

---

## Real Example 3: Staging Namespace - The Default Deny Egress Problem

### The Problem

Nothing works! All connections fail. Also can't even reach google.com.

### Investigation

**Test internal connectivity:**
```bash
kubectl exec -it frontend-pod -n staging -- /bin/sh

# Inside pod:
curl api-service:8080
# Timeout! Should work!

telnet dbservice 3306
# Timeout! Should work!
```

**Test external connectivity:**
```bash
# Inside pod:
ping google.com
# Timeout! (DNS resolution fails)
```

**This suggests egress is blocked entirely.**

### Check Network Policies

```bash
kubectl get networkpolicies -n staging
```

Output:
```
NAME               POD-SELECTOR
test-policy        <empty>
dbnp               app=db
api-np             app=api
```

**Examine the suspicious "test-policy":**

```bash
kubectl get networkpolicy test-policy -n staging -o yaml
```

Output:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-policy
spec:
  podSelector: {}  # ← Empty = matches ALL pods
  policyTypes:
  - Ingress
  - Egress    # ← THIS IS THE PROBLEM!
  # No egress rules = deny ALL egress
```

**The Issue Found!**

`test-policy` applies to ALL pods with:
- Ingress policy (OK)
- **Egress deny all** (WRONG!)

There are specific ingress rules (dbnp, api-np) that can override the egress default deny, but there are **NO specific egress rules to allow traffic**.

**Result:**
- Ingress: Specific policies apply (works for those pods)
- Egress: Default deny applies to ALL (nothing can connect out!)

### The Solution: Remove or Fix Egress Policy

**Option 1: Remove egress policy entirely**

```bash
kubectl edit networkpolicy test-policy -n staging
```

Change:
```yaml
# BEFORE:
policyTypes:
- Ingress
- Egress

# AFTER:
policyTypes:
- Ingress
```

Now it's just a default deny ingress (which is safe).

### Verify the Fix

```bash
# Inside api pod:
telnet dbservice 3306
# Connection successful! ✓

ping google.com
# Response! ✓
```

---

## Real Example 4: Staging Namespace - Cross-Namespace Leak

### The Problem

Frontend pod in qa-test namespace can connect to database in staging namespace (should be blocked!).

### Investigation

**Test from QA to Staging:**
```bash
kubectl exec -it frontend-pod -n qa-test -- /bin/sh

# Inside pod:
telnet dbservice.staging.svc.cluster.local 3306
# Connection successful! ✗ Should fail!
```

(Using fully qualified domain name: `service.namespace.svc.cluster.local`)

### Check Staging Database Network Policy

```bash
kubectl describe networkpolicy dbnp -n staging
```

Output:
```yaml
name: dbnp
podSelector:
  matchLabels:
    app: db
ingress:
- from:
  - namespaceSelector: {}  # ← Empty selector = ALL namespaces!
  - podSelector:
      matchLabels:
        app: api
```

**Two Issues Here:**

1. **Empty namespaceSelector:** Matches ALL namespaces
2. **OR logic:** List of selectors means either rule matches

**Result:** Frontend from qa-test namespace matches "all namespaces" rule → allowed!

### The Fix: Match Specific Namespace with AND

```yaml
# BEFORE (OR + ALL namespaces):
ingress:
- from:
  - namespaceSelector: {}          # ALL namespaces
  - podSelector:
      matchLabels:
        app: api                   # Any api pod

# AFTER (AND + specific namespace):
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        name: staging              # ONLY staging namespace
    podSelector:
      matchLabels:
        app: api                   # AND api pod
```

Apply the fix:
```bash
kubectl edit networkpolicy dbnp -n staging
```

### Verify the Fix

```bash
# Inside frontend pod in qa-test:
telnet dbservice.staging.svc.cluster.local 3306
# Now times out ✓ Correct!
```

---

## Complete Network Policy Reference

### AND Pattern (Preferred)

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: staging
    podSelector:
      matchLabels:
        app: api
    # BOTH conditions required
```

**Use when:** You want traffic only from specific pods in specific namespaces.

### OR Pattern (Use Carefully)

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: staging
  - podSelector:
      matchLabels:
        app: api
    # EITHER condition sufficient
```

**Use when:** You want traffic from (this namespace) OR (these pods anywhere).

### Multiple AND Rules

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: staging
    podSelector:
      matchLabels:
        app: api
- from:
  - namespaceSelector:
      matchLabels:
        environment: prod
    podSelector:
      matchLabels:
        app: admin
    # Either rule can match (each rule is AND internally, but rules are OR)
```

**Logic:** `(staging AND api) OR (prod AND admin)`

---

## Network Policy Best Practices

### 1. Always Use Default Deny

```yaml
# Default deny ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

Apply to every namespace where you use network policies.

### 2. Be Explicit with Selectors

```yaml
# DON'T: Empty/broad selectors
namespaceSelector: {}  # Matches ALL namespaces!

# DO: Specific selectors
namespaceSelector:
  matchLabels:
    name: staging  # Specific namespace
```

### 3. Understand AND vs OR

```yaml
# DON'T: Ambiguous intent
- namespaceSelector:
    matchLabels: ...
- podSelector:
    matchLabels: ...

# DO: Clear intent
# AND pattern:
- namespaceSelector:
    matchLabels: ...
  podSelector:
    matchLabels: ...

# OR pattern (explicit):
- from:
  - namespaceSelector: ...
- from:
  - podSelector: ...
```

### 4. Use Multiple Rules Instead of Complex Lists

```yaml
# DON'T: Complex OR logic
- from:
  - namespaceSelector: ...
  - podSelector: ...
  - ...

# DO: Multiple AND rules (easier to read)
- from:
  - namespaceSelector: ...
    podSelector: ...
- from:
  - namespaceSelector: ...
    podSelector: ...
```

### 5. Document Network Policies

```yaml
metadata:
  name: api-ingress
  annotations:
    description: "Allow frontend to call api"
    allowed-from: "frontend pods in staging namespace"
    denied-from: "all other pods and namespaces"
```

### 6. Test Network Policies

```bash
# Test from within pod
kubectl exec -it <pod> -- telnet <target> <port>
kubectl exec -it <pod> -- curl <target>:8080

# Test cross-namespace
kubectl exec -it <pod> -n ns1 -- telnet <svc>.ns2.svc.cluster.local

# Verify policies exist
kubectl get networkpolicies
kubectl describe networkpolicy <name>
```

---

## Troubleshooting Checklist

```bash
# 1. List all network policies
kubectl get networkpolicies -n <namespace>

# 2. Examine each policy
kubectl describe networkpolicy <name> -n <namespace>

# 3. Identify pod selectors
# Look for: podSelector: app=X

# 4. Identify namespace selectors
# Look for: namespaceSelector: environment=X

# 5. Check for empty selectors
# Empty selector: {} means ALL

# 6. Test connectivity from pod
kubectl exec -it <pod> -- telnet <target> <port>

# 7. Test cross-namespace
kubectl exec -it <pod> -n ns1 -- telnet <svc>.ns2.svc.cluster.local <port>

# 8. Check default deny policies
kubectl get networkpolicies -n <namespace> | grep deny

# 9. Verify AND vs OR logic
# Look at YAML structure (list vs single rule)

# 10. Check namespace labels
kubectl get namespace -o yaml | grep labels -A 5
```

---

## Interview Story: Network Policy Debugging

**The Scenario:**
"I had to troubleshoot network policies across two namespaces. In QA, connections were happening that shouldn't (frontend reaching database). In staging, nothing worked, plus cross-namespace traffic was leaking."

**First Issue - OR Logic Problem:**
"I checked the database network policy and found it had two rules in a list: namespace selector and pod selector. This was OR logic, so any pod in the qa namespace could reach the database, not just the API. The fix was combining those selectors into a single rule (AND logic)."

**Second Issue - Missing Default Deny:**
"The frontend pod could be reached by anything because there was no network policy protecting it. I applied a default deny ingress policy to all pods. Because network policies are additive, specific policies still worked while everything else was denied."

**Third Issue - Default Deny Egress:**
"In staging, nothing worked at all, not even external connectivity. The network policy had egress deny-all with no specific egress rules. Since there's no way to override a global egress deny with specific policies (like you can with ingress), I removed the egress policy and made it ingress-only deny."

**Fourth Issue - Cross-Namespace with Empty Selector:**
"The database policy had an empty namespace selector (matching ALL namespaces) combined with an OR selector. This allowed pods from other namespaces to reach it. The fix was specifying only the staging namespace and using AND logic."

**Key Insight:**
"The critical lesson: understand AND vs OR in network policies. List of selectors = OR. Combined selectors = AND. Also, be explicit with selectors—empty selectors are dangerous. And finally, network policies are additive: specific rules override generic ones, except for egress where there's no override mechanism."

---

## Key Takeaways

1. **AND vs OR is critical:** List = OR, combined = AND

2. **Empty selectors are dangerous:** `{}` matches everything

3. **Network policies are additive:** Specific overrides generic for ingress

4. **Default deny is essential:** Protect frontend pods and other exposed services

5. **Egress policies are absolute:** No override mechanism, all pods affected

6. **Cross-namespace requires FQDN:** `service.namespace.svc.cluster.local`

7. **Test thoroughly:** Use `telnet`, `curl`, `ping` to verify connectivity

8. **Document intent:** Annotations explain why rules exist

9. **One rule per policy is clearer:** Multiple policies easier than complex rules

10. **Staging is not QA:** Test in actual environment before production