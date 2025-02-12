<style type="text/css">
p {text-align:justify}
li {text-align:justify}
blockquote.note
{
background-color:#E0E0E0;
padding-left: 15px;
padding-right: 15px;
padding-top: 1px;
padding-bottom: 1px;
}
code
{
color:#000000;
}
ins {background-color:#A0FFA0}
del {background-color:#FFA0A0}
table {border-collapse: collapse;}
table, th, td {
border: 1px solid black;
border-collapse: collapse;
}
</style>

| Document Number: | d3624r0            |
| ---------------- | ------------------ |
| Date:            | 2025-02-11         |
| Target:          | SG1, LEWG          |
| Revises:         |                    |
| Reply to:        | Gor Nishanov (gorn@microsoft.com), Lucian Radu Teodorescu (lucteo@lucteo.ro) |


# get_system_scheduler (with concurrent forward progress*)

## Abstract 

A system context is an execution resource in std::execution that offers concurrent forward progress guarantees. An instance of <code>system_scheduler</code> allows scheduling work on the system context. 
There is exactly one system context within a program.
A system context represents a shared process wide thread pool implementation
and/or an interface to an OS-provided system thread pool.

## Overview

Win32 and Darwin offer platform supported global system threadpools with the following features:
1. Post work items for execution.
2. Schedule a work item to run at a particular time (or after a delay).
3. Associate an I/O operation with a threadpool, so that the handler processing completion will be executed by the threadpool.
4. Bulk execution (Darwin only), though a bulk algorithm running over non-post work item results in better performance (due to ability to chunk, inline and vectorize loops of chunks, whereas bulk execution in libdispatch is behind ABI boundary and such optimizations are not easily possible).
5. Maintain an optimal number of threads processing work items:
   1. If a thread gets blocked, another one is released/created to maintain the desired number of active threads.
   2. The number of active threads is kept proportional to the number of cores.
   3. Guards against thread explosion (when newly created threads get blocked).
   4. Shrinks the number of threadpool threads to zero when not needed.

On Windows and Darwin platforms, optimal of number of threads is attained via cooperation with the operating system. On **Linux**, Apple's libdispatch library achieve similar characteristics via periodic sampling of the state of threadpool threads by reading /proc/self/task/TID/stat

For C++26, we only offer a system scheduler that supports posting for execution a single or a bulk work items.
Implementations are encouraged to expose `native_handle()` member on a scheduler that would allow
adding timer and I/O injections into C++26 threadpool while standardization catches up.

## WG21 Lore

This paper is a spin off from https://wg21.link/p2079r6 System execution context paper, based on SG1 session in Hagenberg 2025.

> A function called get_system_scheduler must return a concurrent progress guarantee scheduler with implementation limits on the number of execution agents (Gor to prove to us that Windows and Mac, and libdispatch on Linux live up to this – up to a limit at least larger than hardware_concurrency).

SF | F | N | A | SA
---|---|---|---|---
3 | 2 | 5 | 1 | 0

Consensus

Forward P2079R6 to LEWG with the following changes towards C++26:

> 1. Move the name get_system_scheduler to a separate paper, we hope with a stronger progress guarantee.

> 2. Rename the current get_system_scheduler to something with ‘parallel’ in the name, like get_parallel_scheduler, and retain the property that it has the parallel progress guarantee.

> 3. Remove run-time replaceability.

SF | F | N | A | SA
---|---|---|---|---
5 | 5 | 0 | 1 | 0

Consensus

## Testing platform threadpools

| Platform | Add thread on block | Add thread on long running CPU bound task | max limit
|----------|-----------------|-----------------------------------------|----------
| Windows  | yes             | yes (after 600ms queue not moving, possibly with backoff) | 500 (with hardware concurrency 2)
| Darwin   | yes             | not observed                            | 64 (with hardware concurrency 12)
| Linux Libdispatch | yes    | not observed                            | 500 (with hardware concurrency 16)     

<!--

## Discussion

### Is it truly concurrent (*)?

One question was raised whether a parallel execution context with a very large number of threads (100 x std::hardware_concurrency()) can act like
windows or darwin threadpools (or libdispatch on Linux). Yes, it can, but not as efficient. The benefits of elastic threadpools that they dynamically optimize for workload
running at the moment, mininizing context switches and the working set of the process. It is true, any elastic threadpool has a limit for maximum amount of threads it can create and thus would violate concurrent forward progress when the limit is reached, but, so is launching a std::thread for every work item. Eventually, we will run out of memory. In practice elastic threadpools behave like concurrent execution context for common workloads and thus is a valuable facility.

### Why system context is not replaceable

Implementations are free to allow replaceability on particular platform.
Standard should not mandate replaceability of the global thread process-wide threadpool.

System scheduler exists to let you compose components that share threads without
being aware of each other. Some components are parts of the OS, or libraries
not rewritten in C++. 
Replaceability does not fulfill the goals of being a shared system for the process.
Even if we exclude all code not in C++, libraries are testing themselves with a
 system scheduler if replaced, unknown how they would work. 
-->

## Overview

At a glance:

```c++
// at a glance
system_scheduler get_system_scheduler();

class system_scheduler() {
public:
  bool operator==(const system_scheduler&) const noexcept
  { 
    return true;
  }
  forward_progress_guarantee get_forward_progress_guarantee() noexcept
  {
    return forward_progress_guarantee::concurrent;
  }

  sender auto schedule();
  sender auto bulk(integral auto i, auto f);
};
```

Windows:

```c++
class system_scheduler {
   ...
   using native_handle_type = TP_CALLBACK_ENVIRON*;
   native_handle_type native_handle() { return nullptr; }

   friend system_scheduler get_system_scheduler();
private:
   explicit system_scheduler() {}
};
```

Darwin
```c++
class system_scheduler {
   ...
   using native_handle_type = dispatch_queue_t;
   native_handle_type native_handle();
 
   friend system_scheduler get_system_scheduler();
private:
   explicit system_scheduler() {}
   native_handle_type handle = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
};
```

## Implementation experience

System threadpool has been extensively used on Windows and Darwin platforms (and less so, on Linux, using Apple's libdispatch implementation) we consider
their usefulness and production use as proven. Here we offer only a thin interface adapter that exposes system threadpool via sender/receive compatible scheduler API. <!-- The interface is similar to libunifex `static_threadpool` that has been in use at Meta and Microsoft for years, the only difference is that system context dynamically adjust number of threads to satisfy the workload whereas static_threadpool has a fixed number of threads.
-->

## Wording (relative to n5001)

In section [version.syn] add `__cpp_lib_system_scheduler` definition as follows:

<div style="margin-left: 30px;">
<code>
#define __cpp_lib_syncbuf                           201803L // also in &lt;syncstream&gt;<br>
<ins>
#define __cpp_lib_system_scheduler                  2025XXL // also in &lt;execution&lt;<br>
</ins>
#define __cpp_lib_text_encoding                     202306L // also in &lt;text_encoding&gt;
<ins>
</code>
</div>

<p></p>

In [execution.syn], add the following at the end of the code block:

<code>
&nbsp;&nbsp;class run_loop;<br><br>
<ins>
&nbsp;&nbsp;// [exec.system.scheduler]<br>
&nbsp;&nbsp;class system_scheduler { unspecified };<br>
&nbsp;&nbsp;system_scheduler get_system_scheduler();<br>
</ins>
}
<code>
<p></p>

Add the following as a new subsection at the end of [exec.ctx]:

<ins>
33.N.M System scheduler [exec.system.scheduler]<br><br>
An instance of <code>system_scheduler</code> allows scheduling work on the system context. 
The system context offers concurrent forward progress guarantee. There is exactly one system context within a program.<br>
<i>Implementation limits</i>: The implementation shall support at least up to 2 * `std::this_thread::hardware_concurrency()` execution agents.
<p></p>
33.N.M.1 execution::get_system_scheduler<br><br>
&nbsp;&nbsp;1. <i>Returns</i>: An instance of a <code>system_scheduler</code> class that can be used to schedule work onto the system context.
<p></p>
33.N.M.2 execution::system_scheduler class<br><br>
&nbsp;&nbsp;1. <code>system_scheduler</code> is a class that models the <i>scheduler</i> concept and provides access to the system execution context.<br><br>
&nbsp;&nbsp;2. Two objects <i>sch1</i> and <i>sch2</i> of type <code>system_scheduler</code> always compare equal.<br><br>
&nbsp;&nbsp;3. If <i>sch</i> is an object of type <code>system_scheduler</code>, then <code>get_forward_progress_guarantee(<i>sch</i>)</code> returns <code>forward_progress_guarantee::concurrent</code>.
<p></p>
33.N.M.3 Associated types [exec.system.scheduler.types]<br><br>
&nbsp;&nbsp;1. Let <i>sch</i> be an expression of type <code>system_scheduler</code>.
The expression <code>schedule(<i>sch</i>)</code> has type <i>system-schedule-sender</i>
and is not potentially-throwing if <i>sch</i> is not potentially-throwing.<br><br>
&nbsp;&nbsp;class <i>system-schedule-sender</i>;<br><br>
&nbsp;&nbsp;2. An instance of <i>system-schedule-sender</i> remains valid for the duration of execution of the program.<br><br>
&nbsp;&nbsp;3. <i>system-schedule-sender</i> is an exposition-only type that satisfies <i>sender</i>. For any type <code>Env</code>, <code>completion_signatures_of_t&lt;<i>system-schedule-sender</i>, Env&gt;</code> is
completion_signatures&lt;set_value_t(), set_error_t(exception_ptr), set_stopped_t()&gt;.<br><br>
&nbsp;&nbsp;4. Let <i>sndr</i> be an expression of type <i>system-schedule-sender</i>, let <i>rcvr</i> be an expression such that <code>receiver_of&lt;decltype((rcvr )), CS&gt; is true where <code>CS</code> is the completion_signatures specialization above. Let <code>C</code>
be either <code>set_value_t</code> or <code>set_stopped_t</code>. Then:<br>
&nbsp;&nbsp;&nbsp;&nbsp;- The expression <code>connect(sndr, rcvr)</code> has type <i>system-schedule-sender-opstate</i>&lt;decay_t&lt;decltype((rcvr))&gt;&gt;
<br>
&nbsp;&nbsp;&nbsp;&nbsp;- The expression <code>get_completion_scheduler&lt;C&gt;(get_env(sndr))</code> is potentially-throwing if and only
if sndr is potentially-throwing.<br><br>
&nbsp;&nbsp;<code>template&lt;class Rcvr&gt;<br>
&nbsp;&nbsp;struct <i>system-schedule-sender-opstate</i>;</code><br><br>
&nbsp;&nbsp;5. Let <i>o</i> be a non-const lvalue of type <i>system-schedule-sender-opstate</i>&lt;Rcvr&gt;, and let REC(o) be a non-const lvalue reference
to an instance of type Rcvr that was initialized with the expression rcvr passed to the invocation of connect
that returned o and <i>o</i> is started. Then:<br>
&nbsp;&nbsp;- The object to which REC(o) refers remains valid for the lifetime of the object to which o refers.<br>
&nbsp;&nbsp;- The expression start(o) is equivalent to:<br>
&nbsp;&nbsp;&nbsp;&nbsp; try {<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<i>submits the operation for an execution
to the system context.</i><br>
&nbsp;&nbsp;&nbsp;&nbsp; } catch(...) {<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; set_error(std::move(REC (o)), current_exception());<br>
&nbsp;&nbsp;&nbsp;&nbsp; }<br>
&nbsp;&nbsp;- When an execution agent of the system context selects <i>o</i> for execution, it behaves as if it executes the following:<br>
&nbsp;&nbsp;&nbsp;&nbsp;if (get_stop_token(REC (o)).stop_requested()) {<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;set_stopped(std::move(REC (o)));<br>
&nbsp;&nbsp;&nbsp;&nbsp;} else {<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;set_value(std::move(REC (o)));<br>
&nbsp;&nbsp;&nbsp;&nbsp;}<br>

<!--
&nbsp;&nbsp;6. An implementation is encouraged to specialize <code>execution::bulk</code> for the <i>system-schedule-sender</i> as required to achieve efficient bulk execution.
-->

## Acknowledgments

Thank you to all who provided valuable comments and feedback!

Hans Boehm, Olivier Giroux, Ruslan Arutyunyan, Lewis Baker, 
JF Bastian and many others.

## References

https://wg21.link/p2079r6 System execution context [To be renamed parallel execution context]

https://github.com/swiftlang/swift-corelibs-libdispatch Grand Central Dispatch on linux

https://www.youtube.com/watch?v=Z86b3Rd09sE Windows threadpool internals


