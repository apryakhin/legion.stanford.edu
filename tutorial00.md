---
layout: page
permalink: /tutorial/hello_world.html
title: Hello World 
---
No tutorial would be complete without a Hello World example. 
Below is the source code for writing Hello World using
the Legion C++ runtime interface. The source code can also
be found in the 'examples' directory of the repository. We
now cover the details of the various Legion runtime API
calls in this example.
{% highlight cpp linenos %}#include <cstdio>
#include "legion.h"

using namespace LegionRuntime::HighLevel;

enum TaskID {
  HELLO_WORLD_ID,
};

void hello_world_task(const Task *task, 
                      const std::vector<PhysicalRegion> &regions,
                      Context ctx, HighLevelRuntime *runtime) {
  printf("Hello World!\n");
}

int main(int argc, char **argv) {
  HighLevelRuntime::set_top_level_task_id(HELLO_WORLD_ID);
  HighLevelRuntime::register_legion_task<hello_world_task>(HELLO_WORLD_ID,
      Processor::LOC_PROC, true/*single*/, false/*index*/);

  return HighLevelRuntime::start(argc, argv);
}
{% endhighlight %}

#### Legion Namespaces ####

All Legion programs begin by including the `legion.h` header 
file (line 2). This file defines the entire Legion C++ runtime 
API.  Line 4 imports the `LegionRuntime::HighLevel`
namespace into the current program making most of the necessary
types for writing Legion programs available.  There are other
Legion runtime namespaces that we will encounter in later examples.
Detailed documentation for Legion runtime namespaces can be
found [here](/doxygen/annotated.html).

#### Registering Legion Tasks ####

On lines 6-8 we define an enumeration for storing the IDs that
we will direct the Legion runtime to associate with each task.  In this example
we only need the single ID which we will associate with our task
that will print 'Hello World'.  Lines 18-19 show how to register 
a Legion task with a `void` return type with the Legion runtime.
In the [next example](/tutorial/tasks_and_futures.html) we'll see how
to register tasks with non-void return types.  The static method 
`register_legion_task` on the `HighLevelRuntime` class
is templated on the function pointer to
be called to run the task (allowing the API to handle polymorphic return
types elegantly).  In this case it is templated on the function pointer to 
the `hello_world_task` on lines 10-14.  Note that all Legion tasks must have 
the same function type (with the exception of different return types).  
We'll look in detail at the arguments passed to Legion tasks in later 
examples.  There `register_legion_task` call takes several parameters: 

* Task ID - the ID to associate with this task.
* Processor kind - does the task run on latency-optimized cores (CPUs)
                   or throughput-optimized cores (GPUs)
* Single - can the task be run as an individual task
           (discussed in the [next example](/tutorial/tasks_and_futures.html))    
* Index - can the task be run as an index space task
           (discussed in a [following example](/tutorial/index_tasks.html))

There are additional parameters that can be passed to `register_legion_task`
call as can be seen in the [documentation](/doxygen/class_legion_runtime_1_1_high_level_1_1_high_level_runtime.html#ab1637aefa97d58e7f066ef43dd56b5a2).
We'll examine some of these additional parameters in later examples.

All Legion programs begin with a _top-level_ task which starts off
a Legion program.  The Legion runtime must be told which task to 
use as the top-level task using the `set_top_level_task_id` static 
method.  Line 17 invokes this method to register the ID we associate
with the `hello_world_task` as being the ID of the top-level task.
The top-level task must always be capable of being run as a single task.

#### Legion Runtime Start-Up ####

When writing Legion programs, all the static member function 
invocations on the `HighLevelRuntime` class must be performed
before the static method `start` is called.  It is imperative that 
this standard be followed for all Legion programs to ensure that 
they operate properly when running as MPI or GASNet programs.  While 
this restriction currently mandates that all Legion tasks be statically 
registered before starting the Legion runtime, forthcoming versions 
of Legion will support dynamically registering JIT-compiled 
[Terra](http://terralang.org) functions as
Legion tasks.  After the runtime has been setup to run, the program
invokes the `start` static method on the `HighLevelRuntime` class.
This call starts the Legion runtime and only returns an error code
if a problem is encountered during execution.

After `start` is called, the Legion runtime will perform any necessary
setup operations and will then invoke the top-level task.  In this
program the top-level task prints "Hello World" and then returns.  After
the top-level task completes, the Legion runtime will tear itself down
gracefully (abiding by GASNet or MPI conventions when necessary) and exit.

Next Example: [Tasks and Futures](/tutorial/tasks_and_futures.html)

