---
layout: page
permalink: /projects/index.html
title: Projects
---

#### Infrastructure ####

 * __Update Sapling__: Finish updating sapling nodes.
 * __Testing__: Add testing infrastructure for both
    high-level and low-level runtimes.

#### High-Level Runtime ####

 * __Composite Instances__: Add support for composite
    instances so we can avoid close copy operations
    and instead lazily generate new instances in 
    different partitions.  Need intersection copies
    from the low-level runtime.
 * __Compound Region Requirements__: Add support for
    region requirements which can take multiple
    regions as well as asking for multiple fields
    on the same region with different privileges.
 * __Resiliency__: Implement pathways for handling
    mis-speculation and reported failures.  Need
    support from low-level runtime for poison bits.
 * __Lua/Terra Support__: Add support for moving 
    around Terra functions, registering generators,
    etc.  Need to see low-level interface.
 * __Union/Intersection Regions__: Probably a little
    ways off since we don't know what the
    abstractions we really want look like yet.
 * __Dependent Partitioning__: Support for doing
    partitions as functions of existing data inside
    of logical regions (e.g. image/pre-image operations).
 * __Mapper Introspection Interface__: Add support
    for the mapper to introspect region trees,
    memory usage, processor usage, etc.
 * __Instance Layout Modifications__: Update the
    high-level runtime to handle the new instance
    layout API once it exists.
 * __Support for Task Fusing__: Add support for
    mapper controlled dynamic fusing of tasks
    and other operations.
 * __Event Graph Optimization__: Perform optimizations
    on the low-level event graph after it has been
    constructed. This could include doing things like
    copy elimination in a global context or identifying
    critical paths.
 * __Replicated Tasks__: Allow for duplicating tasks
    to generate multiple instances with the same version
    number instead of having one task generate the instance
    and then copying. Draw inspiration from Halide's
    approach to avoiding communication by performing
    extra computation.
 * __Aliased Region Requirements__: Give a semantics
    for what happens when we have aliased region requirements.

#### Low-Level Runtime ####

 * __Lua/Terra Support__: Add support for Lua/Terra
    processors on every node as well as the ability
    to JIT Terra functions and give back an event
    for when the function is actually done JIT-ing
    and ready to use.
 * __Intersection Copies__: Apparently this already
    exists, but it needs to be exposed for the
    high-level runtime to use for composite instances.
 * __Instance Layout API__: Propose how to describe
    instance layouts including serialization order,
    field blocking, etc.  Figure out how this interacts
    with Lua/Terra and accessors.
 * __NUMA Awareness__: Add support for making NUMA
    domains explicit memories.
 * __Processor Groups__: Add support for user directed
    creation of groups of processor queues.
 * __OpenGL Support for Viz__: Create visualization
    processors that are running OpenGL.  Incorporate
    support for instances that are OpenGL primitive
    buffers.
 * __Profiling Support__: Figure out the interesting
    sets of performance primitives that the runtime
    can capture for a task.  Expose them so we can
    make them available to the mapper interface.
 * __Resiliency Support__: Determine abstractions
    for capturing error detection signals from
    both hardware and software.  Figure out interface
    for telling the high-level runtime.  Add support
    for poison bits in the event graph.
 * __Multi-Hop Copies__: Figure out how to handle
    copies between memories which aren't immediately
    adjacent.
 * __Dynamic Phase Barriers__: Fix outstanding bug
    with phase barriers that doesn't allow the
    pending arrival count to fluctuate both up and down.
 * __Distributed Instance Allocators__: Be able to 
    support distributed pointer allocation for index spaces.
 * __Composite Accessors__: Support for accessors that
    know about multiple instances (possible with
    different layouts), so that we don't have to
    build a single instance for every region requirement.
 * __Persistent Memories__: Figure out how to encompass
    disks, solid-state drives, and NVRAM as different
    kinds of memories.
 * __Modular Low-Level Rollout__: Actually roll-out
    the new modular low-level runtime so we can plug
    and play different hardware components.
 * __Communication Layer__: Get rid of GASNet and
    build our own communication layer with our own
    flow control.
 * __Library__: Make the Legion runtime an explicit
    stand-alone library so we don't have to have
    our dumb build system.
 * __Auto-Discovery__: Make the low-level runtime smart
    enough to automatically introspect the hardware
    and create the machine object without programmers
    needing to specify it through command line flags.

#### Mapper Interface ####

 * __Instance Layout Interface__: Update the mapper
    interface to handle the instance layout API
    once it exists.
 * __Fusing Interface__: Update the mapper interface
    to have a call for deciding when operations
    should be dynamically fused together.
 * __Update Default Mapper Implementation__: Update
    default mapper implementation to better reflect
    how applications should be mapped based on everything
    we've learned about mapping as well as new 
    runtime features.
 * __Mapping Tools Update__: Clean-up the mapping
    tools interface and internalize common mapping
    idioms from example mappers.

#### Documentation ####

 * __Performance Tuning Tutorial__: Write the
    tutorial on how to go about tuning legion
    applications for various architectures.
 * __Mapper Tutorial__: Write the tutorial on 
    how to build Legion mappers.  Give examples
    of all the Legion mapper calls.

