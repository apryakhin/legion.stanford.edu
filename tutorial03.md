---
layout: page
permalink: /tutorial/hybrid.html
title: Hybrid Programming Model
---

While using the Legion runtime API enables 
programmers to write Legion applications 
in C++ there are several restrictions which 
must be followed. In this short example we 
illustrate the hybrid nature of the Legion 
programming model and call attention to the 
necessary conventions which Legion C++ 
applications must follow. 

#### Programming Model ####

By design Legion is a hybrid programming model
that is coarse-grained functional and fine-grained
imperative. Similar to functional programming
languages like ML and Haskell, Legion tasks are
pure with covariant and contra-variant arguments
being passed in and out by value respectively. The 
one difference between functions in functional 
languages and Legion tasks is that Legion tasks 
are permitted to have declared side-effects on
logical regions. In this way logical regions
can operate as heaps with tasks mutating their
state. We'll see examples of how this works
in later examples in this tutorial.<br/> 
<br/>
Unlike pure-functional models, the Legion 
programming model allows
for the implementation of tasks to be either
functional or imperative. This freedom makes
it possible to implement the Legion programming 
model in C++. However, it also places some
restrictions on how Legion C++ applications
must be structured.

#### Global Variables and Constants ####

One of the fundamental restrictions of the
Legion programming model is that no global
variables are permitted (line 11). This is necessary
to comply with the functional nature of the
Legion programming model. Furthermore, since
there is no way for the runtime to know
global variables exist, then there is no
mechanism for Legion to ensure that global variables 
can be accessed by tasks running on different
nodes in a distributed system. Variables which
need to be accessed by many tasks should
be allocated in logical regions which we
cover in the next example.

The one exception to rule concerning global
variables is that global constants are
permitted (line 14). Since the values of
global constants remain the same throughout
the execution of an application on all
nodes, then they are safe to use despite
the runtime's ignorance concerning their
existence.

One important detail to be cognizant of when
writing Legion applications is that function
pointers are another form of global variable.
Function pointer values, such as the pointer
to the function `foo` (line 16), can have 
different values in different processes
executing in a distributed system. Therefore
passing function pointers around Legion
applications is strongly discouraged. Instead
Legion provides method calls for registering
tasks and other useful functions (e.g. reduction
functions) statically with the runtime before
the `start` method is invoked in the `main`
function.

#### Other C++ Restrictions ####

There are several other restrictions regarding
the usage of C and C++ features in Legion
applications. In general, Legion applications
should not allocate memory directly using
C or C++ conventions (lines 29-30).
Instead logical regions should be used for storing
data that needs to be persistent across sub-tasks
or escapes from a task's context. The one exception 
to this is that tasks can use C and C++ memory 
allocation routines as long as the lifetimes of 
the allocations do not exceed the lifetime of the 
task. Furthermore, any pointers referencing the 
allocation should not be passed to sub-tasks or
escape the task's context. Violating either
of the these conditions will result in a Legion
application with undefined behavior.

Most other restrictions on the usage of C
and C++ features are concerned with when it
is safe to store pointers or references to
physical instances of logical regions and
will be covered in a later example. Applications
are free to use all other features
of C and C++ within Legion tasks.

Next Example: [Logical Regions](/tutorial/logical_regions.html)
<br/>
Previous Example: [Index Space Tasks](/tutorial/index_tasks.html)

{% highlight cpp linenos %}#include <cstdio>
#include "legion.h"

using namespace LegionRuntime::HighLevel;

enum TaskIDs {
  TOP_LEVEL_TASK_ID,
};

// ILLEGAL
int global_var = 0;

// LEGAL
const int global_constant = 4;

void foo(void) {

}

void top_level_task(const Task *task,
                    const std::vector<PhysicalRegion> &regions,
                    Context ctx, HighLevelRuntime *runtime) {
  printf("The value of global_var %d is undefined\n", global_var);

  printf("The value of global_constant %d will always be the same\n", global_constant);

  printf("The function pointer to foo %p may be different on different processors\n", foo);

  void *some_memory = malloc(16*sizeof(int));
  free(some_memory);
}

int main(int argc, char **argv) {
  HighLevelRuntime::set_top_level_task_id(TOP_LEVEL_TASK_ID);
  HighLevelRuntime::register_legion_task<top_level_task>(TOP_LEVEL_TASK_ID,
      Processor::LOC_PROC, true/*single*/, false/*index*/);
  return HighLevelRuntime::start(argc, argv);
}
{% endhighlight %}
