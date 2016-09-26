---
layout: page
permalink: /tutorial/custom_mappers.html
title: Custom Mappers
---

One of the primary goals of Legion is to
make it easy to remap applications onto
different architectures. Up to this point
all of our applications have been mapped
by the `DefaultMapper` implementation that
is distributed with Legion. The `DefaultMapper`
class provides heuristics for performing mappings
that are good, but regularly not optimal for
specific applications or architectures. By
creating custom mappers programmers can make
application- or architecture-specific mapping
decisions. Furthermore, the mapping interface
isolates mapping decisions from application
code, allowing the same Legion applications
to be targeted at different architectures without
having to modify the application source code.

A common problem encountered when writing code 
for large distributed and heterogeneous machines 
is how mapping impacts correctness. In Legion all
mapping decisions are orthogonal to correctness.
Legion will always perform the necessary operations
to ensure that data is accessed safely with regards
to the specific privilege and coherence constraints
specified in the application. In cases where
mappers generate invalid responses to mapping
queries (e.g. a mapper maps a physical region for
a task into a memory not visible from the target
processor), then the Legion runtime will notify the
mapper that it tried to perform an illegal mapping
and allow it to retry.

To introduce how to write a custom mapper we'll
implement two custom mappers. The first will be
an adversarial mapper that makes random mapping 
decisions designed to stress-test the Legion runtime.
We'll also have the adversarial mapper report the chosen
mapping decisions (which will change with each run
of the application) to further emphasize the orthogonality
of mapping decisions from correctness. The second mapper
that we'll create will be used for partitioning to decide
how many sub-regions to create using <em>tunable</em> 
variables. The application code for DAXPY is nearly 
identical to the code from 
an [earlier example](/tutorial/partitioning.html)
and is therefore omitted. We only show the new
code required for creating and implementing the
custom mappers.

#### Mapper Objects and Registration ####

Mappers are classes that implement the mapping interface
declared in the abstract `Mapper` objection in the
legion.h header file. Legion provides a default
implementation of this interface defined by the 
`DefaultMapper` class (from the default_mapper.h
header file on line 2). The default mapper allows
applications to be developed without being
concerned with mapping decisions. Once correctness
has been established, programmers can implement
custom mappers to achieve higher performance. 
Custom mappers can either extend the `DefaultMapper`
class or implement the `Mapper` interface from
scratch. In this example, we extend the 
`DefaultMapper` to create both our `AdversarialMapper`
(line 13) and `PartitioningMapper` (line 25). We then override 
four of the mapping interface calls in the `AdversarialMapper`
(lines 18-22) and one in the `PartitioningMapper` (lines 30-32)
to show how they work. We'll detail the semantics and our implementation
of them in the coming sections.

Mappers objects are instantiated after the
Legion runtime starts but before the application
begins executing. To instantiate mappers, the application
registers a callback function for the runtime to
invoke prior to beginning execution of the application. In this
example we create the `mapper_registration`
function (lines 35-45) and register the function
with runtime using the static method `set_registration_callback`
on the `HighLevelRuntime` (line 279). All callback
functions must have the same type as the `mapper_registration`
function which allows the runtime to pass the 
necessary parameters for creating new mappers.

The model for Legion is that a specific kind of
mapper is identified by a `MapperID` (unsigned integer)
and there should be a single instance of each kind of mapper
for every processor in the system. Having a single
instance for each processor guarantees that processors
can map tasks in parallel without needing to be
concerned about contention for a single mapper object.
For mappers that are stateful, this can sometimes
lead to mapper instances having incomplete information,
but this would occur naturally in a distributed system
anyway as there would at least have to be different
mapper instances for processes on different nodes.
In practice, we find that having one mapper of each
kind for every processor is a straight-forward model.

When the callback function is invoked, it can
instantiate an arbitrary number of kinds of mappers.
For each kind, it should create one instance for
every processor in the set `local_procs` which
describes the set of processors on the local node
(the callback is invoked in parallel by the
instance of the Legion runtime in every process
with the correct set of processors local to the process).
Zero is reserved as a special `MapperID` for
the `DefaultMapper`, but applications can replace
the default mapper using the `replace_default_mapper`
method on the `HighLevelRuntime` (lines 40-41).
By replacing the default mapper, our tasks will
automatically use the new `AdversarialMapper`
to handle mapping queries. We register the 
instances of the `PartitioningMapper` with 
the `add_mapper` method and pass
`PARTITIONING_MAPPER_ID` as the `MapperID`
which is set to be `1`. We'll show how to use
mapper IDs to determine which mapper is invoked
momentarily. Finally, notice that we iterate over
all the processors in the set of `local_procs`
to create a separate instance of both the `AdversarialMapper`
and the `PartitioningMapper` for each processor (lines 40-43).

