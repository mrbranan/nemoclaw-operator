# Refactor Plan: OpenClaw Operator → NemoClaw Operator

Status: draft, pending implementation
Branch: `claude/refactor-nemoclaw-instances-pWVFk`

## Context

Today this operator reconciles `OpenClawInstance` CRs in group `openclaw.rocks/v1alpha1` — a single pod running the OpenClaw harness directly, with K8s-native hardening (NetworkPolicy, runAsNonRoot, drop ALL caps, seccomp RuntimeDefault, workspace PVC, Tailscale/Chromium/ingress/backup side-features).

The refactor pivots the operator to manage **NemoClaw** instances. NemoClaw is NVIDIA's secure deployment wrapper: it runs a harness (OpenClaw today, Hermes next) inside a long-lived **OpenShell** gateway plus on-demand **sandbox pods** reconciled by the upstream `kubernetes-sigs/agent-sandbox` controller, with policy/blueprint YAML governing egress, filesystem, process, and inference routing.

This is a greenfield rewrite — no deployed users, no migration, no conversion webhook. The refactor doubles as an opportunity to (a) raise the security bar, (b) drop single-harness assumptions, and (c) rebrand the API group.

## Decisions locked with user

1. **Single CRD with a harness discriminator.** `NemoclawInstance.spec.harness.type` ∈ `{openclaw, hermes}` (extensible). One reconciler, one webhook, one docs surface.
2. **Kitchen-sink spec.** Every current `OpenClawInstanceSpec` field survives the rename. New blocks are additive.
3. **Strict secrets bar.** No inline secrets anywhere in the spec. Only `SecretRef` / `SecretKeyRef`. Projected volumes, mode `0400`, per-instance ServiceAccount, auto-generated + rotated gateway token, audit annotations on every managed Secret.
4. **Greenfield.** Break freely. No deprecation window. Old `OpenClawInstance` CRs are not migrated.
5. **API group rename** `openclaw.rocks` → `skygpt.io`. Finalizer, annotations, labels, group in CRD metadata all move.
6. **Auxiliary CRDs renamed.** `OpenClawSelfConfig` → `NemoclawSelfConfig`, `OpenClawClusterDefaults` → `NemoclawClusterDefaults`.
7. **No nested k3s / DinD.** The operator talks to the real cluster; OpenShell gateway is a regular K8s workload and delegates sandbox creation to the `agents.x-k8s.io/v1alpha1` Sandbox CRD.

## Upstream research — runtime topology

Confirmed by reading `deploy/helm/openshell/templates/` in `NVIDIA/OpenShell`:

| Upstream resource | Shape |
|---|---|
| `statefulset.yaml` | **Single-container** pod; ports `grpc`, `health`, optional `metrics`; 1 Gi PVC at `/var/openshell`; probes on `/healthz`, `/readyz`; TLS via projected Secret; conditional `hostAliases` for `host.docker.internal` / `host.openshell.internal`. Key env: `OPENSHELL_SANDBOX_NAMESPACE`, `OPENSHELL_GRPC_ENDPOINT`, TLS paths, SSH handshake secret. |
| `service.yaml` | Configurable type; ports `grpc` (appProtocol `gRPC`) + optional `metrics`. |
| `networkpolicy.yaml` | Targets pods labeled `openshell.ai/managed-by: openshell` (sandbox pods). Allows ingress **only** from the gateway pod on **TCP 2222 (SSH)**. No egress rules — OpenShell enforces egress in-sandbox (Landlock / seccomp / netns). |
| `role.yaml` | `agents.x-k8s.io` — `sandboxes`, `sandboxes/status` — verbs `create,delete,get,list,patch,update,watch`; core `""` — `events` — verbs `get,list,watch`. **No pod/secret/configmap perms.** |
| `serviceaccount.yaml` + `rolebinding.yaml` | Per-release SA bound to the `-sandbox` Role. |

**Architecture layers:**

```
NemoclawInstance  (our CRD, group skygpt.io/v1alpha1)
  └── OpenShell Gateway StatefulSet         (we create)
        └── Sandbox CRs, agents.x-k8s.io    (gateway creates at runtime)
              └── Sandbox pods              (agent-sandbox controller creates)
                    └── Harness container
                          - OpenClaw + `nemoclaw` npm plugin, or
                          - Hermes with hermes.yaml + persona.yaml
```

