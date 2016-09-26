---
layout: page
permalink: /tutorial/multiple.html
title: Multiple Partitions
---

We deviate from our running DAXPY application 
in this example to illustrate how Legion permits 
applications to create multiple logical partitions 
of the same logical region, thereby enabling multiple
views onto the same set of data. To do this, we'll
construct a simple program to compute the derivative
of some data by performing a simple 1-D 5-point stencil
on a discretized space using the standard formula.
<br/><br/>
`f'(x) = (-f(x+2) + 8f(x+1) - 8f(x-1) + f(x-2))/12`
<br/><br/>
To perform the stencil computation we create the
`stencil_lr` logical region with two fields: one 
field containing the input values and a second field 
to store the computed derivative value at each point 
(lines 38-48). After creating the logical regions 
we partition the `stencil_lr` logical region in
two different ways which we cover in the next
section. After partitioning the logical region
we initialize the data in our value field using
the `init_field_task` (lines 111-117, identical 
to the one from our DAXPY example). We then launch 
an index space of tasks to perform the stencil 
computation in parallel (lines 119-129). Finally, 
a single task is launched to check the result of 
the stencil computation (lines 131-139).

#### Creating and Using Multiple Partitions ####

To be able to compute the stencil computation in
parallel, we need to create two partitions: one
disjoint partition with each logical sub-region 
describing the points that each tasks will write, 
and a second partition with each logical sub-region
describing the points that each task will need to 
read from to perform the stencil computation.
While the first partition will be disjoint, the
second partition will be _aliased_ because each
sub-region will additionally require two _ghost_ cells on
each side of the set of elements in
each sub-region in the disjoint partition.
The need for these ghost cells means that some
cells will exist in multiple sub-regions and
therefore the partition will be aliased.

We create two separate `DomainColoring` objects
that are used for storing the colorings for
each of our two partitions (line 58). We then
compute the `Rect` value each sub-region for both 
partitions for that we want
to create (lines 62-101). Note there are two
different cases to handle for the disjoint
partition and four for the aliased partition
(see comments in the code for more details).
After we've computed the two colorings we
create the two partitions of the index space
(lines 97-100) and then obtain the corresponding
logical partitions (lines 103-106). The following
figure shows the resulting logical region tree
for the application.

![](/images/stencil_partition.svg)
<br/><br/>
Legion's support of multiple partitions for 
logical regions enables applications to use
different views of the same data even within
the same task. Consider the index space task
launch for performing the stencil computation.
We pass two projection region requirements
in the launcher object. The first region
requirement requests `READ_ONLY` privileges
on the aliased `ghost_lp` logical partition
for the `FID_VAL` field (lines 121-124).
The second region requirement requests
`READ_WRITE` privileges on the `disjoint_lp`
logical partition for the `FID_DERIV` field
(lines 125-128). In the next section we describe
how Legion proves that all of the stencil
tasks can be run in parallel.

#### Privilege Non-Interference ####

To prove that all of the stencil tasks can run
in parallel, the Legion runtime relies on all
three kinds of non-interference including one
we have not yet covered: privilege non-interference. 
First, field non-interference ensures that any
point task's first region requirement is disjoint
from any other point task's second region requirement
as the two region requirements use different fields.
Next, logical region non-interference ensures all 
tasks are non-interfering with respect to every other
task's second region requirement because a disjoint
partition is used. The last case where non-interference
must be shown is with regards to the first region
requirement between any pair of tasks. Neither field-level
nor logical region non-interference applies. However,
since all tasks are only requesting `READ_ONLY`
privileges, then the tasks are non-interfering on
privileges because tasks that are only reading data cannot
interfere with each other. Consequently, all the
point tasks in the stencil index space launch
can be run in parallel.

Next Example: [Custom Mappers](/tutorial/custom_mappers.html)
<br/>
Previous Example: [Partitioning](/tutorial/partitioning.html)

{% highlight cpp linenos %}#include <cstdio>
#include <cassert>
#include <cstdlib>
#include "legion.h"
using namespace LegionRuntime::HighLevel;
using namespace LegionRuntime::Accessor;

enum TaskIDs {
  TOP_LEVEL_TASK_ID,
  INIT_FIELD_TASK_ID,
  STENCIL_TASK_ID,
  CHECK_TASK_ID,
};

enum FieldIDs {
  FID_VAL,
  FID_DERIV,
};

