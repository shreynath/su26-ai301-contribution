# AI301 Open Source Contribution Log
**Student ID:** 153102  
**Program:** CodePath AI301 — AI Open Source Capstone, Summer 2026  
**Section:** 1B | Wednesdays 4PM–6PM PT

---

## Phase I: Issue Selection

### Issue

🔗 **https://github.com/pytorch/ao/issues/3637**

**Repository:** pytorch/ao (torchao)  
**Organization:** PyTorch (Meta Open Source)  
**Languages:** Python, C++, CUDA  
**Labels:** `topic: documentation`, `good first issue`, `module: inference`

---

### Problem Summary

torchao is PyTorch's native quantization and sparsity library, developed under the PyTorch org at Meta and listed on Meta's official open source project page. It is one of the most actively developed quantization libraries in the LLM inference ecosystem — powering int4, int8, float8, and sparse inference optimizations across Llama, Gemma, Flux, and dozens of other models.

The issue is a documentation update: the `static_quantization.rst` doc page still references the old `AffineQuantizedTensor` (AQT) workflow for static int8 quantization, which has been deprecated in favor of the new `Int8StaticActivationInt8WeightConfig` API introduced in torchao v0.15+. The new config-based API (`quantize_(model, Int8StaticActivationInt8WeightConfig(...))`) is simpler, composable, and consistent with the rest of torchao's modern quantization surface. The existing documentation creates confusion for users who follow the tutorial and then find the recommended pattern doesn't match the current codebase. The fix is to update `docs/source/static_quantization.rst` to demonstrate the `Int8StaticActivationInt8WeightConfig` workflow instead of the deprecated AQT-based path — specifically showing both weight-only (`Int8WeightOnlyConfig`) and static (`Int8StaticActivationInt8WeightConfig`) quantization using the calibration pattern PR #3687 established.

---

### Why I Chose This Issue

torchao sits at the intersection of two things that matter to me: it is a Meta open source project (meaning a contribution here counts as a Meta OSS contribution), and it is directly relevant to AI systems work — quantization is the core technique enabling large model deployment at scale. Contributing to torchao puts a real AI infrastructure project on a resume, not a toy app.

