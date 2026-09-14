# Final stack validation

The following checks passed on the x86-64 benchmark host for the complete four-PR implementation. The current source snapshot is [`52a460c`](https://github.com/suryuhh/salt-optimizations/commit/52a460c12b03e6c352f7c9a4cb207e38fc244bbf). Its source and tests match the previously published PR head byte-for-byte; the measured revision differs only in a documentation comment. No executable code, build inputs, or tests changed during republication.

| Check | Exact command | Exit code | Seconds |
|---|---|---:|---:|
| fmt | `cargo fmt --all -- --check` | 0 | 0.265 |
| sort | `cargo sort --check --workspace --grouped --order package,workspace,lints,profile,bin,benches,dependencies,dev-dependencies,features` | 0 | 0.064 |
| check all targets | `cargo check --all-targets --locked -j8` | 0 | 1.819 |
| clippy | `cargo clippy --all-targets --locked -j8 -- -D warnings` | 0 | 2.57 |
| tests | `cargo test --workspace --locked -j8 -- --test-threads=8` | 0 | 83.29 |
| bucket resize | `cargo test --features test-bucket-resize --locked -j8 -- --test-threads=8` | 0 | 79.413 |
| random stress | `cargo test -p salt --features test-bucket-resize test_e2e_random_stress --locked -j8 -- --ignored --nocapture` | 0 | 34.122 |
| riscv no std | `cargo check -p salt --target riscv64imac-unknown-none-elf --no-default-features --locked -j8` | 0 | 0.966 |
| no default tests | `cargo test --no-default-features --locked -j8 -- --test-threads=8` | 0 | 114.256 |
| no default bucket | `cargo test --no-default-features --features test-bucket-resize --locked -j8 -- --test-threads=8` | 0 | 111.829 |

The host checks completed at 2026-09-14T03:04:01Z. GitHub’s Rust workflow also passed on all four original PR heads before republication. These are the recorded checks of the unchanged implementation; they are not represented as new workflow executions on the replacement repository.
