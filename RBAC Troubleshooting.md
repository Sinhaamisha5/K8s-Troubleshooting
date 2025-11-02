# RBAC Troubleshooting: Finding and Fixing Permission Leaks

## The Scenario: The RBAC Security Leak

### The Setup

A company wants to give summer interns minimal access to a staging cluster:
- **Intended:** Can only `get`, `watch`, `list` pods in `staging` namespace
- **Actual:** Can access **all resources** in **all namespaces** (!)

This is a security breach—interns have far more access than intended.

---

## Understanding RBAC Components

### The Three Components

```
1. Role / ClusterRole
   ├─ Defines what actions are allowed
   ├─ What resources can be accessed
   └─ What verbs (get, list, create, delete, etc.)

2. RoleBinding / ClusterRoleBinding
   ├─ Binds a Role to a user/group
   ├─ Specifies WHO gets these permissions
   └─ Can reference Roles or ClusterRoles

3. User / Group / ServiceAccount
   ├─ The entity being granted permissions
   ├─ Can be a user (Bob, Alice)
   ├─ Can be a group (developers, interns)
   └─ Can be a service account (API)

Permission = User + RoleBinding + Role
```

### Scope: Role vs ClusterRole

```
Role: Limited to a specific namespace
├─ Example: "interns can list pods in staging namespace only"

ClusterRole: Applies cluster-wide
├─ Example: "interns can list pods in ALL namespaces"
└─ WARNING: Very powerful!
```

---

## Real Example: Finding the RBAC Leak

### Step 1: Check Intended Permissions

**List Roles in staging namespace:**

```bash
kubectl get roles -n staging
```

Output:
```
NAME           CREATED AT
intern-role    2024-10-22T09:00:00Z
```

**Examine the role:**

```bash
kubectl describe role intern-role -n staging
```

Output:
```
Name:         intern-role
Namespace:    staging
Labels:       <none>

Rules:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  pods       []                  []              [get list watch]
```

**This looks correct:** Interns can only `get`, `list`, `watch` pods in staging.

**Check RoleBinding:**

```bash
kubectl get rolebinding -n staging
```

Output:
```
NAME                   ROLE              AGE
intern-role-binding    Role/intern-role  5d
```

**Examine the RoleBinding:**

```bash
kubectl describe rolebinding intern-role-binding -n staging
```

Output:
```
Name:         intern-role-binding
Namespace:    staging
Labels:       <none>

Role:
  Kind:  Role
  Name:  intern-role

Subjects:
  Kind   Name      Namespace
  ----   ----      ---------
  Group  interns   
```

**This also looks correct:** The `interns` group is bound to `intern-role` in the staging namespace.

### Step 2: Test Intended Permissions (Should Work)

```bash
# Test: Can Bob (in interns group) list pods in staging?
kubectl auth can-i list pods -n staging \
  --as=bob \
  --as-group=interns
```

Output:
```
yes
```

✅ **Expected behavior—Bob can list pods in staging.**

### Step 3: Test Unintended Permissions (Shouldn't Work)

```bash
# Test: Can Bob get secrets in top-secret namespace?
kubectl auth can-i get secrets -n top-secret \
  --as=bob \
  --as-group=interns
```

Output:
```
yes
```

❌ **Unexpected! Bob shouldn't have access to secrets in top-secret namespace!**

### Step 4: Find the RBAC Leak (With Verbosity)

Use verbosity flag to see which role is granting unwanted access:

```bash
# Same test but with high verbosity
kubectl auth can-i get secrets -n top-secret \
  --as=bob \
  --as-group=interns \
  -v 10
```

Output (relevant parts):
```
...
Decision:
  verb:get
  resource:secrets
  namespace:top-secret
  allowed: true
  reason: RBAC rule matched
  by: clusterrolebinding/developer-binding
  
ClusterRoleBinding: developer-binding
  ClusterRole: developer-role
  Subjects: Group=developers, Group=interns
  
ClusterRole: developer-role
  Rules: apiGroups: ["*"], resources: ["*"], verbs: ["*"]
```

