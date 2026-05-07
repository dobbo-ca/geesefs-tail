# geesefs-tail Helm Chart Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a Helm chart that renders an on-demand privileged Pod which streams a chosen AKS node's geesefs systemd journal logs to `kubectl logs`.

**Architecture:** Single `Pod` resource pinned to a user-supplied node, `hostPID: true` + `privileged: true`, container starts with `apk add --no-cache util-linux` then `nsenter -t 1` into PID 1 to run `journalctl --since … -fu <glob>`. Optional preambles (discovered units, journal header) render conditionally from values. Optional `Namespace` resource carries PSA `privileged` enforce label.

**Tech Stack:** Helm 3, alpine:3.20 image, `nsenter` from util-linux, `journalctl` (host's), helm-unittest for template tests, kubeconform for schema validation.

**Spec:** `docs/superpowers/specs/2026-05-07-geesefs-log-tail-design.md`

---

## File Structure

Files this plan creates:

| File | Purpose |
|------|---------|
| `README.md` | Install/use/uninstall/troubleshoot docs |
| `.gitignore` | Ignore helm cache, OS junk |
| `Makefile` | `make lint`, `make template`, `make test`, `make ci` |
| `chart/Chart.yaml` | Helm chart metadata |
| `chart/values.yaml` | All tunable inputs with documented defaults |
| `chart/templates/pod.yaml` | The Pod resource |
| `chart/templates/namespace.yaml` | Optional Namespace with PSA labels (gated by value) |
| `chart/tests/pod_test.yaml` | helm-unittest cases for pod.yaml |
| `chart/tests/namespace_test.yaml` | helm-unittest cases for namespace.yaml |

The pod template is the heart of the chart and grows incrementally across Tasks 2–7. Each task shows the full updated `pod.yaml` so it can be read in isolation.

---

## Task 1: Bootstrap repo scaffolding & test tooling

**Files:**
- Create: `README.md`
- Create: `.gitignore`
- Create: `Makefile`
- Create: `chart/Chart.yaml`
- Create: `chart/values.yaml`
- Create: `chart/templates/.gitkeep`
- Create: `chart/tests/.gitkeep`

- [ ] **Step 1: Create `.gitignore`**

```
# Helm
chart/charts/
chart/Chart.lock
*.tgz

# OS
.DS_Store
```

- [ ] **Step 2: Create `chart/Chart.yaml`**

```yaml
apiVersion: v2
name: geesefs-tail
description: On-demand privileged pod that streams a host AKS node's geesefs systemd journal logs to kubectl logs.
type: application
version: 0.1.0
appVersion: "0.1.0"
kubeVersion: ">=1.27.0-0"
home: https://github.com/dobbo-ca/geesefs-tail
sources:
  - https://github.com/dobbo-ca/geesefs-tail
keywords:
  - geesefs
  - debugging
  - systemd
  - journalctl
maintainers:
  - name: dobbo-ca
```

- [ ] **Step 3: Create `chart/values.yaml`**

```yaml
# Target node. Required unless nodeSelector is set. Mutually exclusive with nodeSelector.
nodeName: ""

# Alternative to nodeName. Mutually exclusive.
nodeSelector: {}

# How far back to dump host journal before live tail begins.
sinceHours: 24

# systemd unit glob passed to `journalctl -u` and `systemctl list-units`.
unitGlob: "geesefs-*"

# Print discovered units at startup so an empty match doesn't look like a hang.
showDiscoveredUnits: true

# Print journalctl --header for storage-mode diagnosis (verbose, default off).
showJournalHeader: false

# Auto-terminate the pod after this many seconds. 0 disables.
activeDeadlineSeconds: 3600

# Where the pod runs. If createNamespace is true the chart manages this namespace
# with a PSA `privileged` enforce label.
namespace: geesefs-debug
createNamespace: true

# Pod metadata.name.
podName: geesefs-tail

image:
  repository: alpine
  tag: "3.20"
  pullPolicy: IfNotPresent
```

- [ ] **Step 4: Create `chart/templates/.gitkeep` and `chart/tests/.gitkeep`**

Both files are empty. They keep the directories tracked before real content lands.

- [ ] **Step 5: Create `Makefile`**

```makefile
CHART := chart
RELEASE := geesefs-tail
NS := geesefs-debug
NODE ?= dummy-node

.PHONY: lint template test kubeconform ci

lint:
	helm lint $(CHART)

template:
	helm template $(RELEASE) $(CHART) --set nodeName=$(NODE)

test:
	helm unittest $(CHART)

kubeconform:
	helm template $(RELEASE) $(CHART) --set nodeName=$(NODE) | kubeconform -strict -summary -

ci: lint test kubeconform
```

- [ ] **Step 6: Create `README.md` skeleton**

```markdown
# geesefs-tail

On-demand Helm chart that streams a chosen AKS node's geesefs systemd journal logs into `kubectl logs`. Useful for diagnosing geesefs S3 mounter issues on the byoc cluster without the manual privileged-debug-pod + chroot dance.

## Status

WIP. Full docs land in Task 9 of the implementation plan.
```

- [ ] **Step 7: Verify scaffolding**

Run:
```
cd ~/work/dobbo-ca/geesefs-tail
helm lint chart
```

Expected output contains `1 chart(s) linted, 0 chart(s) failed`. (Lint will warn about no templates; that's fine at this stage.)

- [ ] **Step 8: Install helm-unittest plugin (one-time per machine)**

Run:
```
helm plugin install https://github.com/helm-unittest/helm-unittest 2>&1 || helm plugin update unittest
```

Verify with `helm unittest --help`.

- [ ] **Step 9: Commit**

```
cd ~/work/dobbo-ca/geesefs-tail
git add .gitignore Makefile README.md chart
git commit -m "chore: scaffold helm chart structure and test tooling"
```

---

## Task 2: Pod template — minimal valid Pod (TDD)

**Files:**
- Create: `chart/tests/pod_test.yaml`
- Create: `chart/templates/pod.yaml`

- [ ] **Step 1: Write failing test**

`chart/tests/pod_test.yaml`:

```yaml
suite: pod template
templates:
  - templates/pod.yaml
tests:
  - it: requires nodeName when nodeSelector is empty
    set:
      nodeName: ""
      nodeSelector: {}
    asserts:
      - failedTemplate:
          errorMessage: "nodeName or nodeSelector is required"

  - it: rejects both nodeName and nodeSelector being set
    set:
      nodeName: aks-test-1
      nodeSelector:
        kubernetes.io/hostname: aks-test-1
    asserts:
      - failedTemplate:
          errorMessage: "nodeName and nodeSelector are mutually exclusive; set exactly one"

  - it: renders a Pod with the configured name in the configured namespace
    set:
      nodeName: aks-test-1
      podName: geesefs-tail
      namespace: geesefs-debug
    asserts:
      - hasDocuments:
          count: 1
      - isKind:
          of: Pod
      - equal:
          path: metadata.name
          value: geesefs-tail
      - equal:
          path: metadata.namespace
          value: geesefs-debug
```

- [ ] **Step 2: Run tests to verify failure**

Run: `helm unittest chart`
Expected: `templates/pod.yaml` not found, all assertions fail.

- [ ] **Step 3: Create `chart/templates/pod.yaml`**

```yaml
{{- if and (not .Values.nodeName) (not .Values.nodeSelector) -}}
{{- fail "nodeName or nodeSelector is required" -}}
{{- end -}}
{{- if and .Values.nodeName .Values.nodeSelector -}}
{{- fail "nodeName and nodeSelector are mutually exclusive; set exactly one" -}}
{{- end -}}
apiVersion: v1
kind: Pod
metadata:
  name: {{ .Values.podName | quote }}
  namespace: {{ .Values.namespace | quote }}
spec:
  containers:
    - name: tail
      image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
      command: ["sleep", "1"]
```

The `command: sleep 1` is a placeholder so the template is structurally valid. Task 6 replaces it.

- [ ] **Step 4: Run tests to verify pass**

Run: `helm unittest chart`
Expected: both tests pass.

- [ ] **Step 5: Commit**

```
git add chart/templates/pod.yaml chart/tests/pod_test.yaml
git commit -m "feat(chart): minimal Pod template with node selection validation"
```

---

## Task 3: Pod template — security + lifecycle settings

**Files:**
- Modify: `chart/templates/pod.yaml`
- Modify: `chart/tests/pod_test.yaml`

- [ ] **Step 1: Add failing tests**

Append to `chart/tests/pod_test.yaml` under `tests:`:

```yaml
  - it: sets hostPID, restartPolicy, tolerations, and privileged container
    set:
      nodeName: aks-test-1
    asserts:
      - equal:
          path: spec.hostPID
          value: true
      - equal:
          path: spec.restartPolicy
          value: Never
      - contains:
          path: spec.tolerations
          content:
            operator: Exists
      - equal:
          path: spec.containers[0].securityContext.privileged
          value: true

  - it: applies activeDeadlineSeconds when non-zero
    set:
      nodeName: aks-test-1
      activeDeadlineSeconds: 600
    asserts:
      - equal:
          path: spec.activeDeadlineSeconds
          value: 600

  - it: omits activeDeadlineSeconds when zero
    set:
      nodeName: aks-test-1
      activeDeadlineSeconds: 0
    asserts:
      - notExists:
          path: spec.activeDeadlineSeconds

  - it: uses nodeSelector when nodeName is empty
    set:
      nodeName: ""
      nodeSelector:
        kubernetes.io/hostname: aks-test-1
    asserts:
      - notExists:
          path: spec.nodeName
      - equal:
          path: spec.nodeSelector["kubernetes.io/hostname"]
          value: aks-test-1
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `helm unittest chart`
Expected: the four new tests fail (fields not yet rendered).

- [ ] **Step 3: Update `chart/templates/pod.yaml`**

Full file:

```yaml
{{- if and (not .Values.nodeName) (not .Values.nodeSelector) -}}
{{- fail "nodeName or nodeSelector is required" -}}
{{- end -}}
{{- if and .Values.nodeName .Values.nodeSelector -}}
{{- fail "nodeName and nodeSelector are mutually exclusive; set exactly one" -}}
{{- end -}}
apiVersion: v1
kind: Pod
metadata:
  name: {{ .Values.podName | quote }}
  namespace: {{ .Values.namespace | quote }}
spec:
  hostPID: true
  restartPolicy: Never
  {{- if .Values.nodeName }}
  nodeName: {{ .Values.nodeName | quote }}
  {{- end }}
  {{- if .Values.nodeSelector }}
  nodeSelector:
    {{- toYaml .Values.nodeSelector | nindent 4 }}
  {{- end }}
  tolerations:
    - operator: Exists
  {{- if gt (.Values.activeDeadlineSeconds | int) 0 }}
  activeDeadlineSeconds: {{ .Values.activeDeadlineSeconds | int }}
  {{- end }}
  containers:
    - name: tail
      image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
      imagePullPolicy: {{ .Values.image.pullPolicy }}
      securityContext:
        privileged: true
      command: ["sleep", "1"]
```

- [ ] **Step 4: Run tests to verify pass**

Run: `helm unittest chart`
Expected: all tests pass.

- [ ] **Step 5: Commit**

```
git add chart/templates/pod.yaml chart/tests/pod_test.yaml
git commit -m "feat(chart): host PID, privileged container, lifecycle settings"
```

---

## Task 4: Pod template — startup script with nsenter + journalctl

**Files:**
- Modify: `chart/templates/pod.yaml`
- Modify: `chart/tests/pod_test.yaml`

- [ ] **Step 1: Add failing tests**

Append to `chart/tests/pod_test.yaml`:

```yaml
  - it: container command runs apk add util-linux then exec nsenter journalctl
    set:
      nodeName: aks-test-1
      sinceHours: 24
      unitGlob: "geesefs-*"
      showDiscoveredUnits: false
      showJournalHeader: false
    asserts:
      - equal:
          path: spec.containers[0].command[0]
          value: /bin/sh
      - equal:
          path: spec.containers[0].command[1]
          value: -c
      - matchRegex:
          path: spec.containers[0].command[2]
          pattern: "apk add --no-cache util-linux"
      - matchRegex:
          path: spec.containers[0].command[2]
          pattern: 'exec nsenter -t 1 -m -u -i -n -p -- journalctl --since "24 hours ago" -f --unit "geesefs-\*"'

  - it: respects custom sinceHours and unitGlob
    set:
      nodeName: aks-test-1
      sinceHours: 6
      unitGlob: "kubelet"
      showDiscoveredUnits: false
      showJournalHeader: false
    asserts:
      - matchRegex:
          path: spec.containers[0].command[2]
          pattern: '--since "6 hours ago"'
      - matchRegex:
          path: spec.containers[0].command[2]
          pattern: '--unit "kubelet"'
```

- [ ] **Step 2: Run tests to verify failure**

Run: `helm unittest chart`
Expected: the two new tests fail (command is still `sleep 1`).

- [ ] **Step 3: Update `chart/templates/pod.yaml` to render the startup script**

Replace the `containers:` section with:

```yaml
  containers:
    - name: tail
      image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
      imagePullPolicy: {{ .Values.image.pullPolicy }}
      securityContext:
        privileged: true
      command:
        - /bin/sh
        - -c
        - |
          set -eu
          apk add --no-cache util-linux >/dev/null
          {{- if .Values.showJournalHeader }}
          echo "--- journal storage header ---"
          nsenter -t 1 -m -u -i -n -p -- journalctl --header | head
          echo "--- end header ---"
          {{- end }}
          {{- if .Values.showDiscoveredUnits }}
          echo "--- discovered units matching {{ .Values.unitGlob }} ---"
          units=$(nsenter -t 1 -m -u -i -n -p -- systemctl list-units --no-legend {{ .Values.unitGlob | quote }} || true)
          if [ -z "$units" ]; then
            echo "no matching units for {{ .Values.unitGlob }}"
          else
            printf '%s\n' "$units"
          fi
          echo "--- end units ---"
          {{- end }}
          exec nsenter -t 1 -m -u -i -n -p -- journalctl --since "{{ .Values.sinceHours }} hours ago" -f --unit "{{ .Values.unitGlob }}"
```

The full `pod.yaml` after this step:

```yaml
{{- if and (not .Values.nodeName) (not .Values.nodeSelector) -}}
{{- fail "nodeName or nodeSelector is required" -}}
{{- end -}}
{{- if and .Values.nodeName .Values.nodeSelector -}}
{{- fail "nodeName and nodeSelector are mutually exclusive; set exactly one" -}}
{{- end -}}
apiVersion: v1
kind: Pod
metadata:
  name: {{ .Values.podName | quote }}
  namespace: {{ .Values.namespace | quote }}
spec:
  hostPID: true
  restartPolicy: Never
  {{- if .Values.nodeName }}
  nodeName: {{ .Values.nodeName | quote }}
  {{- end }}
  {{- if .Values.nodeSelector }}
  nodeSelector:
    {{- toYaml .Values.nodeSelector | nindent 4 }}
  {{- end }}
  tolerations:
    - operator: Exists
  {{- if gt (.Values.activeDeadlineSeconds | int) 0 }}
  activeDeadlineSeconds: {{ .Values.activeDeadlineSeconds | int }}
  {{- end }}
  containers:
    - name: tail
      image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
      imagePullPolicy: {{ .Values.image.pullPolicy }}
      securityContext:
        privileged: true
      command:
        - /bin/sh
        - -c
        - |
          set -eu
          apk add --no-cache util-linux >/dev/null
          {{- if .Values.showJournalHeader }}
          echo "--- journal storage header ---"
          nsenter -t 1 -m -u -i -n -p -- journalctl --header | head
          echo "--- end header ---"
          {{- end }}
          {{- if .Values.showDiscoveredUnits }}
          echo "--- discovered units matching {{ .Values.unitGlob }} ---"
          units=$(nsenter -t 1 -m -u -i -n -p -- systemctl list-units --no-legend {{ .Values.unitGlob | quote }} || true)
          if [ -z "$units" ]; then
            echo "no matching units for {{ .Values.unitGlob }}"
          else
            printf '%s\n' "$units"
          fi
          echo "--- end units ---"
          {{- end }}
          exec nsenter -t 1 -m -u -i -n -p -- journalctl --since "{{ .Values.sinceHours }} hours ago" -f --unit "{{ .Values.unitGlob }}"
```

- [ ] **Step 4: Run tests to verify pass**

Run: `helm unittest chart`
Expected: all tests pass.

- [ ] **Step 5: Commit**

```
git add chart/templates/pod.yaml chart/tests/pod_test.yaml
git commit -m "feat(chart): startup script runs nsenter + journalctl follow"
```

---

## Task 5: Pod template — preamble toggles (discovered units, journal header)

**Files:**
- Modify: `chart/tests/pod_test.yaml`

The preamble logic itself was rendered in Task 4 (gated by `showDiscoveredUnits` and `showJournalHeader`). This task locks the toggles down with tests so future edits can't silently regress them.

- [ ] **Step 1: Add failing tests**

Append to `chart/tests/pod_test.yaml`:

```yaml
  - it: showDiscoveredUnits=false omits the discovered-units preamble
    set:
      nodeName: aks-test-1
      showDiscoveredUnits: false
      showJournalHeader: false
    asserts:
      - notMatchRegex:
          path: spec.containers[0].command[2]
          pattern: "discovered units matching"

  - it: showDiscoveredUnits=true includes the empty-units fallback line
    set:
      nodeName: aks-test-1
      showDiscoveredUnits: true
      unitGlob: "geesefs-*"
    asserts:
      - matchRegex:
          path: spec.containers[0].command[2]
          pattern: "no matching units for geesefs-\\*"

  - it: showJournalHeader=true includes journalctl --header
    set:
      nodeName: aks-test-1
      showJournalHeader: true
    asserts:
      - matchRegex:
          path: spec.containers[0].command[2]
          pattern: "journalctl --header"

  - it: showJournalHeader=false omits journalctl --header
    set:
      nodeName: aks-test-1
      showJournalHeader: false
    asserts:
      - notMatchRegex:
          path: spec.containers[0].command[2]
          pattern: "journalctl --header"
```

- [ ] **Step 2: Run tests**

Run: `helm unittest chart`
Expected: all tests pass on first run (template already supports these toggles from Task 4).

If a test fails, it indicates a real bug in Task 4's template — fix `chart/templates/pod.yaml` rather than weakening the test.

- [ ] **Step 3: Commit**

```
git add chart/tests/pod_test.yaml
git commit -m "test(chart): lock down preamble toggles"
```

---

## Task 6: Namespace template with PSA labels

**Files:**
- Create: `chart/templates/namespace.yaml`
- Create: `chart/tests/namespace_test.yaml`

- [ ] **Step 1: Write failing tests**

`chart/tests/namespace_test.yaml`:

```yaml
suite: namespace template
templates:
  - templates/namespace.yaml
tests:
  - it: renders nothing when createNamespace is false
    set:
      createNamespace: false
      namespace: geesefs-debug
      nodeName: aks-test-1
    asserts:
      - hasDocuments:
          count: 0

  - it: renders a Namespace with PSA privileged labels when createNamespace is true
    set:
      createNamespace: true
      namespace: geesefs-debug
      nodeName: aks-test-1
    asserts:
      - hasDocuments:
          count: 1
      - isKind:
          of: Namespace
      - equal:
          path: metadata.name
          value: geesefs-debug
      - equal:
          path: metadata.labels["pod-security.kubernetes.io/enforce"]
          value: privileged
      - equal:
          path: metadata.labels["pod-security.kubernetes.io/audit"]
          value: privileged
      - equal:
          path: metadata.labels["pod-security.kubernetes.io/warn"]
          value: privileged
```

- [ ] **Step 2: Run tests to verify failure**

Run: `helm unittest chart`
Expected: namespace tests fail because the template doesn't exist.

- [ ] **Step 3: Create `chart/templates/namespace.yaml`**

```yaml
{{- if .Values.createNamespace -}}
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .Values.namespace | quote }}
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
{{- end -}}
```

- [ ] **Step 4: Run tests to verify pass**

Run: `helm unittest chart`
Expected: all tests pass.

- [ ] **Step 5: Commit**

```
git add chart/templates/namespace.yaml chart/tests/namespace_test.yaml
git commit -m "feat(chart): optional namespace with PSA privileged labels"
```

---

## Task 7: Validate full render against Kubernetes schema

**Files:** none (verification only)

- [ ] **Step 1: Install kubeconform if missing**

Run:
```
which kubeconform || brew install kubeconform
```

- [ ] **Step 2: Render and validate with both nodeName and createNamespace true**

Run:
```
make kubeconform NODE=aks-real-node
```

Expected output: `Summary: ... 0 invalid, 0 errors`. If `apiVersion: v2` chart issues surface, fix the relevant template inline.

- [ ] **Step 3: Render and validate with nodeSelector instead**

Run:
```
helm template geesefs-tail chart \
  --set 'nodeSelector.kubernetes.io/hostname=aks-real-node' \
  --set createNamespace=false \
  | kubeconform -strict -summary -
