# APF → Red Hat SAW: governance-interceptor drop-in

**Audience:** Red Hat Secure Agent Workspace (SAW) maintainers. **Goal:** replace the reference
`governance-interceptor` with **Agent Policy Fabric (APF)** to gain *signed* policy bundles, provenance,
Cedar effects, and a correlated audit trail — the capabilities SAW's docs list as **Phase II, not
implemented** — with **no change to the gateway VM**.

## TL;DR — two flags

APF ships the same OpenShell `GatewayInterceptor` gRPC contract as
[`charts/governance-interceptor`](https://github.com/validatedpatterns-sandbox/secure-agent-workspace/tree/main/charts/governance-interceptor)
(port `18081`, `failure_policy=fail_closed`, `timeout=500ms`, `gatewayEndpoint=https://127.0.0.1:17670`).
Install APF where that chart went, with two flags to match SAW's expectations exactly:

```bash
helm install governance-interceptor <APF-chart> -n <saw-namespace> \
  --set fullnameOverride=governance-interceptor \   # Service DNS the gateway.toml already points at
  --set networkPolicy.enabled=true                  # ingress from the KubeVirt gateway VM + setup Job
```

- **`fullnameOverride=governance-interceptor`** → APF's Service is named `governance-interceptor`, so the
  gateway VM's `gateway.toml` resolves it **unchanged**:
  `grpc_endpoint = http://governance-interceptor.<ns>.svc.cluster.local:18081`.
- **`networkPolicy.enabled=true`** → APF renders the same ingress NetworkPolicy SAW ships (allow
  `kubevirt.io: virt-launcher` + `app.kubernetes.io/name: openshell-saw` on `18081`). Only needed on
  default-deny namespaces (SAW's GitOps clusters).

## What APF replaces — and what it does NOT touch

```
  KubeVirt VM (Fedora golden image)              Kubernetes pod (Helm/ArgoCD)
  ┌───────────────────────────┐                  ┌──────────────────────────────┐
  │ OpenShell gateway         │ ──gRPC 18081──▶  │  governance-interceptor  ⇦ APF │
  │ (cloud-init gateway.toml) │  fail_closed     │  ← THIS is what APF replaces   │
  └───────────────────────────┘                  └──────────────────────────────┘
        ▲ unchanged                                     ▲ signed bundle in
        │ (no VM rebuild)                                 (replaces plain ConfigMaps)
```

- **Unchanged:** the per-user KubeVirt **gateway VM**, its golden image, cloud-init, and `gateway.toml`.
  APF is a cluster pod the VM calls over the Service DNS name — the VM boundary is unaffected.
- **Replaced:** the `governance-interceptor` **Deployment + Service** (and, optionally, its
  NetworkPolicy) — all reproduced by APF's chart.
- **Superseded:** the `governance-policy` chart's plain-YAML ConfigMaps (see below).

## Feeding APF its policy — signed bundle vs. `governance-policy` ConfigMaps

SAW's `governance-policy` chart mounts plain YAML (`/config/policy/policy.yaml`, `/config/profiles/*.yaml`)
and hot-reloads on file change. **APF instead consumes one cryptographically *signed* bundle** (a tarball
in a ConfigMap; an init container unpacks and the policy-service verifies the Ed25519 signature before
loading). That signing/provenance **is** the Phase-II value-add — but it changes the admin workflow from
*"drop a YAML in Git"* to *"sign a bundle, commit the signed artifact."*

Two ways to fit that into GitOps:

1. **Sign locally, commit the artifact** — run `apf-compile genfixture` (or `docker run … apf-compile`) to
   produce the signed bundle, commit it, and point the chart at it with `--set-file bundle.tarGz=…` (or a
   `bundle.existingConfigMap`). Deterministic and auditable; the signed bundle is the reviewed unit.
2. **Sign in-cluster** — enable APF's signing Job (`--set signingJob.enabled=true --set-file
   signingJob.sourceTarGz=…`): it runs `apf-compile` in-cluster, writes the signed bundle to a ConfigMap,
   and the chart consumes it. No local Docker needed. (An ArgoCD PreSync hook is the natural home.)

Provider profiles and sandbox policy carry the **same semantics** as SAW's `profiles/*.yaml` +
`policy.yaml` — APF just delivers them signed and verifies them at load.

## Compatibility matrix (APF vs. SAW `governance-interceptor`)

| Dimension | SAW governance-interceptor | APF | Status |
|---|---|---|---|
| gRPC port | `18081` | `18081` | ✅ match |
| `failure_policy` | `fail_closed` | `fail_closed` | ✅ match |
| `timeout` | `500ms` | `500ms` | ✅ match |
| `gatewayEndpoint` | `https://127.0.0.1:17670` | identical | ✅ match |
| resources | 256Mi/500m · 128Mi/100m | identical | ✅ match |
| Service (ClusterIP, name) | `governance-interceptor` | `governance-interceptor` *(via `fullnameOverride`)* | ✅ with flag |
| NetworkPolicy (virt-launcher + setup Job) | ships one | `networkPolicy.enabled=true` | ✅ with flag |
| startupProbe on the port | tcpSocket 18081 | tcpSocket `interceptor` | ✅ match |
| Policy delivery | plain ConfigMaps + file-watch | **signed bundle** + verify | ⬆ APF adds provenance |
| Signed bundles · Cedar · egress middleware · audit | *Phase II, not implemented* | **implemented** | ⬆ APF value-add |

## Running on OpenShift

Add `--set openshift=true`: APF then pins no UID/fsGroup (the SCC assigns them), runs no root init
container, adds no capabilities, and sets `seccompProfile: RuntimeDefault` — deploying cleanly under the
default **`restricted-v2`** SCC. Use `oc` for `kubectl`; nothing else changes.

## Verify the drop-in

```bash
# 1) Service name the gateway.toml expects exists on 18081
kubectl -n <ns> get svc governance-interceptor -o wide

# 2) NetworkPolicy admits the gateway VM + setup Job
kubectl -n <ns> get networkpolicy governance-interceptor -o yaml | grep -A6 'from:'

# 3) pod healthy, bundle verified, startupProbe passing
kubectl -n <ns> get pod -l app.kubernetes.io/name=apf
kubectl -n <ns> logs deploy/governance-interceptor -c policy-service | grep 'bundle verified and loaded'

# 4) the gateway VM's gateway.toml is unchanged — it already points at:
#    grpc_endpoint = http://governance-interceptor.<ns>.svc.cluster.local:18081
```

---
*APF is the NVIDIA Agent Policy Fabric. This guide targets the SAW reference architecture at
`validatedpatterns-sandbox/secure-agent-workspace`. Full tutorial: see the project README.*
