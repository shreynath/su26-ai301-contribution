# AI301 Open Source Contribution Log
**Student ID:** 153102
**Program:** CodePath AI301 — AI Open Source Capstone, Summer 2026
**Section:** 1B | Wednesdays 4PM–6PM PT

---

## Phase I: Issue Selection

### Issue

🔗 **https://github.com/pytorch/ao/issues/1920**

**Repository:** pytorch/ao (torchao)
**Organization:** PyTorch (Meta Open Source)
**Languages:** Python, CUDA
**Labels:** `good first issue`, `triaged`

---

### Problem Summary

torchao is PyTorch's native library for quantization and sparsity, developed under the PyTorch org and used to accelerate model training and inference across the LLM ecosystem. Issue #1920, opened by maintainer `jcaip`, comes out of a training recipe for 2:4 activation sparsity: FFNs built with Squared-ReLU activations (instead of the more common SwiGLU) exhibit greater than 85% activation sparsity with minimal accuracy loss. The original plan was to accelerate the resulting sparse-activation × dense-weight matmul using 2:4 structured sparsity, which caps theoretical speedup at 2x because it only enforces 50% sparsity — well below the 85%+ actually present in the activations.

Maintainer `janeyx99` proposed a different angle: instead of (or in addition to) accelerating the matmul, use **activation compression** to exploit the actual sparsity level for memory savings. The idea is to compress the sparse Squared-ReLU activations with NVIDIA's **nvcomp** library before storing them for the backward pass, then decompress on the way into the backward computation. This trades compute for memory, which matters because activation memory (not FLOPs) is often the binding constraint for training large FFNs. The concrete next step, agreed on by `jcaip`, is to benchmark nvcomp's compression ratio and runtime overhead on sparse Squared-ReLU activation tensors before anyone commits to building the actual `quantize_`/tensor-subclass integration into torchao.

---

### Why I Chose This Issue

This issue sits closer to systems/ML-infra work than a typical documentation fix, and it let me engage with a real open research question instead of a self-contained bug. The scope is well-bounded for a first contribution: the maintainers have explicitly said the first deliverable is *benchmarking numbers*, not a production PR — `jcaip` wrote that he isn't currently working on this and "would gladly accept a PR" once the overhead/compression-ratio picture is understood. That means "done" for Phase I–III is unambiguous: a reproducible benchmark script and a written summary of nvcomp's compression ratio and latency overhead on representative sparse activation tensors.

I also liked that the issue has an active, multi-year discussion thread (opened March 2025, still being triaged as of mid-2026) with several contributors probing different angles — CPU inference, unstructured sparsity, kernel-level 2:4 acceleration — which gave me real context to read before jumping in. Rather than fabricate a starting point, I actually posted on the thread to confirm the benchmarking approach was still the right entry point and hadn't gone stale; `janeyx99` looped in `vkuzo` (co-owner of quantization/sparsity efforts in torchao) to confirm relevance, and I'm currently waiting on that confirmation before opening any code. This mirrors how real open-source contribution starts: aligning with maintainers before writing throwaway code.

The technical surface — GPU memory profiling, activation sparsity patterns, and a compression library's Python bindings — is a good stretch for my background in binary/systems work from SEFCOM without requiring me to write new CUDA kernels, which the maintainers themselves flagged as the harder, longer-term part of this issue.

---

## Phase II: Reproduce & Plan

### Environment Setup

**OS:** Ubuntu 24.04
**Python:** 3.11
**PyTorch:** 2.7+ (nightly recommended for latest torchao compatibility)
**CUDA:** 12.4
**GPU:** required — nvcomp is a GPU-side compression library, so this benchmark cannot run on CPU-only hardware

**Setup steps:**
```bash
# 1. Fork and clone the repo
git clone https://github.com/<your-username>/ao.git
cd ao

# 2. Create a virtual environment
python -m venv .venv
source .venv/bin/activate

# 3. Install torch + torchao in editable mode
pip install torch --index-url https://download.pytorch.org/whl/cu124
pip install -e ".[dev]"

# 4. Install nvCOMP's Python bindings (NVIDIA-published wheel, matched to CUDA 12)
pip install nvidia-nvcomp-cu12

# 5. Sanity check the nvcomp import and version
python -c "from nvidia import nvcomp; print('nvcomp version:', nvcomp.__version__)"

# 6. Confirm torchao's sparsity utilities are importable (for generating 2:4/unstructured
#    sparse test tensors to benchmark against)
python -c "from torchao.sparsity import sparsify_; print('OK')"
```

Setup is expected to take about 30–45 minutes, mostly for the PyTorch nightly and CUDA toolkit matching. The main risk flagged in the issue thread itself is a CUDA/driver version mismatch between the nvcomp wheel and whatever CUDA build of PyTorch is installed, since nvcomp ships CUDA-version-specific wheels (`nvidia-nvcomp-cu11` vs `nvidia-nvcomp-cu12`).

