---
layout: page
permalink: /traces/index.html
title: Traces
---

#### S3D Traces ####

This page describes how to download and interpret several traces 
that we have captured of the Legion implementation of S3D running 
on [Titan](https://www.olcf.ornl.gov/titan/). Each trace contains
both the start-up and execution of four time steps. Traces are 
captured at the Realm interface on which the Legion runtime is
built. We will detail the types of Realm operations and their
corresponding logging calls in [Realm Event Logging](#eventlogging).

### Downloading Traces ###

Below are the current set of traces that we have been captured. Each
trace is a separate run and comes in a tar ball. The size of the tar
balls is given on the URLs. When unpacked some of the tar balls will
expand to several GBs of data.

All the traces share a common naming convention: mechanism name 
followed by mapping strategy followed by node count. The chemical
mechansim is either DME or Heptane.  DME is the smaller mechanism
containing only 30 species while Heptane is the larger mechnaism
with 52 species. The mapping strategy is listed as either `all` or
`mixed`. The `all` mapping strategy is used to place the majority
of the computation on the GPU (hence all-GPU), while the `mixed`
mapping strategy is designed to mix work evenly between CPUs and
GPUs. Finally, the node count says how many nodes where used as
part of the run.

  * [DME - ALL - 2 Nodes](/s3dtraces/dme_all_2nodes.tar.gz) (5.1 MB)
  * [DME - MIXED - 2 Nodes](/s3dtraces/dme_mixed_2nodes.tar.gz) (16 MB)
  * [DME - ALL - 4 Nodes](/s3dtraces/dme_all_4nodes.tar.gz) (11 MB)
  * [DME - MIXED - 4 Nodes](/s3dtraces/dme_mixed_4nodes.tar.gz) (32 MB)
  * [DME - ALL - 32 Nodes](/s3dtraces/dme_all_32nodes.tar.gz) (85 MB)
  * [DME - MIXED - 32 Nodes](/s3dtraces/dme_mixed_32nodes.tar.gz) (255 MB)
  * [Heptane - ALL - 32 Nodes](/s3dtraces/hept_all_32nodes.tar.gz) (98 MB)
  * [Heptane - MIXED - 32 Nodes](/s3dtraces/hept_mixed_32nodes.tar.gz) (175 MB)

### <a name="eventlogging"></a> Realm Event Logging ###

Traces are captured at the Realm interface inside the Legion runtime.
Realm is the low-level component of the Legion runtime. Its design and
architecture are laid out in [this paper](/pdfs/realm2014.pdf) which we
assume as prior knowledge for the description of the logging calls. The
benefit of capturing traces at the Realm level is that we can easily see
the low-level structure of an application in terms of its most basic
operations (e.g. tasks, data movement, and synchronization) without
needing to understand any of the high-level Legion semantics. Furthermore,
these traces can easily be replayed on different machine architectures
as operations and data movement can mapped and executed in any order that
is consistent with the generated event graph. This makes Realm traces
particularly attractive when considering the design of future architectures.

Each trace below is composed of a collection of files. Each file contains
logging calls for each of the different Realm operations invoked on a
different node. While the files contain operations that are local to a
node, the handles and operations are not required to remain local to a
node. For example, a task registered on one node, may end being run on 
a different node depending on the processor that it was directed to.
Similarly an event created on one node, may serve as the precondition to
an operation launched on a different node. As we describe in the Realm
paper, we guarantee that all handles are globally unique with each handle
refering to exactly one globally unique thing.

There are 14 logging calls which can appear in the log files. Each logging
call is proceeded with an human readable name which can be used to parse
each logging call unambiguously using a top-down LL parsing algorithm.

 1. Record the existence of a given processor: <b>Processor: %1 %2</b> 
    1. Processor ID
    2. Processor Kind (CPU, GPU, Utility)

 2. Record a processor group based on a set of processors: <b>Group: %1 %2 %3 %4 ...</b>
    1. Processor Group ID
    2. Number of contained processors
    3. First Processor ID
    4. Second Processor ID
    ... the given number of contained processors in the group

 3. Record the existence of a given memory: <b>Memory: %1 %2</b>
    1. Memory ID
    2. Processor Kind (GASNet, System, Registered, Socket, Zero-Copy, Framebuffer)

 4. Record a task lauch: <b>Task Request: %1 %2 (%3,%4) (%5,%6) (%7,%8) %9 %10 %11</b>
    1. Function ID for the task to run
    2. Target processor for task
    3. ID of the termination event
    4. Generation of the termination event
    5. ID of the precondition event
    6. Generation of the precondition event
    7. ID of the termination event for the enclosing task
    8. Generation of the termination event for the enclosing task
    9. Priority for the task
    10. Pointer for the arguments
    11. Argument length in bytes

 5. Record the time taken to execute the task: <b>Task Time: (%1,%2) %3</b>
    1. ID of the termination event for the task
    2. Generation of the termination event for the task
    3. Execution time in microseconds

 6. Record when a task waits for an event internally: <b>Task Wait: (%1,%2) (%3,%4) %5</b>
    1. ID of the termination event for the task
    2. Generation of the termination event for the task
    3. ID of the event being waited on
    4. Generation of the event being waited on
    5. Time spent waiting on an event

 7. Record the beginning of event merge: <b>Event Merge: (%1,%2) %3</b>
    1. ID of the merged event
    2. Generation of the merged event
    3. Number of precondition events

 8. Record an event precondition for a merged event: <b>Event Precondition (%1,%2) (%3,%4)</b>
    1. ID of the merged event
    2. Generation of the merged event
    3. ID of the precondition event
    4. Generation of the precondition event

 9. Record an event trigger operation: <b>Event Trigger: (%1,%2) (%3,%4) (%5,%6)</b>
    1. ID of the triggered event
    2. Generation of the triggered event
    3. ID of the precondition event
    4. Generation of the precondition event
    5. ID of the termination event for the enclosing task
    6. Generation of the termination event for the enclosing task

 10. Record the creation of a phase barrier: <b>Barrier Creation: %1 %2</b>
     1. ID of the barrier
     2. Initial expected number of arrivals for all generations

 11. Record an alteration of the expected barrier arrival count: <b>Barrier Alter: (%1,%2) (%3,%4) %5</b>
     1. ID of the barrier
     2. Generation of the barrier
     3. ID of the termination event for the enclosing task
     4. Generation of the termination event for the enclosing task
     5. Delta by which to alter the expected arrival count

 12. Record an arrival on a phase barrier <b>Barrier Arrive: (%1,%2) (%3,%4) (%5,%6) %7</b>
     1. ID of the barrier
     2. Generation of the barrier
     3. ID of the precondition event
     4. Generation of the precondition event
     5. ID of the termination event for the enclosing task
     6. Generation of the termination event for the enclosing task
     7. Arrival count

 13. Record the a copy request between physical regions: <b>Copy Request: (%1,%2) (%3,%4) (%5,%6) %7 %8</b>
     1. ID of the copy termination event
     2. Generation of the copy termination event
     3. ID of the precondition event
     4. Generation of the precondition event
     5. ID of the termination event for the enclosing task
     6. Generation of the termination event for the enclosing task
     7. Source Memory
     8. Destination Memory

 14. Record the total size of a copy in bytes: <b>Copy Size: (%1,%2) %3</b>
     1. ID of the copy termination event
     2. Generation of the copy termination event
     3. Total bytes transfered