void top_level_task(const Task *task,
                    const std::vector<PhysicalRegion> &regions,
                    Context ctx, HighLevelRuntime *runtime) {
  int num_elements = 1024;
  int num_subregions = 4;
  // Check for any command line arguments
  {
    const InputArgs &command_args = HighLevelRuntime::get_input_args();
    for (int i = 1; i < command_args.argc; i++) {
      if (!strcmp(command_args.argv[i],"-n"))
        num_elements = atoi(command_args.argv[++i]);
      if (!strcmp(command_args.argv[i],"-b"))
        num_subregions = atoi(command_args.argv[++i]);
    }
  }
  printf("Running stencil computation for %d elements...\n", num_elements);
  printf("Partitioning data into %d sub-regions...\n", num_subregions);

  Rect<1> elem_rect(Point<1>(0),Point<1>(num_elements-1));
  IndexSpace is = runtime->create_index_space(ctx, 
                          Domain::from_rect<1>(elem_rect));
  FieldSpace fs = runtime->create_field_space(ctx);
  {
    FieldAllocator allocator = 
      runtime->create_field_allocator(ctx, fs);
    allocator.allocate_field(sizeof(double),FID_VAL);
    allocator.allocate_field(sizeof(double),FID_DERIV);
  }
  LogicalRegion stencil_lr = runtime->create_logical_region(ctx, is, fs);
  
  Rect<1> color_bounds(Point<1>(0),Point<1>(num_subregions-1));
  Domain color_domain = Domain::from_rect<1>(color_bounds);

  IndexPartition disjoint_ip, ghost_ip;
  {
    const int lower_bound = num_elements/num_subregions;
    const int upper_bound = lower_bound+1;
    const int number_small = num_subregions - (num_elements % num_subregions);
    DomainColoring disjoint_coloring, ghost_coloring;
    int index = 0;
    // Iterate over all the colors and compute the entry
    // for both partitions for each color.
    for (int color = 0; color < num_subregions; color++) {
      int num_elmts = color < number_small ? lower_bound : upper_bound;
      assert((index+num_elmts) <= num_elements);
      Rect<1> subrect(Point<1>(index),Point<1>(index+num_elmts-1));
      disjoint_coloring[color] = Domain::from_rect<1>(subrect);
      // Now compute the points assigned to this color for
      // the second partition.  Here we need a superset of the
      // points that we just computed including the two additional
      // points on each side.  We handle the edge cases by clamping
      // values to their minimum and maximum values.  This creates
      // four cases of clamping both above and below, clamping below,
      // clamping above, and no clamping.
      if (index < 2) {
        if ((index+elmts_per_subregion+2) > num_elements) {
          // Clamp both
          Rect<1> ghost_rect(Point<1>(0),Point<1>(num_elements-1));
          ghost_coloring[color] = Domain::from_rect<1>(ghost_rect);
        } else {
          // Clamp below
          Rect<1> ghost_rect(Point<1>(0),Point<1>(index+elmts_per_subregion+1));
          ghost_coloring[color] = Domain::from_rect<1>(ghost_rect);
        }
      } else {
        if ((index+elmts_per_subregion+2) > num_elements) {
          // Clamp above
          Rect<1> ghost_rect(Point<1>(index-2),Point<1>(num_elements-1));
          ghost_coloring[color] = Domain::from_rect<1>(ghost_rect);
        } else {
          // Normal case
          Rect<1> ghost_rect(Point<1>(index-2),Point<1>(index+elmts_per_subregion+1));
          ghost_coloring[color] = Domain::from_rect<1>(ghost_rect);
        }
      }
      index += elmts_per_subregion;
    }
    disjoint_ip = runtime->create_index_partition(ctx, is, color_domain,
                                    disjoint_coloring, true/*disjoint*/);
    ghost_ip = runtime->create_index_partition(ctx, is, color_domain,
                                    ghost_coloring, false/*disjoint*/);
  }

  LogicalPartition disjoint_lp = 
    runtime->get_logical_partition(ctx, stencil_lr, disjoint_ip);
  LogicalPartition ghost_lp = 
    runtime->get_logical_partition(ctx, stencil_lr, ghost_ip);

  Domain launch_domain = color_domain;
  ArgumentMap arg_map;

  IndexLauncher init_launcher(INIT_FIELD_TASK_ID, launch_domain,
                              TaskArgument(NULL, 0), arg_map);
  init_launcher.add_region_requirement(
      RegionRequirement(disjoint_lp, 0/*projection ID*/,
                        WRITE_DISCARD, EXCLUSIVE, stencil_lr));
  init_launcher.add_field(0, FID_VAL);
  runtime->execute_index_space(ctx, init_launcher);

  IndexLauncher stencil_launcher(STENCIL_TASK_ID, launch_domain,
       TaskArgument(&num_elements, sizeof(num_elements)), arg_map);
  stencil_launcher.add_region_requirement(
      RegionRequirement(ghost_lp, 0/*projection ID*/,
                        READ_ONLY, EXCLUSIVE, stencil_lr));
  stencil_launcher.add_field(0, FID_VAL);
  stencil_launcher.add_region_requirement(
      RegionRequirement(disjoint_lp, 0/*projection ID*/,
                        READ_WRITE, EXCLUSIVE, stencil_lr));
  stencil_launcher.add_field(1, FID_DERIV);
  runtime->execute_index_space(ctx, stencil_launcher);

  TaskLauncher check_launcher(CHECK_TASK_ID, 
      TaskArgument(&num_elements, sizeof(num_elements)));
  check_launcher.add_region_requirement(
      RegionRequirement(stencil_lr, READ_ONLY, EXCLUSIVE, stencil_lr));
  check_launcher.add_field(0, FID_VAL);
  check_launcher.add_region_requirement(
      RegionRequirement(stencil_lr, READ_ONLY, EXCLUSIVE, stencil_lr));
  check_launcher.add_field(1, FID_DERIV);
  runtime->execute_task(ctx, check_launcher);

  runtime->destroy_logical_region(ctx, stencil_lr);
  runtime->destroy_field_space(ctx, fs);
  runtime->destroy_index_space(ctx, is);
}

