# CIS Hardening on Managed Clusters (GKE / EKS / AKS)

The single most common mistake with CIS on managed Kubernetes is burning
weeks on control plane findings you cannot change — and then skipping the
policy sections you actually own. This is the shared-responsibility split as
it works out in practice.

## What the provider already handles

On all three major providers the control plane is theirs: you cannot see or
modify API server flags, etcd, or the scheduler/controller-manager, and the
generic CIS sections 1–3 mostly do not apply as *your* findings.

| Concern | GKE | EKS | AKS |
|---|---|---|---|
| API server flags (anonymous auth, admission plugins, profiling) | Provider | Provider | Provider |
| etcd TLS + encryption at rest | Encrypted by default (Google-managed keys, CMEK optional) | Encrypted by default (KMS-backed since 2023 for new clusters; verify on old ones) | Encrypted by default (platform keys, KMS/BYOK optional) |
| Control plane audit logging | Cloud Audit Logs (admin activity on by default; enable data access) | Opt-in per log type (`api`, `audit`, `authenticator`…) — **off by default** | Opt-in via Diagnostic Settings — **off by default** |
| Control plane upgrades/patching | Provider | Provider | Provider |
| Kubelet baseline config on managed node images | Hardened defaults (COS) | Hardened defaults (AL2023/Bottlerocket) | Hardened defaults (Azure Linux/Ubuntu) |

"Provider handles it" still deserves verification once: run kube-bench with
the provider-specific benchmark (`gke-*`, `eks-*`, `aks-*` — see
[../kube-bench/config-notes.md](../kube-bench/config-notes.md)), which
encodes this split and only scores what you can affect.

## What remains your responsibility

This list is the actual job, and no provider does any of it for you:

1. **RBAC (CIS 5.1).** Cloud IAM integration grants coarse roles;
   within-cluster RBAC sprawl is yours. Audit `cluster-admin` bindings,
   wildcard rules, and service accounts with `secrets get/list`.
2. **Pod Security Standards (5.2).** Label namespaces
   (`pod-security.kubernetes.io/enforce=baseline` as the floor) or enforce
   equivalents via Kyverno/Gatekeeper. Nothing is enforced by default.
3. **Network policies (5.3).** All providers *support* NetworkPolicy; none
   deploys a default-deny for you. On GKE/AKS confirm the network policy /
   dataplane option is actually enabled — policies silently no-op otherwise.
4. **Enabling the opt-in audit logs.** EKS and AKS ship with control plane
   audit logging off. Turning it on (and shipping it somewhere queryable) is
   step zero of incident readiness.
5. **Service account token hygiene (5.1.5–5.1.6).**
   `automountServiceAccountToken: false` on the default SA per namespace;
   workload identity federation (GKE Workload Identity / EKS Pod Identity /
   AKS Workload Identity) instead of long-lived cloud keys in Secrets.
6. **Secrets encryption above the default.** Default encryption uses
   provider-managed keys. If compliance requires customer-managed keys:
   CMEK (GKE), KMS envelope encryption (EKS), KMS/BYOK (AKS) are opt-in.
7. **Node pools you customize.** The moment you use custom node images,
   self-managed node groups (EKS), or DaemonSets that touch node config,
   section 4 (kubelet, file permissions) is yours again — scan those pools
   with `--targets node`.
8. **Everything above the cluster line:** admission policy, image
   provenance, runtime detection, workload securityContext — the subjects of
   the sibling policy/falco/supply-chain repositories.

## Practical workflow

1. Run kube-bench with the provider benchmark → small, relevant finding set.
2. Diff findings into three buckets: *fix*, *provider-verified*, *accepted
   with reason*. Store the bucket list in the repo next to the scan date.
3. Re-run after every minor cluster upgrade — provider defaults change
   between versions, usually in your favour, occasionally not.
