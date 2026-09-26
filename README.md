# karpenter-topology-spread-issue

When scheduling pods with topology spread constraints, Karpenter evaluates the constraints against all topology domains (e.g. availability zones) known to the cluster, not just the domains of the NodePools the pod is actually compatible with. If a pod can only run on a subset of NodePools (via nodeSelector, node affinity, or tolerations), the domains that only the other NodePools can produce permanently stay at zero pods, so Karpenter judges the skew as violated and fails to provision capacity for pods that should be schedulable.

## Git Repositories

karpenter-topology-spread-issue project is composed of the following git repositories.

* [aws-terraform](https://github.com/ssup2-playground/karpenter-topology-spread-issue_aws-terraform) : Terraform to reproduce issue

## My Contributions

* Analyze why the previous two fixes were reverted (memory and CPU regressions at high NodePool counts) and implement a third fix which is faster and lighter than the current code on every benchmark.
* Provide multi-axis benchmarks (NodePools, instance types, zones, taint groups) and a KWOK e2e test to prevent future regressions.
* Related issues and pull requests.
  * Github Issue : https://github.com/kubernetes-sigs/karpenter/issues/2227
  * Github Issue (report) : https://github.com/kubernetes-sigs/karpenter/issues/2623
  * Github Pull Request : https://github.com/kubernetes-sigs/karpenter/pull/3181
  * Reverted attempts : https://github.com/kubernetes-sigs/karpenter/pull/2639 (memory regression), https://github.com/kubernetes-sigs/karpenter/pull/2671 (CPU regression)

## Cause of Issue

Karpenter simulates scheduling to decide which nodes to create. For a pod with a topology spread constraint, it builds the set of topology domains to spread over from **every** NodePool's requirements and instance type offerings, without checking whether the pod could ever schedule to nodes of those NodePools.

The skew calculation compares each domain's pod count against the global minimum across all domains. A domain that only an incompatible NodePool can produce can never receive the pod, so its count stays zero forever and pins the global minimum at zero. With `maxSkew: 1` and `whenUnsatisfiable: DoNotSchedule`, every reachable domain is then limited to one pod, and the remaining replicas stay `Pending` even though the compatible NodePool has plenty of capacity.

For example, with a NodePool `2az` (restricted to two zones) and a NodePool `3az` (all three zones), a deployment targeting `2az` via its pool label with a zone spread constraint schedules only two replicas (one per reachable zone): the third zone which only `3az` can produce is counted as a domain, pinning the global minimum at zero.

## Previously Reverted Attempts

Both prior fixes filtered domains correctly, but stored scheduling metadata **per domain**, so their cost multiplied with NodePool, instance type, and domain counts, and both were reverted after production regressions.

### Attempt 1 ([#2639](https://github.com/kubernetes-sigs/karpenter/pull/2639)) : deep copy per domain → memory regression ([#2779](https://github.com/kubernetes-sigs/karpenter/issues/2779))

Every domain stored its own deep copy of every producing NodePool's requirements. The same requirements were duplicated at `NodePool x InstanceType x Domain` cardinality on every scheduling loop.

```text
NodePool A (requirements: ~KBs)          NodePool B (requirements: ~KBs)
     |                                        |
     |  deep copy per domain                  |  deep copy per domain
     v                                        v
zone-a : [ Copy(Reqs A) ] [ Copy(Reqs B) ]
zone-b : [ Copy(Reqs A) ] [ Copy(Reqs B) ]
zone-c : [ Copy(Reqs B) ]

memory = NodePools x InstanceTypes x Domains copies
         (100 NodePools x 400 instance types -> GBs allocated per loop)
```

### Attempt 2 ([#2671](https://github.com/kubernetes-sigs/karpenter/pull/2671)) : serialize-to-dedup on every insert → CPU regression ([#2954](https://github.com/kubernetes-sigs/karpenter/issues/2954))

Each domain stored `DomainSource{Requirements, Taints}` slices, and `Insert()` deduplicated by serializing the full requirements to a string — re-serializing every already-stored source on **every insert**. `Insert()` is called `NodePools x InstanceTypes x Domains` times per scheduling loop.

```text
Insert(zone-a, source)
  -> serialize(source)                      // expensive
  -> compare vs serialize(stored source 1)  // re-serialized every time
             vs serialize(stored source 2)
             vs ...

+1875% ~ +33440% slower as NodePool count grows (benchmarked at 1~100 NodePools)
```

## How to solve this issue

The fix ([kubernetes-sigs/karpenter#3181](https://github.com/kubernetes-sigs/karpenter/pull/3181)) tracks which NodePools can produce each domain, and counts a domain for a pod only if at least one producing NodePool passes the pod's `nodeTaintsPolicy` (the pod tolerates the NodePool's taints) and `nodeAffinityPolicy` (the NodePool's requirements are compatible with the pod's node selector and required node affinity).

Unlike the reverted attempts, nothing is stored per domain. One `topologyNodePool{requirements, taints}` is constructed **per NodePool** and shared by pointer across every domain the NodePool can produce; domains only append 8-byte pointers, and deduplication is an O(1) pointer comparison.

```text
topologyNodePool A (built once)     topologyNodePool B (built once)
        ^                                   ^
        |  every ptr->A below is the same   |  every ptr->B below is the same
        |  single object, never copied      |  single object, never copied

zone-a : [ ptr->A , ptr->B ]
zone-b : [ ptr->A , ptr->B ]
zone-c : [ ptr->B ]

memory = NodePools metadata + one 8-byte pointer per (domain, producer)
```

Per-pod filtering memoizes each NodePool's eligibility by pointer, so a pod costs one taint/affinity evaluation per NodePool instead of per domain:

```text
ForEachDomain(pod):
  eligible = {}                        // memo, keyed by pointer
  zone-a: eligible[A]? -> evaluate(A)=true   -> count zone-a
  zone-b: eligible[A]? -> memo hit (true)    -> count zone-b
  zone-c: eligible[B]? -> evaluate(B)=false  -> skip zone-c

evaluations per pod = NodePools (2), not Domains x Producers
```

This makes the fix faster and lighter than the unfixed code on every measured axis (NodePools 1-200, instance types 100-1000, zones 3-50, taint groups 1-20).

## Issue Test

### How Test

1. Create EKS cluster with Karpenter and two NodePools (`2az`, `3az`) with [aws-terraform](https://github.com/ssup2-playground/karpenter-topology-spread-issue_aws-terraform)
2. Run `test/test.sh` in [aws-terraform](https://github.com/ssup2-playground/karpenter-topology-spread-issue_aws-terraform), which deploys 4 replicas targeting the `2az` NodePool with a zone topology spread constraint
3. Count `Running` / `Pending` pods

### Result

* Total pod count : 4

|Karpenter|Running pod count|Pending pod count|
|---|---|---|
|Without fix (v1.14.1)|2|2 (unsatisfiable topology constraint)|
|With [#3181](https://github.com/kubernetes-sigs/karpenter/pull/3181)|4|0|

Karpenter's provisioner error for the pending pods shows the unreachable zone being counted (`us-east-1c` is only producible by the `3az` NodePool, yet appears in the domain counts and pins the global minimum at zero):

```text
could not schedule pod ... unsatisfiable topology constraint for topology spread,
key=topology.kubernetes.io/zone (counts = us-east-1a: 1, us-east-1b: 1, us-east-1c: 0,
podDomains = topology.kubernetes.io/zone Exists,
nodeDomains = topology.kubernetes.io/zone In [us-east-1a us-east-1b])
```

With the fixed controller image (`ghcr.io/ssup2-playground/karpenter:pr3181`, built from karpenter-provider-aws with #3181 applied), the two pending pods are provisioned immediately and the deployment spreads 2:2 across the `2az` NodePool's zones, with zero `could not schedule pod` errors.

Full captures are in the [aws-terraform](https://github.com/ssup2-playground/karpenter-topology-spread-issue_aws-terraform) repository under `test/result_*`.