// The standard initialize field task from earlier examples
void init_field_task(const Task *task,
                     const std::vector<PhysicalRegion> &regions,
                     Context ctx, HighLevelRuntime *runtime) {
  assert(regions.size() == 1); 
  assert(task->regions.size() == 1);
  assert(task->regions[0].privilege_fields.size() == 1);

  FieldID fid = *(task->regions[0].privilege_fields.begin());
  const int point = task->index_point.point_data[0];
  printf("Initializing field %d for block %d...\n", fid, point);

  RegionAccessor<AccessorType::Generic, double> acc = 
    regions[0].get_field_accessor(fid).typeify<double>();

  Domain dom = runtime->get_index_space_domain(ctx, 
      task->regions[0].region.get_index_space());
  Rect<1> rect = dom.get_rect<1>();
  for (GenericPointInRectIterator<1> pir(rect); pir; pir++) {
    acc.write(DomainPoint::from_point<1>(pir.p), drand48());
  }
}

void stencil_task(const Task *task,
                  const std::vector<PhysicalRegion> &regions,
                  Context ctx, HighLevelRuntime *runtime)
{
  assert(regions.size() == 2);
  assert(task->regions.size() == 2);
  assert(task->regions[0].privilege_fields.size() == 1);
  assert(task->regions[1].privilege_fields.size() == 1);
  assert(task->arglen == sizeof(int));
  const int max_elements = *((const int*)task->args);
  const int point = task->index_point.point_data[0];
  
  FieldID read_fid = *(task->regions[0].privilege_fields.begin());
  FieldID write_fid = *(task->regions[1].privilege_fields.begin());

  RegionAccessor<AccessorType::Generic, double> read_acc = 
    regions[0].get_field_accessor(read_fid).typeify<double>();
  RegionAccessor<AccessorType::Generic, double> write_acc = 
    regions[1].get_field_accessor(write_fid).typeify<double>();

  Domain dom = runtime->get_index_space_domain(ctx,
      task->regions[1].region.get_index_space());
  Rect<1> rect = dom.get_rect<1>();
  const DomainPoint zero = DomainPoint::from_point<1>(Point<1>(0));
  const DomainPoint max = DomainPoint::from_point<1>(Point<1>(max_elements-1));
  const Point<1> one(1);
  const Point<1> two(2);
  // If we are on the edges of the entire space we are 
  // operating over, then we're going to do the slow
  // path which checks for clamping when necessary.
  // If not, then we can do the fast path without
  // any checks.
  if ((rect.lo[0] == 0) || (rect.hi[0] == (max_elements-1))) {
    printf("Running slow stencil path for point %d...\n", point);
    // Note in the slow path that there are checks which
    // perform clamps when necessary before reading values.
    for (GenericPointInRectIterator<1> pir(rect); pir; pir++) {
      double l2, l1, r1, r2;
      if (pir.p[0] < 2)
        l2 = read_acc.read(zero);
      else
        l2 = read_acc.read(DomainPoint::from_point<1>(pir.p-two));
      if (pir.p[0] < 1)
        l1 = read_acc.read(zero);
      else
        l1 = read_acc.read(DomainPoint::from_point<1>(pir.p-one));
      if (pir.p[0] > (max_elements-2))
        r1 = read_acc.read(max);
      else
        r1 = read_acc.read(DomainPoint::from_point<1>(pir.p+one));
      if (pir.p[0] > (max_elements-3))
        r2 = read_acc.read(max);
      else
        r2 = read_acc.read(DomainPoint::from_point<1>(pir.p+two));
      
      double result = (-l2 + 8.0*l1 - 8.0*r1 + r2) / 12.0;
      write_acc.write(DomainPoint::from_point<1>(pir.p), result);
    }
  } else {
    printf("Running fast stencil path for point %d...\n", point);
    // In the fast path, we don't need any checks
    for (GenericPointInRectIterator<1> pir(rect); pir; pir++) {
      double l2 = read_acc.read(DomainPoint::from_point<1>(pir.p-two));
      double l1 = read_acc.read(DomainPoint::from_point<1>(pir.p-one));
      double r1 = read_acc.read(DomainPoint::from_point<1>(pir.p+one));
      double r2 = read_acc.read(DomainPoint::from_point<1>(pir.p+two));

      double result = (-l2 + 8.0*l1 - 8.0*r1 + r2) / 12.0;
      write_acc.write(DomainPoint::from_point<1>(pir.p), result);
    }
  }
}

