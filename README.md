Notes for a presentation on "C is Not a Low-level Language" by David Chisnall,
from the March/April 2018 edition of ACM Queue.

This document includes key quotes from the paper, which can be found in its
entirety here: https://queue.acm.org/detail.cfm?id=3212479

### Overview

In the wake of the recent Meltdown and Spectre vulnerabilities, it’s worth
spending some time looking at root causes. Both of these vulnerabilities
involved processors speculatively executing instructions past some kind of
access check and allowing the attacker to observe the results via a side
channel.

### What is a Low-level Language?

Computer science pioneer Alan Perlis defined low-level languages this way:
> “A programming language is low level when its programs require attention to
> the irrelevant.”

Low-level languages are “close to the metal,” whereas high-level languages are
closer to how humans think. For a language to be “close to the metal,” it must
provide an abstract machine that maps easily to the abstractions exposed by
the target platform. It’s easy to argue that C was a low-level language for
the PDP-11.

### Fast PDP-11 Emulators

The root cause of the Spectre and Meltdown vulnerabilities was that processor
architects were trying to build not just fast processors, but fast processors
that expose the same abstract machine as a PDP-11.

Creating a new thread is a library operation known to be expensive, so
processors wishing to keep their execution units busy running C code rely on
ILP (instruction-level parallelism). They inspect adjacent operations and
issue independent ones in parallel.

The quest for high ILP was the direct cause of Spectre and Meltdown. A modern
Intel processor has up to 180 instructions in flight at a time.

A typical heuristic for C code is that there is a branch, on average, every
seven instructions. If you wish to keep such a pipeline full from a single
thread, then you must guess the targets of the next 25 branches. This, again,
adds complexity; it also means that an incorrect guess results in work being
done and then discarded, which is not ideal for power consumption. This
discarded work has visible side effects, which the Spectre and Meltdown
attacks could exploit.

Consider another core part of the C abstract machine’s memory model: flat
memory. This hasn’t been true for more than two decades. A modern processor
often has three levels of cache in between registers and main memory, which
attempt to hide latency.

### Optimizing C


One of the common attributes ascribed to low-level languages is that they’re
fast. In particular, they should be easy to translate into fast code without
requiring a particularly complex compiler.

...the levels of performance expected by C programmers are achieved only as a
result of incredibly complex compiler transforms. The Clang compiler,
including the relevant parts of LLVM, is around 2 million lines of code.

__Optimizing Loops__

For example, in C, processing a large amount of data means writing a loop that
processes each element sequentially. To run this optimally on a modern CPU, the
compiler must first determine that the loop iterations are independent.

Once the compiler has determined that loop iterations are independent, then the
next step is to attempt to vectorize the result, because modern processors get
four to eight times the throughput in vector code that they achieve in scalar
code.

Optimizers at this point must fight the C memory layout guarantees.

__SROA and Loop Unswitching__

Consider two of the core optimizations that a C compiler performs: SROA (scalar
replacement of aggregates) and loop unswitching.

SROA attempts to replace structs (and arrays with fixed lengths) with
individual variables. This then allows the compiler to treat accesses as
independent and elide operations entirely if it can prove that the results
are never visible. This has the side effect of deleting padding in some cases
but not others.

The second optimization, loop unswitching, transforms a loop containing a
conditional into a conditional with a loop in both paths. This changes flow
control, contradicting the idea that a programmer knows what code will
execute when low-level language code runs.

__memalloc Example__

In C, a read from an uninitialized variable is an unspecified value and is
allowed to be any value each time it is read.

...for example, on FreeBSD the malloc implementation informs the operating
system that pages are currently unused, and the operating system uses the
first write to a page as the hint that this is no longer true. A read to
newly malloced memory may initially read the old value; then the operating
system may reuse the underlying physical page; and then on the next write to
a different location in the page replace it with a newly zeroed page. The
second read from the same location will then give a zero value.

Consider the loop-unswitching optimization, this time in the case where the
loop ends up being executed zero times. In the original version, the entire
body of the loop is dead code. In the unswitched version, there is now a
branch on the variable, which may be uninitialized. Some dead code has now
been transformed into undefined behavior. This is just one of many
optimizations that a close investigation of the C semantics shows to be
unsound.

### Understanding C

One of the key attributes of a low-level language is that
programmers can easily understand how the language’s
abstract machine maps to the underlying physical machine.

There is a common myth in software development that parallel programming is
hard.

It’s more accurate to say that parallel programming in a language with a
C-like abstract machine is difficult, and given the prevalence of parallel
hardware, from multicore CPUs to many core GPUs, that’s just another way of
saying that C doesn’t map to modern hardware very well.
