# KEP-NNNN: ReplicaSet Consolidation-Aware Scale-In Strategy

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [The Spreading Heuristic Works Against Consolidation](#the-spreading-heuristic-works-against-consolidation)
  - [No Coordination Mechanism Exists](#no-coordination-mechanism-exists)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1: HPA-Driven Consolidation](#story-1-hpa-driven-consolidation)
    - [Story 2: Respecting Do-Not-Disrupt Signals](#story-2-respecting-do-not-disrupt-signals)
    - [Story 3: Cost Optimization During Off-Peak](#story-3-cost-optimization-during-off-peak)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Feature Gate](#feature-gate)
  - [Modified Pod Deletion Ranking](#modified-pod-deletion-ranking)
  - [Node Informer Integration](#node-informer-integration)
  - [Do-Not-Disrupt Annotation Handling](#do-not-disrupt-annotation-handling)
  - [Interaction with Existing Scale-Down Logic](#interaction-with-existing-scale-down-logic)
  - [Worked Examples](#worked-examples)
  - [Future: Drift-Aware Consolidation (Post-Alpha)](#future-drift-aware-consolidation-post-alpha)
  - [Test Plan](#test-plan)
    - [Prerequisite Testing Updates](#prerequisite-testing-updates)
    - [Unit Tests](#unit-tests)
    - [Integration Tests](#integration-tests)
    - [End-to-End Tests](#end-to-end-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [Extend PodDeletionCost (KEP-2255)](#extend-poddeletioncost-kep-2255)
  - [Pluggable Scale-Down Framework](#pluggable-scale-down-framework)
  - [Webhook-Based Pod Selection](#webhook-based-pod-selection)
  - [Karpenter Pod Deletion Cost Controller (Annotation-Based Approach)](#karpenter-pod-deletion-cost-controller-annotation-based-approach)
- [Infrastructure Needed](#infrastructure-needed)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (CLI, key scenarios, etc.)
  - [ ] (R) Ensure GA://Ensure GA e2e tests meet requirements for [Coverage Onboarding Checklist](https://github.com/kubernetes/community/blob/master/sig-testing/coverage-onboarding-checklist.md)
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Coverage Onboarding Checklist items](https://github.com/kubernetes/community/blob/master/sig-testing/coverage-onboarding-checklist.md) are met
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

This KEP introduces an opt-in consolidation-aware heuristic to the ReplicaSet
controller's scale-down pod selection algorithm. When the `ConsolidatingScaleDown`
feature gate is enabled, the controller prefers deleting pods on nodes with fewer
total active pods during scale-down, enabling workload consolidation onto fewer
nodes. It also respects do-not-disrupt signals so that pods on protected nodes
are deprioritized for deletion.

## Motivation

### The Spreading Heuristic Works Against Consolidation

The current ReplicaSet scale-down algorithm prefers deleting pods on nodes with
*more* colocated replicas of the same ReplicaSet (a spreading heuristic). While
this promotes even distribution, it actively works against node consolidation.

### No Coordination Mechanism Exists

Node autoscalers such as Karpenter and cluster-autoscaler can reclaim empty or
underutilized nodes, but only when workloads consolidate during scale-down. When
an HPA scales a Deployment from 20 replicas to 10, the current spreading
heuristic distributes deletions evenly across nodes, leaving every node partially
occupied and preventing any node from being reclaimed.

KEP-2255 (PodDeletionCost) provides a mechanism for influencing pod deletion
order via annotations, but it requires an external controller to continuously
update annotations before scale-down events occur. This is operationally complex,
does not integrate with HPA-driven scale-down, and continuously updating
annotations places unnecessary load on the API server.

The magnitude of consolidation improvement depends on workload shape (uniform
vs skewed replica counts), cluster topology (number of nodes, pods per node),
and the node autoscaler's consolidation policy (empty-node-only vs
underutilization-based). The consolidation heuristic is most effective when
scale-down events remove enough replicas to empty or nearly empty at least one
node. For small scale-down events (e.g., removing 1-2 replicas from a large
deployment), the improvement over the spreading heuristic may be minimal.

**Value by consolidation policy:** For node autoscalers configured with an
empty-node-only consolidation policy (e.g., Karpenter's `ConsolidateWhenEmpty`),
the benefit is direct — concentrating deletions creates empty nodes that qualify
for removal. For underutilization-based policies (e.g., Karpenter's
`WhenUnderutilized`, cluster-autoscaler's default), the benefit is indirect but
still meaningful — concentrating deletions on already-underutilized nodes pushes
them closer to the utilization threshold faster, reducing the number of
consolidation moves (and therefore disruptions) needed to reach optimal state.
The magnitude of improvement is larger for empty-node-only policies.

### Goals

- Provide a feature-gated, consolidation-aware pod deletion heuristic for
  ReplicaSet scale-down that prefers removing pods from nodes with fewer total
  active pods.
- Respect do-not-disrupt signals from node autoscalers via a generic annotation,
  deprioritizing deletion of pods on protected nodes.
- Maintain full backward compatibility: when the feature gate is disabled,
  behavior is identical to the existing spreading heuristic.
- Complement (not replace) KEP-2255 PodDeletionCost — both mechanisms can
  coexist, with PodDeletionCost taking precedence in the existing sort order.

### Non-Goals

- **Not modifying scheduling behavior.** Topology spread enforcement remains at
  schedule time only. This KEP does not change how pods are placed on nodes.
- **Not providing per-workload opt-in/opt-out.** The feature gate is
  cluster-wide. Individual Deployments or ReplicaSets cannot selectively enable
  or disable the consolidation heuristic.
- **Not replacing or deprecating PodDeletionCost (KEP-2255).** PodDeletionCost
  retains precedence in the sort order (step 4) and remains the mechanism for
  explicit, per-pod deletion priority.
- **Not providing resource-aware scoring.** The heuristic uses pod count as a
  proxy for node utilization. CPU and memory utilization are not considered.
  Resource-aware scoring is a potential beta/GA enhancement.
- **Not handling StatefulSet scale-down.** StatefulSets have different ordering
  semantics (ordinal-based). This KEP applies only to ReplicaSet-managed pods.

## Proposal

### User Stories

#### Story 1: HPA-Driven Consolidation

As a platform engineer running workloads with HPA on Karpenter-managed nodes, I
want scale-down events to consolidate pods onto fewer nodes so that Karpenter can
reclaim empty nodes or nearly empty nodes and reduce my cloud spend, without requiring me to deploy and
maintain a sidecar controller that manages PodDeletionCost annotations. I want this reduction in cloud spend while
minimizing the rate of pod disruption in the cluster. 

#### Story 2: Respecting Do-Not-Disrupt Signals

As a platform engineer, I have nodes running long-lived batch jobs annotated with
do-not-disrupt. When my web-tier Deployment scales down, I want the ReplicaSet
controller to avoid deleting pods from those protected nodes, even if they have
fewer total pods, so that the scaled in pods come from nodes that can actually be reclaimed by Karpenter
via consolidation.

#### Story 3: Cost Optimization During Off-Peak

As a cost-conscious operator, I run a Deployment that scales from 50 replicas
during peak to 10 replicas off-peak. I want the scale-down to preferentially
empty out nodes so that my cluster autoscaler can remove nodes instead of
leaving up to 10 nodes each running a single pod.

### Risks and Mitigations

**Risk: Coupling to Karpenter-specific annotation.**
The initial implementation checks `karpenter.sh/do-not-disrupt`. This couples
core Kubernetes to a specific autoscaler's annotation.

*Mitigation:* We propose introducing a generic, Kubernetes-native annotation
`controller.kubernetes.io/do-not-disrupt` that any autoscaler or operator can
set. The implementation will check both the generic annotation and the
Karpenter-specific annotation during alpha, with the Karpenter-specific
annotation deprecated in beta. This gives the ecosystem time to migrate.

**Risk: Conflict with pod topology spread constraints.**
The consolidation heuristic may concentrate pods on fewer nodes, potentially
violating `topologySpreadConstraints` configured on the workload.

*Mitigation:* Pod topology spread constraints are enforced at scheduling time,
not at deletion time. When pods are deleted and rescheduled, the scheduler
enforces topology spread. However, during scale-down (net pod reduction), no
rescheduling occurs — pods are simply removed. The consolidation heuristic
affects *which* pods are removed, not where new pods are placed. If a user has
topology spread constraints, the remaining pods may violate the desired spread.
For workloads that scale down and stay at the lower replica count, this violation
is persistent — spread is only restored when new pods are scheduled (e.g., on
the next scale-up event). This is the same behavior as the current spreading
heuristic (which also does not guarantee topology spread compliance during
scale-down). We will document this interaction clearly. Factoring topology spread
constraints into the deletion ranking is deferred to future work.

**Risk: Unexpected behavior change for existing workloads.**
Users who depend on the current spreading behavior may be surprised if they
enable the feature gate.

*Mitigation:* The feature gate is disabled by default in alpha. Users must
explicitly opt in. Documentation will clearly describe the behavioral change.

## Design Details

### Feature Gate

- Name: `ConsolidatingScaleDown`
- Component: `kube-controller-manager`
- Default: `false` (alpha)
- Disable-supported: `true`

When disabled, the ReplicaSet controller uses the existing spreading heuristic
with no behavioral change.

### Modified Pod Deletion Ranking

The ReplicaSet controller's `ActivePodsWithRanks.Less()` function determines pod
deletion order during scale-down. The existing sort order is:

1. Unassigned pods (no node) before assigned pods
2. Pending pods before running pods
3. Not-ready pods before ready pods
4. Pods with lower deletion cost (KEP-2255) before higher cost
5. **More co-located pods from the replicaset on a node before few co-located pods on a node** (spreading heuristic)
6. Younger pods before older pods (random selection for remaining 'tied' pods from the replicaset)

When `ConsolidatingScaleDown` is enabled, step 5 is modified:

1. Unassigned pods before assigned pods
2. Pending pods before running pods
3. Not-ready pods before ready pods
4. Pods with lower deletion cost before higher cost
5. **[NEW] Pods on non-protected nodes before pods on nodes with do-not-disrupt pods (from any source even outside the current replicaset) co-located on them**
6. **[CHANGED] Pods on nodes with fewer total active pods before pods on nodes
   with more total active pods** (consolidation heuristic — inverted from
   spreading, note that this considers pods outside the replicaset replica pods)
7. Younger pods before older pods

The rank for each pod is computed by counting all active pods on the same node
(across all namespaces and controllers, not just the current ReplicaSet). This
provides a global view of node utilization for the consolidation decision.

**Architectural note: global pod counting.** Step 6 counts all active pods on
the same node across all namespaces and controllers. This means the ReplicaSet
controller makes deletion decisions based on workloads it does not own, which is
a departure from the current model where the controller reasons only about its
own replicas. This is an intentional design choice: the goal is to consolidate
onto fewer *nodes*, which requires a node-level view rather than a
ReplicaSet-level view.

**Limitation:** Pod count is a heuristic proxy for node utilization, not a
direct measure. Nodes running many DaemonSet pods or system workloads will
appear "full" even if they have low resource utilization. This means the
heuristic may deprioritize pods on nodes that are actually good consolidation
candidates from the node autoscaler's perspective. We accept this trade-off for
alpha because: (1) DaemonSet pod counts are typically uniform across nodes, so
they add a constant offset that does not change the relative ranking; (2) a
resource-aware scoring model is planned for beta but requires the node informer
infrastructure being introduced here; and (3) the heuristic is still strictly
better than the spreading heuristic, which ignores node-level signals entirely.

### Node Informer Integration

When `ConsolidatingScaleDown` is enabled, the ReplicaSet controller initializes
a node informer via the shared informer factory. In alpha, the node informer
serves a concrete purpose: validating that a pod's assigned node still exists
before including it in the consolidation ranking (pods on deleted nodes should
not influence the ranking of other pods). The node informer is also required for
the drift-aware consolidation enhancement planned for beta (reading node
conditions to identify drifted nodes).

The node informer is conditionally initialized — when the feature gate is
disabled, no node informer is created and there is zero additional overhead.

### Do-Not-Disrupt Annotation Handling

When computing disruption cost ranks, the controller pre-computes a
`nodeHasDoNotDisrupt` boolean map in a single pass over all pods on candidate
nodes before the sort begins. This avoids per-candidate annotation scanning
during the sort comparator. The pre-computation is O(P) where P is the total
number of pods on candidate nodes (one indexer lookup per unique node, each
returning the pods on that node). The sort itself then checks the pre-computed
map in O(1) per comparison.

**Alpha behavior:** Checks `karpenter.sh/do-not-disrupt: "true"`.

**Planned beta behavior:** Checks both `controller.kubernetes.io/do-not-disrupt: "true"`
(new generic annotation) and `karpenter.sh/do-not-disrupt: "true"` (deprecated
but still honored).

**Planned GA behavior:** Only checks `controller.kubernetes.io/do-not-disrupt: "true"`.
The Karpenter-specific annotation is no longer checked by the controller.

### Interaction with Existing Scale-Down Logic

The consolidation heuristic modifies only the ranking step of pod deletion
selection. All other aspects of scale-down remain unchanged:

- **PodDeletionCost (KEP-2255):** Evaluated at step 4, before the consolidation
  rank at step 6. PodDeletionCost takes precedence over the consolidation
  heuristic.
- **Pod phase and readiness:** Steps 1-3 are unchanged. Unassigned, pending, and
  not-ready pods are still preferred for deletion regardless of consolidation.
- **Pod age tiebreaker:** Step 7 is unchanged. Among pods with equal
  consolidation rank, younger pods are still preferred.
- **Burst deletion:** The `BurstReplicas` limit on concurrent deletions is
  unchanged.
- **PodDisruptionBudgets (PDBs):** PDBs are enforced at deletion time by the
  eviction API, not at ranking time. The consolidation heuristic selects which
  pods to *attempt* to delete; PDB enforcement may reject some of those
  deletions. This is the same interaction model as the existing spreading
  heuristic. If a PDB blocks deletion of the highest-ranked pod, the controller
  retries on the next sync cycle. The heuristic does not attempt to predict PDB
  availability — doing so would require tracking PDB state across all namespaces
  and add significant complexity for marginal benefit.
- **Pod priority and preemption:** Pod priority is not considered in the
  consolidation heuristic. Priority-based preemption is a scheduler concern
  (scheduling time), not a scale-down concern (deletion time). The ReplicaSet
  controller deletes its own replicas regardless of priority. If priority-aware
  scale-down is desired, PodDeletionCost (step 4) can be used to encode priority
  preferences via an external controller.

### Worked Examples

#### Clean drain: Scale from 9 to 6 replicas

A Deployment with 9 replicas across 3 nodes:

```
Node A: 3 pods (5 total active pods on node)
Node B: 3 pods (8 total active pods on node)
Node C: 3 pods (7 total active pods on node)
```

With `ConsolidatingScaleDown` enabled, the RS controller ranks pods by total
active pods on their node (ascending). Node A has the fewest total active pods
(5), so its 3 replica pods are deleted first. Node A is now empty of this
Deployment's pods, and if the other 2 pods on Node A are also scaled down or
belong to other shrinking workloads, the node becomes reclaimable.

**Without the feature:** The spreading heuristic distributes 3 deletions across
all 3 nodes (1 per node). Every node retains 2 replicas. No node moves closer
to empty.

#### Partial drain: Scale from 9 to 7 replicas

Same cluster. The RS controller removes 2 pods, both from Node A (fewest total
active pods). Node A still has 1 replica — not empty yet. But Node A is now the
least-occupied node for this Deployment, so on the next scale-down event, its
remaining pod is removed first. Convergence takes multiple events, but each
event moves the system toward consolidation.

**Without the feature:** The 2 deletions spread across 2 nodes. No node is
closer to empty than before.

### Future: Drift-Aware Consolidation (Post-Alpha)

Node autoscalers mark nodes for replacement when they drift from their desired
state (e.g., outdated AMI, changed configuration). Karpenter uses a `Drifted`
node condition; other autoscalers may use different signals. Scale-down should
prefer deleting pods from drifted nodes, since those nodes need replacement
regardless — draining them via scale-down avoids additional disruption from a
separate drain operation.

This extends the current two-tier model to three tiers:

1. **Drifted nodes** (highest deletion priority) — pods here are deleted first
2. **Normal nodes** (middle priority) — standard consolidation targets
3. **Do-not-disrupt nodes** (lowest deletion priority) — protected

This enhancement is deferred to beta for two reasons: (1) it limits alpha scope
to the core consolidation heuristic, and (2) the design of a generic drift
signal should be informed by the generic `controller.kubernetes.io/do-not-disrupt`
annotation work planned for beta. In the interim, the Karpenter Pod Deletion
Cost Controller provides three-tier drift ranking via PodDeletionCost
annotations, which takes precedence over the in-tree heuristic in the sort order.

### Test Plan

[x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

#### Prerequisite Testing Updates

The following existing tests require updates to accommodate the new sort step:

- `pkg/controller/controller_utils_test.go`: Existing `ActivePodsWithRanks`
  tests must be updated to verify that the spreading heuristic (step 5) is
  preserved when `ConsolidatingScaleDown` is disabled, and that the new
  consolidation heuristic (step 6) and do-not-disrupt deprioritization (step 5)
  are applied when the gate is enabled.
- `pkg/controller/replicaset/replica_set_test.go`: Existing scale-down tests
  must verify that pod deletion order is unchanged when the feature gate is off.

#### Unit Tests

- Pod deletion ranking with `ConsolidatingScaleDown` enabled and disabled:
  consolidation rank ordering, do-not-disrupt deprioritization, interaction
  with PodDeletionCost, swap correctness
- Pod-per-node counting: correct indexer usage, do-not-disrupt annotation
  detection, unassigned pods, feature gate toggle

Specific file paths and coverage targets will be documented in the
implementation PR.

#### Integration Tests

- `test/integration/replicaset/replicaset_test.go`: Integration test verifying
  that with `ConsolidatingScaleDown` enabled, scale-down of a ReplicaSet
  preferentially removes pods from nodes with fewer total pods.

#### End-to-End Tests

- E2e test (beta requirement) verifying consolidation behavior in a multi-node
  cluster with HPA-driven scale-down.

### Graduation Criteria

#### Alpha

- Feature gate `ConsolidatingScaleDown` implemented and disabled by default
- Unit tests for consolidation ranking and do-not-disrupt logic
- Integration tests for basic consolidation behavior
- KEP at `implementable` status
- PRR approval

#### Beta

- Address feedback from alpha users
- Introduce generic `controller.kubernetes.io/do-not-disrupt` annotation
- Add metrics for consolidation-aware deletions (see [Monitoring Requirements](#monitoring-requirements))
- E2e tests
- Documentation on kubernetes.io
- PRR re-review

#### GA

- Deprecate and remove Karpenter-specific annotation check
- Conformance tests
- 2-week flake-free test window
- All known bugs fixed
- Decide whether feature gate to be enabled by default based on feedback

### Upgrade / Downgrade Strategy

**Upgrade:** Enabling the feature gate changes scale-down pod selection order.
No data migration is required. The change takes effect on the next scale-down
event after the kube-controller-manager restarts with the gate enabled.

**Downgrade:** Disabling the feature gate reverts to the spreading heuristic.
No cleanup is required. Pods previously deleted under the consolidation heuristic
are already gone; the change only affects future scale-down decisions.

### Version Skew Strategy

The feature is entirely within the kube-controller-manager. There is no
version skew concern between control plane components because only one instance
of the ReplicaSet controller runs at a time (leader election). Kubelets and
API servers are unaffected.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [x] Feature gate
  - Feature gate name: `ConsolidatingScaleDown`
  - Components depending on the feature gate: `kube-controller-manager`

###### Does enabling the feature change any default behavior?

Yes. When enabled, ReplicaSet scale-down prefers deleting pods on nodes with
fewer total active pods (consolidation) instead of pods on nodes with more
colocated replicas (spreading). Pods on nodes with do-not-disrupt annotations
are deprioritized for deletion.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. Disabling the feature gate and restarting kube-controller-manager reverts
to the spreading heuristic. No state cleanup is required.

###### What happens if we reenable the feature if it was previously rolled back?

The consolidation heuristic resumes on the next scale-down event. No state
is persisted between enablements.

###### Are there any tests for feature enablement/disablement?

Yes. Unit tests verify that `ActivePodsWithRanks.Less()` behaves correctly
with the feature gate both enabled and disabled. Integration tests verify
end-to-end scale-down behavior under both configurations.

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?

A rollout cannot fail in a way that impacts running workloads. The feature only
affects the *order* in which pods are selected for deletion during scale-down.
If the feature gate fails to enable (e.g., typo in gate name), the existing
spreading heuristic is used. Running pods are never affected — only future
scale-down decisions change.

###### What specific metrics should inform a rollback?

- Unexpected increase in pod churn or rescheduling events
- Cluster autoscaler unable to consolidate nodes (indicating the heuristic is
  not working as expected)
- Increase in topology spread constraint violations reported by users (these should stick with current spreading heuristic)

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

Will be tested during alpha. The feature is stateless — toggling the gate
and restarting kube-controller-manager is sufficient.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, machines, permissions, or service account bindings?

No.

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?

- Check if the `ConsolidatingScaleDown` feature gate is enabled on
  kube-controller-manager.
- (Beta) A new metric `replicaset_consolidation_scale_down_total` will count
  the number of scale-down events that used the consolidation heuristic.
- (Beta) A new metric `replicaset_do_not_disrupt_deprioritizations_total` will
  count pods deprioritized due to do-not-disrupt annotations.

###### How can someone using this feature know that it is working for their instance?

- Observe that scale-down events preferentially remove pods from nodes with
  fewer total pods (visible via `kubectl get pods -o wide` before and after
  scale-down).
- Observe that cluster autoscaler or Karpenter reclaims nodes after scale-down
  events (node count decreases). Also should see increased 'emptiness' reason in counts metrics of node disruption. Should see decreases in voluntary pod disruption rates when ConsolidateWhenEmptyOrUnderUtilized is enabled on a nodepool.
- (Beta) Check the `replicaset_consolidation_scale_down_total` metric.

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

- [x] Metrics
  - Metric name: `replicaset_consolidation_scale_down_total` (beta)
  - Components exposing the metric: `kube-controller-manager`
- [x] Other
  - Node count reduction after scale-down events (observable via cluster
    autoscaler metrics or cloud provider billing)

###### Are there any missing metrics that would be useful to have in this category?

- Per-node pod density distribution after scale-down (future consideration, maybe something like Shannon's Entropy or the like)
- Consolidation efficiency ratio (pods removed from eventually-empty nodes vs
  total pods removed)

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

- Scale-down latency should not increase by more than 5% compared to the
  spreading heuristic (the additional pod-per-node counting is O(N) where N is
  the number of candidate pods).
- No increase in ReplicaSet controller error rate.

###### What are the failure modes for the enhancement?

| Failure Mode | Impact | Detection | Mitigation |
|---|---|---|---|
| Pod indexer returns error for node lookup | Falls back to rank 0 for affected pods (treated as empty node) | Controller error logs | Self-healing: next sync cycle retries |
| Node informer fails to sync | `nodeListerSynced` returns false; controller waits for sync | kube-controller-manager startup logs | Restart kube-controller-manager |
| Do-not-disrupt annotation on all nodes | All pods deprioritized equally; falls through to age-based tiebreaker | No pods consolidated | Expected behavior — no mitigation needed |

###### What steps should be taken if SLOs are not being met to determine the problem?

1. Check kube-controller-manager logs for errors related to the ReplicaSet
   controller or node informer.
2. Verify the feature gate is enabled: check kube-controller-manager flags.
3. Check pod indexer health: verify pods are indexed by node name.
4. Disable the feature gate and restart kube-controller-manager to revert to
   spreading heuristic.

### Dependencies

###### Does this feature depend on any specific services running in the cluster?

No. The feature uses only the existing pod informer/indexer and optionally the
node informer, both of which are part of the standard kube-controller-manager
informer factory.

### Scalability

###### Will enabling / using this feature result in any new API calls?

No new API calls. The consolidation ranking uses the existing pod indexer
(in-memory) to count pods per node. The node informer (when enabled) uses the
standard shared informer factory LIST/WATCH, which is already used by other
controllers.

###### Will enabling / using this feature result in introducing new API types?

No.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

No. No new fields are added to any API objects. The do-not-disrupt annotation
is read-only from the controller's perspective.

###### Will enabling / using this feature result in increasing time taken by any operations?

The pod deletion ranking computation adds an O(N + P) pre-computation pass
where N is the number of candidate pods and P is the total number of pods on
candidate nodes (for building the `nodePodCounts` and `nodeHasDoNotDisrupt`
maps via indexer lookups). The sort itself is O(N log N) with O(1) map lookups
per comparison. For typical ReplicaSet sizes (tens to hundreds of pods), this
adds negligible latency (microseconds).

###### Will enabling / using this feature result in non-negligible increase of resource usage?

- **Memory:** The node informer adds memory proportional to the number of nodes
  in the cluster. For a 5,000-node cluster, this is approximately 5-10 MB
  additional memory in kube-controller-manager. The `nodePodCounts` and
  `nodeHasDoNotDisrupt` maps are ephemeral (allocated per scale-down event and
  garbage collected).
- **CPU:** Negligible. The pod-per-node counting is O(N) per scale-down event.

###### Can enabling / using this feature result in resource exhaustion of some node resources?

No. The feature only affects which pods are deleted during scale-down. It does
not affect scheduling, resource requests, or resource limits.

###### Will enabling / using this feature result in any new resource claim usage?

No.

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?

The feature does not make additional API calls. If the API server is unavailable,
the ReplicaSet controller cannot perform any scale-down operations regardless of
this feature. The pod indexer and node informer operate on cached data.

###### What are other known failure modes?

See [failure modes table](#what-are-the-failure-modes-for-the-enhancement) above.

###### What steps should be taken if SLOs are not being met to determine the problem?

See [SLO troubleshooting](#what-steps-should-be-taken-if-slos-are-not-being-met-to-determine-the-problem) above.

## Implementation History

- 2026-03-27: Initial KEP draft (provisional)
- `consolidation-strategy` branch: Reference implementation with feature gate,
  modified ranking logic, node informer integration, and unit/integration tests

## Drawbacks

- **Increased complexity in pod deletion ordering.** The sort comparator gains
  additional conditional branches, making it harder to reason about deletion
  order. Mitigated by clear documentation and comprehensive tests.

- **Node informer memory overhead.** Even though the alpha implementation
  primarily uses the pod indexer, the node informer is initialized when the gate
  is enabled. In very large clusters (5,000+ nodes), this adds non-trivial
  memory. Mitigated by conditional initialization (no overhead when gate is off).

- **Potential for uneven pod distribution.** The consolidation heuristic
  intentionally concentrates pods on fewer nodes, which may conflict with
  availability goals. Users who want both consolidation and spreading must
  rely on pod topology spread constraints at scheduling time.

## Alternatives

### Extend PodDeletionCost (KEP-2255)

Instead of a new heuristic, extend PodDeletionCost with an automated controller
that sets annotations based on node utilization.

**Why not chosen:** This requires deploying and maintaining an external
controller, does not integrate with HPA-driven scale-down (annotations must be
set *before* the scale-down event), and continuously updating annotations places
load on the API server. Previous proposals along these lines (#107598, #123541)
were closed as stale without implementation.

### Pluggable Scale-Down Framework

Add a `scaleConfig` field to ReplicaSetSpec defining a sequence of sorting
methods for scale-down pod selection (as proposed in #107598).

**Why not chosen:** Too broad in scope. Previous attempts at pluggable frameworks
were closed without implementation. A focused, single-heuristic approach behind a
feature gate is more likely to gain SIG approval and be maintainable long-term.
If the community later wants pluggability, this KEP's consolidation heuristic
can become one option in that framework.

### Webhook-Based Pod Selection

Add a webhook extension point that allows external systems to influence or
override pod deletion selection.

**Why not chosen:** Adds latency to the scale-down path, introduces a new
failure mode (webhook unavailability), and significantly increases complexity.
The in-tree heuristic approach is simpler and more reliable.

### Karpenter Pod Deletion Cost Controller (Annotation-Based Approach)

The Karpenter project has an RFC for a Pod Deletion Cost Controller that
achieves similar goals via a different mechanism: a sidecar controller that
ranks nodes by consolidation preference and writes `pod-deletion-cost`
annotations. This approach works with the existing RS controller sort order
(step 4: PodDeletionCost) without requiring changes to Kubernetes core.

**Relationship to this KEP:** The two approaches are complementary, not
competing. The annotation-based approach can ship independently of Kubernetes
release cycles and provides additional capabilities (three-tier drift ranking,
configurable strategies). This KEP provides a zero-dependency, in-tree solution
that requires no external controller. When both are active, the annotation-based
PodDeletionCost (sort step 4) takes precedence over the in-tree consolidation
heuristic (sort step 6), allowing the external controller to override or refine
the in-tree behavior.

**Deprecation path:** If this KEP reaches GA and the community adopts it widely,
the annotation-based controller becomes optional — useful for advanced ranking
strategies (drift-aware, resource-weighted) but not required for basic
consolidation. The annotation-based approach remains the recommended path for
users who need capabilities beyond the in-tree heuristic.

## Infrastructure Needed

None. The feature is entirely within the existing kube-controller-manager binary
and uses existing informer infrastructure.
