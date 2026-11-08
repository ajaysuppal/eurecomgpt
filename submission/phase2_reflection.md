# Phase 2 — Reflection: where does parallelism stop helping?

Copy this file to `submission/phase2_reflection.md`, fill in every answer slot, and commit it:

```bash
cp phase-2-tiny-gpt/reflection_template.md submission/phase2_reflection.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or your threading plot freely.

Numbers in section 1 must **match `submission/phase2_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->245.662<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->0.972<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->463587.0<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->12636.6<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4685<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2348.57 ms, 2 threads: 2223.756 ms, 4 threads: 2298.026 ms, 8 threads: 2481.057 ms.
Fastest: 2 threads (2223.756 ms).
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The naive version runs 2 × 256 × 256 ≈ 131k interpreted loop iterations. Each one pays Python dispatch, object allocation and a tiny 64-element np.dot call, so the CPU mostly does bookkeeping, not math (245.662 ms). The vectorized version makes a few calls into OpenBLAS. OpenBLAS runs compiled, cache-blocked kernels with AVX2/AVX-512 SIMD instructions (X86_V3/V4 detected), which process 4–8 doubles per instruction. Result: 0.972 ms, 252.7× faster.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
The curve bottoms out at 2 threads (2223.756 ms, only 5.3% faster than 1 thread) and then rises: 4 threads take 2298.026 ms and 8 take 2481.057 ms, 11.6% slower than 2. lscpu shows 2 vCPUs but just 1 physical core with 2 hyperthreads. The two threads share one set of execution units and caches, so the second adds little. Beyond 2, threads are oversubscribed. They time-slice, context-switch and wait at the barriers after every parallel op, and fight for the same memory bandwidth.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
A training step is dominated by large dense matrix multiplies: the QKV projections, attention scores and MLP layers, forward and backward. These are thousands of independent multiply-adds, which the T4's 2,560 CUDA cores run in SIMT lockstep with high memory bandwidth. The GPU took 12636.6 ms vs 463587.0 ms on the CPU, a 36.7× speed-up. Sequential, branchy, low-arithmetic work would not benefit, for example the naive Python-loop attention, or a batch-size-1 token-by-token generation loop dominated by kernel launches.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Each experiment hit a different limit.

- SIMD: vectorizing removed the interpreter overhead and gave 252.7× (245.662 → 0.972 ms). Below that, what remains is work that cannot be divided further: the softmax's row-wise max and sum are reductions, and the fixed per-call overhead now dominates a sub-millisecond kernel (Amdahl's serial fraction).
- Threads: the limit was hardware I do not have, plus coordination. With one physical core, 2 threads gained only 5.3% (2348.57 → 2223.756 ms). At 8 threads, synchronization and context switching cost more than they saved (2481.057 ms).
- GPU: the 36.7× speed-up (463587.0 → 12636.6 ms) was capped by data that has to move and by fixed costs. Every step copies a batch host→device. Each small kernel pays launch latency. CUDA initialization and kernel loading are paid inside only 100 steps. With 32×256 tokens, the T4's cores are under-occupied, so the GPU is partly starved.
Parallelism stops helping when the remaining serial, coordination or transfer cost is larger than the work being split.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
Sweep the GPU batch size (8 → 256) with a warm-up step excluded and torch.cuda.synchronize() around the timer, measuring ms per token. If per-token time keeps falling steeply as batch grows, the T4 was starved by small batches and launch/transfer overhead. If it stays flat, the GPU was already compute-bound and 36.7× is the true ceiling.
<!--/answer-->
