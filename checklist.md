# CIS Kubernetes Benchmark — Working Checklist

A condensed, opinionated pass through the CIS Kubernetes Benchmark sections
that matter most in practice. For each item: **what** it is, **why** it
matters, and **how to check** it. Full remediation detail lives in
`remediation/`. Section numbers follow the CIS Kubernetes Benchmark v1.x
layout (exact numbering shifts slightly between benchmark versions — verify
against the version kube-bench selects for your cluster).

On managed clusters (GKE/EKS/AKS) sections 1–3 are largely the provider's
problem — see [docs/managed-clusters.md](docs/managed-clusters.md).

---

## 1. Control Plane Components

### 1.1 Control plane node files

- **What:** Ownership/permissions on manifests in `/etc/kubernetes/manifests`,
  the PKI directory, and kubeconfigs (`600`/`644`, `root:root`).
- **Why:** World-writable static pod manifests are instant cluster takeover —
  kubelet will run whatever appears there.
- **Check:**
  ```bash
  stat -c '%a %U:%G %n' /etc/kubernetes/manifests/* /etc/kubernetes/pki -R
  ```

### 1.2 API server flags

The highest-value single section of the benchmark. Key items:

- **Anonymous auth disabled** (`--anonymous-auth=false`) — anonymous requests
  should never reach authorization.
- **Authorization mode** includes `Node,RBAC`, never `AlwaysAllow`.
- **Admission plugins**: `NodeRestriction` enabled; on 1.25+ Pod Security
  admission replaces the removed PodSecurityPolicy.
- **Profiling disabled** (`--profiling=false`) — debug endpoints leak
  internals.
- **Audit logging configured** (`--audit-log-path`, `--audit-policy-file`,
  retention flags) — without it, incident response is archaeology.
- **etcd client certs** (`--etcd-certfile/--etcd-keyfile`) and TLS everywhere.
- **Check:**
  ```bash
  ps -ef | grep kube-apiserver | tr ' ' '\n' | grep '^--'
  ```
  Details and rationale per flag: [remediation/apiserver-flags.md](remediation/apiserver-flags.md).

### 1.2.x Encryption at rest

- **What:** `--encryption-provider-config` with `aescbc`/`kms` for Secrets.
- **Why:** Default etcd storage is base64, not encryption; anyone with an
  etcd backup has every Secret.
- **Check:** flag present, and config file lists `secrets` with a provider
  other than `identity` first.

## 2. etcd

- **What:** Peer and client TLS (`--cert-file`, `--peer-cert-file`,
  `--client-cert-auth=true`, `--peer-client-cert-auth=true`), unique CA.
- **Why:** etcd *is* the cluster — full read access to etcd equals full
  cluster compromise, RBAC never enters the picture.
- **Check:**
  ```bash
  ps -ef | grep etcd | tr ' ' '\n' | grep -E 'cert|auth'
  ```

## 3. Control Plane Configuration

- **What:** Audit policy coverage and log retention; authentication is not
  left on client certificates alone (no revocation) — prefer OIDC/short-lived
  tokens for humans.
- **Why:** Client certs handed to engineers are forever-credentials.
- **Check:** review `--audit-policy-file` content — it should capture
  request-level detail for Secrets access and RBAC changes, metadata for the
  rest.

## 4. Worker Nodes

### 4.1 Node files

- **What:** Permissions on kubelet config, kubeconfig, and CA file
  (`600`/`644`, `root:root`).
- **Check:**
  ```bash
  stat -c '%a %U:%G %n' /var/lib/kubelet/config.yaml /etc/kubernetes/kubelet.conf
  ```

### 4.2 Kubelet configuration

The most commonly failed section on self-managed nodes:

- **`authentication.anonymous.enabled: false`** — the kubelet API can exec
  into pods; anonymous access to it is node compromise.
- **`authorization.mode: Webhook`** (not `AlwaysAllow`) — delegate kubelet
  API authz to the API server.
- **`readOnlyPort: 0`** — the legacy 10255 port serves pod specs (including
  env vars) unauthenticated.
- **`protectKernelDefaults: true`**, cert rotation on, strong
  `tlsCipherSuites`.
- **Check:**
  ```bash
  curl -sk https://<node>:10250/pods    # must be 401/403, never JSON
  curl -s  http://<node>:10255/pods     # must fail to connect
  ```
  Full config: [remediation/kubelet-config.md](remediation/kubelet-config.md).

## 5. Policies (every cluster, managed or not)

- **5.1 RBAC:** no wildcard rules; `cluster-admin` bound to break-glass
  identities only; no `secrets get/list` for service accounts that do not
  need it. Check with `kubectl auth can-i --list --as=system:serviceaccount:<ns>:<sa>`
  or a tool like `rbac-lookup`.
- **5.2 Pod Security Standards:** namespaces labelled with
  `pod-security.kubernetes.io/enforce` (baseline as floor, restricted where
  possible).
- **5.3 Network Policies:** every namespace has at least a default-deny;
  check `kubectl get netpol -A` against the namespace list.
- **5.7 General:** default service account `automountServiceAccountToken:
  false`; workloads in dedicated namespaces, not `default`.

---

## How to use this checklist

1. Run kube-bench ([kube-bench/job.yaml](kube-bench/job.yaml)) for the
   automated subset; it maps findings to these section numbers.
2. Walk section 5 manually — most of it is not flag-checkable and is where
   real clusters actually bleed.
3. Track exceptions in writing. "We fail 1.2.x because our cloud KMS handles
   encryption" is fine; an unexplained FAIL is not.