**Found the leak!** ClusterRoleBinding `developer-binding` is binding:
- `interns` group (too broad!)
- To `developer-role` (which has `*` access to everything)

---

## Understanding the RBAC Leak

### The Problem Visualized

```
RBAC Decision Tree:

User: Bob
Group: interns

1. Check Roles in current namespace (staging):
   ├─ RoleBinding: intern-role-binding
   └─ Role: intern-role
      └─ Allows: [get, list, watch pods] ✓ Matched

2. Check ClusterRoles:
   ├─ ClusterRoleBinding: developer-binding
   │  ├─ ClusterRole: developer-role
   │  ├─ Subjects: [developers, interns]  ← interns here!
   │  └─ Rules: [all resources, all verbs]
   └─ Result: ✓ Matched!

3. RBAC Decision:
   Bob is in "interns" group
   "interns" is bound to developer-role via developer-binding
   developer-role allows ALL access
   Result: ALLOWED
```

### The Root Cause

```
Who added interns to developer-binding?
├─ Originally: developers group only
├─ Someone added: interns group (mistake!)
└─ Result: Interns now have admin-level access
```

---

## The Fix: Remove Interns from ClusterRoleBinding

### Step 1: Edit the ClusterRoleBinding

```bash
kubectl edit clusterrolebinding developer-binding
```

Original (with leak):
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: developer-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: developer-role
subjects:
- kind: Group
  name: developers
- kind: Group
  name: interns     # ← REMOVE THIS (the problem!)
```

Fixed (without leak):
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: developer-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: developer-role
subjects:
- kind: Group
  name: developers
# ← interns group removed
```

Save and exit.

### Step 2: Verify the Fix

**Test 1: Bob should NOT have access to secrets in top-secret**

```bash
kubectl auth can-i get secrets -n top-secret \
  --as=bob \
  --as-group=interns
```

Output:
```
no
```

✅ **Fixed! Bob no longer has access to top-secret.**

**Test 2: Bob should NOT have access to secrets in staging**

```bash
kubectl auth can-i get secrets -n staging \
  --as=bob \
  --as-group=interns
```

Output:
```
no
```

✅ **Correct. The Role binding in staging doesn't allow secrets.**

**Test 3: Bob SHOULD still have access to pods in staging**

```bash
kubectl auth can-i list pods -n staging \
  --as=bob \
  --as-group=interns
```

Output:
```
yes
```

✅ **Correct. Bob still has intended access.**

---

## kubectl auth can-i: The Diagnostic Command

### Basic Syntax

```bash
kubectl auth can-i <verb> <resource> [flags]
```

### Examples

```bash
# Am I allowed to list pods in default namespace?
kubectl auth can-i list pods

# Can user "bob" delete deployments in staging?
kubectl auth can-i delete deployments -n staging \
  --as=bob

# Can the "developers" group get secrets?
kubectl auth can-i get secrets \
  --as=bob \
  --as-group=developers

# Can service account "myapp" create pods?
kubectl auth can-i create pods \
  --as=system:serviceaccount:default:myapp

# Why is "bob" allowed to do something? (high verbosity)
kubectl auth can-i get secrets -n top-secret \
  --as=bob \
  --as-group=interns \
  -v 10  # or -v 9 or -v 8 for more detail
```

### Verbosity Levels

```bash
# Default (no flags)
- Shows only: yes / no

# -v 1 to -v 5
- Shows basic decision info
- Shows which role matched

# -v 6 to -v 8
- Shows detailed RBAC rules
- Shows matching process

# -v 9 to -v 10
- Shows complete RBAC decision tree
- Shows all considered roles
- Shows exact matching rules
- BEST FOR DEBUGGING LEAKS!
```

---

## Common RBAC Mistakes

### Mistake 1: Binding Group to ClusterRole

```yaml
# DON'T: This gives everyone in group admin access
ClusterRoleBinding:
  metadata:
    name: admin-binding
  subjects:
  - kind: Group
    name: all-users  # ← Too broad!
  roleRef:
    name: cluster-admin
```

