![Build Status](https://github.com/Kaiepi/ra-Gauge/actions/workflows/test.yml/badge.svg)

NAME
====

Gauge - Iterative polling

SYNOPSIS
========

```raku
use v6.d;
use Gauge;

#|[ Jury-rigged benchmark that keys it/s counts of an evaluated code block. ]
sub MAIN(
    **@code,
    #|[ The number of threads to dedicate to benchmarking. ]
    UInt:D :j(:$jobs) = $*KERNEL.cpu-cores.pred,
    #|[ The duration in seconds over which timestamps will be aggregated. ]
    Real:D :p(:$period) = 1,
    #|[ The cooldown in seconds performed between each individual benchmark. ]
    Real:D :c(:$cooldown) = (try 2 / 3 * $jobs * $period orelse 2 / 3 * $*KERNEL.cpu-cores.pred),
    #|[ Whether or not ANSI 24-bit SGR escape sequences should be suppressed.
        These highlight blocks of it/s counts opneing with newfound maximums. ]
    Bool:D :m(:$mono) = False,
--> Nil) {
    use MONKEY-SEE-NO-EVAL;
    map $mono ?? &mono !! &poly,
        Gauge(EVAL Qa[-> { @code.join() }])
            .poll($period)
            .pledge($jobs)
            .throttle($cooldown);
}
#=[ Benchmark threads are run once an iteration of any existing threads has
    been exhausted. This is staggered by the cooldown, and by default, allows
    for multiple benchmarks to be taken with a brief overlap of threaded work,
    reducing the time needed to aggregate results while keeping low overhead. ]

sub poly(Int:D $next --> Empty) {
    my constant @mark = «\e[48;5;198m \e[48;5;202m \e[48;5;178m \e[48;5;41m \e[48;5;25m \e[48;5;129m»;
    state $mark is default(-1);
    state $peak is default(0);

    my $jump := $peak < $next;
    $mark += $jump;
    $mark %= @mark;
    $peak max= $next;

    my $note = @mark[$mark];
    $note ~= $jump ?? '⊛' !! '∗';
    $note ~= " \e[m";
    $note ~= $next;

    put $note
}

sub mono(Int:D $next --> Empty) {
    state $peak is default(0);

    my $jump := $peak < $next;
    $peak max= $next;

    my $note = $jump ?? '⊛' !! '∗';
    $note ~= ' ';
    $note ~= $next;

    put $note
}
```

DESCRIPTION
===========

```raku
class Gauge is Seq { ... }
```

`Gauge`, in general, provides an interface for a collection of temporal, lazy, non-deterministic iterators. At its base, it attempts to measure durations of calls to a block with as little overhead as possible in order to avoid unnecessary influence over results. This can stack operations mapping its iterator until it decays to a `Seq` via `Iterable` or the `iterator` itself.

METHODS
=======

CALL-ME
-------

```raku
method CALL-ME(::?CLASS:_: Block:D $block --> ::?CLASS:D)
```

Produces a new `Gauge` sequence of native `uint64` durations of a call to the given block. Measurements of each duration are **not** monotonic, thus leap seconds and hardware errors will skew results.

poll
----

```raku
method poll(::?CLASS:D: Real:D $seconds --> ::?CLASS:D)
```

Returns a new `Gauge` sequence that produces an `Int:D` count of iterations of the former totalling a duration of `$seconds`. This will take longer than the given argument to complete due to the overhead of iteration, which may be measured by `Gauge` itself in combination with `head`.

throttle
--------

```raku
method throttle(::?CLASS:D: Real:D $seconds --> ::?CLASS:D)
```

Returns a new `Gauge` sequence that will sleep `$seconds` between iterations of its former. If a `poll` is applied later, this will incorporate the time waited into the time it takes to complete an iteration, but not any steps in between required to caculated. The total time throttled is allowed to exceed any poll applied to a `Gauge`, but never by any more than one iteration.

pledge
------

```raku
method pledge(::?CLASS:D: UInt:D $length --> ::?CLASS:D)
```

Returns a new `Gauge` sequence that, across an array of threads of breadth `$length`, pledges that a form of iteration will be continuously invoked across a threaded repetition of this iterator's component iterators in the process. When an empty length is given, this will not change the iteration.

When a length of `1` is given, this will produce a `covenant` wrapping the iterator. This is a thread that upon receiving a query to perform an iteration, will produce its result, then go ahead and perform another iteration while waiting for the next query. This means switching from `skip` to `head` may perform an extra `skip` whose result is discarded while waiting for `head`. Such a thread will finish in a sink context, but is allowed to wait until the end of a program after decaying to another `Seq` when it doesn't play nice.

When a length of `2` or more is given, a *contract* wrapping this number of covenants will be formed. This carries a particular cycle followed to accomodate any throttling performed beforehand. A thread must be spawned in order to work, but any prior threads may complete an iteration during this timespan, and will be waited on in reverse as each covenant is run in its original ordering.

AUTHOR
======

Ben Davies (Kaiepi)

COPYRIGHT AND LICENSE
=====================

Copyright 2023 Ben Davies

This library is free software; you can redistribute it and/or modify it under the Artistic License 2.0.

