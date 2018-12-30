# async-profiler

This project is a low overhead sampling profiler for Java
that does not suffer from [Safepoint bias problem](http://psy-lob-saw.blogspot.ru/2016/02/why-most-sampling-java-profilers-are.html).
It features HotSpot-specific APIs to collect stack traces
and to track memory allocations. The profiler works with
OpenJDK, Oracle JDK and other Java runtimes based on HotSpot JVM.

async-profiler can trace the following kinds of events:
 - CPU cycles
 - Hardware and Software performance counters like cache misses, branch misses, page faults, context switches etc.
 - Allocations in Java Heap
 - Contented lock attempts, including both Java object monitors and ReentrantLocks

## Download

Latest release:

 - Linux x64: [async-profiler-1.4-linux-x64.tar.gz](https://github.com/jvm-profiling-tools/async-profiler/releases/download/v1.4/async-profiler-1.4-linux-x64.tar.gz)
 - macOS x64: [async-profiler-1.4-macos-x64.tar.gz](https://github.com/jvm-profiling-tools/async-profiler/releases/download/v1.4/async-profiler-1.4-macos-x64.tar.gz)

[Previous releases](https://github.com/jvm-profiling-tools/async-profiler/releases)

## Supported platforms

- **Linux** / x64 / x86 / ARM / AArch64
- **macOS** / x64

Note: macOS profiling is limited to user space code only.

## CPU profiling

In this mode profiler collects stack trace samples that include **Java** methods,
**native** calls, **JVM** code and **kernel** functions.

The general approach is receiving call stacks generated by `perf_events`
and matching them up with call stacks generated by `AsyncGetCallTrace`,
in order to produce an accurate profile of both Java and native code.
Additionally, async-profiler provides a workaround to recover stack traces
in some [corner cases](https://bugs.openjdk.java.net/browse/JDK-8178287)
where `AsyncGetCallTrace` fails.

This approach has the following advantages compared to using `perf_events`
directly with a Java agent that translates addresses to Java method names:

* Works on older Java versions because it doesn't require
`-XX:+PreserveFramePointer`, which is only available in JDK 8u60 and later.

* Does not introduce the performance overhead from `-XX:+PreserveFramePointer`,
which can in rare cases be as high as 10%.

* Does not require generating a map file to map Java code addresses to method
names.

* Works with interpreter frames.

* Does not require writing out a perf.data file for further processing in
user space scripts.

## ALLOCATION profiling

Instead of detecting CPU-consuming code, the profiler can be configured
to collect call sites where the largest amount of heap memory is allocated.

async-profiler does not use intrusive techniques like bytecode instrumentation
or expensive DTrace probes which have significant performance impact.
It also does not affect Escape Analysis or prevent from JIT optimizations
like allocation elimination. Only actual heap allocations are measured.

The profiler features TLAB-driven sampling. It relies on HotSpot-specific
callbacks to receive two kinds of notifications:
 - when an object is allocated in a newly created TLAB;
 - when an object is allocated on a slow path outside TLAB.

This means not each allocation is counted, but only allocations every _N_ kB,
where _N_ is the average size of TLAB. This makes heap sampling very cheap
and suitable for production. On the other hand, the collected data
may be incomplete, though in practice it will often reflect the top allocation
sources.

Unlike Java Mission Control which uses similar approach, async-profiler
does not require Java Flight Recorder or any other JDK commercial feature.
It is completely based on open source technologies and it works with OpenJDK.

The minimum supported JDK version is 7u40 where the TLAB callbacks appeared.

Heap profiler requires HotSpot debug symbols. Oracle JDK already has them
embedded in `libjvm.so`, but in OpenJDK builds they are typically shipped
in a separate package. For example, to install OpenJDK debug symbols on
Debian / Ubuntu, run
```
# apt-get install openjdk-8-dbg
```
On Gentoo the ``icedtea`` OpenJDK package can be built with the per-package setting
``FEATURES="nostrip"`` to retain symbols.

### Wall-clock profiling

`-e wall` option tells async-profiler to sample all threads equally every given
period of time regardless of thread status: Running, Sleeping or Blocked.
For instance, this can be helpful when profiling application start-up time.

Wall-clock profiler is most useful in per-thread mode: `-t`.

Example: `./profiler.sh -e wall -t -i 5ms -f result.svg 8983`

## Building

Build status: [![Build Status](https://travis-ci.org/jvm-profiling-tools/async-profiler.svg?branch=master)](https://travis-ci.org/jvm-profiling-tools/async-profiler)

Make sure the `JAVA_HOME` environment variable points to your JDK installation,
and then run `make`. GCC is required. After building, the profiler agent binary
will be in the `build` subdirectory. Additionally, a small application `jattach`
that can load the agent into the target process will also be compiled to the
`build` subdirectory.

## Basic Usage

As of Linux 4.6, capturing kernel call stacks using `perf_events` from a non-
root process requires setting two runtime variables. You can set them using
sysctl or as follows:

```
# echo 1 > /proc/sys/kernel/perf_event_paranoid
# echo 0 > /proc/sys/kernel/kptr_restrict
```

To run the agent and pass commands to it, the helper script `profiler.sh`
is provided. A typical workflow would be to launch your Java application,
attach the agent and start profiling, exercise your performance scenario, and
then stop profiling. The agent's output, including the profiling results, will
be displayed in the Java application's standard output.

Example:

```
$ jps
9234 Jps
8983 Computey
$ ./profiler.sh start 8983
$ ./profiler.sh stop 8983
```

Alternatively, you may specify `-d` (duration) argument to profile
the application for a fixed period of time with a single command.

```
$ ./profiler.sh -d 30 8983
```

By default, the profiling frequency is 100Hz (every 10ms of CPU time).
Here is a sample of the output printed to the Java application's terminal:

```
--- Execution profile ---
Total samples:           687
Unknown (native):        1 (0.15%)

--- 6790000000 (98.84%) ns, 679 samples
  [ 0] Primes.isPrime
  [ 1] Primes.primesThread
  [ 2] Primes.access$000
  [ 3] Primes$1.run
  [ 4] java.lang.Thread.run

... a lot of output omitted for brevity ...

          ns  percent  samples  top
  ----------  -------  -------  ---
  6790000000   98.84%      679  Primes.isPrime
    40000000    0.58%        4  __do_softirq

... more output omitted ...
```

This indicates that the hottest method was `Primes.isPrime`, and the hottest
call stack leading to it comes from `Primes.primesThread`.

## Launching as an Agent

If you need to profile some code as soon as the JVM starts up, instead of using the `profiler.sh` script,
it is possible to attach async-profiler as an agent on the command line. For example:

```
$ java -agentpath:/path/to/libasyncProfiler.so=start,svg,file=profile.svg ...
```

Agent library is configured through the JVMTI argument interface. The format of the arguments string is described [in the source code](https://github.com/jvm-profiling-tools/async-profiler/blob/af94b0e55178c46e17c573a65c498d25b58b641b/src/arguments.cpp#L26). The `profiler.sh` script actually
converts command line arguments to the that format.

For instance, `-e alloc` is converted to `event=alloc`, `-f profile.svg` is converted to `file=profile.svg` and so on. But some arguments are processed directly by `profiler.sh` script. E.g. `-d 5` results in 3 actions: 1) attaching profiler agent with start command, sleeping for 5 seconds, and then attaching the agent again with stop command.

## Flame Graph visualization

async-profiler provides out-of-the-box [Flame Graph](https://github.com/BrendanGregg/FlameGraph) support.
Specify `-o svg` argument to dump profiling results as an interactive SVG
immediately viewable in all mainstream browsers.
Also, SVG output format will be chosen automatically if the target
filename ends with `.svg`.

```
$ jps
9234 Jps
8983 Computey
$ ./profiler.sh -d 30 -f /tmp/flamegraph.svg 8983
```

![Example](https://github.com/jvm-profiling-tools/async-profiler/blob/master/demo/SwingSet2.svg)

## Profiler Options

The following is a complete list of the command-line options accepted by
`profiler.sh` script.

* `start` - starts profiling in semi-automatic mode, i.e. profiler will run
until `stop` command is explicitly called.

* `stop` - stops profiling and prints the report.

* `status` - prints profiling status: whether profiler is active and
for how long.

* `list` - show the list of available profiling events. This option still
requires PID, since supported events may differ depending on JVM version.

* `-d N` - the profiling duration, in seconds. If no `start`, `stop`
or `status` option is given, the profiler will run for the specified period
of time and then automatically stop.  
Example: `./profiler.sh -d 30 8983`

* `-e event` - the profiling event: `cpu`, `alloc`, `lock`, `cache-misses` etc.
Use `list` to see the complete list of available events.

  In allocation profiling mode the top frame of every call trace is the class
of the allocated object, and the counter is the heap pressure (the total size
of allocated TLABs or objects outside TLAB).

  In lock profiling mode the top frame is the class of lock/monitor, and
the counter is number of nanoseconds it took to enter this lock/monitor.  

  Two special event types are supported on Linux: hardware breakpoints
and kernel tracepoints:
  - `-e mem:<func>[:rwx]` sets read/write/exec breakpoint at function
  `<func>`. The format of `mem` event is the same as in `perf-record`.
  Execution breakpoints can be also specified by the function name,
  e.g. `-e malloc` will trace all calls of native `malloc` function.
  - `-e trace:<id>` sets a kernel tracepoint. It is possible to specify
  tracepoint symbolic name, e.g. `-e syscalls:sys_enter_open` will trace
  all `open` syscalls.

* `-i N` - sets the profiling interval in nanoseconds or in other units,
if N is followed by `ms` (for milliseconds), `us` (for microseconds)
or `s` (for seconds). Only CPU active time is counted. No samples
are collected while CPU is idle. The default is 10000000 (10ms).  
Example: `./profiler.sh -i 500us 8983`

* `-j N` - sets the Java stack profiling depth. This option will be ignored if N is greater 
than default MAX_STACK_FRAMES.  
Example: `./profiler.sh -j 30 8983`

* `-b N` - sets the frame buffer size, in the number of Java
method ids that should fit in the buffer. If you receive messages about an
insufficient frame buffer size, increase this value from the default.  
Example: `./profiler.sh -b 5000000 8983`

* `-t` - profile threads separately. Each stack trace will end with a frame
that denotes a single thread.  
Example: `./profiler.sh -t 8983`

* `-s` - print simple class names instead of FQN.

* `-o fmt[,fmt...]` - specifies what information to dump when profiling ends.
This is a comma-separated list of the following options:
  - `summary` - dump basic profiling statistics;
  - `traces[=N]` - dump call traces (at most N samples);
  - `flat[=N]` - dump flat profile (top N hot methods);
  - `jfr` - dump events in Java Flight Recorder format readable by Java Mission Control.
  This *does not* require JDK commercial features to be enabled.
  - `collapsed[=C]` - dump collapsed call traces in the format used by
  [FlameGraph](https://github.com/brendangregg/FlameGraph) script. This is
  a collection of call stacks, where each line is a semicolon separated list
  of frames followed by a counter.
  - `svg[=C]` - produce Flame Graph in SVG format.
  - `tree[=C]` - produce call tree in HTML format.  
     --reverse option will generate backtrace view. 
  
  `C` is a counter type:
  - `samples` - the counter is a number of samples for the given trace;
  - `total` - the counter is a total value of collected metric, e.g. total allocation size.
  
  The default format is `summary,traces=200,flat=200`.

* `--title TITLE`, `--width PX`, `--height PX`, `--minwidth PX`, `--reverse` - FlameGraph parameters.  
Example: `./profiler.sh -f profile.svg --title "Sample CPU profile" --minwidth 0.5 8983`

* `-f FILENAME` - the file name to dump the profile information to.  
Example: `./profiler.sh -o collapsed -f /tmp/traces.txt 8983`

* `--all-user` - include only user-mode events. This option is helpful when kernel profiling
is restricted by `perf_event_paranoid` settings.  
`--all-kernel` is its counterpart option for including only kernel-mode events.

* `-v`, `--version` - prints the version of profiler library. If PID is specified,
gets the version of the library loaded into the given process.

## Restrictions/Limitations

* On most Linux systems, `perf_events` captures call stacks with a maximum depth
of 127 frames. On recent Linux kernels, this can be configured using
`sysctl kernel.perf_event_max_stack` or by writing to the
`/proc/sys/kernel/perf_event_max_stack` file.

* Profiler allocates 8kB perf_event buffer for each thread of the target process.
Make sure `/proc/sys/kernel/perf_event_mlock_kb` value is large enough
(more than `8 * threads`) when running under unprivileged user.
Otherwise the message _"perf_event mmap failed: Operation not permitted"_
will be printed, and no native stack traces will be collected.

* There is no bullet-proof guarantee that the `perf_events` overflow signal
is delivered to the Java thread in a way that guarantees no other code has run,
which means that in some rare cases, the captured Java stack might not match
the captured native (user+kernel) stack.

*  You will not see the non-Java frames _preceding_ the Java frames on the
stack. For example, if `start_thread` called `JavaMain` and then your Java
code started running, you will not see the first two frames in the resulting
stack. On the other hand, you _will_ see non-Java frames (user and kernel)
invoked by your Java code.

* No Java stacks will be collected if `-XX:MaxJavaStackTraceDepth` is zero
or negative.

* Too short profiling interval may cause continuous interruption of heavy
system calls like `clone()`, so that it will never complete;
see [#97](https://github.com/jvm-profiling-tools/async-profiler/issues/97).
The workaround is simply to increase the interval.

* When agent is not loaded at JVM startup (by using -agentpath option) it is
highly recommended to use `-XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints` JVM flags.
Without those flags the profiler will still work correctly but results might be
less accurate e.g. without `-XX:+DebugNonSafepoints` there is a high chance that simple inlined methods will not appear in the profile. When agent is attached at runtime `CompiledMethodLoad` JVMTI event
enables debug info, but only for methods compiled after the event is turned on.

## Troubleshooting

```
Failed to change credentials to match the target process: Operation not permitted
```
Due to limitation of HotSpot Dynamic Attach mechanism, the profiler must be run
by exactly the same user (and group) as the owner of target JVM process.
If profiler is run by a different user, it will try to automatically change
current user and group. This will likely succeed for `root`, but not for
other users, resulting in the above error.

```
Could not start attach mechanism: No such file or directory
```
The profiler cannot establish communication with the target JVM through UNIX domain socket.

For the profiler to be able to access JVM, make sure
 1. `/tmp` directory of Java process is physically the same directory as `/tmp` of your shell.
 2. Attach socket `/tmp/.java_pidNNN` has not been deleted.
 3. JVM is not run with `-XX:+DisableAttachMechanism` option.

```
Failed to inject profiler into <pid>
```
The connection with the target JVM has been established, but JVM is unable to load profiler shared library.
Make sure the user of JVM process has permissions to access `libasyncProfiler.so` by exactly the same absolute path.
For more information see [#78](https://github.com/jvm-profiling-tools/async-profiler/issues/78).

```
Perf events unavailble. See stderr of the target process.
```
`perf_event_open()` syscall has failed. The error message is printed to the error stream
of the target JVM.

Typical reasons include:
 1. `/proc/sys/kernel/perf_event_paranoid` is set to restricted mode (>=2).
 2. seccomp disables perf_event_open API in a container.
 3. OS runs under a hypervisor that does not virtualize performance counters.
 4. perf_event_open API is not supported on this system, e.g. WSL.

```
[frame_buffer_overflow]
```
This message in the output means there was not enough space to store all call traces.
Consider increasing frame buffer size with `-b` option.