#### Legion Machine Model ####

In order to target a specific architecture, mappers need
access to a description of the underlying hardware.
Legion provides a static singleton `Machine` object
that can be used to make queries concerning the
underlying hardware. We show how to make some of these
queries as part of the constructor for our `AdversarialMapper`
object (lines 47-173).

A pointer to the `Machine` singleton is passed as part
of the arguments to any mapper constructor, but it
can also always be obtained by calling the static method
`get_machine` of the `Machine` type from anywhere in the
application. The `Machine` type
actually comes from the Legion low-level runtime, but
typedefs are used to ensure the proper types are all
available for application use as well. In our constructor
we begin by obtaining a reference to the STL set of
all the processors in the machine (line 51). The
`Processor` objects are simply light-weight handles
for naming all of the processors in the machine. The
set of all processors can depend on which version of
the low-level runtime as well as command line arguments
such as the `-ll:cpu <#>` flag which specifies the
number of CPU processors to create. (It is on our
TODO list to have the low-level runtime inspect the 
hardware to automatically discover the underlying
machine architecture. More details on how to 
configure the Legion low-level runtime can be 
found [here](/profiling/index.html#machine).)

We now want to print out information about the
underlying processors and memories for our machine.
To do this we first add the conditional on line 31
to ensure that only one mapper performs the output.
Recall a separate instance of the `AdversarialMapper`
will be made for every processor. We only have the 
mapper whose local processor is the first one in
the set of all processors do the printing. We then
iterate over the set of all processors and print out
their ID and type (lines 54-83). Most objects
obtained from the `Machine` object have an associated
`id` field that can uniquely identify them (it is
useful to print these IDs in hexadecimal format).
There are currently three types of processors supported
by the Legion runtime: latency-optimized processors
(`LOC_PROC`) are CPU processors, throughput-optimized
processors (`TOC_PROC`) are GPUs, and utility processors
are special CPU processors used for performing Legion
runtime tasks. (Utility processors can also be used
for running Legion tasks, but may suffer from longer
latencies as they will be interleaved with runtime-level
operations.)

We also obtain and print the list of
memories as well as their kinds and sizes (lines 84-158).
Note memory sizes are controlled by command line
values and may not accurately reflect the actual
underlying hardware (e.g. the L1-cache values in
our shared-memory-only low-level runtime). Different
architectures should use different command line
arguments to properly set these values until
Legion learns to discover the underlying hardware.
More information on configuring the Legion low-level
runtime parameters with command line flags can
be found [here](/profiling/index.html#machine).

A useful way to think about the Legion machine
model encapsulated by the `Machine` objects is as a
graph with processors and memories as two kinds
of nodes. There are two kinds of edges in this graph:
processor-memory edges and memory-memory edges. An
edge between a processor and a memory indicates that
the processor can directly perform load and store
operations to that memory. Memory-memory edges indicate
that data movement can be directly performed between
the two memories (either by a processor or by a
DMA engine). 
{::comment}
Below is an example graph for the
shared-memory-only low-level runtime with four
processors (default values). We model separate
L1 caches to give the illusion of blocking for
per-processor caches, but still allow all processors
to access any memory because of cache coherence.
{:/comment}
Edges between nodes are called affinities in the
machine model. Affinities are currently dimensionless
and are approximate indications of the latency
and bandwidth between two nodes (again, having the
low-level runtime automatically discover the
actual values is a long-term goal). Line 160 uses
the `get_visible_memories` method to obtain the
set of memories visible from the local processor.
We then print out the affinities between the local
processor and each of these memories using the
`get_proc_mem_affinity` method (line 167).

#### Assigning Tasks to Processors ####

The first mapper call that we override is the 
`select_task_options` call (lines 175-284). This
mapper call is performed on every task launch
immediately after it is made. The call asks the
mapper to specify some of the important properties
for the task by setting fields on the `Task`
object. (Most mapper queries are answered by 
mutating the mapping fields of the `Task` object.)

* __inline_task__: whether the task should be
  inlined directly into the parent task's context
  by using the parent task's physical regions.
* __spawn_task__: whether the task is eligible
  for stealing (based on Cilk-style semantics).
* __map_locally__: whether the task should be
  mapped by the processor on which it was launched 
  or whether it should be mapped by the processor
  where it will run.
* __profile_task__: should the runtime collect
  profiling information about the task while
  it is executing to provide feedback to the 
  mapper.
* __target_proc__: which processor should the task
  be sent to once all of its mapping dependences
  have been satisfied and it is ready to map.

For our adversarial mapper, we perform the default
choice for all options except the last one. When
we set the `target_proc` field we select a random
processor to which the task will be sent. This
is done using a static utility method from `DefaultMapper`
which will pick a random processor from a given
STL set (lines 182-183).

#### Slicing Index Task Spaces ####

The second call that we override is the `slice_domain`
method (lines 186-205). The `slice_domain` method is
used by the runtime to query the mapper about the 
best way to distribute the point tasks from an index
space task launch through the machine. The mapper is
given the domain to slice and then asked to generate
sub-domains to be sent to different processors in the
form of `DomainSplit` objects which we also refer
to as _slices_. `DomainSplit` objects describe the 
sub-domain, the target processor, whether the slice
can be stolen, and finally whether `slice_domain` should
be recursively invoked on the slice when it arrives
at its destination. Using this call mappers can choose
both the granularity at which index space tasks are
handled and distributed. Furthermore, it also gives
the mapper the ability to hierarchically decompose
an index space by recursively calling `slice_domain`
when distributing domains across very large machines.

For our `AdversarialMapper` implementation we again use 
another utility method from 
the `DefaultMapper` called `decompose_index_space`
to decompose our domain into two slices and send the
two slices to two random processors. We continue
recursively dividing domains in half until there
is only one point in each domain. Lines 188-183 choose
the two random processors and lines 195-196 perform
the slicing. We then check to see how many points
are left in each slice and mark that `domain_split`
should recursively be called unless there is only
one point left (line 197-204). Overall, this creates a
tree of slices of depth log(N) in the number of
points in the domain with each slice in the tree being
sent to a random processor.

#### Selecting Memories for Physical Instances ####

The next mapping call that we override is the `map_task`
method (lines 207-229). Once a task has been assigned
to map on a specific processor (the `target_proc` field
in the `Task` object), then this method is called by
the runtime to select the memories in which to create
the physical instances of the logical regions requested
by the task's `RegionRequirement` objects. The mapper
specifies the target memories by modifying the mapping
fields on the `RegionRequirement` objects (lines 219,
222-225). The memories containing currently valid physical
instances for each `RegionRequirement` are provided
by the runtime to the mapper in the `current_instances`
field. The mapper specifies an ordered list of memories
for the runtime to try when searching for either an 
existing physical instance or creating a new instance
in the `target_ranking` field of each `RegionRequirement`.
If the runtime fails to find or make a physical instance
in any of the memories, then the mapping fails and
the mapper will be notified that the task failed to map
using the `notify_mapping_failed` mapper call. If the
mapper does nothing in the `notify_mapping_failed` call
then the task is placed back on the list of tasks eligible
to be selected for mapping by the `select_tasks_to_schedule`
mapper call. In addition to the `target_ranking` field
there are other fields which the mapper can set which
we do not cover here (lines 222-225).

For our `AdversarialMapper` implementation, the mapper
finds the set of all memories visible from the target
processor and then puts them in a random order as the
target ranking for the runtime to use, thereby challenging
the Legion runtime to maintain correctness of data
that will need to be moved through random sets of memories
as each task runs. Note finally that we return true
as the result of the `map_task` method. This instructs
the runtime to call the `notify_mapping_result`
method if the mapping of the task succeeds.

#### Reporting Results ####

The last mapper call that we override for our
`AdversarialMapper` is the 
`notify_mapping_result` method (lines 231-242).
We use this method to report the chosen mapping
for each task, but it is also useful for memoizing
mapping results and for knowing the result of mapping
decisions when profiling tasks. Before invoking
this method, the runtime sets the `selected_memory`
field on each `RegionRequirement` object in the
vector for each `Task`. The mapper can then see
in which memory a physical instance for each 
`RegionRequirement` resides. For our `AdversarialMapper`
implementation we print the mapping for each logical
region of every task to show that the assignment
of regions to memories is truly random. Every run
of the application should report a different
mapping, but will always report that the correct
answer is computed illustrating the Legion mapping
decisions are orthogonal to correctness.


#### Handling Tunable Variables ####

When writing tasks, there are often many cases where
variables depend on the underlying nature of the 
machine. In Legion we refer to these variables as
<em>tunable variables</em> because they often need
to be specifically tuned to different architectures.
Since these variables are machine dependent and likely
to affect performance, we prefer to make these variables
explicit. To do this we provide a separate mapping call
`get_tunable_value` to explicitly request that the
mapper pick the value for this variable. We override
this call in our `PartitioningMapper` on lines 30-32.

We make a slight modification to our DAXPY code to
make the number of sub-regions to create a tunable 
variable so that the mapper can pick a value based 
on the number of processors on the target machine. Note that the
`top_level_task` explicitly invokes the `get_tunable_value`
to find the number of sub-regions to create (see
the code sample below). When this call is made, we
pass the `PARTITIONING_MAPPER_ID` as the value to the
`MapperID` field, indicating that an instance of
our `PartitioningMapper` should be used to handle 
the request and not an instance of the `AdversarialMapper`.
When we make the call we also pass in a `TunableID`
which is used to identify the name of the tunable
variable that should be set. This value can by
static or dynamic, so long as the mapper that it
is being sent to knows how to determine it. In
this case we pass `SUBREGION_TUNABLE` as the
integer name for tunable variable. On lines
251-264 we can see the implementation of the
`get_tunable_value` method for the `PartitioningMapper`.
We see that this mapper class knows how handle
queries for values of the `SUBREGION_TUNABLE`
tunable variable only. The `PartitioningMapper`
instance looks up the total number of processors
in the machine and specifies that as the number
of sub-regions to create.

{% highlight cpp %}
int num_subregions = 
        runtime->get_tunable_value(ctx, SUBREGION_TUNABLE, 
                                   PARTITIONING_MAPPER_ID);
{% endhighlight %}

For the moment tunable variables must be unsigned
integers, but this is primarily a side effect of
C++ not supporting templated virtual methods.
If users have compelling examples of code that
needs other types of tunable variables then 
please let us know.

#### What Next? ####

Congratulations! You've reached the end of the
Legion tutorial as it currently exists. There
are many features already implemented in Legion 
which are not covered by this tutorial including:

* reduction operations, reduction privileges,
  and reduction instances
* high performance region accessors
* runtime debugging modes and flags
* runtime performance tuning knobs
* unmapping and remapping optimizations
* explicit cross-region copy operations
* additional mapping calls and settings
* close operations and composite instances (pending)
* mapper profiling tools
* relaxed coherence modes
* acquire and release operations for simultaneous coherence
* reservations and phase barriers for synchronization
  in a deferred execution model
* predicated operations
* support for speculative execution (in progress)
* inner and idempotent tasks
* efficient data-centric resiliency and recovery (in progress)

If you are interested in learning more about how to
use these features of Legion or you have questions
regarding how to use them, please post to the
[discussion board](http://legion.stanford.edu/forum).
<br/><br/>
Previous Example: [Multiple Partitions](/tutorial/multiple.html)

{% highlight cpp linenos %}#include "legion.h"
#include "default_mapper.h"

enum {
  SUBREGION_TUNABLE,
};

enum {
  PARTITIONING_MAPPER_ID = 1,
};


class AdversarialMapper : public DefaultMapper {
public:
  AdversarialMapper(Machine *machine, 
      HighLevelRuntime *rt, Processor local);
public:
  virtual void select_task_options(Task *task);
  virtual void slice_domain(const Task *task, const Domain &domain,
                            std::vector<DomainSplit> &slices);
  virtual bool map_task(Task *task); 
  virtual void notify_mapping_result(const Mappable *mappable);
};

class PartitioningMapper : public DefaultMapper {
public:
  PartitioningMapper(Machine *machine,
      HighLevelRuntime *rt, Processor local);
public:
  virtual int get_tunable_value(const Task *task,
                                TunableID tid,
                                MappingTagID tag);
};

void mapper_registration(Machine *machine, HighLevelRuntime *rt,
                          const std::set<Processor> &local_procs) {
  for (std::set<Processor>::const_iterator it = local_procs.begin();
        it != local_procs.end(); it++)
  {
    rt->replace_default_mapper(
        new AdversarialMapper(machine, rt, *it), *it);
    rt->add_mapper(PARTITIONING_MAPPER_ID,
        new PartitioningMapper(machine, rt, *it), *it);
  }
}

AdversarialMapper::AdversarialMapper(Machine *m, 
                                     HighLevelRuntime *rt, Processor p)
  : DefaultMapper(m, rt, p) // pass arguments through to DefaultMapper
{
  const std::set<Processor> &all_procs = machine->get_all_processors();
  if ((*(all_procs.begin())) == local_proc) {
    printf("There are %ld processors:\n", all_procs.size());
    for (std::set<Processor>::const_iterator it = all_procs.begin();
          it != all_procs.end(); it++)
    {
      // For every processor there is an associated kind
      Processor::Kind kind = machine->get_processor_kind(*it);
      switch (kind) {
        // Latency-optimized cores (LOCs) are CPUs
        case Processor::LOC_PROC:
          {
            printf("  Processor ID %x is CPU\n", it->id); 
            break;
          }
        // Throughput-optimized cores (TOCs) are GPUs
        case Processor::TOC_PROC:
          {
            printf("  Processor ID %x is GPU\n", it->id);
            break;
          }
        // Utility processors are helper processors for
        // running Legion runtime meta-level tasks and 
        // should not be used for running application tasks
        case Processor::UTIL_PROC:
          {
            printf("  Processor ID %x is utility\n", it->id);
            break;
          }
        default:
          assert(false);
      }
    }
    const std::set<Memory> &all_mems = machine->get_all_memories();
    printf("There are %ld memories:\n", all_mems.size());
    for (std::set<Memory>::const_iterator it = all_mems.begin();
          it != all_mems.end(); it++) {
      Memory::Kind kind = machine->get_memory_kind(*it);
      size_t memory_size_in_kb = machine->get_memory_size(*it) >> 10;
      switch (kind) {
        // RDMA addressable memory when running with GASNet
        case Memory::GLOBAL_MEM:
          {
            printf("  GASNet Global Memory ID %x has %ld KB\n", 
                    it->id, memory_size_in_kb);
            break;
          }
        // DRAM on a single node
        case Memory::SYSTEM_MEM:
          {
            printf("  System Memory ID %x has %ld KB\n",
                    it->id, memory_size_in_kb);
            break;
          }
        // Pinned memory on a single node
        case Memory::REGDMA_MEM:
          {
            printf("  Pinned Memory ID %x has %ld KB\n",
                    it->id, memory_size_in_kb);
            break;
          }
        // A memory associated with a single socket
        case Memory::SOCKET_MEM:
          {
            printf("  Socket Memory ID %x has %ld KB\n",
                    it->id, memory_size_in_kb);
            break;
          }
        // Zero-copy memory betweeen CPU DRAM and
        // all GPUs on a single node
        case Memory::Z_COPY_MEM:
          {
            printf("  Zero-Copy Memory ID %x has %ld KB\n",
                    it->id, memory_size_in_kb);
            break;
          }
        // GPU framebuffer memory for a single GPU
        case Memory::GPU_FB_MEM:
          {
            printf("  GPU Frame Buffer Memory ID %x has %ld KB\n",
                    it->id, memory_size_in_kb);
            break;
          }
        // Block of memory sized for L3 cache
        case Memory::LEVEL3_CACHE:
          {
            printf("  Level 3 Cache ID %x has %ld KB\n",
                    it->id, memory_size_in_kb);
            break;
          }
        // Block of memory sized for L2 cache
        case Memory::LEVEL2_CACHE:
          {
            printf("  Level 2 Cache ID %x has %ld KB\n",
                    it->id, memory_size_in_kb);
            break;
          }
        // Block of memory sized for L1 cache
        case Memory::LEVEL1_CACHE:
          {
            printf("  Level 1 Cache ID %x has %ld KB\n",
                    it->id, memory_size_in_kb);
            break;
          }
        default:
          assert(false);
      }
    }

    const std::set<Memory> vis_mems = machine->get_visible_memories(local_proc);
    printf("There are %ld memories visible from processor %x\n",
            vis_mems.size(), local_proc.id);
    for (std::set<Memory>::const_iterator it = vis_mems.begin();
          it != vis_mems.end(); it++) {
      std::vector<ProcessorMemoryAffinity> affinities;
      int results = 
        machine->get_proc_mem_affinity(affinities, local_proc, *it);
      assert(results == 1);
      printf("  Memory %x has bandwidth %d and latency %d\n",
              it->id, affinities[0].bandwidth, affinities[0].latency);
    }
  }
}

void AdversarialMapper::select_task_options(Task *task) {
  task->inline_task = false;
  task->spawn_task = false;
  task->map_locally = false;
  task->profile_task = false;
  task->task_priority = 0;
  const std::set<Processor> &all_procs = machine->get_all_processors();
  task->target_proc = 
    DefaultMapper::select_random_processor(all_procs, Processor::LOC_PROC, machine);
}

void AdversarialMapper::slice_domain(const Task *task, const Domain &domain,
                                     std::vector<DomainSplit> &slices) {
  const std::set<Processor> &all_procs = machine->get_all_processors();
  std::vector<Processor> split_set;
  for (unsigned idx = 0; idx < 2; idx++) {
    split_set.push_back(DefaultMapper::select_random_processor(
                        all_procs, Processor::LOC_PROC, machine));
  }

  DefaultMapper::decompose_index_space(domain, split_set, 
                                        1/*splitting factor*/, slices);
  for (std::vector<DomainSplit>::iterator it = slices.begin();
        it != slices.end(); it++) {
    Rect<1> rect = it->domain.get_rect<1>();
    if (rect.volume() == 1)
      it->recurse = false;
    else
      it->recurse = true;
  }
}

bool AdversarialMapper::map_task(Task *task) { 
  const std::set<Memory> &vis_mems = 
      machine->get_visible_memories(task->target_proc);  
  assert(!vis_mems.empty());
  for (unsigned idx = 0; idx < task->regionssize(); idx++) {
    std::set<Memory> mems_copy = vis_mems;  
    // Assign memories in a random order
    while (!mems_copy.empty()) {
      unsigned mem_idx = (lrand48() % mems_copy.size());
      std::set<Memory>::iterator it = mems_copy.begin();
      for (unsigned i = 0; i < mem_idx; i++)
        it++;
      task->regions[idx].target_ranking.push_back(*it);
      mems_copy.erase(it);
    }
    task->regions[idx].virtual_map = false;
    task->regions[idx].enable_WAR_optimization = false;
    task->regions[idx].reduction_list = false;
    task->regions[idx].blocking_factor = 1;
  }
  // Report successful mapping results
  return true;
}

void AdversarialMapper::notify_mapping_result(const Mappable *mappable) {
  if (mappable->get_mappable_kind() == Mappable::TASK_MAPPABLE) {
    const Task *task = mappable->as_mappable_task();
    assert(task != NULL);
    for (unsigned idx = 0; idx < task->regions.size(); idx++) {
      printf("Mapped region %d of task %s (ID %lld) to memory %x\n",
              idx, task->variants->name, 
              task->get_unique_task_id(),
              task->regions[idx].selected_memory.id);
    }
  }
}

PartitioningMapper::PartitioningMapper(Machine *m,
                                       HighLevelRuntime *rt,
                                       Processor p)
  : DefaultMapper(m, rt, p)
{
}

int PartitioningMapper::get_tunable_value(const Task *task,
                                          TunableID tid,
                                          MappingTagID tag)
{
  if (tid == SUBREGION_TUNABLE)
  {
    const std::set<Processor> &cpu_procs = 
      machine_interface.filter_processors(Processor::LOC_PROC);
    return cpu_procs.size();
  }
  // Should never get here
  assert(false);
  return 0;
}

int main(int argc, char **argv) {
  HighLevelRuntime::set_top_level_task_id(TOP_LEVEL_TASK_ID);
  HighLevelRuntime::register_legion_task<top_level_task>(TOP_LEVEL_TASK_ID,
      Processor::LOC_PROC, true/*single*/, false/*index*/);
  HighLevelRuntime::register_legion_task<init_field_task>(INIT_FIELD_TASK_ID,
      Processor::LOC_PROC, true/*single*/, true/*index*/);
  HighLevelRuntime::register_legion_task<daxpy_task>(DAXPY_TASK_ID,
      Processor::LOC_PROC, true/*single*/, true/*index*/);
  HighLevelRuntime::register_legion_task<check_task>(CHECK_TASK_ID,
      Processor::LOC_PROC, true/*single*/, true/*index*/);

  // Here is where we register the callback function for 
  // creating custom mappers.
  HighLevelRuntime::set_registration_callback(mapper_registration);

  return HighLevelRuntime::start(argc, argv);
}
{% endhighlight %}
