# Kubernetes CIS Hardening Kit

A working kit for assessing and remediating Kubernetes clusters against the
CIS Kubernetes Benchmark — distilled from doing this on real self-managed and
managed clusters. It is a reference/cookbook, not a compliance product: the
value is in the prioritisation and the "why", not in chasing a 100% score.

## Structure

```
.
├── checklist.md                    # Condensed benchmark walk-through: what / why / how to check
├── kube-bench/
│   ├── job.yaml                    # One-shot in-cluster kube-bench Job
│   └── config-notes.md             # Benchmark selection, targets, result interpretation
├── remediation/
│   ├── apiserver-flags.md          # Control plane flags with rationale (self-managed)
│   └── kubelet-config.md           # Hardened KubeletConfiguration + verification
└── docs/
    └── managed-clusters.md         # GKE/EKS/AKS: provider's job vs yours
```

## Usage

```bash
# 1. Scan worker nodes (pick the right benchmark on managed clusters —
#    see kube-bench/config-notes.md)
kubectl create namespace kube-bench
kubectl apply -f kube-bench/job.yaml
kubectl -n kube-bench wait --for=condition=complete job/kube-bench
kubectl -n kube-bench logs job/kube-bench

# 2. Classify every FAIL: real gap / provider-managed / accepted exception
# 3. Remediate using remediation/, re-scan, keep the exception list in git
```

Then walk `checklist.md` for the manual sections (RBAC, Pod Security,
NetworkPolicies) — the automated scan covers the smaller half of the
benchmark's value.

## Opinions baked in

- **Section 5 over section 1.** On managed clusters the control plane
  findings are mostly not yours; RBAC sprawl and missing default-deny
  network policies are, and they are where clusters actually get hurt.
- **Verify live state, not config files.** Flags override kubelet config
  files; `configz` and `ps` output are authoritative, files are intent.
- **A documented exception beats a silent pass.** The kit assumes you keep a
  dated fix/verified/accepted bucket list under version control.

## Scope

Strictly defensive: benchmark checks, hardening configuration, and
verification commands. No exploit tooling.

## License

MIT — see [LICENSE](LICENSE).
