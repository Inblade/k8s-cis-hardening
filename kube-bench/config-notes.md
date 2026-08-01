# kube-bench Configuration Notes

Practical notes on getting trustworthy results out of kube-bench, gathered
from running it across self-managed and managed clusters.

## Pick the right benchmark

kube-bench autodetects the Kubernetes version, but on managed clusters
autodetection selects the *generic* CIS benchmark, which produces a wall of
false FAILs for controls the provider handles. Pass the matching profile
explicitly:

| Cluster | Flag |
|---|---|
| GKE | `--benchmark gke-1.6.0` |
| EKS | `--benchmark eks-1.5.0` |
| AKS | `--benchmark aks-1.7` |
| kubeadm / self-managed | let autodetect run, verify with `kube-bench version` output |

Check `kube-bench` release notes for the newest profile matching your
cluster version — profiles lag new Kubernetes releases by a few months.

## Choose targets deliberately

- `--targets node` — the only thing that makes sense on managed clusters;
  you cannot see the control plane anyway.
- `--targets master,etcd,controlplane,node,policies` — self-managed control
  plane nodes. The `master` target must run **on** a control plane node
  (schedule with the control plane toleration + nodeSelector).

## Why the Job needs what it needs

- `hostPID: true` — checks read live process command lines
  (`/proc/<pid>/cmdline`) to verify flags actually in effect, not just what
  a config file says.
- Read-only hostPath mounts of `/etc/kubernetes`, `/var/lib/kubelet`,
  `/etc/systemd` — file permission and config-content checks.
- None of the mounts are writable and the container needs no privileged
  mode. If your admission policy (rightly) blocks hostPath, add a scoped
  exception for the scanner namespace rather than weakening the policy.

## Interpreting results

- **FAIL ≠ act immediately.** First classify: real gap / provider-managed /
  deliberate exception. The score only means something after that pass.
- **WARN items are manual checks**, not passes. The RBAC and policy sections
  (5.x) are mostly WARN and are where the important findings usually are.
- Compare runs over time, not absolute scores between clusters — different
  benchmarks have different check counts.

## Operational pattern

- Run on demand or as a CronJob (weekly is plenty); `ttlSecondsAfterFinished`
  keeps completed jobs from accumulating.
- Ship JSON output (`--json`) to object storage if you need an audit trail:
  `kubectl logs job/kube-bench > results/$(date +%F).json`.
- One job per node pool is enough for permission checks — nodes in a pool
  are images of each other. Scan every pool, not every node.
