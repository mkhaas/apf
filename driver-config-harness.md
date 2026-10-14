# APF — governed harness-configuration mount (`driver_config` allowlist)

**Status:** implemented, unit-tested, and **validated live on a real Kubernetes OpenShell gateway**. Shared for Red Hat (Secure Agent Workspace) review.

## The problem this solves

A Secure Agent Workspace wants to inject a reviewed **harness** — a vetted set of agent skills + MCP servers — into a user's sandbox at runtime. OpenShell has no API to push files into a sandbox, so the harness arrives as a **volume mount** via OpenShell's `driver_config` on `CreateSandbox`.

But once the gateway allows `driver_config` at all (`allow_driver_config=true`), OpenShell's built-in guards only ensure the volume is *attachable*. **Any signed-in user could then mount any attachable volume, any way** — the capability that delivers the skills is also an attack surface.

**What APF adds:** it *validates* the `driver_config` at admission against a **signed allowlist** in the SandboxPolicy — admit the exact sanctioned `saw-harness-*` mount by an admin, deny everything else, deny-by-default, and the rule is gated by the Ed25519 **signing key** (stronger than an unsigned ConfigMap).

## Driver-agnostic by design

The rule describes **the mount**, not one driver's wire syntax. APF normalizes each compute driver's own mount config onto one abstract shape (`type` / `source` / `target` / `read_only`) and matches *that*, so a single signed rule governs **podman, docker, and kubernetes**.

**The rule (authored once, signed):**
```yaml
driver_config:
  deny_templates: true
  allowed:
    - require_role: openshell-admin          # only an admin identity may mount
      drivers: [podman, docker, kubernetes]  # one abstract shape covers all three
      max_mounts: 1
      mounts:
        - { type: volume, source_prefix: saw-harness-, target: /sandbox/harness, read_only: true }
```

| Rule field | Means | podman / docker | kubernetes |
|---|---|---|---|
| `source_prefix` | *what* is mounted must start with this | `source` (volume name) | PVC `claim_name` |
| `target` | *where* it lands in the sandbox | `target` | `mount_path` |
| `read_only` | must be read-only | `read_only` | `read_only` (volume or mount) |

The same harness mount, expressed per driver — all three **ADMIT**:

```jsonc
// ① podman / ② docker (spec.template.driver_config) — flat inline mounts:
{ "podman": { "mounts": [
  { "type": "volume", "source": "saw-harness-abc123", "target": "/sandbox/harness", "read_only": true }
] } }

// ③ kubernetes — PVC volume joined by name to a container volume mount:
{ "kubernetes": {
  "volumes": [ { "name": "harness",
      "persistent_volume_claim": { "claim_name": "saw-harness-abc123", "read_only": true } } ],
  "containers": { "agent": { "volume_mounts": [
      { "name": "harness", "mount_path": "/sandbox/harness", "read_only": true } ] } }
} }
```

## Safety properties (acceptance criteria)

| # | Property | How APF meets it |
|---|---|---|
| 1 | **Deny by default** | No `driverConfig` rule in the signed bundle ⇒ any present driver config denied |
| 2 | **Identity from the gateway only** | Role from `principal["roles"]` (verified OIDC / mTLS cert OU), never a request field |
| 3 | **Refuse templates** | A template-based create carrying/needing driver config is refused (its config can't be checked pre-resolution) |
| 4 | **Exact match** | Exact target, read-only required, mount-count cap, source prefix — on the *normalized* mount, so the same rule holds across drivers |
| 6 | **Audited** | Every allow/deny → APF audit record with a correlation id |
| 7 | **Only the gateway's driver** | Exactly one driver key, equal to the configured compute driver |

> **On criterion 5 (every RPC that can carry driver config):** verified live — OpenShell 0.1.x marks `openshell.v1.OpenShell/CreateSandboxTemplate` as a **non-interceptable RPC** and rejects the whole interceptor if a binding targets it. APF therefore does not bind it; the template threat is covered at `CreateSandbox` Validate (criterion 3). The handler is retained for when/if OpenShell makes the RPC interceptable.

## Validated live on Kubernetes

Deployed on a real single-node cluster (OpenShell `compute_driver=kubernetes`, APF registered as its **fail-closed** interceptor) and driven with a real `openshell sandbox create --driver-config-json`:

| Request (admin caller) | APF decision |
|---|---|
| PVC `saw-harness-*` → `/sandbox/harness`, read-only | ✅ `driver_config.allowed` (create then proceeds to the k8s driver) |
| `mount_path: /etc` | ❌ `driver_config.denied.no_match` |
| non-`saw-harness-` PVC `claim_name` | ❌ `driver_config.denied.no_match` |
| writable (`read_only: false`) | ❌ `driver_config.denied.no_match` |
| plain sandbox (no driver_config) | ✅ allowed untouched — **created** (no regression) |

Confirmed the kubernetes PVC shape normalizes correctly live (`volumes` joined to `containers.agent.volume_mounts`), the role gate holds, and exact-match holds.

**Tier-1 only.** An admission-only policy (driver_config + filesystem, no egress/content rules) injects **no** supervisor middleware, so this governs on an interceptor-only (tier-1) deployment with nothing else to register. APF wires its egress content middleware **only** when the policy carries L7 network/content rules.

## Deploy / redeploy

You need new **images** and a signed **bundle** — **no change to the released chart line** (this ships as a preview).

1. **Images** — pull **`0.1.0-driverconfig.3`** (multi-arch amd64 + arm64) for all three: `apf-policy-gateway`, `apf-policy-service`, `apf-compile`. (The gateway enforces, policy-service mints the rule into the provenance token, apf-compile lowers the authored rule.)
2. **Chart** — install/upgrade with the **preview chart `0.2.1-driverconfig`**, which wires the rule + compute driver:
   ```
   helm install apf oci://ghcr.io/mkhaas/apf/charts/apf --version 0.2.1-driverconfig \
     -n apf --create-namespace --set image.pullSecrets[0].name=ghcr-pull
   ```
   Already on it? Just bump the image tag to `.3`.
3. **Compute driver** — `--set interceptor.computeDriver=kubernetes` (or `podman`), matching your gateway.
4. **Bundle** — an example SandboxPolicy with the `driver_config` block ships in the demo bundle (`examples/demo-bundle/sandbox/default-policy.yaml`). Re-sign with `apf-compile` to produce the signed bundle.
5. **Existing policies/sandboxes** — no action. Deny-by-default means bundles without the rule keep today's behavior; normal creates are unaffected.

Then a `CreateSandbox` with the sanctioned mount is admitted and everything else is denied + audited.
