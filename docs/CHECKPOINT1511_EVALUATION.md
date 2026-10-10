# Isolated evaluation of prompt-cache allocation reuse

This branch evaluates [Strata PR #1511](https://github.com/Niko1221/Strata/pull/1511), by **InB4DevOps**, against the adaptive `.42` source at `1dd738fbd241dd1249ca462533fca162bd65be78`. It is not a production promotion or a new competing feature proposal.

The port retains the original author and commit references:

- `3c662499b11e56c619d2e68834adb95dbeeadbc8`: phase tracing and reusable running-state host allocations.
- `bd6d3bf62c116b105bf7fd44d2e172f3ffb4b212`: move a checkpoint chain into its parked snapshot when the caller replaces it.

This is adapted Strata code under the repository's MIT license. No implementation from llama.cpp, ggml or AI2 is copied. Their related research is separate prior art; no performance result from another machine is presented as a local result.

Conflict resolution preserves `.42`'s minimum-token admission, current host-memory/commit sampling and bounded physical-memory eviction loop, including post-capture admission. SYCL keeps its current available-host-memory API. Live-memory and resource-lease controls are unchanged.

The local follow-up corrects one accounting edge: a copied snapshot estimate includes checkpoint sizes, while a moved chain keeps its capacities. A checked helper replaces the logical checkpoint contribution with the actual owned capacity before cache-budget admission, then credits only that ownership when calculating new physical allocation. Overflow or an invalid layer-split chain refuses the move estimate. This avoids subtracting spare capacity that the original estimate never included.

The upstream switches are explicit evaluation controls:

| Variable | `0` | `1` |
|---|---|---|
| `STRATA_PROMPT_CACHE_TRACE` | Disable detailed trace | Record capture and restore phases |
| `STRATA_RUNNING_STATE_REUSE` | Reallocate running-state capture buffers | Reuse validated owned allocations and overwrite every state byte |
| `STRATA_CHECKPOINT_MOVE` | Copy the outgoing checkpoint chain | Move it only where the caller will replace/reset it |

Use the same trace setting for comparison arms. Running-state reuse and chain move are default-on in the original PR; the evaluation must set both explicitly. The shell workload runner is Linux-oriented, disables thinking and accepts returned text without checking its content. It is tooling, not a Windows/high-reasoning correctness gate. Synthetic traces are never performance evidence.

The CPU cache test passed **4,226 checks**, and the upstream offline tooling passed **19 tests**. A Windows CUDA build then passed **14 selected native tests**, including the snapshot regressions, plus the IQ3_S fused/MMQ reference parity check. The new regressions check all five overwritten running-state fields, checkpoint-chain ownership after rejected insertion, native restore of the rolled-back checkpoint, and invalid capture without consuming the source chain or replacing the published image. The spare-capacity fixture also reproduces the original under-admission calculation and checks that the corrected physical admission remains exactly the fresh 97-byte contribution. HIP and SYCL are source-reviewed only; they have not been built or run.

## Projection-off local qualification

Measured on 2026-10-10 with Windows, Ryzen 9 7940HS, RTX 4070 Laptop 8 GB and 64 GB DDR5-5600. The model was ISTA-DASLab Qwen3.8-Flash-Next GSQ-RCO IQ3_S, with a 65,536-token configured context, high reasoning, CPU vision, MTP, a 2,048 MiB conversation cache and two cache slots. The native build used CUDA 13.4.59, MSVC 19.51.36260 (toolset 14.51.36231) and `sm_89`. Experimental speed projection was disabled in sampling and no native control-vector argument was loaded.

All six runs used the same executable at evaluation source `66c247147ae8e7e636eac72f496ce94a9e47a811`, with tracing enabled in every arm. The control explicitly disabled both reuse and move in that executable; **it was not an unmodified public-release binary**. The only experimental switches were `STRATA_RUNNING_STATE_REUSE` and `STRATA_CHECKPOINT_MOVE`. Each run used a fresh process and cache. Execution order was control, reuse, move, combined, combined, control.

Each run sent the same 13 fixed-history HTTP requests: one image question, two interleaved short conversations and their repeated prefixes, then a function call and its result. Request hashes, server/source/binary hashes and prefix-reuse decisions matched across all runs. Requests used seed 42, high reasoning with an 8,192-token reasoning budget and a 12,288-token total output cap. Actual prompts were **139–390 tokens**, so this is a short cache-switching qualification, **not a 64K prompt benchmark or a Hermes autonomy test**.

| Run | Reuse / move | Passed requests | Output tokens | HTTP request time (s) | Park capture total (ms) | Restore total (ms) |
|---|---|---:|---:|---:|---:|---:|
| Control 1 | 0 / 0 | 13 / 13 | 691 | 69.683 | 837.1 | 323.4 |
| Reuse only | 1 / 0 | 13 / 13 | 689 | 61.172 | 668.6 | 241.4 |
| Move only | 0 / 1 | 13 / 13 | 692 | 67.546 | 331.7 | 149.7 |
| Combined 1 | 1 / 1 | 13 / 13 | 712 | 69.079 | 261.4 | 126.2 |
| Combined 2 | 1 / 1 | 13 / 13 | 688 | 68.249 | 246.6 | 144.6 |
| Control 2 | 0 / 0 | 13 / 13 | 721 | 62.703 | 748.4 | 269.6 |

Every run stored 12 parked snapshots, created nine checkpoints and restored eight snapshots. Native and vision process identities stayed unchanged through each run; owned processes were confirmed gone after cleanup. The sampled minimum available RAM across the six runs was **10.069 GiB**, and minimum native-reported free VRAM was **324 MiB**, above the 3 GiB / 250 MiB guards.

The useful measured result is narrower than total task speed: the combined setting reduced total park capture time by **68.8% in the first comparison and 67.0% in the reverse-order comparison**, about **0.50–0.58 seconds across 12 parks**. Trace accounting also shows the intended paths were exercised. Running-state reuse raises retained allocation capacity beyond the control's existing KV reuse; checkpoint move credits already owned checkpoint storage. Cumulative additional-allocation admission estimates were 4,316.825 / 4,316.663 MiB for the two controls and 601.539 / 600.957 MiB for the combined runs. These are **sums of per-event admission estimates, not concurrent RAM savings or measured allocator traffic**.

No end-to-end speedup is established. The first image request alone varied from **6.438 to 13.890 seconds**, before these cache optimizations could be reused; its host-side difference dominated several totals. Output lengths varied even with identical seeded requests. Small disclosed read-only inspection activity also overlapped some runs. The HTTP column includes image processing and request overhead but excludes process startup/cleanup, and is not a clean throughput benchmark. Park save time includes capture time and must not be added to it. Two control and two combined runs, with only one run of each singular setting, do not establish a population estimate or a reduction in peak RAM. Earlier projection-enabled or interrupted fixtures are separate evidence and are not included here.
