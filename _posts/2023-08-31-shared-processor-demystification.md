---
layout: post
title:  "Shared processors demystification"
date:   2023-08-31 15:00:00 -0400
categories: power hmc phyp
---

On a POWER server, LPAR can be configured with dedicated or shared processors. While dedicated processors are pretty straitforward to setup, shared processors have several parameters that are not always intuitive and it's very easy to create a suboptimal configuration. This post will describe those parameter and explain how to set them up.

# Processor

While configuring partitions, the term processor refers to a core, not a sockat.

# Processing Units

**Short description:** The garanted CPU time for the partition.

**Also known as as:** Entitled capacity or entitlmeent.

The POWER hypervisor works with a dispatch window of 10 ms: every 10 ms, it ensure all shared processor partitions have their fair share of time using the processor cores. The "processing units" setting indicates how much time using the cores is garanteed to be available for the partition during each 10ms dispatch window: 1.0 processing unit means the partition is garanteed to be able to spend 10ms in the cores during each dispatch window. 1.5 processing unit means the partition is garanteed to be able spend 15ms in the cores during each 10ms dispatch window (for example 7.5ms in 2 cores).

A partition doesn't necesseraly use all the garanteed time in the CPU (if it's waiting for I/O for example), it can yield it's time to the hypervisor.

The minimum processing units for a partition is 0.05 (since POWER7+, was 0.1 before), above this value, the granularity is 0.01.

This value can be changed dynamically between the minimum and maximum value. Changing the minimum or maximum value requires a partition activation.

# Capped/Uncapped and Weight

If a partition is capped, it will not be able to use more CPU time than what is allowed by it's processing units setting even if cycles are available and the partition would have a use for those cycles.

If a partition is uncapped it may be able to use more CPU time if available and it has a use for them.

During each dispatch window, after (that's not always true but the result is the same) all the partitions have been offered their garanteed CPU time, the hypervisor will dispatch the remaining CPU time to the uncapped partitions that need more CPU time proportionnaly to their weight value.

If the weight of a partition is set to 0, the partition is actually capped. It is often more interesting to have the partition set to uncapped with a weight of 0 than setting it to capped because the weight can be changed dynamically while changing the capped/uncapped setting requires a partition activation.

If the VIOS use shared processors, it is recommended to set their weight to 255 because a CPU starved VIOS will generally have very negative performance impact.

# Shared processor Pool

A processor pool can limit the CPU time of a set of partitions during each dispatch window. If a processor pool has a size of 2 cores, all the partitions belonging to that pool will be able to collectively use at most 20ms of CPU time during each 10ms dispatch window.

Processor pools are often used for software license compliance (IBM i or Oracle for example).

A POWER server always have a default pool that has no limit. You can create up to 63 additional pools.

A partition always belongs to one pool.

A processor pool does not reserve physical cores, it only limits how many cores a group of partitions can use at the same time. It is possible to have 2 processor pools with 8 processors each on a system with 12 cores for example.

Processor pools can be changed dynamically and a partition can switch pool dynamically.

# Virtual Processors

The "Virtual Processors" value indicates the number of processors the partition will "think" it has. This is always a whole number.

The minimum value is the entitlement rounded up (for example with an entitlement of 1.3, the minimum VP is 2). 

The maximum value is the entitlement divided the minimum entitlement for a partition (0.05 since POWER7+, 0.1 before) rounded down, for example a POWER 9 with 0.49 entitlement has a maximum VP of 9.

For a capped partition, there is no point in having more VP than the minimum needed for the entitlement.

For an uncapped partition, the VP value indicates how many physical cores the partition can use at most. This means that a partition with 0.5 entitlement and 2 VPs will be able to use at most 2 physical cores.

There is no point in having a partition with more VP than enabled physical cores in the system or the number of processors in the shared processor pool.

It is generally not advisable to have a very high number of VP with a very low entitlement.

This value can be changed dynamically between the minimum and maximum value. Changing the minimum or maximum value requires a partition activation.