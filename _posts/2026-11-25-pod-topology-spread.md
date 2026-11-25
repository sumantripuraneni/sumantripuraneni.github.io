---
title: "Pod Topology Spread: Three Replicas Isn't High Availability"
description: "Why replica count alone doesn't protect you, and the topology spread setup I put on every production workload."
date: 2026-11-25 14:00:00 -0500
categories: [Kubernetes, Scheduling]
tags: [kubernetes, openshift, scheduling, high-availability, topology]
---

I've reviewed a lot of clusters where a team says "we're HA, we run three replicas." Then you check where those replicas actually run, and all three are
on the same node. One kernel panic, one bad drain, and the service is gone.

Replica count is capacity. **Where** those replicas land is availability. The scheduler won't sort that out for you unless you tell it to, and the tool for
that is `topologySpreadConstraints`.

## What it does

You tell the scheduler: keep pods with these labels evenly balanced across these domains. A domain is whatever a node label says it is: a node, a zone,
a rack.

Here's the setup I start with on almost every production Deployment:

```yaml
spec:
  template:
    metadata:
      labels:
        app: web
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: web
          matchLabelKeys:
            - pod-template-hash
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: web
          matchLabelKeys:
            - pod-template-hash
```

In plain terms:

- **Zones are a hard rule.** Never let one zone get more than one pod ahead   of another. If that can't be met, the pod waits.
- **Nodes are a soft rule.** Spread across nodes when you can, but don't   leave pods Pending just because the node pool is tight.

## The fields that matter

- **`topologyKey`**: the node label that defines a domain.
  `kubernetes.io/hostname` gives you per-node, `topology.kubernetes.io/zone` per-zone. On bare metal I use a custom rack label (more on that below).
- **`labelSelector`**: which pods get counted. It has to match the pod's own labels. Get this wrong and the constraint quietly does nothing.
- **`maxSkew`**: how uneven you'll tolerate. I use `1` almost everywhere.
- **`whenUnsatisfiable`**: `DoNotSchedule` is a hard rule, `ScheduleAnyway` is a preference.
- **`matchLabelKeys: [pod-template-hash]`**: only count pods from the same rollout. Explained below; always add it on Deployments.

## How the scheduler decides

It's simple arithmetic. For each zone, the scheduler asks: if I put the pod here, how far ahead of the emptiest zone does this one get? If that's more
than `maxSkew`, the zone is out.

Current state, `maxSkew: 1`:

| Zone   | Pods |
|--------|------|
| zone-a | 2    |
| zone-b | 1    |
| zone-c | 1    |

Put the next pod in zone-a and it's 3 vs 1, a gap of 2. Not allowed. Zone-b or zone-c gives a gap of 1. Fine. That's the whole algorithm.

## See it in action

Six workers, two per zone:

```console
$ kubectl get nodes -L topology.kubernetes.io/zone
NAME        STATUS   ROLES    AGE   VERSION   ZONE
worker-01   Ready    worker   41d   v1.33.4   zone-a
worker-02   Ready    worker   41d   v1.33.4   zone-a
worker-03   Ready    worker   41d   v1.33.4   zone-b
worker-04   Ready    worker   41d   v1.33.4   zone-b
worker-05   Ready    worker   41d   v1.33.4   zone-c
worker-06   Ready    worker   41d   v1.33.4   zone-c
```

### Without constraints

```console
$ kubectl get pods -l app=web -o wide
NAME                   READY   STATUS    RESTARTS   AGE   IP            NODE        NOMINATED NODE   READINESS GATES
web-7c9d8f6b5d-2xkqp   1/1     Running   0          45s   10.128.2.14   worker-01   <none>           <none>
web-7c9d8f6b5d-8hj4m   1/1     Running   0          45s   10.128.2.15   worker-01   <none>           <none>
web-7c9d8f6b5d-c6wzt   1/1     Running   0          45s   10.128.2.16   worker-01   <none>           <none>
web-7c9d8f6b5d-k2v9n   1/1     Running   0          45s   10.129.0.22   worker-02   <none>           <none>
web-7c9d8f6b5d-p7rtx   1/1     Running   0          45s   10.129.0.23   worker-02   <none>           <none>
web-7c9d8f6b5d-x4lqs   1/1     Running   0          45s   10.131.0.31   worker-03   <none>           <none>
```

Five of six pods in zone-a. Lose that zone and you lose most of the service.

### With constraints

```console
$ kubectl get pods -l app=web -o wide
NAME                   READY   STATUS    RESTARTS   AGE   IP            NODE        NOMINATED NODE   READINESS GATES
web-5f6b9c7d8e-4mtrw   1/1     Running   0          38s   10.128.2.21   worker-01   <none>           <none>
web-5f6b9c7d8e-9qzxk   1/1     Running   0          38s   10.129.0.27   worker-02   <none>           <none>
web-5f6b9c7d8e-bn2hv   1/1     Running   0          38s   10.131.0.35   worker-03   <none>           <none>
web-5f6b9c7d8e-f8sdl   1/1     Running   0          38s   10.130.0.18   worker-04   <none>           <none>
web-5f6b9c7d8e-l6cpj   1/1     Running   0          38s   10.131.2.12   worker-05   <none>           <none>
web-5f6b9c7d8e-w3ynk   1/1     Running   0          38s   10.130.2.19   worker-06   <none>           <none>
```