```

Expected: clean summary, 0 invalid.

- [ ] **Step 4: Confirm `make ci` passes end-to-end**

Run: `make ci`
Expected: lint clean, all unittests green, kubeconform clean.

- [ ] **Step 5: Commit (only if Steps 1-4 surfaced any fixes)**

If everything passed without changes, skip commit. Otherwise:

```
git add -p
git commit -m "fix(chart): resolve issues found by kubeconform validation"
```

---

## Task 8: README — full usage and troubleshooting

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Replace `README.md` content**

```markdown
# geesefs-tail

On-demand Helm chart that streams a chosen AKS node's geesefs systemd journal logs into `kubectl logs`. Built for the byoc cluster, where geesefs runs as a host-level systemd unit and its logs do not flow into the cluster log pipeline by default.

## What it does

`helm install` renders a single privileged `Pod` pinned to the node you pick. The pod uses `hostPID: true` and `nsenter` to step into PID 1 on the host, then runs `journalctl --since "<N> hours ago" -fu '<glob>'` against the host journal. Output goes to container stdout, so `kubectl logs -f` (and the cluster log pipeline) picks it up.

## Requirements

- Helm 3.12+
- Cluster admin (or equivalent) on the target cluster
- `kubeconform` and `helm-unittest` only if you want to run the test suite locally

