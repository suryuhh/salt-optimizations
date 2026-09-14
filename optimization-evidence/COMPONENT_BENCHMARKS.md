# Historical validator replay and kernel evidence

The report's primary evidence is the [direct benchmark of each PR against its immediate base](REAL_WITNESS_BENCHMARKS.md). This page retains the earlier validator replays and the committer-kernel check cited in the report.

## Validator replay

These paired tests replayed the same 128 recorded dense blocks through a validator. Each row uses the preceding implementation as its control, so the effects are historical steps rather than additive gains or measurements of the complete published PRs.

| Measured step | Control | Pairs | Validator throughput | CPU per block | Peak process memory |
|---|---|---:|---:|---:|---:|
| Initial batch decoder | Same pipeline with parallel decoding but scalar point arithmetic | 6 | **+35.46% [25.56, 45.36]** | **-26.32%** | No established change |
| Specialized scalar decoder | Same later decoder, with IFMA disabled in both arms | 6 | **+18.17% [14.60, 21.74]** | **-15.93%** | No established change |
| Batched commitment, window 8 precursor | Pipeline already containing batch decoding and vector MSM | 6 | **+5.56% [1.78, 9.33]** | **-6.36%** | **-357 MB (-17.10%)** |

The scalar-decoder row is a forced-scalar sensitivity test on the same x86-64 host, not a native non-IFMA hardware measurement. The published trie PR uses window 7, whose direct SALT benchmark appears in the main report.

## Committer kernel check

The final window-7 implementation passed the scalar/vector equivalence test. In the same run, its precomputed table built in **0.03 seconds**, versus **0.31 seconds** for window 11, and its eight-lane multiplication took **2,936 ns per point**, versus **8,178 ns** for scalar multiplication (**2.78× faster**). These kernel timings explain the mechanism; the report's five-thread trie table is the end-to-end SALT benchmark.

<details>
<summary>Equivalence-test result and kernel timings</summary>

```text
running 1 test
committer w=11: table build 0.31 s, scalar 6018 ns/mul, lanes 2025 ns/mul, x2.97
committer w=9: table build 0.10 s, scalar 6845 ns/mul, lanes 2258 ns/mul, x3.03
committer w=8: table build 0.05 s, scalar 7297 ns/mul, lanes 2418 ns/mul, x3.02
committer w=7: table build 0.03 s, scalar 8178 ns/mul, lanes 2936 ns/mul, x2.78
test ifma_committer::tests::bench_mul_index_batch ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 33 filtered out; finished in 0.87s
```

</details>