**Hard dependency:** `kubernetes-sigs/agent-sandbox` controller must be installed in the cluster. We will document this as a prereq and optionally bundle it in the Helm chart behind a flag.

## Upstream research — NemoClaw plugin

`nemoclaw/openclaw.plugin.json`:

```json
{
  "id": "nemoclaw",
  "name": "NemoClaw",
  "version": "0.1.0",
  "description": "Migrate and run OpenClaw inside OpenShell with optional NIM-backed inference",
  "configSchema": {
    "type": "object",
    "properties": {
      "blueprintVersion":  { "type": "string", "default": "latest" },
      "blueprintRegistry": { "type": "string", "default": "ghcr.io/nvidia/nemoclaw-blueprint" },
      "sandboxName":       { "type": "string", "default": "openclaw" },
      "inferenceProvider": { "type": "string", "default": "nvidia" }
    },
    "additionalProperties": false
  }
}
```

Key facts:

- NemoClaw is shipped as an **OpenClaw plugin** (npm, TypeScript, Node ≥22.16). It runs **inside the harness**, not on the host.
- Subdirs under `nemoclaw/src/`: `blueprint` (OCI pull / version pin), `commands` (slash commands), `lib`, `onboard`, `security` (SSRF and redaction validators).
- NemoClaw also ships a top-level `Dockerfile`, so there is a real image we can reference for default values (not a placeholder).
- For `harness.type: openclaw` the operator ensures `nemoclaw` is present in `spec.plugins` and projects the four plugin config values into the OpenClaw container env.
- For `harness.type: hermes` there is no equivalent plugin; configuration is file-based (`hermes.yaml`, `persona.yaml`) mounted from a ConfigMap.

## API surface (group `skygpt.io/v1alpha1`)

Keep every existing `OpenClawInstanceSpec` field. Rename struct to `NemoclawInstanceSpec`. Drop `Ollama` (replaced by `Nemotron`). Add:

```go
type NemoclawInstanceSpec struct {
    // ... all existing fields, unchanged ...
    Harness         HarnessSpec          `json:"harness"`
    Gateway         GatewaySpec          `json:"gateway,omitempty"`
    Blueprint       *BlueprintSpec       `json:"blueprint,omitempty"`
    SandboxTemplate *SandboxTemplateSpec `json:"sandboxTemplate,omitempty"`
    InferenceRouter *InferenceRouterSpec `json:"inferenceRouter,omitempty"`
    Nemotron        *NemotronSpec        `json:"nemotron,omitempty"`
}

type HarnessSpec struct {
    Type    string    `json:"type"`                        // openclaw | hermes
    Image   ImageSpec `json:"image,omitempty"`             // harness image (OpenClaw or Hermes)
    Plugins []string  `json:"plugins,omitempty"`           // npm plugins (OpenClaw only); auto-includes "nemoclaw"
    Config  RawConfig `json:"config,omitempty"`            // passthrough to harness-specific config file
}

type GatewaySpec struct {
    Image       ImageSpec                   `json:"image,omitempty"`    // defaults to NVIDIA/OpenShell image
    Replicas    *int32                      `json:"replicas,omitempty"` // default 1
    Storage     StorageSpec                 `json:"storage,omitempty"`  // PVC at /var/openshell (default 1Gi)
    TLS         *GatewayTLSSpec             `json:"tls,omitempty"`      // SecretRef only; self-signed issuer optional
    Resources   corev1.ResourceRequirements `json:"resources,omitempty"`
    ExtraEnvRef []EnvFromRef                `json:"extraEnvRef,omitempty"` // SecretRef / ConfigMapRef only
}

type BlueprintSpec struct {
    // Never inline. Either a pinned OCI blueprint or an in-cluster ConfigMap reference.
    OCI          *BlueprintOCIRef             `json:"oci,omitempty"`          // registry + version
    ConfigMapRef *corev1.LocalObjectReference `json:"configMapRef,omitempty"` // mutually exclusive with oci
    Key          string                       `json:"key,omitempty"`          // default "blueprint.yaml"
}

type SandboxTemplateSpec struct {
    // Projected into a ConfigMap the gateway reads when spawning Sandbox CRs.
    Namespace   string                      `json:"namespace,omitempty"`   // defaults to instance namespace
    Isolation   string                      `json:"isolation,omitempty"`   // gvisor | kata | runc (agent-sandbox backend)
    Resources   corev1.ResourceRequirements `json:"resources,omitempty"`
    GPU         *GPUSpec                    `json:"gpu,omitempty"`
    Storage     StorageSpec                 `json:"storage,omitempty"`     // sandbox workspace PVC
    NodeSelector map[string]string          `json:"nodeSelector,omitempty"`
    Tolerations []corev1.Toleration         `json:"tolerations,omitempty"`
}

type InferenceRouterSpec struct {
    Enabled       bool               `json:"enabled,omitempty"`
    Mode          string             `json:"mode,omitempty"`       // local | cloud | auto
    LocalEndpoint string             `json:"localEndpoint,omitempty"`
    CloudProvider *CloudProviderRef  `json:"cloudProvider,omitempty"` // SecretRef for keys only
    PrivacyRules  []PrivacyRule      `json:"privacyRules,omitempty"`
}

type NemotronSpec struct {
    Enabled   bool                        `json:"enabled,omitempty"`
    Image     ImageSpec                   `json:"image,omitempty"`
    Model     string                      `json:"model,omitempty"`
    GPU       *GPUSpec                    `json:"gpu,omitempty"`
    Resources corev1.ResourceRequirements `json:"resources,omitempty"`
    Storage   StorageSpec                 `json:"storage,omitempty"`
}
```

