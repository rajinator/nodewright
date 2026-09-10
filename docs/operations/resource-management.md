# Resource Management in NodeWright

NodeWright provides flexible resource management for the pods it creates: it uses a namespace LimitRange for defaults when one is present (the chart installs one by default, which you can disable) and allows per-package overrides. This document explains how resource defaults and overrides work, and what validation rules are enforced.

---

## 1. Namespace Defaults with LimitRange

By default, NodeWright uses a [Kubernetes LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/) to set default CPU and memory requests/limits for all containers in the namespace where NodeWright operates.

**Example LimitRange:**
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: skyhook-default-limits
  namespace: <your-namespace>
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 250m
        memory: 256Mi
```

- If a pod/container does **not** specify its own resources, these defaults are applied.
- You can configure these values via the Helm chart or Kustomize overlays.

---

## 2. Per-Package Resource Overrides

You can override the default resource requests/limits for each package in your NodeWright Custom Resource (CR). This is done in the `resources` field for each package:

**Example:**
```yaml
spec:
  packages:
    mypackage:
      version: 1.0.0
      image: ghcr.io/nvidia/skyhook-packages/shellscript
      resources:
        cpuRequest: "200m"
        cpuLimit: "400m"
        memoryRequest: "128Mi"
        memoryLimit: "256Mi"