void check_task(const Task *task,
                const std::vector<PhysicalRegion> &regions,
                Context ctx, HighLevelRuntime *runtime)
{
  assert(regions.size() == 2);
  assert(task->regions.size() == 2);
  assert(task->regions[0].privilege_fields.size() == 1);
  assert(task->regions[1].privilege_fields.size() == 1);
  assert(task->arglen == sizeof(int));
  const int max_elements = *((const int*)task->args);

  FieldID src_fid = *(task->regions[0].privilege_fields.begin());
  FieldID dst_fid = *(task->regions[1].privilege_fields.begin());

  RegionAccessor<AccessorType::Generic, double> src_acc = 
    regions[0].get_field_accessor(src_fid).typeify<double>();
  RegionAccessor<AccessorType::Generic, double> dst_acc = 
    regions[1].get_field_accessor(dst_fid).typeify<double>();

  Domain dom = runtime->get_index_space_domain(ctx,
      task->regions[1].region.get_index_space());
  Rect<1> rect = dom.get_rect<1>();
  const DomainPoint zero = DomainPoint::from_point<1>(Point<1>(0));
  const DomainPoint max = DomainPoint::from_point<1>(Point<1>(max_elements-1));
  const Point<1> one(1);
  const Point<1> two(2);

  // This is the checking task so we can just do the slow path
  bool all_passed = true;
  for (GenericPointInRectIterator<1> pir(rect); pir; pir++) {
    double l2, l1, r1, r2;
    if (pir.p[0] < 2)
      l2 = src_acc.read(zero);
    else
      l2 = src_acc.read(DomainPoint::from_point<1>(pir.p-two));
    if (pir.p[0] < 1)
      l1 = src_acc.read(zero);
    else
      l1 = src_acc.read(DomainPoint::from_point<1>(pir.p-one));
    if (pir.p[0] > (max_elements-2))
      r1 = src_acc.read(max);
    else
      r1 = src_acc.read(DomainPoint::from_point<1>(pir.p+one));
    if (pir.p[0] > (max_elements-3))
      r2 = src_acc.read(max);
    else
      r2 = src_acc.read(DomainPoint::from_point<1>(pir.p+two));
    
    double expected = (-l2 + 8.0*l1 - 8.0*r1 + r2) / 12.0;
    double received = dst_acc.read(DomainPoint::from_point<1>(pir.p));
    if (expected != received)
      all_passed = false;
  }
  if (all_passed)
    printf("SUCCESS!\n");
  else
    printf("FAILURE!\n");
}

int main(int argc, char **argv) {
  HighLevelRuntime::set_top_level_task_id(TOP_LEVEL_TASK_ID);
  HighLevelRuntime::register_legion_task<top_level_task>(TOP_LEVEL_TASK_ID,
      Processor::LOC_PROC, true/*single*/, false/*index*/);
  HighLevelRuntime::register_legion_task<init_field_task>(INIT_FIELD_TASK_ID,
      Processor::LOC_PROC, true/*single*/, true/*index*/);
  HighLevelRuntime::register_legion_task<stencil_task>(STENCIL_TASK_ID,
      Processor::LOC_PROC, true/*single*/, true/*index*/);
  HighLevelRuntime::register_legion_task<check_task>(CHECK_TASK_ID,
      Processor::LOC_PROC, true/*single*/, true/*index*/);

  return HighLevelRuntime::start(argc, argv);
}
{% endhighlight %}