The scope is ideal. The issue is labeled `good first issue` by a core maintainer (`jerryzh168`), the task is concrete — update one `.rst` file to reflect the modern `Int8StaticActivationInt8WeightConfig` API — and "done" is unambiguous: the rendered documentation matches current best practices and a maintainer approves. There is already a merged reference PR (#3655, float8 static quant) that demonstrates the exact documentation pattern to follow for the int8 case.

The stack is Python, which I can navigate, and the contribution requires understanding the quantization API surface well enough to write correct example code — which demonstrates genuine technical engagement, not just a text edit. I specifically avoided the heavier torchao issues (FSDP QAT, Tensor Parallelism, CUDA kernels) because they require deep distributed systems knowledge outside the scope of a 3–4 week contribution cycle. Documentation with working code examples is a realistic, high-value contribution.

---

## Phase II: Reproduce & Plan

### Environment Setup

**OS:** Ubuntu 24.04 (WSL2)  
**Python:** 3.11  
**PyTorch:** 2.7+ (nightly recommended for latest torchao compatibility)  
**CUDA:** 12.4 (GPU optional — CPU path works for int8 static quant)

**Setup steps:**
```bash
# 1. Fork and clone the repo
git clone https://github.com/<your-username>/ao.git
cd ao

# 2. Create a virtual environment
python -m venv .venv
source .venv/bin/activate

# 3. Install torchao in editable mode with dev dependencies
pip install torch --index-url https://download.pytorch.org/whl/cu124
pip install -e ".[dev]"

# 4. Confirm the current static_quantization doc builds
cd docs
pip install -r requirements.txt
make html
# Open docs/build/html/static_quantization.html in a browser
# Confirm it renders the OLD AQT-based workflow (this is the bug state)

# 5. Confirm Int8StaticActivationInt8WeightConfig is importable
python -c "from torchao.quantization import Int8StaticActivationInt8WeightConfig; print('OK')"
```

Setup took approximately 30 minutes including PyTorch nightly install. No blockers; the `docs/` directory has a working Sphinx setup with a `make html` target.

**Working branch:**  
`https://github.com/<your-username>/ao/tree/fix-static-quant-docs`

---

### Steps to Reproduce

The "bug" is documentation that references a deprecated workflow. Reproduction means confirming the mismatch:

1. Clone the repo and open `docs/source/static_quantization.rst`
2. Observe that the tutorial walks through the old `AffineQuantizedTensor` / `to_affine_quantized_intx_static` path for quantizing a model
3. Now open `torchao/quantization/quant_api.py` and search for `Int8StaticActivationInt8WeightConfig` — this is the current recommended API
4. **Expected (post-fix):** The `static_quantization.rst` doc demonstrates the modern `Int8StaticActivationInt8WeightConfig` flow, matching the pattern established in PR #3655 for float8 static quant
5. **Actual (current state):** The doc uses the deprecated `to_affine_quantized_intx_static` path and `StaticQuantConfig` custom class that the issue tracker explicitly says should be updated

Additionally: run the existing example script to confirm it still executes correctly as a baseline:
```bash
python examples/quantization/static_quant.py
```

---

### Solution Approach

**Understand:**  
The `static_quantization.rst` tutorial currently teaches users to build their own `ObservedLinear`, `QuantizedLinear`, and `StaticQuantConfig` classes from scratch using the low-level `to_affine_quantized_intx_static` API. While this is instructive for understanding internals, it obscures the recommended modern workflow. The issue asks to update this to use `Int8StaticActivationInt8WeightConfig` — the high-level config-based API that handles calibration and quantization in a composable way via `quantize_(model, config)`.

The reference implementation to follow is PR #3655, which added the float8 static quant flow to the docs. That PR's structure — calibration with a `MinMaxObserver`, then `quantize_()` with a built-in config — is the exact pattern to replicate for int8.

**Plan:**
1. Read PR #3655 (float8 static quant docs) and issue #3687 (which landed `Int8StaticActivationInt8WeightConfig`) in full to understand the exact API surface
2. Read the current `docs/source/static_quantization.rst` end to end and annotate every section that references the deprecated path
3. Rewrite the tutorial to demonstrate the modern workflow:
   - Keep the `ToyLinearModel` setup (it is still correct)
   - Replace the custom `ObservedLinear`/`QuantizedLinear`/`StaticQuantConfig` sections with the `AffineQuantizedMinMaxObserver` + `Int8StaticActivationInt8WeightConfig` pattern
   - Show both weight-only quantization (`Int8WeightOnlyConfig`) and static activation+weight quantization (`Int8StaticActivationInt8WeightConfig`) as the maintainer suggested in the issue thread
   - Update all code snippets to import from current public API paths
4. Run the updated code snippets locally end-to-end to confirm they execute without errors
5. Run `make html` in `docs/` and visually verify the rendered page looks correct
6. Check that no existing doc tests break: `pytest docs/` if applicable

**Review:**  
torchao uses standard Python style with `pre-commit` hooks. Will run `pre-commit run --all-files` before pushing. Commit message follows the repo convention: `docs: update static quantization tutorial to use Int8StaticActivationInt8WeightConfig`.

**Evaluate:**  
- Updated code examples execute end-to-end without errors on CPU
- Rendered HTML doc clearly shows the modern `Int8StaticActivationInt8WeightConfig` workflow
- Both weight-only and static quant paths are demonstrated, as requested in the issue thread
- `pre-commit` passes with no violations
- Maintainer (`jerryzh168` or `namgyu-youn`) reviews and confirms the update matches established conventions

---

## Phase III: Implementation

*(To be filled in during Week 3)*

- [ ] Updated `docs/source/static_quantization.rst` with `Int8StaticActivationInt8WeightConfig` workflow
- [ ] Code examples verified running end-to-end locally
- [ ] `make html` confirmed — rendered doc looks correct
- [ ] `pre-commit` passing

---

## Phase IV: Pull Request

- **PR Link:** *(pending)*
- **PR Summary:** *(pending)*
- **Maintainer Feedback Log:** *(pending)*
