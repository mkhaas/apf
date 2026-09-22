# APF (Agent Policy Fabric) — GTC Berlin PoC

## Contents

- [Overview](#overview)
- [Access — private registry (start here)](#access--private-registry-start-here)
- [Fit with the Red Hat SAW governance-interceptor pattern](#fit-with-the-red-hat-saw-governance-interceptor-pattern)
- [Details](#details)
- [Prerequisite — create the signed bundle (do this before Helm)](#prerequisite--create-the-signed-bundle-do-this-before-helm)
- [The Helm chart](#the-helm-chart)
  - [What it deploys](#what-it-deploys)
  - [Values to run](#values-to-run)
- [The demo](#the-demo)
- [Two install paths](#two-install-paths)
  - [Brownfield — you already run OpenShell and have your own sandbox policy](#brownfield--you-already-run-openshell-and-have-your-own-sandbox-policy)
  - [Greenfield — from scratch](#greenfield--from-scratch)
- [Authoring & signing](#authoring--signing)
  - [A minimalistic OpenShell sandbox policy](#a-minimalistic-openshell-sandbox-policy)
  - [Sign it — `docker run`](#sign-it--docker-run)
  - [Deploy + watch APF inject it](#deploy--watch-apf-inject-it)
  - [Change the policy — re-sign live](#change-the-policy--re-sign-live)
  - [Provider config — show APF *rejecting* a bad change](#provider-config--show-apf-rejecting-a-bad-change)
  - [APF *owns* policy (no runtime side-door)](#apf-owns-policy-no-runtime-side-door)
- [Audit & insights](#audit--insights)
- [Operations quick-reference](#operations-quick-reference)
- [Appendix — All Helm values supported](#appendix--all-helm-values-supported)
- [Signing — reference](#signing--reference)
  - [How signing works](#how-signing-works)
  - [Signing in-cluster with a Job (no local Docker)](#signing-in-cluster-with-a-job-no-local-docker)
- [Concepts](#concepts)

## Overview

**Agent Policy Fabric (APF)** aims to provide a reference blueprint for governing autonomous agents by
facilitating standardized interfaces across identity, risk, and runtime enforcement points such as OpenShell.
By compiling security policies into cryptographically signed, verifiable bundles, APF projects centralized
intent into native execution layers — eliminating decision drift and delivering trusted, immutable
auditability. Currently, APF provides four capabilities:

1. **Federated policy governance** — security, compliance, and tool owners contribute policy under
   enterprise trust and review.
2. **Signed policy projection** — one canonical bundle compiles into controls each enforcement surface
   can consume.
3. **Layered runtime verification** — containment, capability, and semantic checks; each answers a
   different risk question.
4. **Traceable evidence and response** — correlated decisions support investigation, review, and agent
   containment.

**This tutorial demonstrates capability #2 — Signed policy projection:** authoring a policy, signing it
into one canonical bundle, and projecting it onto the OpenShell enforcement surface. It also **partially**
shows **capability #4 — Traceable evidence**.
(Runs on a Kubernetes or OpenShift cluster.)

## Access — private registry (start here)

The APF images and Helm chart are **private** on GHCR (`ghcr.io/mkhaas/apf`). Get access and authenticate first:

1. **Ask the owner (@mkhaas)** to grant your GitHub user access to the packages.
2. **Create a GitHub PAT** with the **`read:packages`** scope (github.com/settings/tokens).
3. **Log in** (paste the PAT as the password):
   ```bash
   docker login ghcr.io -u <your-github-username>
   helm registry login ghcr.io -u <your-github-username>
   ```
4. **Create a cluster pull secret** (once):
   ```bash
   kubectl create namespace apf 2>/dev/null
   kubectl -n apf create secret docker-registry ghcr-pull \
     --docker-server=ghcr.io --docker-username=<your-github-username> --docker-password=<your-PAT>
   ```
   Every `helm install` / `helm upgrade` in this doc already carries `--set image.pullSecrets[0].name=ghcr-pull`.

## Fit with the Red Hat SAW governance-interceptor pattern

APF implements the **same OpenShell `GatewayInterceptor` gRPC contract** as the Red Hat **Secure Agent
Workspace (SAW)** [`governance-interceptor`](https://github.com/validatedpatterns-sandbox/secure-agent-workspace)
chart (port 18081, `failure_policy=fail_closed`). So APF **drops into that slot** — and adds what that
pattern lists as *Phase II, not implemented*: signed bundles, provenance, Cedar, egress content middleware,
and this audit trail.

**Running on OpenShift:** install with `--set openshift=true`. The chart then pins no UID/fsGroup (the SCC
assigns them), runs no root init container, adds no added capabilities, and sets `seccompProfile:
RuntimeDefault` — so it deploys cleanly under the default **`restricted-v2`** SCC. Use `oc` in place of
`kubectl`; everything else is identical.

## Details

APF ships as **three pre-built container images and a Helm chart**:

| Artifact | Role | Purpose | Location |
|---|---|---|---|
| `apf-policy-service` (image) | PAP | policy authoring/serving + signing | `ghcr.io/mkhaas/apf/apf-policy-service:0.2.0` |
| `apf-policy-gateway` (image) | projector | OpenShell gateway interceptor — projects the signed policy | `…/apf/apf-policy-gateway:0.2.0` |
| `apf-compile` (image) | CLI | signs the policy bundle | `…/apf/apf-compile:0.2.0` |
| `apf` (Helm chart) | deploy | installs and configures APF (pod: service + gateway) | `oci://…/apf/charts/apf:0.2.0` |

Add an image pull secret (`--set image.pullSecrets[0].name=<secret>`) if your cluster needs registry
credentials. See [The Helm chart](#the-helm-chart) for what it deploys and its values. *(`apf-compile` is a
CLI you run with `docker` — not a deployed container; the pod itself runs just `policy-service` +
`policy-gateway`.)*

Prerequisites: a Kubernetes **or** OpenShift cluster with **OpenShell** installed, `kubectl`/`helm`, and a
signing key pair (a dev key is included; use your own for production).

## Prerequisite — create the signed bundle (do this before Helm)

**Which case are you?**

| You want… | What to do |
|---|---|
| **APF demo policy + APF demo key** — just try it | **Skip this prerequisite — apply Helm directly:** `helm install apf oci://ghcr.io/mkhaas/apf/charts/apf --version 0.2.0 -n apf --create-namespace --set image.pullSecrets[0].name=ghcr-pull` |
| **Your own key** (keep the APF demo policy, or use yours) | Sign the policy with your key, then install — the **3 steps below** (skip the editing if you keep the APF demo policy). |
| **Your own policy** (with the APF demo key or yours) | Edit the policy, then sign + install — the **3 steps below**. |

*A bundle is bound to the key that signs it, so changing the key means re-signing — there's no key-only shortcut.*

> **On OpenShift:** add **`--set openshift=true`** to every `helm install` / `helm upgrade` in this doc (and
> use `oc` for `kubectl`). It renders the `restricted-v2`-safe variant — no pinned UID/fsGroup, no root init,
> no added capabilities.

To run your own policy or keys, you create the signed bundle first (before Helm). You only ever edit two kinds of file —
your **sandbox policy** (`sandbox/default-policy.yaml`) and your **provider profiles**
(`provider-profiles/*.yaml`); everything else is a working default. **Three steps:**

**Step 1 — get the demo bundle from the chart.** It's already inside the chart — no separate download.
Extract the demo bundle and copy the dev keys into working folders:

```bash
# pull the chart — gives ./apf with the demo bundle + dev keys inside:
helm pull oci://ghcr.io/mkhaas/apf/charts/apf --version 0.2.0 --untar
mkdir -p policy-bundle keys
tar xzf apf/files/default-bundle.tar.gz -C policy-bundle
cp apf/files/apf-dev.seed apf/files/apf-dev.pub keys/
```

That gives you:
```
policy-bundle/
├── sandbox/default-policy.yaml   # ←  edit: your sandbox policy (egress, providers)
├── provider-profiles/*.yaml      # ←  edit: per-provider rules (the demo tampers one)
├── rules/ · annotators/ · effects/ · findings/   #  defaults — exposed, change only if you want
├── demo-bundle.template.yaml     #    leave as-is (the manifest; signing fills it in)
└── demo-bundle.yaml              #    the signed demo bundle
keys/  apf-dev.seed (private) · apf-dev.pub (public)
```

**Step 2 — edit what you need:** your **sandbox policy** and/or a **provider profile**. Just trying it out?
Edit nothing — the defaults run as-is.

**Step 3 — sign your bundle, then install it.**
```bash
# 1) sign your policy — apf-compile writes the signed demo-bundle.yaml into policy-bundle/:
docker run --rm -v "$PWD/policy-bundle:/work" -v "$PWD/keys:/keys" \
  ghcr.io/mkhaas/apf/apf-compile:0.2.0 \
  genfixture --key /keys/apf-dev.seed --keyid apf-dev \
    --template /work/demo-bundle.template.yaml --base-dir /work --out /work/demo-bundle.yaml
# 2) package the signed folder and install it (on OpenShift, add: --set openshift=true):
( cd policy-bundle && tar czf /tmp/bundle.tgz * )
helm install apf oci://ghcr.io/mkhaas/apf/charts/apf --version 0.2.0 -n apf --create-namespace --set image.pullSecrets[0].name=ghcr-pull --set-file bundle.tarGz=/tmp/bundle.tgz
```

> **No Docker on your workstation?** The chart can sign your policy **in-cluster** with a one-shot Job
> instead (`--set signingJob.enabled=true`) — it signs, then you install APF onto the result. Full steps:
> [Signing in-cluster with a Job](#signing-in-cluster-with-a-job-no-local-docker).

> **Additional note — use your own keys for anything real.** The dev keys baked into the chart are a
> **shared demo key: everyone who has the chart has the same one**, and it doubles
> as the trust root — so a default install can be forged against (the chart prints a warning). For real use,
> **generate your own** Ed25519 (EdDSA) pair (`--generate-key`, or `head -c 32 /dev/urandom | base64`), then
> pass both: `--set-file signing.seed=keys/apf-dev.seed` (private) and `--set-file trustRoot.pub=keys/apf-dev.pub`
> (public). *(Already have a key? Drop `--generate-key`.)*

---

## The Helm chart

### What it deploys
A single `helm install apf oci://…/charts/apf --version 0.2.0` creates:

| Object | Purpose |
|---|---|
| **Deployment** (1 pod) | init container `prepare` (untars the signed bundle) + two app containers: `policy-service` (PAP) and `policy-gateway` (the interceptor/projector), sharing a Unix socket |
| **Service** | exposes the interceptor on `:18081` (and the egress middleware on `:9099` when `tier=2`) |
| **PersistentVolumeClaim** (optional) | durable audit volume when `audit.persistence=true` |

The chart **sets up everything it needs for you** — you don't create any Kubernetes objects by hand.

### Values to run

**None are required.** `helm install apf oci://ghcr.io/mkhaas/apf/charts/apf --version 0.2.0 -n apf --create-namespace --set image.pullSecrets[0].name=ghcr-pull` runs out of the box with
a baked demo bundle + dev key. To run *your* policy, pass your files (the chart turns them into the
Kubernetes objects):

| Value | What it does | How to pass it |
|---|---|---|
| `bundle.tarGz` | your signed bundle | `--set-file bundle.tarGz=/tmp/bundle.tgz` |
| `trustRoot.pub` | your public key (omit → baked dev key) | `--set-file trustRoot.pub=keys/apf-dev.pub` |
| `signing.seed` | your private key (omit → baked dev key) | `--set-file signing.seed=keys/apf-dev.seed` |

*(Add `--set image.pullSecrets[0].name=<secret>` if your cluster needs credentials to pull the images.
Everything else — ports, tier, audit, security — is optional; see the appendix only if you need it.)*

## The demo

| # | Demo | Section |
|---|---|---|
| 1 | Edit an OpenShell sandbox policy → sign → deploy → APF injects + enforces | [Authoring & signing](#authoring-signing) |
| 2 | Change the policy → re-sign → redeploy (revision update, provenance-stamped) | [Change the policy live](#change-the-policy-re-sign-live) |
| 3 | Tamper a provider config → APF rejects it | [Provider rejection](#provider-config-show-apf-rejecting-a-bad-change) |
| 4 | Try a direct policy edit → refused: "policy is managed by APF" | [APF owns policy](#apf-owns-policy-no-runtime-side-door) |
| 5 | Audit logs: merged OpenShell+APF timeline + live signature verification | [Audit & insights](#audit-insights) |

Deep-dive concepts are at the bottom ([Concepts](#concepts)).

---

> ### ☑ Kubernetes vs OpenShift — the only three deltas
> Every command below is **identical** on OpenShift. The differences are exactly:
> 1. **`oc` ≡ `kubectl`** — use either.
> 2. **`helm install apf … --set openshift=true`** — omits the pinned uid/fsGroup (the SCC assigns them),
>    uses no root init and no added capabilities, sets `seccompProfile: RuntimeDefault`. Verified
>    `restricted-v2`-compliant by render.
> 3. Nothing else — Service DNS, images, interceptor wiring, audit paths are the same.
>
> *The outputs shown in this doc were captured on a vanilla-K8s cluster; on OpenShift they are identical.
> The APF chart is designed + render-verified for `restricted-v2`; your `oc apply` on the OpenShift cluster
> is the final confirmation.*

## Two install paths

### Brownfield — you already run OpenShell and have your own sandbox policy
You keep everything you have; you just **add APF** and point OpenShell's interceptor at it.

**Step 1 — put your existing policy into a signed bundle** (details in [Authoring & signing](#authoring-signing)): copy your
current OpenShell sandbox policy into `policy-bundle/sandbox/default-policy.yaml` (and any providers into
`policy-bundle/provider-profiles/`), sign it (same `docker run` as the
[Prerequisite](#prerequisite-create-the-signed-bundle-do-this-before-helm)), then package it:
```bash
( cd policy-bundle && tar czf /tmp/bundle.tgz * )     # package the signed bundle for Helm
```

**Step 2 — install APF** (the chart sets up everything it needs from your files):
```bash
helm install apf oci://ghcr.io/mkhaas/apf/charts/apf --version 0.2.0 -n apf --create-namespace --set image.pullSecrets[0].name=ghcr-pull \
  --set-file bundle.tarGz=/tmp/bundle.tgz \
  --set-file trustRoot.pub=keys/apf-dev.pub \
  --set-file signing.seed=keys/apf-dev.seed
# OpenShift: add  --set openshift=true
# (Images default to the published 0.2.0 tags; override with --set gateway.image=… / policyService.image=… if needed.)
```

**Step 3 — register APF into your OpenShell gateway.** Add this block to your OpenShell `gateway.toml` —
the chart's `NOTES.txt` prints this exact block **and** the ready-to-run patch command for your release:
```toml
[[openshell.gateway.interceptors]]
name           = "apf-policy-adapter"
grpc_endpoint  = "http://apf-apf.apf.svc.cluster.local:18081"
order          = 0
failure_policy = "fail_closed"
allow_insecure_transport = true      # dev/tier 1; remove and use TLS for production
```
Then recreate the OpenShell gateway pod so it reloads the config.

> **Two different dev toggles (don't confuse them):** `allow_insecure_transport` above governs the
> **OpenShell → APF interceptor hop** and applies to **both** brownfield and greenfield — set it `false` +
> supply TLS for production. `--set server.disableTls=true` (greenfield) is separate: it turns off TLS on
> the **OpenShell server's own API** so the `openshell` CLI can connect over plain HTTP. Both are dev-only.

Your existing sandbox policy now flows through APF **signed + provenance-stamped + audited** — you didn't
rewrite anything, you wrapped it.

### Greenfield — from scratch
```bash
# 1) Install OpenShell + the open-source agent-sandbox CRDs
kubectl apply -f https://github.com/kubernetes-sigs/agent-sandbox/releases/download/v1.0.2/sandbox-with-extensions.yaml
helm install openshell oci://ghcr.io/nvidia/openshell/helm-chart --version <ver> -n openshell --create-namespace \
  --set server.disableTls=true    # OpenShell dev option only (no TLS on the OpenShell server); local testing, not required
# 2) then follow the Brownfield steps 1–3 above to add APF.
```

*(Both paths end the same way: OpenShell calls APF's interceptor on every gateway operation.)*

---

## Authoring & signing

### A minimalistic OpenShell sandbox policy
A bundle is a small tree; the heart is the sandbox policy. Excerpt **from the default
`sandbox/default-policy.yaml`** — deny-default egress, each host pinned to an access level (`github` is
read-only; `httpbin` adds an L7 rule):
```yaml
apiVersion: nvidia.agent-policy-fabric.sandbox/v1alpha1
kind: SandboxPolicy
metadata: { name: default }
spec:
  process: { run_as_user: sandbox, run_as_group: sandbox }
  network_policies:                       # deny-default: ONLY these hosts are reachable
    github:
      name: github
      binaries: [{ path: /usr/bin/gh }, { path: /usr/bin/curl }]
      endpoints:
        - { host: api.github.com, port: 443, protocol: rest, access: read-only, enforcement: enforce }
    httpbin:
      name: httpbin
      endpoints:
        - host: httpbin.org
          port: 443
          protocol: rest
          enforcement: enforce
          rules:
            - allow: { method: POST, path: /anything }   # L7: only POST /anything
```

And a **provider profile** (`provider-profiles/github.yaml`) — the allowed provider, its endpoints and
binaries, and how its credential is injected:
```yaml
apiVersion: nvidia.agent-policy-fabric.policy/v1alpha1
kind: Provider
metadata: { name: github, title: GitHub }
spec:
  type: source-control
  endpoints:
    - { host: api.github.com, port: 443, protocol: rest, access: read-only, enforcement: enforce }
  binaries: [/usr/bin/gh, /usr/bin/git]
  credentials:
    - name: api_token
      envVars: [GITHUB_TOKEN]
      required: true
      style: bearer                         # injected as: Authorization: Bearer $GITHUB_TOKEN
      source: { kind: host-env, name: GITHUB_TOKEN }
```

Plus a manifest template `demo-bundle.template.yaml` (lists members with `sha256: REPLACE_WITH_DIGEST`
and `policy_revision: 1`).

### Sign it — `docker run`
Sign with the **same one command** as the
[Prerequisite](#prerequisite-create-the-signed-bundle-do-this-before-helm).
**Captured on the devbox:**
```
manifest  sandbox/default-policy.yaml: sha256=…
manifest  provider-profiles/github.yaml: sha256=622e83a44b8038…
manifest  rules/security-content-gate.yaml: sha256=8e852fe5680e93…
signed bundle written to /work/demo-bundle.yaml
# result: id: cedar-demo-bundle   policy_revision: 1   (Ed25519 signature over the manifest)
```
It hashes every member file and Ed25519-signs the manifest — just `docker run`, no local toolchain to install.
**OpenShift:** identical — `docker`/`podman` on your workstation; the bundle is platform-neutral.

### Deploy + watch APF inject it
```bash
( cd policy-bundle && tar czf /tmp/bundle.tgz * )
helm upgrade apf oci://ghcr.io/mkhaas/apf/charts/apf --version 0.2.0 -n apf --set image.pullSecrets[0].name=ghcr-pull --set-file bundle.tarGz=/tmp/bundle.tgz   # chart auto-rolls the pod
# create a sandbox, then inspect it:
openshell --gateway-endpoint http://127.0.0.1:8080 --gateway-insecure sandbox create --name demo1 --from <img> -- sleep 3600
openshell ... sandbox get demo1     # shows spec.policy (your rules) + apf.nvidia.com/* provenance annotations
```
**Captured on the devbox (`openshell sandbox get`):**
```
Annotations:
  apf.nvidia.com/bundle-id:        cedar-demo-bundle
  apf.nvidia.com/policy-version:   1
  apf.nvidia.com/document-digest:  f1444a1db30c2d669de4…
  apf.nvidia.com/signer-identity:  apf-dev
  apf.nvidia.com/provenance-token: eyJ0eXAiOiJKV1Q…       (EdDSA JWT)
Policy:
  network_policies: anthropic, github, openai, nvidia-inference, httpbin, atg   # your signed rules
```
The sandbox now carries **your policy** (`spec.policy`) **+ APF's provenance stamps** (`metadata.annotations`).
**OpenShift:** identical output.

### Change the policy — re-sign live
Example change **from the default policy** — drop `github` from the allowed hosts:
```diff
   network_policies:
-    github:
-      name: github
-      binaries: [{ path: /usr/bin/gh }, { path: /usr/bin/curl }]
-      endpoints:
-        - { host: api.github.com, port: 443, protocol: rest, access: read-only, enforcement: enforce }
     httpbin:
       name: httpbin
```
Then bump `policy_revision`, re-sign (same command as the
[Prerequisite](#prerequisite-create-the-signed-bundle-do-this-before-helm)), and redeploy:
```bash
# edit sandbox/default-policy.yaml (as above), bump policy_revision to 2, re-sign, then:
( cd policy-bundle && tar czf /tmp/bundle.tgz * )
helm upgrade apf oci://ghcr.io/mkhaas/apf/charts/apf --version 0.2.0 -n apf --set image.pullSecrets[0].name=ghcr-pull --set-file bundle.tarGz=/tmp/bundle.tgz   # chart auto-rolls the pod
# (recreate the OpenShell gateway pod so it re-reconciles the new policy)
# new sandboxes now show policy-version: 2 — the change is live + attributed.
```
> **Note:** bump `policy_revision` on every change — APF's anti-rollback counter refuses a re-used revision.

### Provider config — show APF *rejecting* a bad change
The bundle's `provider-profiles/*.yaml` are the **allowed providers**. APF rejects a bad provider change two ways:

| Bad change | APF response |
|---|---|
| Tamper a `provider-profiles/*.yaml` *after* signing | policy-service refuses the bundle: `manifest_hash_mismatch` |
| Runtime `CreateProvider` not in the signed profiles (or asks `read-write` where the profile says `read-only`) | `provider-instance-create-verify` (Validate) **denies** it; OpenShell fails the operation |

**The signed profile (from the default policy) — `provider-profiles/github.yaml`:**
```yaml
kind: Provider
metadata: { name: github }
spec:
  type: source-control
  endpoints:
    - { host: api.github.com, port: 443, protocol: rest, access: read-only, enforcement: enforce }
  binaries: [/usr/bin/gh, /usr/bin/git]
```

**Case 1 — tamper after signing** (edit the signed file, which changes its hash):
```diff
       endpoints:
-        - { host: api.github.com, port: 443, protocol: rest, access: read-only,  enforcement: enforce }
+        - { host: api.github.com, port: 443, protocol: rest, access: read-write, enforcement: enforce }   # edited AFTER signing
```
→ the file no longer matches the manifest's declared `sha256`, so policy-service refuses the **whole bundle**:
`manifest_hash_mismatch: provider-profiles/github.yaml`.

**Case 2 — disallowed provider at runtime:** a `CreateProvider` that isn't in the signed profiles, or asks
for more than the profile grants (e.g. `read-write` on `api.github.com` where the signed profile says
`read-only`), is denied by the `provider-instance-create-verify` binding (Validate phase) — OpenShell fails
the operation.

**Captured on the devbox — tampered provider profile:**
```
# after signing, edit provider-profiles/github.yaml: access read-only -> read-write, redeploy
APF policy-service:
  Error: bundle verification failed:
    ["manifest_hash_mismatch: provider-profiles/github.yaml:
        declared 622e83a44b8038…  actual 5d02818a1f68ed…"]
APF pod endpoints: (none)          # policy-service refuses the bad bundle -> OpenShell fail-closes
```
APF pinpoints the exact tampered member and refuses to serve. **OpenShift:** identical.

---

### APF *owns* policy (no runtime side-door)

Once APF governs the gateway, OpenShell defers **all** policy authority to it. A direct runtime mutation is
**refused** — captured on the devbox:
```
$ openshell policy update padv --add-allow 'api.github.com:443:PUT:/repos/NVIDIA/OpenShell/contents/docs/**'
Error: 'The caller does not have permission to execute the specified operation'
       "direct policy mutation not permitted; policy is managed by APF"
```
**The only way to change agent policy is a signed, provenance-stamped, audited APF bundle**
([Authoring & signing](#authoring-signing)) — no runtime side-door. *(OpenShift: identical — the refusal
comes from APF governance, not the platform.)*

---

## Audit & insights

Every APF component writes a structured **JSONL** trail (one file per component) under the audit volume:
```bash
POD=$(kubectl -n apf get pod -l app.kubernetes.io/name=apf -o jsonpath='{.items[0].metadata.name}')
kubectl -n apf exec $POD -c policy-gateway -- ls /var/log/apf/audit
#   apf-policy-service-audit.jsonl   apf-policy-gateway-audit.jsonl   apf-policy-gateway-interceptor-audit.jsonl
```
Each event carries: `event_type`, `correlation_id`, `user_id`/`agent_id`/`session_id`, `bundle_id`/`_digest`,
`policy_revision`, `signer_id`, `outcome`, `reason_code`, and the exact `effects.patches`. (Full field-source
table: see the audit walkthrough doc.)

**Every governance decision is recorded** (schema `apf.audit.event.v1`). Denials that used to leave no
trace — a blocked config/approval-mode change, a rejected provider op — now emit a `config_governance`
record that shares the deny's `correlation_id`. **Captured live on the devbox** (attempting an approval-mode
change → refused):
```json
{
  "schema_version": "apf.audit.event.v1",
  "event_id": "evt_5496f117a4a9433e99b21b42e1c16f3b",
  "timestamp": "2026-09-21T19:58:18.644299559Z",
  "event_type": "config_governance",
  "correlation_id": "dfeeeda7-526d-4a52-9c68-db5abd1a8c07",
  "user_id": "unauthenticated-local-dev",
  "enforcement_point_id": "apf-policy-gateway:interceptor",
  "runtime_substrate": "openshell",
  "component": { "name": "apf-policy-gateway-interceptor", "version": "0.1.0",
                 "instance_id": "apf-policy-gateway:interceptor@apf-apf-597f777b69-t786k" },
  "trust_root_id": "apf-policy-service:signing",
  "resource": { "type": "openshell.sandbox.v1", "id": "default" },
  "verb": "interceptor.update_config",
  "outcome": "refused",
  "reason_code": "interceptor.govern_denied.approval_mode_change",
  "error": { "code": "interceptor.govern_denied.approval_mode_change",
             "message": "approval mode is managed by APF", "retryable": false }
}
```
The gateway's deny log carries the **same** `correlation_id` (`dfeeeda7…`), so decision → evidence is 1:1.

**Insight 1 — one request, end to end.** Join OpenShell's gateway log and APF's audit on the shared
`correlation_id` (APF puts it in OpenShell's `log_annotations` *and* its own audit), sort by timestamp.
**Captured on the devbox** (join key `518a4313…`):
```
23:22:07.274  APF/interceptor   policy_injection      (interceptor.policy_injected)
23:22:07.276  OpenShell         CreateSandbox -> interceptor decision="allow"
23:22:07.277  APF/interceptor   admission_validation  (interceptor.validate_allowed)
```
**Insight 2 — provable origin (verified live).** The `provenance-token` is an Ed25519 JWT; verify vs APF's
public key. **Captured on the devbox:**
```
claims: sub=apf-dev  apf/bundle-id=cedar-demo-bundle  apf/version=1  jti=518a4313 (== correlation_id)
SIGNATURE VALID  ✓        genuinely signed by apf-dev (ed25519:wHEcbtZJatNB+9Kg…)
flip 1 bit -> REJECTED ✓  tamper-evident + non-repudiable
```
This ties *this sandbox → this signed policy revision → this signer*, verifiable by anyone with the public
key. **OpenShift:** identical (audit paths, correlation_id, and verification are platform-neutral).

For durable audit across pod restarts, install with `--set audit.persistence=true`.

---

## Operations quick-reference
- **Update policy:** edit `sandbox/`+`provider-profiles/` → `apf-compile genfixture` (bump revision) → `tar czf /tmp/bundle.tgz *` → `helm upgrade apf … --set-file bundle.tarGz=/tmp/bundle.tgz` → recreate OpenShell gateway pod.
- **OpenShift:** `--set openshift=true` (SCC-safe: arbitrary UID, no root init).
- **Multiple governance extensions:** APF is one `[[openshell.gateway.interceptors]]` block; add more (DLP, audit) as more blocks, chained by `order`.

---

## Appendix — All Helm values supported

> **You don't need this section to run APF.** `helm install` works with no values, and the [Values to
> run](#values-to-run) table above covers the common overrides. This is the full reference — skip it unless
> you're tuning ports, tier 2 (egress middleware), audit persistence, or OpenShift specifics.

**Interceptor (projection) — aligned to the standard OpenShell `GatewayInterceptor` config**

| Value | Default | Meaning |
|---|---|---|
| `enabled` | `true` | deploy the interceptor |
| `gateway.image` | `…/apf-policy-gateway:0.2.0` | the interceptor (projector) image |
| `gateway.pullPolicy` | `IfNotPresent` | image pull policy |
| `interceptor.port` | `18081` | GatewayInterceptor gRPC port the OpenShell gateway calls |
| `interceptor.gatewayEndpoint` | `https://127.0.0.1:17670` | OpenShell gateway URL APF calls back to (reconcile/refresh) |
| `interceptor.failurePolicy` | `fail_closed` | deny operations if the interceptor is unreachable |
| `interceptor.timeout` | `500ms` | interceptor evaluation timeout |
| `interceptor.allowInsecureTransport` | `true` | dev; set `false` + `interceptor.tlsSecret` for TLS/SPIFFE |
| `resources` | 256Mi/500m · 128Mi/100m | gateway container resources |

**Policy plane + inputs**

| Value | Default | Meaning |
|---|---|---|
| `policyService.image` | `…/apf-policy-service:0.2.0` | the PAP (authoring/signing) sidecar image |
| `policyService.httpPort` | `19200` | health + JWKS the gateway verifies provenance against |
| `bundle.tarGz` | `""` → baked demo bundle | your signed bundle, via `--set-file bundle.tarGz=…`; chart creates the ConfigMap |
| `trustRoot.pub` | `""` → baked dev key | your public key, via `--set-file trustRoot.pub=…`; chart creates the ConfigMap |
| `signing.seed` | `""` → baked dev key | your private key, via `--set-file signing.seed=…`; chart creates the Secret |
| `bundle.existingConfigMap` | `""` | *override:* reference a ConfigMap you already made (skips auto-create) |
| `trustRoot.existingConfigMap` | `""` | *override:* reference a trust ConfigMap you already made |
| `signing.existingSecret` | `""` | *override:* reference a signing Secret you already made |
| `bundle.manifest` | `demo-bundle.yaml` | manifest filename inside the tree |
| `signing.keyId` | `apf-dev` | key id advertised for signing/provenance |
| `signingJob.enabled` | `false` | render the in-cluster signing Job (+ RBAC) — sign in-cluster instead of via local Docker |
| `signingJob.sourceTarGz` / `signingJob.seed` | `""` / baked | your unsigned bundle + signing key for the Job, via `--set-file` |
| `compile.image` | `…/apf-compile:0.2.0` | the `apf-compile` image the signing Job runs |
| `surfaces` | 7 projection surfaces | what the policy-service serves (`openshell.*` + `apf.*`) |
| `document.kind` / `document.name` | `openshell.sandbox.v1` / `default` | the sandbox policy the gateway injects |

**Tiers, audit, service**

| Value | Default | Meaning |
|---|---|---|
| `tier` | `1` | `1` = interceptor only (admission); `2` = + egress content middleware (TLS) |
| `middleware.port` / `.tlsSecret` / `.maxPayloadBytes` / `.timeout` | `9099` / `""` / `4194304` / `10s` | tier-2 supervisor middleware |
| `audit.persistence` | `false` | `true` = durable audit PVC (keeps JSONL across restarts) |
| `audit.mountPath` | `/var/log/apf/audit` | where the JSONL evidence trail is written |
| `service.type` | `ClusterIP` | how the interceptor is exposed |

**Platform / security**

| Value | Default | Meaning |
|---|---|---|
| `openshift` | `false` | **`true` on OpenShift** — omits pinned uid/fsGroup (SCC assigns them); no root init, no added caps; seccomp `RuntimeDefault` |
| `securityContext.{runAsUser,runAsGroup,fsGroup}` | `65532` | applied only when `openshift=false` |
| `image.pullSecrets` | `[]` | pull secrets for a private registry |


## Signing — reference

### How signing works

You don't need this to get started (you only edit `sandbox/default-policy.yaml`), but here is what the one
signing command actually does. It takes three paths:

| Flag | Meaning |
|---|---|
| `--template …/demo-bundle.template.yaml` | The **manifest** — a "packing list" that names every policy file in the bundle. Each file's digest starts as a placeholder (`sha256: REPLACE_WITH_DIGEST`). You normally leave this alone; you'd only edit it to *add or remove* a policy file. |
| `--base-dir …/policy-bundle` | The **folder holding the actual policy files** the manifest lists (`sandbox/`, `rules/`, `provider-profiles/`, …). The signer reads them here to compute each file's real SHA-256. |
| `--out …/demo-bundle.yaml` | The **signed bundle it writes** — the same manifest with every placeholder replaced by the real digest, plus a signature block. This is the file the Helm chart runs. |

So: **template (in) → hash every file in base-dir → fill in the digests → sign → signed bundle (out).**
The digests bind the manifest to exact file contents, and the signature binds the whole thing to your key —
so any later tampering is detectable. That is the "signed" in *signed policy projection*.

### Signing in-cluster with a Job (no local Docker)

Don't want Docker on your workstation? Sign **inside the cluster**. The chart ships a one-shot Job (with its
own RBAC) that runs `apf-compile`, signs your bundle, and writes the signed bundle into a ConfigMap. It signs
with the baked dev key, or your own via `--set-file signingJob.seed`. **Run it as its own step, then install
APF onto the result** (this order is deterministic — QA-verified end-to-end):

```bash
# 1) package your (unsigned) policy folder:
( cd policy-bundle && tar czf /tmp/bundle.tgz * )

# 2) pull the chart, then render + apply the signing Job — it signs in-cluster
#    and writes the signed bundle into configmap/apf-apf-bundle:
helm pull oci://ghcr.io/mkhaas/apf/charts/apf --version 0.2.0 --untar
helm template apf ./apf --set signingJob.enabled=true \
  --set-file signingJob.sourceTarGz=/tmp/bundle.tgz \
  --set image.pullSecrets[0].name=ghcr-pull \           # lets the Job pull apf-compile (private)
  --show-only templates/signing-job.yaml | kubectl -n apf apply -f -
kubectl -n apf wait --for=condition=complete job/apf-apf-signing --timeout=150s

# 3) install APF onto the signed bundle the Job produced:
helm install apf oci://ghcr.io/mkhaas/apf/charts/apf --version 0.2.0 \
  -n apf --create-namespace --set image.pullSecrets[0].name=ghcr-pull --set bundle.existingConfigMap=apf-apf-bundle
```
*(Private registry? The `--set image.pullSecrets[0].name=…` above lets the Job pull `apf-compile` (or use
`--set compile.image=…` for a reachable image). The Job runs two containers: `sign` — `apf-compile
genfixture` — and `publish` — writes the ConfigMap.)*

## Concepts

- **Bundle** = your policy, authored once and **signed**. A *tree* (manifest + `sandbox/`, `rules/`, `provider-profiles/`, `annotators/`, `findings/`, `effects/`). **One bundle governs many sandboxes.**
- **APF = two processes:** *policy-service (PAP)* signs/serves; *policy-gateway* is the OpenShell interceptor that **projects** the signed policy — injects it and stamps provenance. Deployed as one pod, two containers sharing a socket.
- **Signing (`apf-compile`)** = SHA-256 each member + Ed25519-sign the manifest → tamper-evident. Change any file later → `manifest_hash_mismatch` → refused.
- **Provenance token** = an Ed25519 JWT minted by policy-service, stamped on each sandbox (`apf.nvidia.com/provenance-token`); binds *sandbox ↔ policy revision ↔ signer*; independently verifiable via APF's public key.
- **Annotations vs policy:** `spec.policy` = the enforced *rules*; `metadata.annotations` (`apf.nvidia.com/*`) = provenance *notes* (not enforced). `openshell sandbox get` shows both.
- **Labels vs annotations:** labels = tags K8s selects by (e.g. `openshell.ai/sandbox-workspace` = the tenant); annotations = data you read (the JWT).
- **The 14 interceptor bindings** — APF governs the whole gateway surface across 3 phases (ModifyOperation = change/inject, Validate = allow/deny, PostCommit = observe):

  | Governs | RPCs | Phase |
  |---|---|---|
  | Sandbox lifecycle | CreateSandbox | ModifyOperation (inject), Validate, PostCommit |
  | Config | UpdateConfig | ModifyOperation, Validate, PostCommit |
  | Providers | CreateProvider, UpdateProvider | Validate |
  | Provider profiles | Import/Update/Delete ProviderProfiles | Validate |
  | Policy analysis | SubmitPolicyAnalysis | Validate |
  | Draft approvals | ApproveDraftChunk, ApproveAllDraftChunks | Validate |
- **Multi-tenant:** `workspace_id` attributes every audit event per tenant; options = shared policy / per-tenant policy in one bundle / per-tenant APF deployment.
- **Fail-closed:** if APF is unreachable or the policy fails verification, the gateway **refuses** the operation rather than run it ungoverned.