```yaml
# DO: Use fine-grained roles instead
RoleBinding:  # ← Namespace-scoped!
  metadata:
    namespace: staging
    name: dev-binding
  subjects:
  - kind: Group
    name: developers
  roleRef:
    name: developer-role
    kind: Role  # ← Not ClusterRole
```

### Mistake 2: Wildcard Resources

```yaml
# DON'T: Allows access to all resources
Role:
  rules:
  - apiGroups: ["*"]
    resources: ["*"]  # ← TOO BROAD
    verbs: ["*"]

# DO: Be specific
Role:
  rules:
  - apiGroups: [""]
    resources: ["pods", "services"]  # ← Specific resources
    verbs: ["get", "list", "watch"]   # ← Specific verbs
```

### Mistake 3: Over-Permissive Verbs

```yaml
# DON'T: Allows deletion
Role:
  rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["*"]  # ← Includes delete!

# DO: Only needed verbs
Role:
  rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]  # ← Read-only
```

### Mistake 4: ClusterRole when Role is Better

```yaml
# DON'T: Affects all namespaces
ClusterRoleBinding:
  roleRef:
    kind: ClusterRole
    name: view  # ← Applies everywhere

# DO: Limit scope
RoleBinding:  # ← In specific namespace
  roleRef:
    kind: Role
    name: viewer
```

---

## Audit Your RBAC: Systematic Approach

### Step 1: List All Roles and ClusterRoles

```bash
# Roles in a namespace
kubectl get roles -n staging

# ClusterRoles (cluster-wide!)
kubectl get clusterroles

# Look for "admin", "edit", "*" wildcards
kubectl get clusterroles -o json | jq '.items[] | select(.rules[0].verbs[]=="*")'
```

### Step 2: List All Bindings

```bash
# Bindings in a namespace
kubectl get rolebindings -n staging

# ClusterRoleBindings (MOST DANGEROUS!)
kubectl get clusterrolebindings

# See what's bound to what
kubectl describe clusterrolebinding <name>
```

### Step 3: Check for Overly Permissive Bindings

```bash
# Look for dangerous patterns:
# - ClusterRole with "*" verbs
# - ClusterRoleBinding to large groups
# - Unnecessary admin bindings

kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.kind=="ClusterRole" and .roleRef.name=="cluster-admin")'
```

### Step 4: Test User Permissions

```bash
# For each user/group, test what they can access
kubectl auth can-i list pods \
  --as=<user> \
  --as-group=<group> \
  -v 10  # ← High verbosity to see why
```

### Step 5: Validate Against Policy

```bash
# Compare actual vs intended:
# Intern: Should only list pods in staging namespace
# Developer: Should only manage deployments in dev/staging
# Admin: Should have full access

# Test each and verify
```

---

## Interview Story: The RBAC Security Audit

**Setup:**
"I was brought in to audit RBAC permissions on a production cluster. Management was concerned about over-permissive access, especially for contractors and interns."

**Initial Check:**
"I created a simple test user and tried to check their permissions using `kubectl auth can-i`. The user claimed they only had read-only access to one namespace, but when I tested, they could delete resources in other namespaces too."

**Investigation:**
"Using the verbosity flag (`-v 10`), I could see exactly which ClusterRoleBinding was granting unwanted access. Turns out their group had been added to an admin ClusterRoleBinding by mistake, and nobody documented it."

**The Problem:**
"The user was bound to a developer ClusterRole that had wildcard permissions on all resources. This happened because someone added the entire 'contractors' group to an admin ClusterRoleBinding, not realizing it would give them cluster-wide access."

**The Solution:**
"I removed the contractors group from the dangerous ClusterRoleBinding and created a proper namespaced RoleBinding with limited permissions. After the fix, they only had access to what they should have."

**Key Insight:**
"The lesson: use namespace-scoped Roles and RoleBindings when possible. Reserve ClusterRoles and ClusterRoleBindings for truly cluster-wide needs. And always verify with `kubectl auth can-i -v 10` to see the exact source of permissions."

