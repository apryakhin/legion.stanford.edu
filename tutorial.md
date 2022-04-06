---
layout: page
permalink: /tutorial/index.html
title: Tutorials
---

## Legion API Tutorial

After completing the [getting started](/starting/) guide this tutorial
will show how to begin writing programs to the Legion C++ runtime
API. The goal of the tutorial is not to describe all the Legion
runtime calls, but rather to introduce to users how to write programs
within the Legion framework. Consequently, the tutorial is not
comprehensive, but when completed users should understand the
methodology behind the API. Furthermore, these code examples are
designed to clearly demonstrate the usage of API calls and should not
be considered idiomatic of actual Legion code. Well designed Legion
applications will extend the types in the API to construct higher
levels of abstractions.

The tutorial is broken into ten examples which introduce concepts from
the Legion programming model. Each example builds upon previous
examples to gradually show how a complete Legion application is
structured. The source code for each example can be found in the
`tutorial` directory of the Legion repository. Users should build and
run these examples while working through the tutorial.

 0. [Hello World](/tutorial/hello_world.html)
 1. [Tasks and Futures](/tutorial/tasks_and_futures.html)
 2. [Index Space Tasks](/tutorial/index_tasks.html)
 3. [Hybrid Programming Model](/tutorial/hybrid.html)
 4. [Logical Regions](/tutorial/logical_regions.html)
 5. [Physical Regions](/tutorial/physical_regions.html)
 6. [Privileges](/tutorial/privileges.html)
 7. [Partitioning](/tutorial/partitioning.html)
 8. [Multiple Partitions](/tutorial/multiple.html)
 9. [Custom Mappers](/tutorial/custom_mappers.html)

## Advanced Examples

In addition to the basic tutorial, we will be gradually adding more
complete examples of programs written in Legion. As we add new
programs they will be registered here along with a brief description
of the features they encompass.

  * [Circuit Simulation](/tutorial/circuit.html) - This is the circuit
    example from our [publications](/publications/). The circuit
    simulation illustrates the use of reduction privileges as well as
    reduction-fold physical instances.  We also cover how to run tasks
    on the GPU, a simple Legion design pattern, and an example of how
    Legion easily enables in-situ analysis of program data.
  * [Explicit Ghost Regions](/tutorial/ghost.html) - An illustration
    of how to use Legion to implement an explicit ghost regions
    algorithm. We cover how to perform inter-region copies and use
    phase barriers to encode producer-consumer relationships in a
    deferred execution environment.  We also show how to employ
    acquire and release operations in conjunction with simultaneous
    coherence to safely manage explicit ghost regions.
  * [Conjugate Gradient](https://github.com/lanl/CODY/tree/master/legion/legion-hpcg) -
    An external project being developed by Los Alamos National Lab
    that illustrates how higher-level abstractions should be
    constructed on top of logical regions and the Legion programming
    model. In this particular case sparse matrix and vector
    abstractions are built on top of Legion as part of the development
    of a conjugate gradient solver.

## Debugging and Profiling

The following pages describe Legion's debugging and profiling
facilities, and how to use them.

  * [Debugging](/debugging/)
  * [Profiling](/profiling/)
