# Kubelet Hardening Configuration

The kubelet is the most attacked node component: its API can exec into any
pod on the node. Prefer the config file (`/var/lib/kubelet/config.yaml`,
`KubeletConfiguration`) over command-line flags — flags override the file,
so hardening in the file with contradicting flags still present is a silent
no-op. Check the live state, not the file:

```bash
ps -ef | grep kubelet | tr ' ' '\n' | grep '^--'
```

## Reference configuration

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration

# --- Authentication / authorization (CIS 4.2.1–4.2.3) ---
authentication:
  anonymous:
    enabled: false            # anonymous kubelet API access = node takeover
  webhook:
    enabled: true             # delegate authn of bearer tokens to the API server
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook               # never AlwaysAllow — SubjectAccessReview per request

# --- Attack surface (CIS 4.2.4–4.2.5) ---
readOnlyPort: 0               # legacy 10255 serves pod specs unauthenticated
protectKernelDefaults: true   # refuse to start if sysctls were tampered with

# --- Certificate hygiene (CIS 4.2.10–4.2.12) ---
rotateCertificates: true      # client cert rotation
serverTLSBootstrap: true      # request serving certs from the cluster CA
tlsMinVersion: VersionTLS12
tlsCipherSuites:
  - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
  - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
  - TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
  - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384

# --- Sensible operational hardening (not CIS, but do it) ---
makeIPTablesUtilChains: true
eventRecordQPS: 5             # 0 disables event rate limiting — keep it non-zero
streamingConnectionIdleTimeout: 4h   # never 0 (unlimited exec/attach sessions)
podPidsLimit: 4096            # fork-bomb containment per pod
```

Apply with a kubelet restart per node (or a node pool roll on managed/IaC
setups):

```bash
systemctl restart kubelet
```

## Notes per decision

- **`authorization.mode: Webhook`** sends a `SubjectAccessReview` for every
  kubelet API request, so `--kubelet-client-certificate` on the API server
  side must be valid or exec/logs breaks cluster-wide. Change both sides in
  one maintenance window.
- **`serverTLSBootstrap: true`** creates CSRs that need approval; pair with
  an approver (e.g. kubelet-csr-approver) or certificates stay pending and
  kubelet serving falls back to self-signed — which reintroduces the MITM
  gap you were closing.
- **`protectKernelDefaults: true`** makes kubelet fail fast if kernel
  parameters differ from its expectations. Set the sysctls in the node image
  first, then flip this on — otherwise fresh nodes crash-loop on boot.
- **`readOnlyPort: 0`** occasionally breaks ancient monitoring agents that
  scrape 10255. Their loss. Migrate them to the authenticated
  `/metrics/resource` endpoint on 10250.

## Verification

```bash
# Authenticated port must reject anonymous requests
curl -sk https://<node-ip>:10250/pods    # expect 401 Unauthorized

# Read-only port must be closed
curl -s --max-time 3 http://<node-ip>:10255/pods; echo "exit=$?"  # expect connection refused

# Live config as the kubelet sees it (proxied through the API server)
kubectl get --raw "/api/v1/nodes/<node-name>/proxy/configz" | jq .
```

The `configz` check is the authoritative one — it reflects merged
flag+file+drop-in state, which is what actually runs.
