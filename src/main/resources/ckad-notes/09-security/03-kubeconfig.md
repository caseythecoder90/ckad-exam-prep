---
section: 09-security
chapter: 03
title: KubeConfig
course: CKAD — Mumshad Mannambeth (KodeKloud/Udemy)
examinable: YES — reading/writing kubeconfig, switching contexts, namespace in context
cka_note: Generating certificates for cluster components is CKA scope; kubeconfig usage is CKAD scope
companion_diagrams:
  - diagrams/03-kubeconfig-structure.png
---

# KubeConfig

> **CKA note:** generating certificates for Kubernetes components (CSR signing, CA, kubelet certs) is CKA/CKS scope. This chapter covers kubeconfig — how to use and manage the credentials file. That's fully in CKAD scope and is tested.

## Why kubeconfig Exists

Without kubeconfig, every kubectl command requires explicit flags:

```bash
kubectl get pods \
  --server https://my-kube-playground:6443 \
  --client-key admin.key \
  --client-certificate admin.crt \
  --certificate-authority ca.crt
```

That's four flags every time, for every command. kubeconfig stores all of that once and kubectl reads it automatically. It also lets a single file manage credentials for multiple clusters and multiple users simultaneously, with a concept called a context that selects which combination is active.

Default location: `$HOME/.kube/config`. No `--kubeconfig` flag needed when using this path.

---

## The Three Sections

![KubeConfig — Structure and Context Wiring](diagrams/03-kubeconfig-structure.png)

kubeconfig has exactly three lists:

### `clusters:`
Describes the clusters you connect to. Each entry has a name and the cluster's connection details: server URL and the CA certificate that signed the API server's TLS cert.

```yaml
clusters:
  - name: production
    cluster:
      server: https://172.17.0.51:6443
      certificate-authority: /etc/kubernetes/pki/ca.crt
      # or inline: certificate-authority-data: <base64>
```

### `users:`
Describes the identities/credentials available. Each entry has a name and the credential material.

```yaml
users:
  - name: admin
    user:
      client-certificate: /etc/kubernetes/pki/users/admin.crt
      client-key: /etc/kubernetes/pki/users/admin.key
```

### `contexts:`
The "marriage" — each context pairs one cluster entry with one user entry. Optionally sets a default namespace. The name is arbitrary but `user@cluster` is the convention.

```yaml
contexts:
  - name: admin@production
    context:
      cluster: production    # must match a name in clusters:
      user: admin            # must match a name in users:
      namespace: finance     # optional — default namespace for this context
```

**The context does not control access.** It's a shortcut that says "when kubectl runs with this context active, use these credentials to talk to this cluster." RBAC on the server side decides what that user can actually do.

---

## current-context — The Active Selection

```yaml
current-context: admin@production
```

This single field determines which context kubectl uses when no `--context` flag is provided. Change it with `kubectl config use-context`.

---

## Full Working Example

```yaml
apiVersion: v1
kind: Config
current-context: admin@production

clusters:
  - name: development
    cluster:
      server: https://dev.example.com:6443
      certificate-authority-data: LS0tLS1CRUdJTi...   # base64 encoded CA cert
  - name: production
    cluster:
      server: https://172.17.0.51:6443
      certificate-authority: /etc/kubernetes/pki/ca.crt

contexts:
  - name: admin@production
    context:
      cluster: production
      user: admin
      namespace: finance
  - name: dev@development
    context:
      cluster: development
      user: dev-user

users:
  - name: admin
    user:
      client-certificate: /etc/kubernetes/pki/users/admin.crt
      client-key: /etc/kubernetes/pki/users/admin.key
  - name: dev-user
    user:
      token: eyJhbGciOiJSUzI1NiIs...   # OIDC JWT or ServiceAccount token
```

---

## Certificates in kubeconfig — Path vs Inline

There are two ways to reference certificate material. They are equivalent; kubectl handles both.

**Option A — file path:**
```yaml
certificate-authority: /etc/kubernetes/pki/ca.crt
client-certificate:    /etc/kubernetes/pki/users/admin.crt
client-key:            /etc/kubernetes/pki/users/admin.key
```
The file must exist on disk at the specified path. Convenient when managing certs separately (e.g., cert rotation replaces the file without touching kubeconfig).

**Option B — base64 inline (`-data` suffix):**
```yaml
certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t...
client-certificate-data:    <base64 of .crt file>
client-key-data:            <base64 of .key file>
```
The cert is embedded directly in the file. The file is self-contained — you can send it to another machine without also copying the cert files. This is what `kubectl config view` shows as `REDACTED` by default and what kubeadm-generated configs use.

To convert a cert file to base64 for pasting into kubeconfig:
```bash
cat ca.crt | base64 -w0    # -w0 prevents line wrapping (Linux)
cat ca.crt | base64         # macOS (no -w flag needed)
```

To see the actual data (not redacted):
```bash
kubectl config view --raw
```

---

## Enterprise OIDC kubeconfig — What You're Looking At

An enterprise kubeconfig has every section present — the file is just long because the tokens are large base64 blobs that push the `users:` section far down. Naming conventions often use `<cluster-url>/<user-email>` for both the user name and context name.

A context's `namespace:` field means every kubectl command in that context defaults to that namespace without needing `-n`.

