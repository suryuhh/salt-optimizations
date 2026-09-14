# 7× faster SALT witness decoding and proof verification

Combined, PRs #2–#5 make SALT witness decoding plus proof verification **7.24× faster** on dense mainnet witnesses and **2.17× faster** on ordinary head witnesses. The trie-update change is measured separately in SALT's existing benchmark. Each PR is also measured against the code immediately before it, so its individual effect is visible below.

In the headline table, one witness/s means one complete SALT witness decoded and its cryptographic proof verified. This is the SALT work for one block witness; it excludes transaction execution, networking and canonical chain advancement, so it is neither validator blocks/s nor TPS. Elapsed time is wall-clock latency; process CPU adds the CPU time used by all worker threads, so it can exceed elapsed time.

| Workload, eight witnesses decoded and verified concurrently | Upstream main + #152 verifier work | After PRs #2–#5 | CPU time per witness, baseline → changed | Peak process memory, baseline → changed |
|---|---:|---:|---:|---:|
| Dense mainnet (10,092 tx/block) | 4.64 witnesses/s | **33.58 witnesses/s (7.24×)** | 1710 → **237 ms (-86.2%)** | **789 → 346 MiB (-56.1%)** |
| Ordinary mainnet head (26 tx/block) | 182.5 witnesses/s | **396.7 witnesses/s (2.17×)** | 41.2 → **19.6 ms (-52.4%)** | **22.4 → 12.4 MiB (-44.6%)** |

Every timed proof verified successfully, every computed state root matched the preceding mainnet block header, and wire encoding and proof-acceptance rules are unchanged. The vector paths require x86-64 AVX-512F and AVX-512IFMA; other CPUs use the scalar implementations.

## 1. Decode points in vector lanes inside each worker