## Install

Pin to a specific node (most common):

```sh
helm install geesefs-tail ./chart \
  --create-namespace \
  --namespace geesefs-debug \
  --set nodeName=aks-n261fc4201a7-31527110-vmss000000
```

Or select via labels:

```sh
helm install geesefs-tail ./chart \
  --create-namespace \
  --namespace geesefs-debug \
  --set 'nodeSelector.kubernetes.io/hostname=aks-n261fc4201a7-31527110-vmss000000'
```

The chart manages the namespace by default (`createNamespace=true`) and applies a Pod Security Admission `privileged` enforce label to it. If you already have a namespace ready, set `createNamespace=false` and `--namespace <yours>`.

## Watch logs

```sh
kubectl -n geesefs-debug logs -f geesefs-tail
```

You'll see, in order:
1. `apk add` output (silenced).
2. (optional) `journalctl --header` block — enable with `--set showJournalHeader=true`.
3. (optional) Discovered units list, or `no matching units for <glob>` if the glob hits nothing.
4. The journal: last 24 hours of history, then live tail.

## Uninstall

```sh
helm uninstall geesefs-tail -n geesefs-debug
```

The pod also self-terminates after `activeDeadlineSeconds` (default 3600s) as a forgotten-install backstop. Set to `0` to disable.

