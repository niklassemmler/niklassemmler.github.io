---
layout: post
title:  "How Flink passes data internally"
date:   2017-05-28
categories: flink system
---

In this post I will explain how Flink processes items and communicates them
between operators distributed over multiple machines. The post will explain how
Flink deploys tasks and how the data is passed inside Flink to a Netty
abstraction layer. It is a loose write-up of my own code-reading and provides
the reader with enough anchors to dive deeper into Flink's code base.

From a top-down perspective, Flink jobs are made up of a sequence of operators.
Every operator receives its input through an InputGate, processes it and emits
it into a ResultPartition. Every operator consists of a set of parallel
instances and so each operator may receive input from multiple sources
(InputChannel's) and send them to different endpoints (ResultSubpartition's).

To keep the discussion easy to follow, we focus always on the simplest case,
i.e. single ungrouped data streams. My analysis is based on the current state of
the code as on Github: https://github.com/apache/flink.

# Setup

Before we come to the part how Flink actually forwards items, let us first
consider how a flink job is set up. A user writes a program, compiles it into a
JAR and distributes it (via _flink run_) to all nodes, that are part of the
Flink cluster.

On each node a TaskManager is running that is coordinating the deployment of the
jobs. As part of the general setup, it will create a NettyConnectionManager and
create BufferPool's to distribute memory segments between the input and output
of every operator. As part of a job setup, it will instantiate a Task object,
which sets the context and invokes an executable, in a streaming environment
StreamTask, in a batch environment a BatchTask. 

Here we focus on StreamTask, which takes a StreamOperator to process input
streams and is run() on initialization.  Focussing on one input,
OneInputStreamTask wraps the StreamOperator in an StreamInputProcessor and calls
its processInput(...) method as long as the task runs. StreamInputProcessor
reads the incoming Buffer from a CheckpointBarrierHandler (either a
BarrierBuffer or BarrierTracker), deserializes, recycles the Buffer and passes
the result as an object to the StreamOperator's processElement method. The most
basic StreamOperator, the StreamMap, executes a userfunction over every single
element and collects the result. 


# Receiver

Now let us take a closer look first at the receiver. The BarrierBuffer mentioned
above, requests buffer from the InputGate and forwards them unless the channel
is blocked, in which case it stores it with a BufferSpiller in the OS cache
(memory \& disk). The channel is only blocked, when a CheckpointBarrier is
received instead of a buffer and this is only the case, when checkpointing is
_manually_ enabled inside of the job. So in the simplest case, we can assume
that no blocking occurs and the InputGate is queried through getNextBufferOrEvent().

(See https://ci.apache.org/projects/flink/flink-docs-release-1.2/dev/stream/checkpointing.html)

We will focus on the SingleInputGate to keep things simple. The SingleInputGate
contains references to multiple InputChannels. When the previous operator
resides on the same machine (LocalInputChannel), the InputChannel is connected directly to the
preceeding ResultSubpartition. We will focus on the case, where the previous
operator resides on a different machine and the data has to pass the network
(RemoteInputChannel).

So getNextBufferOrEvent() is called on the InputGate. If the partitions have not
already been requested, it calls a requestSubpartitions on each InputChannel.
Using NettyConnectionManager and the PartitionRequestClientFactory, a
PartitionRequestClient is created and the subpartition is requested. The
PartitionRequestClient assigns all InputChannel's to a PartitionRequestHandler.

When the subpartition is available, getNextBufferOrEvent() in the
SingleInputGate *waits* till an InputChannel with data is queued. (This means
that the thread will block here when no data is available.) Then it calls
getNextBuffer() on the input channel. The RemoteInputChannel contains a Queue
(an ArrayDeque) polls a queue containing the buffers and returns it.

From a network perspective, the PartitionRequestHandler (an independent thread!)
triggers when data arrives (channelRead), decodes it (decodeMsg), request a
Buffer from the InputGate's BufferProvider and forwards it to the onBuffer
function of the InputChannel. The inputChannel is then queued in the
SingleInputGate (notifyChannelNonEmpty).

The situation is a bit more difficult, when data arrives and no MemorySegment
is available. In this case the automatic reading of the channel is disabled and
a BufferListenerTask is added as listener to the BufferProvider of the
InputGate. As soon as buffer is recycled the buffer is used to store the
previous data and automatic reading resumes. When the time in between is long
enough, this should directly impact TCP's congestion window size and new data
should arrive slower.

Finally, when the task has received all data, it will additionally receive an
EndOfPartitionEvent and gracefully tear down the connection.


# Sender side

On the sender side results are forwarded from an OutputCollector to a
RecordWriter. From there a channel is selected through a ChannelSelector and the
record is written towards a ResultSubpartition (by passing first the
ResultPartitionWriter and the ResultPartition). Before that happens, a Buffer is
requested from the BufferPool and the data is serialized. Here RecordWriter will
*block and wait* till a Buffer becomes available.

ResultSubpartitions come in two forms SpillableSubpartition, which spills to
disk and a PipelinedSubpartition, which keeps data entirely in memory. Both are
accessed via Views. (When a SpillableSubpartition actually spills data to disk,
this data will be accessible via a SpilledSubpartitionView.)
SpillableSubpartition are chosen, when data needs to be collected in its
entirety before it can be processed. (This is necessary for the classical
reduce, but might as well be necessary for windows.)

Again from a network perspective (= indendent execution thread), when the
connection is established, the PartitionRequestServerHandler creates a
SequenceNumberingViewReader and a view of the subpartition by querying first the
ResultpartitionProvider / ResultpartitionProvider, then the ResultPartition
(createSubpartitionView) and finally the ResultSubpartition.
SequenceNumberingViewReader adds itself as a listener to the newly created queue
and when notified (notifyBuffersAvailable) calls notifyReaderNonEmpy on the
PartitionRequestQueue. Buffers are then requested in the
writeAndFlushNextMessageIfPossible method of the PartitionRequestQueue, where
the Buffer is written to the Netty channel.


# Summary

So far for now. I hope the post gives a good idea on how Flink communicates data
over the network. The key points to focus on are: 

- Each Flink Task deserializes the data in the StreamInputProcessor and serializes it in the RecordWriter. 
- When checkpointing is enabled (not the default), the task will block in the
  BarrierBuffer till a CheckPoint is processed.
- In general a task will only block, when not enough data segments are
  available, to store records before writing them out as buffers.
- On the receiver side, if the Task has not enough Buffers to retrieve items, it
  will deactivate automatic reading of the channel and wait for new Buffers.
This will impact TCP and thereby the sending and processing rate of the
preceeding operator.

The post is still missing some figures. If I have the time, I
will try to complete it later.

Topics that I would like to touch on later are:
- Scheduling - How node placement is decided and how the deployment details can
  be accessed at runtime.
- Latency Markers - Where they are created and what exacltly they measure.