**Users section** (below the long token blob) with the legacy OIDC format:
```yaml
users:
- name: <cluster-url>/<user>@<ad-domain>
  user:
    auth-provider:
      config:
        client-id: <registered-OIDC-app-id>
        client-secret: ""
        id-token: eyJ0eXAi...                       # current JWT (huge base64)
        idp-issuer-url: https://idp.example.com/adfs # ADFS (AD Federation Services)
        refresh-token: xeLlx...                     # gets a new id-token when it expires
      name: oidc
```

A few things to note here:

**`auth-provider: name: oidc`** — this is the *legacy* OIDC authentication format in kubeconfig, predating the `exec:` plugin approach. It was deprecated in client-go v1.22 and removed in v1.26. Enterprises may keep it working via a pinned/internal kubectl build. The concept is identical to the OIDC exec plugin — just an older API shape.

**`idp-issuer-url: .../adfs`** — ADFS stands for Active Directory Federation Services, Microsoft's enterprise SSO/OIDC provider. It provides OIDC tokens backed by Active Directory. A login tool authenticates with ADFS (often via Kerberos — see ch02), and ADFS issues the `id-token`.

**`id-token`** — the big base64 blob. This is the JWT that kubectl presents to the API server as your credential. Paste it into [jwt.io](https://jwt.io) to see the identity claims (user, groups, expiry). **This is sensitive data** — anyone with this token can impersonate you until it expires (often ~1 hour).

**`refresh-token`** — when the `id-token` expires, kubectl uses the refresh token to get a new `id-token` from the IdP without re-prompting. If the refresh token also expires, you must re-run the login tool.

---

## kubectl config Commands — Complete Reference

```bash
# ── Viewing ──────────────────────────────────────────────────────
kubectl config view                     # print kubeconfig (certs redacted)
kubectl config view --raw               # print with actual cert data
kubectl config view --minify            # print only current context's entries
kubectl config view --kubeconfig=<file> # view a specific file

# ── Contexts ─────────────────────────────────────────────────────
kubectl config current-context          # show active context name
kubectl config get-contexts             # list all contexts (active marked with *)
kubectl config use-context <name>       # switch active context
kubectl config delete-context <name>    # remove a context

# ── Set default namespace for current context ────────────────────
kubectl config set-context --current --namespace=<ns>
# This is probably the most-used config command day-to-day.
# After this, kubectl get pods works without -n <ns>

# ── Modifying entries ────────────────────────────────────────────
kubectl config set-cluster <name> \
  --server=https://api.example.com:6443 \
  --certificate-authority=/path/to/ca.crt

kubectl config set-credentials <name> \
  --client-certificate=/path/to/user.crt \
  --client-key=/path/to/user.key

kubectl config set-context <name> \
  --cluster=<cluster-name> \
  --user=<user-name> \
  --namespace=<ns>

# ── Per-command overrides ─────────────────────────────────────────
kubectl get pods --context=dev@development          # use different context once
kubectl get pods --kubeconfig=/tmp/other-config     # use a different file entirely
kubectl -n kube-system get pods                     # override namespace for one command

# ── Renaming ─────────────────────────────────────────────────────
kubectl config rename-context <old> <new>
```

---

## Multiple kubeconfig Files

You don't have to cram everything into one file. The `KUBECONFIG` environment variable accepts a colon-separated list of paths. kubectl merges them at runtime:

```bash
export KUBECONFIG=~/.kube/config:~/.kube/work-config:~/.kube/personal-config
kubectl config get-contexts   # shows contexts from all three files merged
```

To permanently set this, add the export to `~/.bashrc` or `~/.zshrc`.

Enterprise login tools may write to a specific file or append to `~/.kube/config`. Check which file your current context is in:
```bash
kubectl config view --raw | grep -A3 current-context
```

---

## The Context → Namespace Mental Model

A context does not restrict what namespaces you can access (that's RBAC). It sets the *default* namespace for commands that need one. The namespace field in a context is equivalent to passing `-n <namespace>` on every kubectl command.

```yaml
contexts:
  - name: casey@prod
    context:
      cluster: prod
      user: casey
      namespace: team-dev   # ← default; override with -n
```

Without this: `kubectl get pods` → scoped to `default` namespace  
With this: `kubectl get pods` → scoped to `team-dev`

You can always override with `-n <ns>` or `-A`/`--all-namespaces`.

---

## Exam-Pattern Gotchas

- **Context names are arbitrary strings.** `admin@production` is just a convention. The actual wiring is the `cluster:` and `user:` fields inside the context object.
- **`--context` vs `use-context`:** `--context` overrides for a single command; `use-context` changes `current-context` in the file permanently.
- **Missing `certificate-authority`** causes `x509: certificate signed by unknown authority`. The file must exist at the path specified or the base64 data must be correct.
- **`kubectl config view` redacts cert data** by default — use `--raw` to see it. On the exam, if asked to copy a CA cert into kubeconfig, use `certificate-authority-data` (base64).
- **`set-context --current`** modifies the currently active context, not a new one. This is the fastest way to change your default namespace.
- **`--kubeconfig` flag** takes a file path; `KUBECONFIG` env var takes a colon-separated list. They behave differently when merging is needed.
- **Namespaces in contexts are not written into manifests.** A `namespace: finance` in a context only affects kubectl default behavior; it doesn't appear in pod YAML.