---

## Best Practices

### 1. Principle of Least Privilege

```yaml
# DON'T: Give everyone cluster-admin
kind: ClusterRoleBinding
subjects:
- kind: Group
  name: all-developers

# DO: Give specific permissions
kind: RoleBinding
metadata:
  namespace: dev
subjects:
- kind: Group
  name: developers
roleRef:
  kind: Role
  name: developer-role
```

### 2. Namespace Isolation

```yaml
# Use Roles (namespace-scoped), not ClusterRoles
kind: Role  # ← Namespace-scoped!
metadata:
  namespace: staging
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

### 3. Specific Verbs

```yaml
# DON'T: Wildcard verbs
verbs: ["*"]  # ← Too permissive

# DO: Only needed verbs
verbs: ["get", "list", "watch"]  # ← Read-only
```

### 4. Specific Resources

```yaml
# DON'T: Wildcard resources
resources: ["*"]  # ← Everything

# DO: Specific resources
resources: ["pods", "services", "configmaps"]
```

### 5. Regular Audits

```bash
# Periodically check:
# Who has what access?
# Are there unnecessary ClusterRoleBindings?
# Are groups too permissive?

# Use kubectl auth can-i to verify
```

---

## RBAC Configuration Examples

### Example 1: Intern with Minimal Access

```yaml
# Role: Read-only pods in staging
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: intern-role
  namespace: staging
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

---
# RoleBinding: Bind to interns group
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: intern-binding
  namespace: staging
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: intern-role
subjects:
- kind: Group
  name: interns
```

### Example 2: Developer with Dev Access

```yaml
# ClusterRole: Dev-level permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer-role
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update"]
- apiGroups: [""]
  resources: ["services", "configmaps"]
  verbs: ["get", "list"]

---
# ClusterRoleBinding: Only to dev/staging namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: developer-role
subjects:
- kind: Group
  name: developers
```

### Example 3: ServiceAccount with Minimal Access

```yaml
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-reader
  namespace: default

---
# Role: Read-only for deployment
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-reader-role
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-reader-binding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-reader-role
subjects:
- kind: ServiceAccount
  name: app-reader
  namespace: default
```

---

## Troubleshooting Checklist

```bash
# 1. Test if user can perform action
kubectl auth can-i <verb> <resource> -n <namespace> \
  --as=<user> \
  --as-group=<group>

# 2. If unexpected result, use high verbosity
kubectl auth can-i <verb> <resource> -n <namespace> \
  --as=<user> \
  --as-group=<group> \
  -v 10

# 3. Check which role/binding granted access
# Look for "allowed by: <role/clusterrole-binding>"

# 4. Review the identified role/binding
kubectl describe role <role-name> -n <namespace>
kubectl describe clusterrolebinding <binding-name>

# 5. Check if group/user should be there
kubectl get rolebinding <binding-name> -o yaml

# 6. Remove unwanted subject if necessary
kubectl patch rolebinding <binding-name> \
  --type json \
  -p='[{"op": "remove", "path": "/subjects/0"}]'

# 7. Re-test to verify fix
kubectl auth can-i <verb> <resource> \
  --as=<user> \
  --as-group=<group>
```

---

## Key Takeaways

1. **Role vs ClusterRole:** Use Role (namespace-scoped) by default

2. **RoleBinding vs ClusterRoleBinding:** RoleBinding is safer

3. **Principle of Least Privilege:** Only grant needed permissions

4. **Test Before Trusting:** Use `kubectl auth can-i` to verify

5. **High Verbosity Reveals Leaks:** `-v 10` shows exact source

6. **Wildcard Resources/Verbs:** Dangerous, be specific

7. **Regular Audits:** Check for over-permissive bindings

8. **Document Policy:** Know who should have what access

9. **Groups Over Users:** Easier to manage at scale

10. **ClusterRoleBindings Are Dangerous:** Use sparingly, review carefully