```

- If **any** of the four fields (`cpuRequest`, `cpuLimit`, `memoryRequest`, `memoryLimit`) are set, **all four must be set** and must be positive values.
- If no override is set, the namespace's LimitRange applies.

---

## 3. Validation Rules

NodeWright enforces the following validation rules (via webhook) for resource overrides:

- If any of the four resource fields are set, **all four must be set**.
- All values must be **positive**.
- `cpuLimit` must be **greater than or equal to** `cpuRequest`.
- `memoryLimit` must be **greater than or equal to** `memoryRequest`.

**Examples:**

| Valid? | cpuRequest | cpuLimit | memoryRequest | memoryLimit | Reason |
|--------|-----------|----------|--------------|-------------|--------|
| ✅     | 200m      | 400m     | 128Mi        | 256Mi       | All set, valid |
| ❌     | 200m      |          | 128Mi        | 256Mi       | Not all fields set |
| ❌     | 200m      | 100m     | 128Mi        | 256Mi       | cpuLimit < cpuRequest |
| ❌     | 200m      | 400m     | 128Mi        | 64Mi        | memoryLimit < memoryRequest |
| ❌     | 0         | 400m     | 128Mi        | 256Mi       | Zero value |

If a resource override is invalid, the NodeWright CR will be **rejected** by the webhook.

---

## 4. Best Practices

- Use LimitRange to set sensible defaults for your namespace.
- Only set per-package overrides if you need different resource requirements for a specific package.
- Review your resource settings to avoid overcommitting or underutilizing cluster resources.
- If you change LimitRange defaults, new pods will use the new defaults unless overridden.

---

## 5. Troubleshooting

- If your NodeWright CR is rejected, check that all four resource fields are set and valid if you are using overrides.
- Use `kubectl describe limitrange -n <namespace>` to see the current defaults.
- Use `kubectl describe skyhook <name>` to see the status and any error messages.

---

## 6. Disabling Resource Defaults (No Limits)

If you do **not** want any default resource requests or limits applied to your Skyhook-managed pods/containers, you can simply **omit the LimitRange** from your namespace:

- **Helm:** Set `limitRange: {}` or remove the `limitRange` section from your `values.yaml`.

If there is **no LimitRange** and you do **not** set resource requests/limits in your package overrides, then:

- Your pods/containers will run with **no resource requests or limits**.
- This means they will be scheduled as "BestEffort" pods, which may be evicted first under resource pressure and may not get guaranteed CPU/memory.

**Note:**

- Disabling resource limits is not recommended for production clusters, as it can lead to resource contention and unpredictable scheduling.
- Only do this if you have a specific reason and understand the implications.

---

## 7. Special Case: Uninstall Pod Resources

Uninstall pods in NodeWright **do not use per-package resource overrides**.  
Instead, their resource requests/limits are determined only by the namespace defaults:

- If a LimitRange is present in the namespace, uninstall pods will use those default CPU and memory requests/limits.
- If there is no LimitRange, uninstall pods will run as "BestEffort" (no resource requests/limits).
- Any `resources:` overrides set for the original package are **not applied** to the uninstall pod.

**Note:**

- Ensure defaults are big enough for uninstall processes if using the uninstall package life cycle.

---

## 8. When a Node Is Already Full: `packagePriorityClassName`

A package stage can fail to start on a node that has no CPU or memory left, even though NodeWright has work to do there. The pod is rejected with `OutOfcpu` (or `OutOfmemory`) and the stage never runs.

```
$ kubectl get pod -n nodewright
NAME                                    READY   STATUS      RESTARTS   AGE
my-skyhook-mypackage-1.0.0-apply-node1  0/3     OutOfcpu    0          25h
```

### Why this is not a scheduling problem

NodeWright pins every package and interrupt pod to its node with `spec.nodeName`. That is deliberate — the pod exists to act on *that* host — but it means the pod never goes through the scheduler. So the usual answers do not apply:

- The scheduler never sees the pod, so **scheduler preemption never runs for it**. Priority alone will not push other pods aside.
- `OutOfcpu` is a *kubelet admission* rejection, not the scheduler's `Unschedulable`. The kubelet is being handed a pod for a node that cannot fit it.

The kubelet has its own admission-time preemption and it will evict lower-priority pods to admit a pod — but only for pods it considers **critical**, which means a priority value of at least `2000000000`.

### Why only the two system classes work

Kubernetes caps user-defined PriorityClasses at `1000000000`:

```
maximum allowed value of a user defined priority is 1000000000
```

That is half the critical threshold, so **a custom PriorityClass can never enable this behaviour**, no matter how high you set it. Only the two built-in classes qualify:

| PriorityClass | Value | Enables kubelet preemption |
|---|---|---|
| `system-node-critical` | 2000001000 | ✅ |
| `system-cluster-critical` | 2000000000 | ✅ |
| any custom class | ≤ 1000000000 | ❌ — affects node-pressure eviction order only |

### Which of the two to use

**Use `system-cluster-critical`.** Both clear the threshold, but they are not interchangeable. The kubelet decides what a pod may evict with `kubetypes.Preemptable`: a critical pod may always evict a non-critical one, and beyond that it is a strict `>` on priority value. `system-node-critical` is 1000 points higher than `system-cluster-critical`, so a package pod running as node-critical can evict pods *in the cluster-critical band* — CoreDNS, metrics-server, and whatever else your cluster puts there. `system-cluster-critical` sits exactly at the floor of that band, so it can still evict ordinary workloads to get a stage moving but cannot displace other critical infrastructure. It is the smaller hammer, and it is the one that fits.

### Enabling it

```yaml
controllerManager:
  manager:
    env:
      packagePriorityClassName: "system-cluster-critical"
```

Unset by default: package pods run at priority 0 and simply wait for room.

> [!WARNING]
> This is not free. A critical package pod will cause the kubelet to **evict running workloads** to make room for it, best-effort first, then burstable, then guaranteed. Turn it on when a stalled node upgrade is worse than a restarted workload — not by default.

**Caveats:**

- Some clusters restrict the system priority classes with a `ResourceQuota` scoped by `PriorityClass`. Where that is in force, pod creation is rejected outright instead of the pod being admitted, which trades a stalled stage for a failing one. Check with `kubectl get resourcequota -n <namespace> -o yaml` before enabling.
- The kubelet only preempts when *every* admission failure is a resource shortfall. If the pod is also rejected for another reason (a taint it does not tolerate, a node selector mismatch), no eviction happens.
- This is a backstop, not a capacity plan. If package pods routinely land on full nodes, size the nodegroup with headroom for upgrades and spread the workloads pinned to it.

---

For more information, see the [Kubernetes documentation on resource management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) and [LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/). 