**Problem.** [#137](https://github.com/megaeth-labs/salt/pull/137) distributes commitments across threads, but each worker still reconstructs and validates its points with scalar field arithmetic.

**Change.** [Decode eight points together](https://github.com/suryuhh/salt-optimizations/pull/2/files#diff-a7bf1ae409bec836399b884497b2a7c4a00c258efc07ec219d77744fdb18ea55R131) with table-based square roots, Jacobi subgroup checks and shared inversions. Unsupported CPUs and tails retain the specialized scalar path. Acceptance rules, order and wire encoding are unchanged.

**Result.** PR #2 against #137's merged parallel decoder, with eight worker threads:

| Dense mainnet witness | #137's decoder | PR #2 | Change |
|---|---:|---:|---:|
| Decode elapsed time | 169.5 ms | **22.8 ms** | **7.43× faster** |
| Process CPU | 1293 ms | **113 ms** | **-91.2%** |

**Why it matters.** #137 reports a roughly 10× sequential-to-parallel gain on an Apple M4. PR #2 is additional: parallel decoding is already in both arms, and it removes arithmetic inside each worker on an AMD IFMA host. An earlier [128-block validator replay](https://github.com/suryuhh/salt-optimizations/blob/codex/docs/optimization-report/optimization-evidence/COMPONENT_BENCHMARKS.md) measured **+35.46% throughput** and **−26.32% CPU per block** from the initial batch decoder. With IFMA disabled on both arms, its specialized scalar arithmetic separately measured **+18.17% validator throughput** and **−15.93% CPU per block** on the same host; that is a scalar-path sensitivity test, not a measurement on native non-IFMA hardware.

## 2. Preserve thread parallelism in the vector MSM

**Problem.** [#152](https://github.com/megaeth-labs/salt/pull/152) parallelizes scalar Pippenger windows, but the arithmetic within each window remains scalar. A vector replacement must preserve that existing thread parallelism to improve both CPU and latency.

**Change.** [Split large MSMs across worker threads](https://github.com/suryuhh/salt-optimizations/pull/3/files#diff-a7bf1ae409bec836399b884497b2a7c4a00c258efc07ec219d77744fdb18ea55R508) and run the vector kernel within each chunk. Inputs below 8,192 points stay on #152's scalar path in multi-thread pools; one-thread use switches at 128 points.

**Result.** PR #3 against PR #2 on dense mainnet proofs:

| Proof verification | PR #2 base | PR #3 | Change |
|---|---:|---:|---:|
| One proof, elapsed | 126.0 ms | **84.1 ms** | **-33.2%** |
| One proof, process CPU | 447 ms | **206 ms** | **-54.0%** |
| One proof, peak memory | 250 MiB | 262 MiB | +4.9% |
| Eight concurrent, verified proofs/s | 17.6 | **38.4** | **2.18×** |

**Why it matters.** The repaired vector path cuts both latency and CPU on large proofs while more than doubling concurrent proof throughput.

**Where it applies.** Ordinary-head proofs contain 377–1,973 commitments, so the cutoff deliberately keeps their unchanged scalar path: 4.38 versus 4.43 ms. The synthetic proof benchmark from #152 also stays scalar through its 4,351-commitment case; its three rounds overlap (13.34 versus 13.63 ms).

**Crossover and correctness.** A direct check is neutral through 8,191 points, then measures **21.6% faster at 8,192**, **29.7% at 16,384**, and **57.8% at 50,000**. Results match arkworks immediately below, above and at the dispatch boundary.

## 3. Batch trie commitment and shrink the precomputed table

**Problem.** [#135](https://github.com/megaeth-labs/salt/pull/135) exposes the table window by platform but keeps window 11 on native hosts. Trie updates still multiply indexed deltas individually.

**Change.** [Batch indexed leaf and internal-node multiplications](https://github.com/suryuhh/salt-optimizations/pull/4/files#diff-1fbba266e052be301a32da6780f9467bb82576975429c3476afbdd4a3eb21073R749) in eight lanes. IFMA hosts use window 7; scalar hosts keep window 11. The smaller table initializes in 0.03 rather than 0.31 seconds in the [kernel equivalence test](https://github.com/suryuhh/salt-optimizations/blob/codex/docs/optimization-report/optimization-evidence/COMPONENT_BENCHMARKS.md#committer-kernel-check).

**Result.** Elapsed time for SALT's existing `update 10000 KVs` benchmark, PR #4 against PR #3:

| Threads | PR #3 | PR #4 | Speedup |
|---:|---:|---:|---:|
| 1 | 193.7 ms | 121.1 ms | **1.60×** |
| 2 | 108.0 ms | 69.6 ms | **1.55×** |
| 4 | 64.8 ms | 44.6 ms | **1.45×** |
| 8 | 42.4 ms | 31.4 ms | **1.35×** |
| 16 | 32.7 ms | 30.5 ms | **1.07×** |

**Why it matters.** This is the state-root update path, and the change is faster at every tested thread count. The earlier [128-block validator replay](https://github.com/suryuhh/salt-optimizations/blob/codex/docs/optimization-report/optimization-evidence/COMPONENT_BENCHMARKS.md) of the precursor window-8 design measured **+5.56% throughput [1.78, 9.33]** and **−357 MB peak memory**.

**Benchmark note.** The benchmark code and all five thread counts match the upstream performance bot. Each Criterion window was shortened to three seconds and run in three alternating rounds. The gain narrows at 16 threads because the baseline already saturates the host.

## 4. Keep witness commitments affine and contiguous

**Problem.** #152 avoids normalization when `Z = 1`, but witnesses still retain 128-byte projective points in a tree map and verification copies or converts them.

**Change.** [Store ordered node IDs beside 64-byte affine points](https://github.com/suryuhh/salt-optimizations/pull/5/files#diff-fe8b4295b0ab97d31152c499ba87211a2b882e5adcdac84419c21858d521edf6R121); verification borrows the array by index. The [four reviewable commits](https://github.com/suryuhh/salt-optimizations/pull/5/commits) separate affine/vector primitives, IPA verification, SALT storage, and the bounded transcript chunk. Wire encoding, ordering and duplicate-ID behavior are preserved.

**Result.** PR #5 against PR #4:

| Proof verification | PR #4 base | PR #5 | Change |
|---|---:|---:|---:|
| Dense, one proof | 87.2 ms | **52.2 ms** | **-40.2%** |
| Dense, one-proof peak | 263 MiB | **170 MiB** | **-35.4%** |
| Dense, 8 concurrent verified proofs/s | 38.2 | **62.9** | **1.64×** |
| Dense concurrent peak | 726 MiB | **273 MiB** | **-62.4%** |
| Ordinary head, one proof | 4.47 ms | **3.79 ms** | **-15.2%** |
| Ordinary-head concurrent peak | 23.0 MiB | **13.1 MiB** | **-42.9%** |

**Why it matters.** The representation change reduces proof-verification time and memory on both dense and ordinary mainnet witnesses.

**Run-to-run variation.** PR #4 does not change proof verification. Its 87.2 ms value comes from a separate alternating run and lies within PR #3's 79.4–87.2 ms range. Comparing PR #3's slowest round with its base's fastest still gives a 29.2% latency reduction.

SALT's repeated-polynomial 16,000-query benchmark is **2.5% slower** (9.06 → 9.29 ms). That fixture coalesces repeated commitments to a small MSM; it is retained as a disclosed narrow regression. Real mainnet proofs above retain hundreds to tens of thousands of commitments and improve in every reported row.

## Measurement and adoption

The dense sample fixes eight positions in blocks 6,900,001–6,900,127 before timing; it averages **10,092 transactions and about 50,000 commitments per witness**. The ordinary-head sample fixes eight positions in blocks 26,000,001–26,000,127; it averages **26.25 transactions and 377–1,973 commitments**. Every proof verified, every computed SALT root matched the preceding mainnet block header, and all arms used identical proof identities.

The benchmark baseline is immutable commit [`7135116`](https://github.com/suryuhh/salt-optimizations/commit/713511618e1db1cc5123398d35a8996352fa6a89), which applies #152's shared CRS, fixed-base proof work, scalar windowed MSM and `Z = 1` normalization skip to upstream main. It excludes #152's node-polynomial cache and refresh path, which optimize witness creation rather than checking the supplied witnesses timed here.

Each table reports the median of three alternating rounds from immutable binaries on one 16-logical-CPU AMD EPYC/Genoa Linux host with AVX-512F/IFMA. Fixed work was prepared before timing; elapsed time and process CPU were measured separately. Memory is process high-water RSS. The two populations show where each optimization engages; they are not a full validator benchmark.

The units follow SALT's existing performance reports: milliseconds for one witness or proof, and the same 10,000-KV trie benchmark used by the repository's performance bot. The headline additionally reports witnesses/s because it runs eight independent witnesses concurrently. Unlike #152's separate `mega-reth` replay measurement, this public harness does not execute or advance complete blocks and therefore does not report blocks/s.

A clean-clone rerun of the public script on the same host reproduced the dense and ordinary-head stack results at **7.37×** and **2.17×**. Across the component rows above, the largest change from the published estimate was 3.7 percentage points.

PR #2 applies directly to upstream main. PRs #2 and #3 also cherry-pick cleanly onto #152's current [`d357608`](https://github.com/megaeth-labs/salt/commit/d357608e784612ed78e30eb79b9a520b20ca3112) head. PR #4 touches the same committer and trie sites as its latest cache refactor, so PRs #4 and #5 remain pinned to the measured base and need conflict resolution after #152 stabilizes. Non-IFMA CPUs keep scalar arithmetic.

[Reproduce every real-witness table from the public 23.6 MB corpus](https://github.com/suryuhh/salt-optimizations/blob/codex/docs/optimization-report/optimization-evidence/REAL_WITNESS_BENCHMARKS.md); the same page records all rounds, input identities and exact source heads. [Final validation](https://github.com/suryuhh/salt-optimizations/blob/codex/docs/optimization-report/optimization-evidence/FINAL_VALIDATION.md) covers formatting, Clippy, default/no-default tests, bucket resizing, random stress and RISC-V no-std.
