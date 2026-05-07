# geesefs Host Log Tail — Design

**Date:** 2026-05-07
**Status:** Approved (pending implementation plan)

## Problem

geesefs runs as a host-level systemd unit on AKS nodes (not as a pod), so its logs do not flow into the cluster log pipeline. Today, surfacing geesefs logs requires a manual sequence: apply a privileged debug pod manifest, exec in, `chroot /host`, find the dynamic systemd unit name with `systemctl | grep geesefs`, then `systemctl status` or `journalctl` against it. This is slow, error-prone, and produces output only inside the debug pod's interactive shell.

## Goal

Provide a one-command, on-demand way to stream geesefs systemd logs from a chosen node into `kubectl logs`, so logs flow through the existing cluster log pipeline (Azure Monitor / Log Analytics) and are viewable, greppable, and shareable like any other pod log.

Non-goals:
- Running continuously (this is a diagnostic tool, not a daemon).
- Centralized log shipping infrastructure.
- Anything beyond geesefs.

## Approach

Helm chart that renders a single privileged `Pod` pinned to a user-chosen node. The pod uses `hostPID: true` and `nsenter` into PID 1 to run `journalctl` against the host's systemd journal. journalctl streams the last N hours of history, then follows live, all to container stdout — picked up by `kubectl logs -f` and the cluster log pipeline.

Selected over two alternatives:
- **Privileged pod + chroot** (current manual flow): more moving parts (host root mount, chroot, awk pipeline to discover units).
- **hostPath read of `/var/log/journal`**: avoids privilege but requires persistent journal enabled and journalctl args that don't follow as cleanly; no real win for a tool that's already a privileged debug aid.

## Architecture

One Helm chart, one resource (`Pod`). Lifecycle:

1. `helm install geesefs-tail ./chart --set nodeName=<aks-node>`
2. `kubectl logs -f geesefs-tail`
3. `helm uninstall geesefs-tail`

Pod spec essentials:
- `hostPID: true`
- `nodeName` (or `nodeSelector`) from values
- `securityContext.privileged: true`
- `restartPolicy: Never`
- `tolerations: [{operator: Exists}]`
- `activeDeadlineSeconds` configurable (default 3600s; 0 disables) as a safety net for forgotten installs

Container:
- Image: `alpine:3.20`
- Startup: `apk add --no-cache util-linux >/dev/null` (installs `nsenter`)
- Exec:
  ```
  nsenter -t 1 -m -u -i -n -p -- \
    journalctl --since "${SINCE}" -f --unit "${UNIT_GLOB}"
  ```
- Optional preamble (gated by values):
  - Discovered units (`showDiscoveredUnits`, default on): runs `nsenter ... systemctl list-units --no-legend "${UNIT_GLOB}"`. If output is empty, the startup script must emit an explicit `no matching units for ${UNIT_GLOB}` line so the stream is not a silent hang.
  - Journal storage header (`showJournalHeader`, default off): `nsenter ... journalctl --header | head` for diagnosing storage mode.

## Helm chart layout

```
geesefs-tail/
├── Chart.yaml
├── values.yaml
└── templates/
    └── pod.yaml
```

`values.yaml`:

```yaml
nodeName: ""                  # required unless nodeSelector set
nodeSelector: {}              # alternative to nodeName; mutually exclusive
sinceHours: 24
unitGlob: "geesefs-*"
showDiscoveredUnits: true
showJournalHeader: false
activeDeadlineSeconds: 3600   # 0 disables
namespace: geesefs-debug
podName: geesefs-tail
image:
  repository: alpine
  tag: "3.20"
  pullPolicy: IfNotPresent
```

Template behavior:
- Fail rendering if both `nodeName` and `nodeSelector` are empty (`{{ required ... }}`).
- Fail rendering if both are set (mutually exclusive).
- `sinceHours` is rendered into the journalctl `--since "<N> hours ago"` argument (the `${SINCE}` placeholder shown in the container exec snippet).

## Security & RBAC

- `privileged: true` + `hostPID: true` are required for `nsenter -t 1`. Without them, nsenter fails with `Operation not permitted`.
- Default namespace is `geesefs-debug` (overridable) so a Pod Security Admission label of `pod-security.kubernetes.io/enforce: privileged` can be scoped narrowly. Production namespaces remain restricted.
- No ServiceAccount or RBAC needed — pod does not call the Kubernetes API.
- Cleanup is user-driven via `helm uninstall`. `activeDeadlineSeconds` provides a backstop.

## Edge cases

| Case | Behavior |
|------|----------|
| `unitGlob` matches no units | Discovered-units preamble prints "no matching units" line so the stream isn't a silent hang. |
| Persistent journal disabled on node | Optional `showJournalHeader` value surfaces storage mode for diagnosis. AKS default is persistent. |
| Bad `nodeName` | Pod stays Pending. `activeDeadlineSeconds` and/or manual `helm uninstall` clean up. |
| Forgotten install | `activeDeadlineSeconds` (default 3600s) terminates the pod automatically. |

## Testing

1. `helm lint chart/`
2. `helm template ... | kubeconform` for schema validity
3. Live smoke test on a dev/staging cluster:
   - Install with a known good `nodeName`, confirm discovered units, history dump, and live follow via `kubectl logs -f`.
   - Trigger a unit restart on the host (via existing manual debug flow) and confirm the event appears in the live tail.
   - `helm uninstall` cleans up.
4. Negative cases:
   - Wrong `nodeName` → Pending pod, cleanup works.
   - `unitGlob` matching nothing → preamble line, no silent hang.
   - `activeDeadlineSeconds: 60` → pod self-terminates.

No automated CI coverage — this is diagnostic tooling; real-cluster smoke tests are the meaningful verification.
