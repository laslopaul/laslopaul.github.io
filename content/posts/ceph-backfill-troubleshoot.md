---
title: "When Ceph Backfill Was Not Actually Stuck"
date: 2026-05-17
draft: false
hideComments: true
tags: ["ceph", "troubleshooting"]
layout: post
---

Recently I had a task that looked simple on paper: resize the root partitions on several Kubernetes nodes running a Rook-Ceph cluster.

The resize itself was not the most interesting part. The interesting part started after new OSDs appeared in the cluster and Ceph began rebalancing data. At that point the cluster looked healthy, but the rebalancing process almost stopped making progress. For more than 8 hours the number of misplaced objects stayed practically on the same figure, while most of the remapped PGs were sitting in `backfill_wait`.

That was the part that made the task interesting. It was not an obvious failure, and it was not a classic “Ceph is down” situation. The cluster was alive, the data was safe, but the background recovery work was moving so slowly that it looked stuck.

At first glance everything looked fine:

```bash
ceph -s
```

The cluster was reporting `HEALTH_OK`. But the PG state told a slightly different story:

```text
24 remapped pgs
705 active+clean
23 active+remapped+backfill_wait
1  active+remapped+backfilling
```

Out of 24 remapped PGs, only one was actually backfilling. The rest were sitting in `backfill_wait`. Nothing was broken, but the cluster was moving at a painfully slow pace. The number of misplaced objects stayed almost unchanged for more than 8 hours, so it stopped looking like normal slow housekeeping and started looking like a real throttling issue.

## Understanding what `backfill_wait` really meant

At the beginning I treated `backfill_wait` as something vague: Ceph wants to backfill, but it is waiting for something.

In practice, it was more specific than that. A PG in `backfill_wait` is queued for backfill, but it does not currently have the required reservation from the involved OSDs. In other words, the PG is ready to move, but some OSD is saying: not now, I do not have a free slot for this work.

The question became: why was Ceph allowing only one backfill at a time?

## The misleading OSD benchmark

The first thing that stood out was one newly added OSD: `osd.8`.

Ceph has a mechanism where an OSD measures the capacity of the underlying device, including its IOPS capacity, and stores that value in the config database. This value is later used by the mClock scheduler to decide how much background work the OSD can handle.

On this cluster, `osd.8` had a very strange value:

```bash
ceph config dump | grep "osd.8" | grep osd_mclock
osd.8   osd_mclock_max_capacity_iops_ssd   349.717058
```

The other SSD-backed OSDs had values in a completely different range. Some were around 40,000–70,000 IOPS.

So Ceph believed that this new OSD was dramatically slower than its neighbors. It was the same type of disk, but from Ceph's point of view it looked like a weak device that should not be given much background work.

The fix was to overwrite the bogus value with a more realistic one:

```bash
ceph config set osd.8 osd_mclock_max_capacity_iops_ssd 40000
```

## mClock ignoring the usual knobs

After that, I expected the usual recovery settings to help:

```bash
# Allow each OSD to run up to 8 backfill operations at the same time.
# This controls how many PGs can be backfilled concurrently per OSD.
ceph config set osd osd_max_backfills 8

# Allow each OSD to run up to 4 active recovery operations at the same time.
# This affects recovery concurrency, for example when replicas need to be rebuilt.
ceph config set osd osd_recovery_max_active 4
```

The settings were visible in the config dump. So they were definitely set.

But the cluster still behaved almost the same way: many PGs in `backfill_wait`, only one actively backfilling.

This is where another Ceph lesson arrived.

With the mClock scheduler enabled, the traditional recovery and backfill knobs are not necessarily honored. Settings like `osd_max_backfills` and `osd_recovery_max_active` may be present in the config database, but mClock can still control the effective limits using its own scheduling logic.

To make mClock honor those settings, this flag has to be enabled:

```bash
ceph config set osd osd_mclock_override_recovery_settings true
```

Without that, the settings looked correct but had no real effect.

## Persistent config is not always immediate config

Even after enabling the override flag and setting higher values, the running OSDs did not immediately behave differently.

The persistent configuration was changed, but the live daemons still needed a push. In this case, the useful command was:

```bash
ceph tell 'osd.*' injectargs --osd-max-backfills=8 --osd-recovery-max-active=4
```

This applies the arguments to the running OSD daemons without restarting them.

So the full picture was actually two-layered:

```bash
# Persistent configuration
ceph config set osd osd_mclock_override_recovery_settings true
ceph config set osd osd_max_backfills 8
ceph config set osd osd_recovery_max_active 4

# Immediate runtime effect
ceph tell 'osd.*' injectargs --osd-max-backfills=8 --osd-recovery-max-active=4
```

The persistent config survives restarts. The injected arguments make the current daemons pick up the values immediately.

After correcting the mClock behavior and injecting the runtime arguments, the cluster PG stats changed to something much more reasonable:

```text
12 backfilling
12 backfill_wait
```

At that point, misplaced objects started dropping visibly and the cluster finally looked like it was making real progress.

## What I took away from this

This was a good Ceph troubleshooting experience because the cluster was not actually broken. It was healthy, but the backfill process had effectively been stuck for more than 8 hours, and the reason was not obvious from the high-level status.

A few things I learned:

- `HEALTH_OK` means the data is safe, not that all background work is finished. A Ceph cluster can be healthy and still spend a lot of time rebalancing or backfilling.

- `backfill_wait` usually means there is a reservation or throttling bottleneck. If many PGs are waiting and only one is backfilling, it is worth checking the effective backfill limits instead of assuming the cluster is stuck.

- mClock changes the troubleshooting model. The old knobs are still there, but they may not do what you expect unless `osd_mclock_override_recovery_settings` is enabled.

- The automatic OSD capacity benchmark matters. If a newly added OSD gets a bad benchmark result, Ceph may treat it as a much slower disk and schedule recovery work very conservatively.

- There is a difference between changing the config database and changing the behavior of running daemons. Sometimes `ceph config set` is not enough for the current situation, and `ceph tell ... injectargs` is needed to apply the change immediately.

## Commands I would check next time

These are the commands I would keep close if I had to troubleshoot a similar situation again:

```bash
ceph -s
ceph health detail
ceph pg dump_stuck
ceph osd df tree
ceph config dump | grep osd_mclock
ceph tell osd.N dump_recovery_reservations
ceph config get osd osd_max_backfills
ceph config get osd osd_recovery_max_active
ceph config get osd osd_mclock_override_recovery_settings
```

And if I need the runtime values to take effect immediately:

```bash
ceph tell 'osd.*' injectargs --osd-max-backfills=8 --osd-recovery-max-active=4
```
