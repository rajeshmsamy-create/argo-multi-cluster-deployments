# Argo CD RBAC Guide

## Introduction

Argo CD RBAC (Role-Based Access Control) defines **who can access Argo CD** and **what actions they can perform**.  
It works by assigning permissions to **roles**, and mapping **users or groups** to those roles.

RBAC config is stored mainly in:

- **argocd-rbac-cm** → Policies, roles, default access  
- **argocd-cm** → Authentication provider (OIDC, LDAP, GitHub, etc.)

---

# RBAC Core Concepts

## 1️⃣ Roles
Describe what actions can be performed.

## 2️⃣ Users / Groups
Authenticated by SSO or Dex.

## 3️⃣ Policies
Rules that bind roles to actions on resources.

RBAC rule syntax:

```
p, <user|group|role>, <resource>, <action>, <object>, <effect>
```

Example:

```
p, role:devops, applications, sync, *, allow
```


# 🛠 Common RBAC Actions

| Action | Meaning |
|--------|---------|
| `get` | View resource |
| `create` | Create resource |
| `update` | Modify resource |
| `delete` | Remove resource |
| `sync` | Sync applications |
| `override` | Sync even with validation warnings |
| `exec` | Open shell into pods |
| `action/*` | Run custom actions |

---

# ⭐ Example RBAC Configurations

## 1. Read-Only Role

Users can only view Argo CD.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    p, role:readonly, applications, get, *, allow
    p, role:readonly, projects, get, *, allow
    p, role:readonly, logs, get, *, allow

  scopes: "[groups]"

  role.readonly: "Read only access"
```

Assign group:

```
g, myteam, role:readonly
```

---

## 2. DevOps Role (Typical Permissions)

Can sync applications and view logs but cannot delete or modify settings.

```yaml
policy.csv: |
  p, role:devops, applications, get, *, allow
  p, role:devops, applications, sync, *, allow
  p, role:devops, logs, get, *, allow
  p, role:devops, exec, create, *, allow
  p, role:devops, settings, get, *, allow

g, devops-team, role:devops
```

---

## 3. Full Admin Access

Argo CD has a built-in role:

```
role:admin
```

Assign your team:

```
g, platform-team, role:admin
```

---

## 4. App-Specific RBAC

Only allow a team to manage *guestbook-dev*:

```yaml
p, role:guestbook-dev-team, applications, get, guestbook-dev, allow
p, role:guestbook-dev-team, applications, sync, guestbook-dev, allow

g, devteam, role:guestbook-dev-team
```

# Applying RBAC Changes

```bash
kubectl apply -f argocd-rbac-cm.yaml -n argocd
kubectl rollout restart deployment argocd-server -n argocd
```

Or:

```bash
kubectl delete pod -l app.kubernetes.io/name=argocd-server -n argocd
```

##Use case:

1. Create a user - Bob

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  # …your existing keys here…

  # New user account that can log in and use API tokens
  accounts.bob: apiKey,login
```

```
kubectl apply -f argocd-cm.yaml -n argocd
kubectl rollout restart deployment argocd-server -n argocd
```

2. Edit argocd-rbac-cm to assign the right policy
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    # --- Project access ---
    # readsync can view only the 'guestbook' project
    p, user:readsync, projects, get, guestbook, allow

    # --- Application access ---
    # readsync can view and sync guestbook-dev
    p, user:readsync, applications, get, guestbook-dev, allow
    p, user:readsync, applications, sync, guestbook-dev, allow

    # readsync can view and sync guestbook-prod
    p, user:readsync, applications, get, guestbook-prod, allow
    p, user:readsync, applications, sync, guestbook-prod, allow

  # Optional but recommended: everyone else is read-only
  policy.default: role:readonly
```

Apply
```
kubectl apply -f argocd-rbac-cm.yaml -n argocd
kubectl rollout restart deployment argocd-server -n argocd
```

3. Generate a token for this user

From a machine where argocd CLI is configured and you can log in as admin (or another powerful user):

Login as admin (if not already):
```
argocd login <argocd-server-url> \
  --username admin \
  --password <admin-password> \
  --insecure   # only if you don't have proper TLS
```

Generate token for bob:

```
argocd account generate-token --account bob
```

You’ll get a long token string like: