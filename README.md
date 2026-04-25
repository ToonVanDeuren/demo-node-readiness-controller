# demo-node-readiness-controller

A hands-on demo showing how to use the [Node Readiness Controller (NRC)](https://github.com/kubernetes-sigs/node-readiness-controller) together with the [Node Problem Detector (NPD)](https://github.com/kubernetes/node-problem-detector) to gate workload scheduling behind a critical daemon.

The goal: workloads must not land on a node until a critical supporting daemon is confirmed running on that node.

## How it works

```
Node boots
  └─ NPD checks every 10s: is demo-critical-supporting-daemon Running on this node?
       ├─ No  → sets CriticalDaemonNotReady=True  → NRC applies taint
       └─ Yes → sets CriticalDaemonNotReady=False → NRC removes taint

Taint present  → demo workload pods stay Pending (no toleration)
Taint absent   → demo workload pods are scheduled normally
```

The critical daemon itself tolerates the taint, so it can start regardless of node readiness state. Once NPD detects it running, the taint is removed and other workloads can schedule.

## Components

| File | What it creates |
|------|----------------|
| `01-namespace.yaml` | `demo` namespace |
| `02-nrc-crds.yaml` | `NodeReadinessRule` CRD |
| `03-nrc-install.yaml` | NRC controller in `nrr-system` namespace |
| `04-node-readiness-rule.yaml` | Rule: taint all nodes until `CriticalDaemonNotReady=False` |
| `05-npd-rbac.yaml` | RBAC for NPD to read pods in `demo` namespace |
| `06-npd-config.yaml` | NPD plugin config + check script |
| `07-npd-daemonset.yaml` | NPD DaemonSet on all Linux nodes |
| `08-demo-critical-daemon.yaml` | Simulated critical daemon (busybox), tolerates the readiness taint |
| `09-demo-workload.yaml` | nginx Deployment with no toleration — waits for node readiness |

## Prerequisites

- Kubernetes 1.19+
- `kubectl` configured against your cluster
- Permissions to create cluster-scoped resources (Nodes, ClusterRoles, CRDs)

## Install

Apply the manifests in order:

```bash
kubectl apply -f demo/
```

Kubernetes will apply files in filename order, so the numbered prefix ensures correct ordering.

## Verify

**Check node conditions** — after NPD starts, the custom condition should appear:

```bash
kubectl get nodes -o custom-columns='NAME:.metadata.name,CONDITIONS:.status.conditions[*].type'
```

**Check NRC rule status** — shows which nodes have the taint applied and their evaluation results:

```bash
kubectl get noderreadinessrule demo-critical-daemon-rule -o yaml
```

**Watch workload scheduling** — pods should remain Pending until the critical daemon is Running:

```bash
kubectl get pods -n demo -w
```

**Inspect the taint directly:**

```bash
kubectl get nodes -o custom-columns='NAME:.metadata.name,TAINTS:.spec.taints'
```

## Configuration

**Change which nodes the rule applies to** (`04-node-readiness-rule.yaml`):

```yaml
nodeSelector:
  matchLabels:
    node-role.kubernetes.io/worker: ""
```

**Change enforcement mode** (`04-node-readiness-rule.yaml`):

- `continuous` (default): taint is reapplied if the daemon stops after initial boot
- `bootstrap-only`: taint is only enforced at node startup, not reapplied later

**Change taint effect** to evict running pods instead of blocking new ones:

```yaml
taint:
  key: "readiness.k8s.io/critical-daemon-not-ready"
  effect: "NoExecute"
```

**Dry-run mode** — test impact without applying taints:

```yaml
spec:
  dryRun: true
```

Check results with `kubectl get noderreadinessrule -o yaml` under `.status.dryRunResults`.

**Adjust check interval** (`06-npd-config.yaml`, `critical-daemon-monitor.json`):

```json
"invoke_interval": "10s"
```

## Key concepts

**Node conditions**: Kubernetes has built-in conditions (`Ready`, `MemoryPressure`, etc.). NPD can add custom conditions via plugins. This demo adds `CriticalDaemonNotReady` using a shell script that queries the Kubernetes API.

**Readiness taints**: Applied by NRC when a node condition indicates a dependency is missing. Pods without the matching toleration will not be scheduled on that node. Pods with the toleration (like the critical daemon itself) are unaffected.

**NPD custom plugin**: A bash script (`check-critical-daemon.sh`) that exits with:
- `0` → daemon is running → condition `False` → taint removed
- `1` → daemon not running → condition `True` → taint applied
- `2` → API error → condition `Unknown` → taint stays

## Versions

| Component | Version |
|-----------|---------|
| Node Readiness Controller | v0.3.0 |
| Node Problem Detector | v0.8.19 |
| NRC API | `readiness.node.x-k8s.io/v1alpha1` |
