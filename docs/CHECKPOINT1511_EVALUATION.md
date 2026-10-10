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

At preparation time, the CPU cache test passed **4,226 checks**, and the upstream offline tooling passed **19 tests**. Added native snapshot regressions check all five overwritten running-state fields, checkpoint-chain ownership after rejected insertion, native restore of the rolled-back checkpoint, and invalid capture without consuming the source chain or replacing the published image. These native regressions have not run yet. CUDA, HIP and SYCL builds, model correctness and performance are still pending. No local speedup or reduction in peak RAM is claimed. Qualification must keep keyed-answer correctness, memory floors, pin/image/steering identity, cancelled-request behavior and safe failures, and must measure full task time separately from capture timing.