One per node, two per zone.

### Check the spread yourself

`-o wide` shows nodes, not zones. This gets you the per-zone count:

```bash
for n in $(kubectl get pods -l app=web -o jsonpath='{.items[*].spec.nodeName}'); do
  kubectl get node "$n" -o jsonpath='{.metadata.labels.topology\.kubernetes\.io/zone}{"\n"}'
done | sort | uniq -c
```

```console
      2 zone-a
      2 zone-b
      2 zone-c
```

And per node:

```console
$ kubectl get pods -l app=web -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort | uniq -c
      1 worker-01
      1 worker-02
      1 worker-03
      1 worker-04
      1 worker-05
      1 worker-06
```

> On OpenShift, swap `kubectl` for `oc`. Everything here works the same.
{: .prompt-tip }

### When the hard rule bites

The Deployment is at 8 replicas, spread 3/3/2, and zone-c is out of CPU. Scale to 9. Zone-c can't take the pod, and zone-a or zone-b would put it two
ahead of zone-c. So it waits:

```console
$ kubectl describe pod web-5f6b9c7d8e-zr8tq | grep -A3 Events
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  12s   default-scheduler  0/6 nodes are available: 2 Insufficient cpu, 4 node(s) didn't match pod topology spread constraints.
```

That's the rule working, not a bug. If you'd rather have the pod running somewhere than wait for balance, that's what `ScheduleAnyway` is for. Make
that call per workload.

## Why it's worth it

- **Outages hurt less.** A node, rack, or zone failure costs you a slice of capacity, not the whole service.
- **Upgrades stop being scary.** Draining a node or rolling a MachineConfigPool on OpenShift only evicts one pod per workload at a time.
  Pair it with a PodDisruptionBudget and upgrades become routine.
- **No hot nodes.** Load is spread out instead of one node running hot while others sit idle.
- **Less cross-zone traffic.** Zonal load balancers have healthy endpoints in every zone.

### Why not just use anti-affinity?

Required `podAntiAffinity` means at most one pod per domain. That's fine for three replicas in three zones. Scale to six and half of them won't schedule.
Topology spread says "keep it balanced," so 2/2/2 is fine. I only reach for anti-affinity when I really do need strict one-per-node, like some stateful
systems.

## Things I've seen bite teams

**Rolling updates skew the spread.** During a rollout, old and new pods both match `app: web`, so the scheduler counts both. Once the old pods are gone you
can be left lopsided. `matchLabelKeys: [pod-template-hash]` makes each ReplicaSet balance itself. Always add it on Deployments.

**Nothing rebalances after the fact.** Spread is only checked at scheduling time. A node comes back or you add a zone, and existing pods stay where they
are. Run the descheduler; on OpenShift that's the **Kube Descheduler Operator** with a topology profile. It evicts pods that break their constraints so the scheduler can place them again.

**The label selector doesn't match.** Copy-paste the constraint from another app, forget to change `app: web`, and it counts zero pods. No error, no
spreading.

**Nodes without the label are skipped.** If a node is missing the `topologyKey` label, pods with that constraint won't land there. Common on
bare metal where nobody labelled the new nodes.

**Tainted nodes still count by default.** If you have tainted nodes that can't run the workload, they can drag the "emptiest zone" count down and block
scheduling. Set `nodeTaintsPolicy: Honor`.

**Small zones need `minDomains`.** If you need at least three zones and a zone isn't available yet, `minDomains: 3` stops pods piling into the zones you
do have. With the cluster autoscaler, it pushes new capacity into the missing zone.

## Bare metal: spread across racks

No cloud zones on bare metal, but you still have racks, power feeds and top-of-rack switches that fail together. Label the nodes yourself:

```bash
oc label node worker-01 worker-02 topology.example.com/rack=rack-a
oc label node worker-03 worker-04 topology.example.com/rack=rack-b
oc label node worker-05 worker-06 topology.example.com/rack=rack-c
```

Then use `topologyKey: topology.example.com/rack`. A rack losing power no longer takes a service down with it.

## What I check in a review

- Every multi-replica workload has a zone (or rack) constraint
- `labelSelector` matches the pod template labels
- `matchLabelKeys: [pod-template-hash]` on Deployments
- Hard rule on the failure domain, soft rule on nodes
- A PodDisruptionBudget next to it
- Descheduler running if the cluster's topology changes much

A few lines of YAML, and a node or zone failure becomes an annoyance instead of an incident.

**Reference:** [Kubernetes docs: Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)