## Values

| Key | Default | Description |
|-----|---------|-------------|
| `nodeName` | `""` | Target node. Required unless `nodeSelector` is set. |
| `nodeSelector` | `{}` | Map of labels to select the target node. Mutually exclusive with `nodeName`. |
| `sinceHours` | `24` | How far back to dump host journal before live tail. |
| `unitGlob` | `"geesefs-*"` | Passed to `journalctl -u` and `systemctl list-units`. |
| `showDiscoveredUnits` | `true` | Print discovered units at startup. |
| `showJournalHeader` | `false` | Print `journalctl --header` for storage diagnosis. |
| `activeDeadlineSeconds` | `3600` | Auto-terminate after N seconds. `0` disables. |
| `namespace` | `geesefs-debug` | Where the pod runs. |
| `createNamespace` | `true` | Whether the chart creates the namespace with PSA labels. |
| `podName` | `geesefs-tail` | `metadata.name` of the rendered pod. |
| `image.repository` | `alpine` | Container image. `nsenter` is installed at startup via `apk`. |
| `image.tag` | `"3.20"` | |
| `image.pullPolicy` | `IfNotPresent` | |

## Troubleshooting

**Pod stuck in Pending.** Check `kubectl describe pod -n geesefs-debug geesefs-tail`. Most likely a wrong `nodeName` (`0/N nodes are available`) or PSA enforcement on the namespace blocking `privileged`. If you're using your own namespace, ensure it carries the `pod-security.kubernetes.io/enforce: privileged` label.

