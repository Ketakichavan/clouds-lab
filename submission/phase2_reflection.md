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
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->753.276<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->0.996<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->426340.4<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->13935.1<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4813<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2302.223 ms, 2 threads: 2058.131 ms, 4 threads: 2643.066 ms, 8 threads: 2579.503 ms. 2 threads was the fastest.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The naive version loops in Python, one number at a time, so most of its time goes on interpreter overhead, type checks and scattered memory reads, not on the actual maths. The vectorized version passes the whole matrix multiply to NumPy's BLAS, compiled code that uses SIMD (AVX) to process several numbers per instruction over contiguous, cache-friendly memory. Same work and same result: 753.276 ms became 0.996 ms, about 756x faster.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
2 threads helped only slightly (2302 -> 2058 ms, about 12%) and 4 and 8 threads were slower (2643 and 2580 ms). My Colab runtime has just 2 vCPUs, so extra threads only take turns on the same cores. They also have to split and sync the work after each operation and share caches and memory bandwidth so past 2 threads that overhead outweighs any gain. On this machine, 2 threads gave the best result.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
A training step is mostly big matrix multiplications made of thousands of independent multiply-adds which is exactly what the T4's thousands of cores and fast memory are built for. That's why it took 13,935 ms against 426,340 ms on the CPU (about 30x). Generating text one token at a time with batch size 1 wouldn't benefit much because each step waits on the previous one and the GPU sits mostly idle.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Parallelism paid off when the work split into many independent pieces and there was hardware to run them. Vectorization gave 756x (753.276 -> 0.996 ms) and the GPU about 30x (426,340 -> 13,935 ms). Threads ran into two limits. The first was hardware: with only 2 vCPUs, 2 threads was the best (2058 ms) and 4 threads was slower (2643 ms). The second was coordination: even 2 threads gained only about 12% because splitting and syncing work costs time. The GPU is limited by data movement and serial work: data has to be copied to the device, and steps like the optimizer update or token-by-token generation can't be spread out which by Amdahl's law caps the speed-up. So parallelism stops helping when the work can't be split, the cores run out, or coordination and data transfer cost more than they save.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would compare GPU and CPU at different batch sizes (1, 8, 32 and 128). I expect the GPU's lead to shrink at small batches, where launch and transfer overhead dominate. If it stayed about 30x faster even at batch size 1, I would drop the idea that overhead is its main limit.
<!--/answer-->