**Working branch:**
`https://github.com/<your-username>/ao/tree/activation-compression-nvcomp-bench`

---

### Steps to Reproduce

There is no existing bug to reproduce here — this is a benchmarking task that starts from scratch — so "reproduction" means establishing the baseline the issue is asking about:

1. Generate a synthetic activation tensor that mimics a Squared-ReLU FFN's activation statistics: apply `x.relu().square()` to a random Gaussian input tensor, which produces the same clipped, highly-sparse distribution (>85% zeros) described in the linked paper.
2. Confirm the sparsity level empirically: `(activation == 0).float().mean()` should land in the 85%+ range referenced in the issue.
3. Feed the same tensor through nvcomp's `Codec` across several supported algorithms (LZ4, GDeflate, Bitcomp, ANS, Zstd) and record the compressed buffer size relative to the uncompressed buffer size for each.
4. **Expected (per the issue's premise):** because the activation tensor is mostly zeros, general-purpose compressors should achieve a compression ratio well beyond the 2x ceiling that 2:4 structured sparsity can offer, at some added compress/decompress latency.
5. **Actual (unknown, to be measured):** this is exactly the open question `jcaip` posed — the compression ratio and, critically, the added latency of compress-then-store / load-then-decompress versus just storing the dense activation, need to be measured before anyone can say whether this is a net win for training throughput.

---

### Solution Approach

**Understand:**
The task is narrower than "add activation compression to torchao" — it is "produce the numbers that tell us whether activation compression is worth building." Two things need measuring: (1) compression ratio on realistic sparse Squared-ReLU activations, and (2) the wall-clock overhead of compression/decompression relative to the FFN forward/backward pass it would sit inside, since GPU compression only helps if the memory savings outweigh the added compute time in a training step.

**Plan:**
1. Read the referenced paper (`https://openreview.net/pdf?id=O5feVk7p6Y`) to pull realistic activation shapes and sparsity statistics for representative FFN sizes, rather than using arbitrary synthetic shapes.
2. Write a small benchmarking script (`benchmarks/nvcomp_activation_compression.py`) that:
   - Constructs Squared-ReLU activation tensors at a few representative sizes (e.g. matching a 7B-class FFN's intermediate dimension)
   - Runs nvcomp's `Codec` across multiple algorithms (LZ4, GDeflate, Bitcomp, ANS, Zstd — the set nvcomp exposes) and records compressed size vs. original size
   - Times compress and decompress calls using CUDA events (not wall-clock `time.time()`, to avoid host/device sync noise) and reports overhead as a percentage of a representative FFN forward+backward step
3. Compare the best-performing algorithm's compression ratio against the 2x ceiling of 2:4 structured sparsity, to directly answer the question the issue raises.
4. Write up results as a short markdown summary (compression ratio table + overhead table) and post it back on issue #1920 before writing any integration code, per `jcaip`'s guidance that a PR should follow, not precede, the benchmarking.
5. Only after maintainer sign-off on the numbers, scope out what a `quantize_`-style or tensor-subclass-based integration into torchao would look like for the actual PR.

**Review:**
torchao uses standard Python style with `pre-commit` hooks; I'll run `pre-commit run --all-files` on the benchmark script before pushing. Since this starts as a benchmarking script rather than a library change, I'll place it under `benchmarks/` rather than `torchao/`, consistent with how other exploratory sparsity benchmarks are organized in the repo.

**Evaluate:**
- Benchmark script runs end-to-end on a CUDA-enabled machine and produces a reproducible compression-ratio + overhead table
- Results directly answer the question posed in the issue: does nvcomp compression beat the 2x ceiling of 2:4 sparsity, and at what overhead cost?
- Maintainers (`jcaip`, `janeyx99`, and/or `vkuzo` once they weigh in) confirm the benchmarking methodology is sound and the direction is still relevant before I scope a follow-up implementation PR

---

## Phase III: Implementation

- [ ] Confirmed with maintainers (`janeyx99`/`vkuzo`) that the nvcomp benchmarking direction is still current — **currently pending reply on the issue thread**
- [ ] `benchmarks/nvcomp_activation_compression.py` written and generating representative sparse Squared-ReLU activation tensors
- [ ] Compression ratio measured across nvcomp algorithms (LZ4, GDeflate, Bitcomp, ANS, Zstd)
- [ ] Compress/decompress overhead measured via CUDA events, expressed relative to a representative FFN forward+backward step
- [ ] Results written up and posted back to issue #1920 for maintainer feedback
- [ ] `pre-commit` passing on the benchmark script

---

## Phase IV: Pull Request

- **PR Link:** *(pending — blocked on maintainer confirmation of direction)*
- **PR Summary:** *(pending)*
- **Maintainer Feedback Log:** *(pending)*
