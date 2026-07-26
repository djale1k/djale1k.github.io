---
title: "Go's Garbage Collector in the AI Era"
draft: false
date: "2026-07-26"
---


Two years ago I wrote [Golang's Garbage Collection Simplified](/posts/golang-gc-simplified/). Since then two things happened at the same time: Go's garbage collector got the biggest overhaul in a decade, and a large share of the Go code being written stopped being typed by hand.

Those two facts are related, but not in the way people usually assume. Let me start by killing a myth I keep hearing.

## No, AI does not manage your memory

I have heard some version of this more than once: *"the model handles the memory stuff now."*

It does not. Garbage collection happens at **runtime**, inside the Go runtime, while your program is running in production. An AI coding assistant operates at **authoring time**, on your laptop, before the program has ever run. They are separated by a compiler and a deployment. A model cannot sweep a heap any more than a linter can pay your AWS bill.

What AI has actually changed is the **volume and shape of the code arriving at the runtime**. That turns out to matter a lot, and it is the real story.

## What actually changed in Go's GC

If you last paid attention around Go 1.19, here is what you missed.

### Go 1.24: cheaper allocation across the board

Go 1.24 rewrote the builtin `map` on top of [Swiss Tables](https://abseil.io/about/design/swisstables), made small-object allocation more efficient, and shipped a new runtime-internal mutex. Together these cut **CPU overhead by 2–3% on average** across the Go team's benchmark suite — for free, on recompile.

It also added the `weak` package and `runtime.AddCleanup`, a saner replacement for `SetFinalizer`.

### Go 1.25: Green Tea arrives as an experiment

The mark phase is where the GC actually burns time — roughly **90% of GC cost is marking**, and about **35% of that is the CPU stalled waiting on memory**.

The reason is that classic mark-sweep chases pointers. It hops from object to object across the heap in essentially random order. Modern CPUs hate this: a main-memory access can be ~100x slower than a cache hit, and the prefetcher cannot guess where you are going next.

[Green Tea](https://go.dev/blog/greenteagc) changes the unit of work from *objects* to *pages*. The work list tracks 8 KiB pages, per-page metadata holds two bits per object ("seen" and "scanned"), and the collector scans many objects on the same page in one contiguous pass. Fewer jumps, better locality, and a FIFO page queue that scales across cores instead of contending on one stack.

You could opt in with `GOEXPERIMENT=greenteagc` at build time.

### Go 1.26: Green Tea is the default

As of Go 1.26, Green Tea is on by default. The headline numbers:

- **10–40% reduction in GC overhead** for programs that lean on the collector, with ~10% being the common case
- An **additional ~10%** on newer amd64 CPUs (Intel Ice Lake, AMD Zen 4 and up), where the collector uses vector instructions to scan small objects

You can still opt out with `GOEXPERIMENT=nogreenteagc`, but that escape hatch is expected to disappear in Go 1.27.

Put concretely: if your service spends 10% of its CPU in GC, you are looking at a 1–4% total CPU win for doing nothing but upgrading. At the scale I work at, that is real money.

## So where does AI actually come in?

Here is the honest connection.

The GC got roughly 10–40% cheaper at collecting garbage. Meanwhile, code generation made it much cheaper to **write and merge** Go — and reviewing that code for runtime behaviour did not get cheaper at the same rate.

I want to be careful here, because the tempting claim is stronger than the evidence. I have not benchmarked generated Go against hand-written Go, and I am not aware of a corpus study that does. Humans return pointers from constructors, skip capacity hints, and over-capture in closures constantly — I have written all of it myself. The honest version of the argument is about throughput, not about models being uniquely wasteful:

> When a team can produce and merge more code per sprint, profile-guided review matters more, not less.

What a model genuinely cannot do is close the loop. It has no view of your production heap profile, and no way to know that this particular function runs 40,000 times a second. That context has always been the reviewer's job; there is now simply more code arriving that needs it.

These are the patterns worth looking for on a hot path, whoever wrote them:

**Returning pointers by reflex.** A constructor like this usually puts its result on the heap:

```go
func newRecord(id string) *Record {
    return &Record{ID: id}
}
```

"Usually" is doing real work in that sentence. Escape analysis is interprocedural and the compiler decides — with inlining, a caller that does not let the pointer escape can keep it on the stack. Do not guess; ask the compiler (see below).

In a hot loop over millions of rows, a value that stays on the stack avoids GC pressure entirely — though it is not literally free, since copying large structs and zeroing memory still cost something.

**Growing slices without a capacity hint.** The one I see most:

```go
var out []Result
for _, r := range records {
    out = append(out, transform(r))
}

// one backing-array allocation instead of repeated grow-and-copy
out := make([]Result, 0, len(records))
```

Good routine advice, with two caveats: it says nothing about what `transform` allocates internally, and if you filter most records out, `len(records)` overallocates badly.

**String concatenation in loops.** `strings.Builder` is often the right answer, and `Grow` with a size estimate helps more. But benchmark it rather than applying it as a rule — for a handful of small strings the compiler may already do fine, and a context-free rewrite rule is exactly the failure mode this post is warning about.

**Interface conversions on hot paths.** Converting a value to `any` *can* cost an allocation, but it is not automatic — pointer-shaped values need none, small integers hit a runtime cache, and a value that does not escape may stay on the stack. Treat it as something to measure, not as a rule that interfaces allocate.

**Closures that capture more than they need**, which can force captured variables to escape.

None of these are *wrong*. Every one is fine in code that runs occasionally. They only bite on hot paths — and neither a model nor a linter can tell your hot path from your startup code.

## The part that is now your job

The bottleneck has moved. Writing Go is cheap; **verifying its runtime behaviour is not**, and that part has not been automated at all.

Ask the compiler what escapes:

```bash
go build -gcflags='-m' ./... 2>&1 | grep 'escapes to heap'
```

Benchmark allocations, not just time — `B/op` and `allocs/op` are the numbers that predict GC pressure:

```bash
go test -bench=. -benchmem ./...
```

Profile a real heap instead of guessing:

```bash
go tool pprof -alloc_objects http://localhost:6060/debug/pprof/heap
```

And if you run in containers, set a soft memory limit so the GC knows its actual ceiling:

```bash
GOMEMLIMIT=900MiB
```

That last one has saved me more OOM kills than any micro-optimisation — but leave headroom rather than setting it equal to the container limit. It is a *soft* limit over the Go heap, and it does not account for every byte your container is charged for: memory owned by C code, some OS-managed mappings, and thread stacks sit outside it.

## Takeaway

Go's garbage collector is in the best shape it has ever been. Green Tea is a substantial architectural improvement to the mark phase, and most teams will get it by upgrading and doing nothing else.

Worth being precise about what it is not: Go's collector is still a non-moving, non-generational mark-sweep collector. Green Tea changed *how* marking traverses memory, not the fundamental design.

But a faster collector is not permission to stop thinking about allocation. Green Tea makes Go more forgiving of allocation-heavy workloads; it does not make allocation behaviour irrelevant.

If there is one line to take away, it is this: **code production is getting cheaper faster than performance verification is.** That is true of AI-assisted development, and it was already true of copy-paste, heavy frameworks, and ordinary deadline pressure. The fix is the same in every case — do not mechanically optimise every allocation. Find the hot paths, measure them, and let the profile decide what to change.

Upgrade to 1.26. Then go look at `allocs/op` on your hottest path. The compiler will tell you the truth if you ask it.
