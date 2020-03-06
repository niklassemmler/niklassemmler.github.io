---
layout: post
title:  Proteus - Time-Interleaved Virtual Cluster
date:   2015-10-24
categories: articles,datacenter,data analysis,cluster scheduling
---

Today I am going to present an article that has caught my attention:

_"The only Constant is Change: Incorporating Time-Varying Network Reservations
in Data Centers"_ by Di Xie, et al., in SIGCOMM 2012

The authors look at the data centers of the time and find that while it is
possible to reserve storage and computing power on demand the same was not true
for network bandwidth.  Existing solutions had already tried to solve this issue
by introducing a virtual clusters abstraction. 

**Virtual Cluster (VC):** A VC consists of all links and nodes in a data center that
are used by a single application (e.g. a Hadoop job). Usually the cluster is
isolated from the rest of the system by virtualization not only on the hosts
(nodes), but additionally the links used by the system. On these links bandwidth
is reserved for traffic of the cluster.

What sets this work apart is a measurement on existing workloads. In the process
the authors find that typical data center applications generate traffic only in
30% to 60% of the time. In contrast most virtual cluster implementations reserve
bandwidth for the whole duration of the application. The authors aim to optimize the
bandwidth $$B$$ usage in a time period $$T$$ by a VC as in $$\int^T_0 B(t)dt$$. To this end
they introduce a new type of VC: Time-Interleaved Virtual Clusters (TIVC). 

Using the workloads: Sort, WordCount, Hive Join and Hive Aggregation as a
baseline, they define four TIVC models: single peak, multiple peaks,
varying-width peaks, varying width and height peaks. The last type is formally
defined as $$<N, T, B_b, P_1, \cdots, P_k>$$ where N defines the number of
nodes, T the start and end time, $$B_b$$ the base bandwidth and $$P_i = (T_{i1},
T_{i2}, B_i)$$ define multiple periods $$T_{i1}$$ to $$T_{i2}$$ where the
bandwidth is changed to $$B_i$$. All other types are simplifications of this
formal definition.

In a separate profile run the bandwidth usage of recurring applications--the
authors state that especially MapReduce are highly regular--in a
data center can be modeled with this specification. On future runs bandwidth can
be reserved accordingly. One might expect that job execution might vary due to
stragglers, and disk failures etc. 

Explain evaluation model

Rate limiting is supposed to be hardly possible on the side of the hardware switches
(core) as it would require a (very costly) upgrade of these same switches. Hence
it is recommended to implement performance guarantees on the hypervisor.

In a simulation with 16,000 machines Proteus found fewer than 26 concurrent jobs per link. Additionally, they found that current switches support up to 16k policers and 64k ACL entries.

How much oversubscription is still working?
How powerful are these switches really?
Statement: Oktopus is state of the art
Statement: Compared to Sort, Hive Join and Hive Aggregation the WordCount
requires significantly less bandwidth

A simulation

System: Oktopus, Gatekeeper
Profiling: Elastisizer, StarFish and CBO (but no networking, just type number of
VMs)
A flexible model for resource management in virtual private networks Duffield,
Goyal

Optimize repeated jobs: Corral
Global Data: Iridium, JetStream