**Strict-secrets invariants** enforced by webhook:

- No `string` fields named `*Token`, `*Key`, `*Password`, `*Secret` anywhere in the spec. Only `SecretKeyRef`.
- `Gateway.ExtraEnvRef` rejects inline env values.
- `CloudProvider.APIKey` is `SecretKeyRef` only.
- Every managed `Secret` is stamped with `skygpt.io/owned-by: <instance>` and `skygpt.io/rotated-at: <ts>`.

## Builders

Greenfield, no harness pod builders to carry over. The target resource set per `NemoclawInstance` is:

| Builder | File | Resource |
|---|---|---|
| `BuildGatewayStatefulSet` | `internal/resources/gateway_statefulset.go` | Single-container pod; ports grpc/health/metrics; PVC at `/var/openshell`; TLS projected; probes on `/healthz` + `/readyz`; runAsNonRoot, drop ALL caps, seccomp RuntimeDefault (the gateway does not need privileges). |
| `BuildGatewayService` | `gateway_service.go` | ClusterIP by default; grpc (appProtocol gRPC) + optional metrics. |
| `BuildGatewayServiceAccount` | `gateway_sa.go` | Per-instance SA, `automountServiceAccountToken: true`. |
| `BuildGatewayRole` | `gateway_rbac.go` | `agents.x-k8s.io/sandboxes{,/status}` CRUD + events read (mirrors upstream). |
| `BuildGatewayRoleBinding` | `gateway_rbac.go` | Binds SA ↔ Role. |
| `BuildBlueprintConfigMap` | `blueprint_configmap.go` | Resolved blueprint YAML, hash in annotation `skygpt.io/blueprint-hash`. |
| `BuildSandboxTemplateConfigMap` | `sandbox_template_configmap.go` | Gateway reads this to construct Sandbox CRs. |
| `BuildGatewayTokenSecret` | existing, reuse | Auto-generated + rotated. |
| `BuildNetworkPolicy` | extend existing | Two policies: (a) gateway ingress allowlist (ingress/auth side), (b) sandbox pods isolation mirroring upstream `openshell.ai/managed-by: openshell` selector on port 2222. |
| `BuildNemotronDeployment` | `nemotron_deployment.go` | Optional NIM-backed local inference; SecretRef for NGC key. |
| Observability, backup, ingress, Tailscale, Chromium, autoupdate | existing, re-target gateway pod | Kitchen-sink preserved. |

The operator does **not** build sandbox pods directly. That is the gateway's job at runtime, delegated to the `agents.x-k8s.io` controller.