**Logs are silent (no entries).** Check the discovered-units preamble — if it says "no matching units", your `unitGlob` is wrong. Set `--set unitGlob='*your-pattern*'`. If discovered units look right but no history shows, set `--set showJournalHeader=true` and confirm the host has persistent journal storage.

**`nsenter: Operation not permitted`.** The pod isn't running privileged. Confirm you didn't override `securityContext` and that the namespace's PSA label permits privileged pods.

## Security note

This chart deploys a privileged pod with `hostPID` access. It can read everything the host's PID 1 can read. Treat it like SSH-as-root on the node. Uninstall when done.

## Local development

```sh
make lint            # helm lint
make test            # helm unittest
make template        # render with NODE=dummy-node
make kubeconform     # render + kubeconform validate
make ci              # all of the above
```

helm-unittest plugin: `helm plugin install https://github.com/helm-unittest/helm-unittest`
kubeconform: `brew install kubeconform`
```

- [ ] **Step 2: Sanity-check rendered markdown**

Open `README.md` in a viewer (or `glow README.md` if installed) and skim for broken tables/code fences.

- [ ] **Step 3: Commit**

```
git add README.md
git commit -m "docs: full install, usage, values, and troubleshooting in README"
```

---

## Task 9: Smoke test on a real cluster

**Files:** none (manual verification)

This task is the spec's "Live smoke test" requirement. It cannot be automated and must run against a real AKS cluster with geesefs deployed.

- [ ] **Step 1: Pick a node**

```
kubectl get nodes -o wide
```

Pick one running geesefs. Note its name.

- [ ] **Step 2: Install**

```
helm install geesefs-tail ./chart \
  --create-namespace \
  --namespace geesefs-debug \
  --set nodeName=<that-node>
