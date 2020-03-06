---
layout: post
title: In-network Aggregation in WSN
date: 2015-03-02
categories: in-network aggregation survey
---

Today I want to introduce the article "In-network Aggregation Techniques for Wireless
Sensor Networks: A Survey" by Elena Fasolo et al. This paper is the only one I found that
gives a wide overview over the many topics connected with data aggregation in the
network. Sensors typically suffer from limited availability of energy. Researchers in the
field aim for restricting the transmitted data to keep the sensors running longer without
external interference. But restricting the transmission is not the only aim. Energy can
also be saved by reducing the window where sensors are listening for data or by reducing
the collisions in case of a shared medium. Fasolo et al. look beyond the classical
routing algorithms and focus on the problem where data needs only to be communicated from
a source to sink node, but where data from multiple source nodes can be aggregated before
it is communicated to the sink node. 

On the setup of their survey Fasolo et al. define three essential ingredients for every
data aggregation in a wireless sensor network (WSN):
1. Routing Protocols
2. Aggregative Functions 
3. Data Representations.

## Networking Protocolos


\
-> aggregation points ->
/

aggregation tree

\
 \
--x
 / \
/   x -> user
   /
--x
 /
/

Data aggregation requires a different forwarding than classical routing algorithms.
Instead of reducing the length of the path the aggregation approach focusses on
forwarding the packet according to its content and the availability of aggregation
points. The aggregation points bring their own needs into the routing algorithm. A
trade-off exists between communicating data quickly to avoid data staleness and waiting
for more incoming data to reduce the communication overhead. Timing/scheduling and
synchronization strategies have to be developed to handle this problem based on the
underlying application. The main strategies developed so far contain:

* Periodic simple aggregation: Aggregating data every pre-defined period.
* Periodic per-hop aggregation: Aggregating data every period, if packets from all
  children have been received.
* Periodic per-hop adjusted aggregation: Timing per node is adjusted due to position in
  aggregation tree

## Aggregation Functions

Aggregation functions can range from simple duplicate detections mechanisms, where to
original data is kept (lossless), to statistical functions. where only a approximate
statement over the data is possible (lossy). 

Especially in WSNs the main challenge lies in specifying the aggregation functions for the
limited processing capacity of the sensor.

## Data representation

The data representation used to store and aggregate the data is not emphasized by Fasolo
et al. They simply state, the used data structure should be optimized for compressing and
aggregating the data.




