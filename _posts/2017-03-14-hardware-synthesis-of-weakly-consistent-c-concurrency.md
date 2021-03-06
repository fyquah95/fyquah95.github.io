---
layout: post
title: Hardware Synthesis Of Weakly Consistent C Concurrency
category: paper-review
subtitle:

excerpt:

---

*Ramanathan, N., Fleming, S.T., Wickerson, J. and Constantinides, G.A., 2017. Hardware Synthesis of Weakly Consistent C Concurrency. -- FPGA 2017*

[Link to paper](http://cas.ee.ic.ac.uk/people/gac1/pubs/NadeshFPGA17.pdf) 

The paper addresses challenges in synthesizing HLS programs to hardware which uses Sequentially consistent (SC) atomics and weakly consistent atomics
with C11 defined atomics. The paper suggests that these atomic-based approaches can be more efficient than using locks both in terms of resources
and runtime.

The work implements atomic accesses on [Legup HLS](http://legup.eecg.utoronto.ca/), which uses C11 atomic locks and pthreads.

## Atomic Consistency Violations

Consistency violation originates from dependency analysis and scheduling.  Legup preserves read-after-write, write-after-write and write-after-read to
memory locations in its constraint solvers. Informally, the constraint solvers preserves the order of memory access to a same memory location. The formal
definition of the constraint is provided in Section 3.1.2 of the paper.

These constraints work well in lock-based and single-threaded scenarios. It is important to note that these reordering and Legup original constraint
solver _are not_ wrong - it simply maps incorrectly to atomic-based operations. The constraint does not address Read-After-Read dependencies which allows
the potential for out of order overlapping scheduling.

## C11 Memory Consistency Model

The C11 consistency model can be roughly understood as the following. Note that this is not the formal definition of the atomic
operations in the C11 specification.:

- atomic load / store cannot be reorder with another atomic load / store to the same location (coherence)
- relaxed atomic load can be reordered anywhere within its thread
- acquire atomic load cannot be reordered with loads or stores _after_ it in program order
- release atomic store cannot be reordered with loads / stores that are sequenced _before it_
- SC atomic load / store cannot be reordered with any other load / stores


## Method

The paper demonstrates atomic HLS synthesis by extending LegUp Pthread flow with three of the memory access semantics.
As mentioned in an earlier section, LegUp memory access scheduling is based on a constraint scheduler. Hence,
the methods here describes the constraints appended to the original constraint solver.

I will not include the formal definitions of the constraints - they are in Section 4.1, 4.2 and 4.3 respectively
of the original paper.

In the methods described below, consider the following example:

~~~python

# Assume that w, x, y and z maps global memory locations.
w = ...
x = ...
y = ...
z = ...

def thread():
  r0 = ld_non_atomic(w)
  r1 = ld_non_atomic(x)
  r2 = ld_acquire(y)
  r3 = ld_non_atomic(z)
~~~

The original LegUp scheduler would schedule the instructions all in the same cycle as there is no dependencies amongst them.

Cycles | 1 | 2
--- | --- | ---
r0 = ld_non_atomic(w) | X | X
r1 = ld_non_atomic(x) | X | X
r2 = ld_acquire(y)    | X | X
r3 = ld_non_atomic(z) | X | X


### (1) Preserving SC Semantics

This method simply serialize all memory operations.

This naive solution to the problem overrides the memory dependencies generated by LegUp. It results in a program that
is semantically correct, but fails to utilize any form of parallelism offered by multi-port memory controllers.

Cycles | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8
--- | --- | --- | --- | --- | --- | --- | ---
r0 = ld_non_atomic(w) | X | X |   |   |   |   |   |
r1 = ld_non_atomic(x) |   |   | X | X |   |   |   |
r2 = ld_acquire(y)    |   |   |   |   | X | X |   |
r3 = ld_non_atomic(z) |   |   |   |   |   |   | X | X

### (2) Exploring Atomics

This method constraints non-atomic memory accesses from being scheduled over atomic memory accesses. In essence,
atomic memory accesses simply act as a memory fence for non atomic operations.

Cycles | 1 | 2 | 3 | 4 | 5 | 6
--- | --- | --- | --- | --- | ---
r0 = ld_non_atomic(w) | X | X |   |   |   |
r1 = ld_non_atomic(x) | X | X |   |   |   |
r2 = ld_acquire(y)    |   |   | X | X |   |
r3 = ld_non_atomic(z) |   |   |   |   | X | X

This solution is less restrictive than (1), as ordering is only enforced w.r.t. atomics. But in the worst case,
where all memory acceses are atomic, it is equivalent to (1).

### (3) Exploiting Weak Atomics

This method defines 4 forms of atomics:

- Sequentially Consistent (SC)
- Acquire (ACQ)
- Release (REL)
- Relaxed (RLX)

The union of all these atomic operations forms the set of atomic operations.

These atomics map to 5 sets of constraints

- Memory accesses before an atomic SC must remain before the atomic SC
- Memory accesses after an atomic SC must remain after the atomic SC
- Memory accesses after an atomic ACQ must remain after the atomic ACQ
- Memory accesses before an atomic REL must remain before the atomic REL
- RAR constraint - ordering on atomic reads on a same memory location must be preserved

Cycles | 1 | 2 | 3 | 4
--- | --- | --- | --- | --- | ---
r0 = ld_non_atomic(w) | X | X |   |
r1 = ld_non_atomic(x) | X | X |   |
r2 = ld_acquire(y)    | X | X |   |
r3 = ld_non_atomic(z) |   |   | X | X


## Results

The paper demonstrates its results on a SPSC Circular Buffer by comparing an implementation without locks
(hand-tuned for correctness), 3 lock-based implementations and 3 lock-free implementations (the 3 methods
discussed earlier). Circular buffer is a data structure commonly used in the linux kernel and boost C++
libraries.

![Throughput and LUT utilization of the implementations](/images/papers/hls_concurrency_1.png)

For more detailed results, refer Section 5 of the paper.

## Conclusions

The paper has shown how weak atomics can be compiled to hardware with HLS - which would be a building block
towards enabling a larger class of lock-free programs to be synthesized to hardware.