```

- [ ] **Step 3: Confirm logs flow**

```
kubectl -n geesefs-debug logs -f geesefs-tail
```

Expected within 30s:
- "discovered units matching geesefs-*" header
- One or more `geesefs-*.service` lines under it
- 24h of historical journal entries
- Live entries continuing after the dump

- [ ] **Step 4: Verify live tail**

In another terminal, on the same node via the existing manual privileged-debug-pod flow, run `systemctl restart <one-of-the-discovered-units>`. Within ~5s the restart entry should appear in the streaming `kubectl logs` output.

- [ ] **Step 5: Verify negative cases**

Wrong glob:
```
helm upgrade geesefs-tail ./chart --reuse-values --set unitGlob='nope-no-such-*'
```

Confirm logs show: `no matching units for nope-no-such-*`. (Note: `helm upgrade` recreates the pod since `restartPolicy: Never` plus an immutable Pod resource means helm will fail or ask for `--force`. Easier path: `helm uninstall && helm install` with the new value.)

activeDeadline cutoff:
```
helm uninstall geesefs-tail -n geesefs-debug
helm install geesefs-tail ./chart \
  --namespace geesefs-debug \
  --set nodeName=<that-node> \
  --set activeDeadlineSeconds=60
```

Wait 60s, confirm pod transitions to `Failed` with reason `DeadlineExceeded`.

- [ ] **Step 6: Cleanup**

```
helm uninstall geesefs-tail -n geesefs-debug
kubectl delete namespace geesefs-debug
```

- [ ] **Step 7: Note results**

If everything passed, no further commit. If issues surfaced, file them as separate issues against the repo and fix in dedicated commits.

---

## Self-Review Notes

(To be completed after writing — see Self-Review section of writing-plans skill.)
