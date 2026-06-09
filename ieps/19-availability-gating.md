---
title: Introducing ServerReadinessRule to ensure Server readiness

iep-number: 19

creation-date: 2026-04-27

status: implementable

authors:
  - '@xkonni'

reviewers:
  - '@ironcore-dev/metal-operator-maintainers'
---

# IEP-19: Availability Gating for IronCore Servers

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Overview](#overview)
  - [ReadinessGateController](#readinessgatecontroller)
  - [ServerReadinessRule CRD](#serverreadinessrule-crd)
  - [Enforcement modes](#enforcement-modes)
  - [Lifecycle examples](#lifecycle-examples)
- [Dependencies](#dependencies)
- [Related](#related)

## Summary

Introduce a generic availability gating mechanism that prevents servers from being claimed until all declared readiness gates pass. A `ServerReadinessRule` declares which servers to target, which condition to monitor, and which taint to manage. The `ReadinessGateController` inside metal-operator watches rules and `Server` conditions and manages taint add/remove. Any entity with the appropriate permissions can write conditions on `Server` objects — they need not be aware that a rule exists.

## Motivation

Servers entering the fleet may fail pre-availability checks (e.g. physical wiring, unclean disks). Without a gating mechanism, a server can reach `Available` state and get claimed before these checks pass, leading to issues at workload runtime.

Other gate types follow the same pattern without requiring changes to metal-operator.

### Goals

- Block servers from being claimed until all declared readiness gates pass
- Support multiple concurrent gate types owned by independent controllers
- Work for both managed and auto-discovered servers
- Be transparent to the workload scheduling layer
- Support re-gating when a server returns from maintenance

### Non-Goals

- Defining the logic run by individual external controllers
- Replacing existing maintenance workflows
- Automatic timeout or force-removal of taints for stuck gates
- Eviction of already-reserved servers based on gate state

## Proposal

### Overview

The taints of `Server` objects can be manipulated via `ServerReadinessRule` objects. These rules monitor the `.status.conditions` of a `Server` for a specific condition type and status.

If the specific condition type is not present in the expected status, a taint specified in the rule is applied to the `Server`.

Once the condition is present in the expected status, the taint is removed.

This makes a `Server` unclaimable to `ServerClaim`s that don't have the required tolerations.

### ReadinessGateController

The `ReadinessGateController` runs inside metal-operator and:

- Watches all `ServerReadinessRule` objects and all `Server` objects
- When a server matches a rule and the condition is not satisfied, applies the corresponding taint
- Watches `Server.status.conditions` and removes a taint when its condition is satisfied
- Manages rule lifecycle: adds a finalizer at rule creation, removes orphaned taints on rule deletion or selector change

Any entity with the appropriate permissions can write conditions on `Server.status.conditions`. They do not need to apply or remove taints and need not be aware of any `ServerReadinessRule` that references their condition type.

### ServerReadinessRule CRD

A `ServerReadinessRule` is a cluster-scoped resource that declares:

- Which servers to target (`spec.serverSelector`)
- Which condition to monitor (`spec.conditions[*].type` + `spec.conditions[*].status`)
- Which taint to manage (`spec.taint`)
- How to enforce the rule (`spec.enforcementMode`)

```yaml
apiVersion: metal.ironcore.dev/v1alpha1
kind: ServerReadinessRule
metadata:
  name: example-one-shot
  finalizers:
    - metal.ironcore.dev/serverreadinessgc
spec:
  serverSelector:
    matchLabels:
      example.com/hardware-type: server-gen2
  enforcementMode: BootstrapOnly
  conditions:
  - type: ExampleCondition
    status: True
  taint:
    key: metal.ironcore.dev/example-not-ready
    effect: NoClaim
---
apiVersion: metal.ironcore.dev/v1alpha1
kind: ServerReadinessRule
metadata:
  name: example-continuous
  finalizers:
    - metal.ironcore.dev/serverreadinessgc
spec:
  serverSelector:
    matchLabels:
      example.com/hardware-type: server-gen2
  enforcementMode: Continuous
  conditions:
  - type: ExampleContinuousCondition
    status: True
  taint:
    key: metal.ironcore.dev/example-continuous-not-ready
    effect: NoClaim
```

**Taint key uniqueness** is enforced by a validating admission webhook. On create and update of a `ServerReadinessRule`, the webhook rejects the request if any other rule declares the same `spec.taint.key`. Each taint key must be owned by exactly one rule.

**Rule lifecycle managed by ReadinessGateController:**

- At creation, adds `metal.ironcore.dev/serverreadinessgc` as a finalizer.
- On deletion, removes the rule's taint from all servers that carry it, then releases the finalizer.
- On `spec.serverSelector` change, removes the rule's taint from servers that no longer match the updated selector.

### Enforcement modes

#### `BootstrapOnly`

The rule is enforced once per server. When a server first matches the rule and the condition is not satisfied, the `ReadinessGateController` applies the taint. Once the condition passes and the taint is removed, the controller records completion via annotation and stops enforcing that rule for that server.

```yaml
metadata:
  annotations:
    metal.ironcore.dev/bootstrap-completed-example-one-shot: 'true'
```

If the condition later flips to `False` at runtime, nothing happens — the taint is not re-applied. Re-gating can be triggered externally by clearing the completion annotation, at which point the controller picks up the server again on its next reconcile.

#### `Continuous`

The rule is always enforced. If the condition is not satisfied at any point, the `ReadinessGateController` applies the taint immediately. When the condition recovers, the taint is removed. Suitable for checks that must hold throughout the server's lifetime.

### Lifecycle examples

#### Server created — bootstrap rule

The server matches the rule. The condition is absent — `ReadinessGateController` applies the taint.

```yaml
# Server at creation
metadata:
  name: compute-01
  labels:
    example.com/hardware-type: server-gen2
spec:
  taints:
    - key: metal.ironcore.dev/example-not-ready # applied by ReadinessGateController
      effect: NoClaim
status:
  conditions: []
```

An external entity sets the condition to `True`. `ReadinessGateController` removes the taint and records bootstrap completion.

```yaml
metadata:
  annotations:
    metal.ironcore.dev/bootstrap-completed-example-one-shot: 'true'
spec:
  taints: []
status:
  conditions:
    - type: ExampleCondition
      status: 'True'
      reason: Passed
      lastTransitionTime: '2026-05-29T10:05:00Z'
```

The server is claimable. The completion annotation prevents the taint from being re-applied even if the condition later flips to `False`.

#### Server created — continuous rule

The server matches the rule. The condition is absent — `ReadinessGateController` applies the taint.

```yaml
spec:
  taints:
    - key: metal.ironcore.dev/example-continuous-not-ready # applied by ReadinessGateController
      effect: NoClaim
status:
  conditions: []
```

An external entity sets the condition to `True`. `ReadinessGateController` removes the taint. Server is claimable.

```yaml
spec:
  taints: []
status:
  conditions:
    - type: ExampleContinuousCondition
      status: 'True'
      reason: Passed
      lastTransitionTime: '2026-05-29T10:12:00Z'
```

The condition later flips to `False`. `ReadinessGateController` re-applies the taint immediately.

```yaml
spec:
  taints:
    - key: metal.ironcore.dev/example-continuous-not-ready
      effect: NoClaim
status:
  conditions:
    - type: ExampleContinuousCondition
      status: 'False'
      reason: Failed
      lastTransitionTime: '2026-05-29T14:00:00Z'
```

## Dependencies

| Responsibility                       | Owner                                                      |
| ------------------------------------ | ---------------------------------------------------------- |
| Taint add/remove                     | ReadinessGateController (metal-operator)                   |
| Rule lifecycle (finalizer, taint GC) | ReadinessGateController (metal-operator)                   |
| Condition reporting                  | Any entity with write access to `Server.status.conditions` |

## Related

- ironcore-dev/enhancements#11 — original issue
- ironcore-dev/metal-operator#878 — Server taints