## Reconciliation

All resources via `controllerutil.CreateOrUpdate` per CLAUDE.md rules. Owner refs on every managed resource. Status conditions follow `meta.SetStatusCondition`, `ObservedGeneration` tracked. Rollout on blueprint change is triggered by the `skygpt.io/blueprint-hash` annotation on the gateway StatefulSet pod template.

## Webhook

- Reject inline secrets (pattern check on field names + type).
- Reject `BlueprintSpec` with both `OCI` and `ConfigMapRef` set, or neither.
- Reject unknown `harness.type`.
- Reject `sandboxTemplate.isolation` values not supported by installed agent-sandbox backend (lookup at admission time via discovery, best-effort).
- Warn when target namespace lacks `pod-security.kubernetes.io/enforce=restricted` (we expect restricted — the gateway is non-privileged).
- Warn when `agents.x-k8s.io/sandboxes` CRD is absent.

## Files

**Renames (`git mv` to preserve blame):**

- `api/v1alpha1/openclawinstance_types.go` → `nemoclawinstance_types.go`
- `api/v1alpha1/openclawselfconfig_types.go` → `nemoclawselfconfig_types.go`
- `api/v1alpha1/openclawclusterdefaults_types.go` → `nemoclawclusterdefaults_types.go`
- `internal/controller/openclawinstance_controller.go` (+ test) → `nemoclawinstance_controller.go`
- `internal/controller/openclawselfconfig_controller.go` (+ test) → `nemoclawselfconfig_controller.go`
- `internal/webhook/openclawinstance_webhook.go` (+ test) → `nemoclawinstance_webhook.go`
- `config/samples/openclaw_v1alpha1_openclawinstance.yaml` (+ `_full.yaml`) → `skygpt_v1alpha1_nemoclawinstance*.yaml`
- `charts/openclaw-operator/` → `charts/nemoclaw-operator/`
- `bundle/manifests/openclaw-operator.*.clusterserviceversion.yaml` → `nemoclaw-operator.*.clusterserviceversion.yaml`
- `config/crd/bases/openclaw.rocks_*.yaml` (regenerated) → `skygpt.io_*.yaml`

**New files:**

- `api/v1alpha1/harness_types.go`, `gateway_types.go`, `blueprint_types.go`, `sandbox_template_types.go`, `inferencerouter_types.go`, `nemotron_types.go`.
- `internal/resources/{gateway_statefulset,gateway_service,gateway_sa,gateway_rbac,blueprint_configmap,sandbox_template_configmap,nemotron_deployment}.go`.
- `docs/openshell-runtime.md`, `docs/blueprint.md`, `docs/harness-adapters.md`, `docs/agent-sandbox-dependency.md`.

**In-place edits:**

- `PROJECT` (group `skygpt.io`, kinds `Nemoclaw*`).
- `cmd/main.go` (register kinds, leader-election ID renamed).
- `Makefile` (chart sync target paths).
- `.goreleaser.yaml`, `Dockerfile` LABELs, `bundle.Dockerfile`.
- `release-please-config.json` extra-files paths.
- `README.md` full rewrite, `docs/api-reference.md` regen, `CHANGELOG.md`, `ROADMAP.md`.
- All `+kubebuilder:rbac` markers — verify no unused perms after builder changes.
- All e2e fixture YAMLs in `test/e2e/` (kinds + apiVersion).
- Every `openclaw.rocks/` annotation / label / finalizer string (codebase-wide `sed` with review).

**Module + image:**

- Module path `github.com/openclawrocks/openclaw-operator` → `github.com/skygpt/nemoclaw-operator` (deferred to a separate cosmetic commit inside this same branch).
- Image `ghcr.io/openclaw-rocks/openclaw-operator` → `ghcr.io/skygpt/nemoclaw-operator`.
- Finalizer `openclaw.rocks/finalizer` → `skygpt.io/finalizer`.

## Phasing (one commit per phase)

