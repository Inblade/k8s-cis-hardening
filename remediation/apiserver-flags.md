# API Server Hardening Flags

Reference for hardening `kube-apiserver` on self-managed control planes
(kubeadm layout assumed: edit
`/etc/kubernetes/manifests/kube-apiserver.yaml`; the kubelet restarts the
static pod on save). On managed clusters these flags are the provider's
responsibility — see [../docs/managed-clusters.md](../docs/managed-clusters.md).

Change one flag group at a time and watch API server health between changes.
A typo in a static pod manifest takes the API server down, and you fix it
over SSH, not kubectl.

## Authentication and authorization

```
--anonymous-auth=false
```
Anonymous requests get username `system:anonymous`. Nothing legitimate needs
it once health checks use authenticated probes; disable it so RBAC mistakes
on `system:unauthenticated` can never matter.

```
--authorization-mode=Node,RBAC
```
`Node` restricts kubelets to their own objects; `RBAC` for everything else.
`AlwaysAllow` must never appear. Order matters: authorizers are tried left
to right.

```
--enable-admission-plugins=NodeRestriction
```
`NodeRestriction` stops a compromised kubelet from modifying other nodes'
objects or labelling itself into scheduling decisions. On 1.25+ rely on Pod
Security admission (built in, on by default) for pod-level policy.

## Attack surface reduction

```
--profiling=false
```
pprof endpoints expose memory contents and timing internals; nobody profiles
a production API server through the public endpoint.

```
--service-account-lookup=true
```
Validates that a presented service account token's secret still exists —
deleted tokens die immediately instead of living until expiry.

```
--kubelet-certificate-authority=/etc/kubernetes/pki/ca.crt
--kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
--kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
```
Without `--kubelet-certificate-authority` the API server does not verify
kubelet serving certs, enabling MITM of exec/logs traffic.

## Audit logging

```
--audit-log-path=/var/log/kubernetes/audit/audit.log
--audit-log-maxage=30
--audit-log-maxbackup=10
--audit-log-maxsize=100
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
```
Non-negotiable. Minimum useful policy: `RequestResponse` level for Secrets,
ConfigMaps, and RBAC objects; `Metadata` for the rest; drop high-volume
read-only noise (`get`/`watch` on endpoints/leases by system components).
Mount both paths into the static pod (`hostPath`) — a flag pointing at an
unmounted path fails silently at best.

## etcd and in-transit encryption

```
--etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
--etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
--etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
--tls-cert-file=/etc/kubernetes/pki/apiserver.crt
--tls-private-key-file=/etc/kubernetes/pki/apiserver.key
--tls-min-version=VersionTLS12
```
etcd client certs must come from the etcd CA (separate CA from the cluster
CA in the kubeadm layout — keep it that way).

## Encryption at rest

```
--encryption-provider-config=/etc/kubernetes/enc/encryption-config.yaml
```
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded 32-byte key>
      - identity: {}
```
Provider order is the write order; `identity` last allows reading legacy
plaintext during migration. After enabling, rewrite existing Secrets so they
are actually encrypted:

```bash
kubectl get secrets -A -o json | kubectl replace -f -
```

Prefer a `kms` provider over `aescbc` where a cloud/HSM KMS is available —
`aescbc` keys live on the control plane disk next to the data they protect.

## Deprecated checks you can skip

Older benchmark versions reference `--insecure-port` and
`--insecure-bind-address`; the insecure port was removed in Kubernetes 1.20+
(current versions refuse to start with a non-zero value), and
`--basic-auth-file` / `--token-auth-file` static auth was removed/should be
absent. If kube-bench flags these on a modern cluster, it is running the
wrong benchmark version.