1. **group-rename** — sweep `openclaw.rocks` → `skygpt.io` everywhere (finalizer, labels, annotations, RBAC markers, CRD group). Regenerate. No behavioral change.
2. **types-rename** — `git mv` API + controller + webhook files; rename Go types (`OpenClawInstance*` → `NemoclawInstance*`); regenerate deepcopy + manifests.
3. **api-additions** — split new spec blocks across `harness_types.go`, `gateway_types.go`, `blueprint_types.go`, `sandbox_template_types.go`, `inferencerouter_types.go`, `nemotron_types.go`; remove `OllamaSpec`; regenerate.
4. **builders** — new gateway/rbac/configmap builders; unit tests in `internal/resources/resources_test.go`.
5. **controller** — wire gateway-first reconciliation (no sandbox pods built by operator); blueprint resolution (OCI fetch or ConfigMap ref) with hash annotation for rollout; status conditions.
6. **webhook** — strict-secrets invariant check; blueprint mutual-exclusion; harness.type enum; isolation discovery warning.
7. **helm/bundle/samples** — rename chart dir, Chart.yaml/values.yaml/templates; rewrite CSV; rewrite sample CRs under `config/samples/`.
8. **docs** — README rewrite, api-reference regen, runbooks under `docs/runbooks/nemoclaw-*.md`, CHANGELOG breaking-change banner, `agent-sandbox-dependency.md`.
9. **module-path** — rename module path + image; fix all imports; update goreleaser, Dockerfiles, CI; retag.

## Verification

```bash
make generate          # regenerate zz_generated.deepcopy.go
make manifests         # regenerate config/crd/bases/*.yaml under new group
make sync-chart-crds   # propagate CRDs into charts/nemoclaw-operator/templates/crds/
go vet ./...
make lint              # golangci-lint v1.64.5
go test ./internal/resources/ -v   # fast unit tests
make test              # full unit + envtest integration
make test-e2e          # kind cluster; requires agent-sandbox controller installed first
```

E2E smoke test (`test/e2e/e2e_nemoclaw_smoke_test.go`):

1. Install `kubernetes-sigs/agent-sandbox` CRDs + controller into kind.
2. Apply `config/samples/skygpt_v1alpha1_nemoclawinstance.yaml` with `harness.type: openclaw` and an in-cluster ConfigMap blueprint.
3. Wait for the gateway StatefulSet pod Ready; probes on `/healthz` + `/readyz` green.
4. Confirm blueprint ConfigMap is mounted at `/etc/nemoclaw/blueprint.yaml` and the pod annotation `skygpt.io/blueprint-hash` matches.
5. Confirm gateway Role grants `agents.x-k8s.io/sandboxes` CRUD.
6. Trigger a sandbox creation (via gateway gRPC) and assert a `Sandbox` CR appears, then its pod, then the harness container Ready.
7. Assert the sandbox-pod NetworkPolicy: ingress-only from gateway on TCP 2222.
8. Rotate the gateway token Secret; confirm the StatefulSet rolls.
9. Delete the `NemoclawInstance` and confirm GC of the gateway StatefulSet, Service, SA, Role, RoleBinding, NetworkPolicies, ConfigMaps, Secrets, and the Sandbox CRs.

## Out of scope (follow-up PRs)

- Bundling the `kubernetes-sigs/agent-sandbox` controller inside our Helm chart (initial release documents it as a prereq).
- Hermes harness end-to-end (types + reconciler paths land in this refactor; Hermes image + e2e ships in a follow-up).
- Inference-router full policy enforcement (initial release passes config through; enforcement lives in the NemoClaw plugin).
- Conversion webhook from old `OpenClawInstance` (none — greenfield break).
- Backup/restore awareness of sandbox workspace PVCs (initial release covers gateway PVC only).
- Warm pool / `SandboxWarmPool` integration.

## Risks / open questions

- **Agent-sandbox controller maturity.** CRD is `v1alpha1`; API may shift. Pin a known-good commit in docs, add CI job that installs + asserts CRD shape.
- **Isolation backend availability.** gVisor / Kata require node-level prereqs. Webhook only warns; runbook covers node setup.
- **Gateway image stability.** Upstream NVIDIA/OpenShell image tags; pin in `values.yaml`, watch for breaking helm chart changes.
- **NemoClaw plugin publishing.** The `nemoclaw` npm package must be published for `spec.plugins` auto-injection to work; confirm release pipeline before GA.
