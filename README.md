# AI301 Open Source Contribution Log
**Student ID:** 153102
**Program:** CodePath AI301 — AI Open Source Capstone, Summer 2026
**Section:** 1B | Wednesdays 4PM–6PM PT

---

## Phase I: Issue Selection

### Issue

🔗 **https://github.com/Eventual-Inc/Daft/issues/7196**

**Repository:** Eventual-Inc/Daft
**Organization:** Eventual (Daft — high-performance data engine for AI/multimodal workloads)
**Languages:** Rust, Python
**Labels:** `bug`, `fix`, `help wanted`, `p1`

---

### Problem Summary

Daft is a Rust-powered, Python-native distributed data engine used for multimodal and AI data processing (images, audio, video, structured data). Issue #7196, opened by maintainer `everettVT`, reports that when a scalar UDF (`@daft.func`) raises an exception, Daft correctly propagates the underlying error to the user. However, when a UDF with a `list[...]` return dtype raises the exact same exception, the real error is swallowed and replaced with a cryptic, unrelated error: `DaftError::ValueError Need at least 1 series to perform concat`.

This was surfaced concretely through `daft.functions.video_frames()`: when the optional `pillow` dependency is missing, the function is supposed to raise a clear `ImportError` telling the user to `pip install daft[video]`. Instead, users see the confusing concat error, with no indication that a missing dependency is the actual cause. `video_metadata()` (a scalar-return function) does not hit this bug, which is what makes the divergence between scalar- and list-return UDF error handling visible.

The maintainer's working hypothesis, stated directly in the issue, is that the executor attempts to concat the (empty) partial/child series results of a list-return UDF batch *before* the worker's raised exception is surfaced — so when there are zero series to concat, that secondary failure fires and masks the original one.

---

### Why I Chose This Issue

This is a very recently opened (July 2, 2026), fully unclaimed issue — no assignee, no linked PR — filed directly by a Daft maintainer with `help wanted` and `p1` labels, meaning it's both wanted and considered a real priority rather than backlog filler. That combination is rare: most other unassigned "good first issue"-style candidates I checked in Daft, vLLM, and torchao already had an open PR against them by the time I looked, even though they still showed as unclaimed in older reference data.

The bug is also explicitly reproducible on macOS (the reporter's own environment is macOS arm64), and it has nothing to do with CUDA or GPU-specific code paths — it's a pure error-propagation / executor-logic bug in how list-return UDF results are assembled after a worker exception. That makes it realistic to fully reproduce, debug, and eventually fix on my own Mac, without needing GPU access or a hardware-gated CI environment, which ruled out several other otherwise-appealing candidates in vLLM and torchao's CUDA-heavy repos.

The issue also comes with a clean, minimal, dependency-free repro (no video files, no `pillow`/`av` needed) contrasting a scalar UDF (works correctly) against a list-return UDF (fails), which gives me a very concrete, bounded starting point: trace why the two code paths diverge. Per the maintainer's own framing, the surface fix is about correct error propagation, not a large new feature, which makes "done" reasonably well-defined for a first Rust-touching contribution to a top-tier data engineering repo.

Before writing any code, I posted on the issue thread to confirm the direction: I noted the likely root cause (concat-before-propagate on an empty list-of-series) and asked whether the fix should live purely in the error-propagation logic (surface the worker's exception before attempting the concat) or whether `Series::concat` itself should handle the empty-list case by constructing an empty child array from the expected list dtype. I'm currently waiting on `everettVT`'s reply before scoping implementation work, mirroring how real open-source contribution starts — aligning with maintainers before writing throwaway code.

---

## Phase II: Reproduce & Plan

### Environment Setup

**OS:** macOS (Apple Silicon or Intel — no GPU/CUDA dependency for this issue)
**Python:** 3.10+
**Rust:** stable toolchain via `rustup` (Daft's core engine is Rust, exposed to Python via PyO3/maturin)
**Build tool:** `maturin`

**Setup steps:**
```bash
# 1. Fork and clone the repo
git clone https://github.com/<your-username>/Daft.git
cd Daft

# 2. Install Rust (if not already installed)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup default stable

# 3. Create a virtual environment
python -m venv .venv
source .venv/bin/activate

# 4. Install maturin and build Daft in editable/develop mode
pip install maturin
maturin develop  # use --release for a slower build but faster runtime while benchmarking

# 5. Sanity check the build
python -c "import daft; print(daft.__version__)"
```

Setup is expected to take roughly 20–30 minutes on first build, mostly for the initial Rust compile (`daft-core`, `daft-io`, `daft-dsl`, etc.). Incremental rebuilds after small Rust changes are much faster with `maturin develop` (debug build) than `--release`.

**Working branch:**
`https://github.com/<your-username>/Daft/tree/list-udf-error-propagation`

---

### Steps to Reproduce

1. Confirm the control case works correctly — a scalar UDF that raises should propagate its real exception:
   ```python
   import daft
   from daft import col

   @daft.func(return_dtype=daft.DataType.int64())
   def scalar_import_error(x):
       raise ImportError("The 'pillow' module is required. Install daft[video].")

   daft.from_pydict({"x": [1]}).with_column("y", scalar_import_error(col("x"))).collect()
   # Expected & actual: DaftError::ComputeError ... ImportError: The 'pillow' module is required...
   ```
2. Reproduce the bug — the same exception raised from a `list[...]`-return UDF should propagate identically, but doesn't:
   ```python
   @daft.func(return_dtype=daft.DataType.list(daft.DataType.int64()))
   def list_import_error(x):
       raise ImportError("The 'pillow' module is required. Install daft[video].")

   daft.from_pydict({"x": [1]}).with_column("y", list_import_error(col("x"))).collect()
   # Actual (buggy): DaftError::ValueError Need at least 1 series to perform concat
   # Expected: the same ImportError as the scalar case
   ```
3. **Expected (per the issue's premise):** list-return UDFs should surface the worker's original exception, exactly like scalar UDFs do.
4. **Actual:** the concat step over the list UDF's (empty, post-failure) child series results fails first, masking the real error entirely — this reproduces cleanly and consistently on macOS with no external dependencies.

---

### Solution Approach

**Understand:**
The bug is narrower than "list UDFs are broken" — it's specifically that error propagation and result-assembly are ordered incorrectly for list-return UDFs. Somewhere in the executor, after a UDF worker raises, the code path for list dtypes tries to concat whatever partial/child series it has *before* checking whether the worker actually failed, whereas the scalar path checks for the worker exception first. The task is to find that divergence point in the Rust source and correct the ordering (or handle the empty-concat case gracefully) without changing the successful-case behavior for either path.

**Plan:**
1. Locate the UDF result-assembly code paths in the Rust source (likely under `src/daft-dsl` or wherever `daft.func`/UDF execution and result concatenation live — e.g. near `Series::concat` usage flagged in the issue) and diff the scalar vs. list dtype handling to find where they diverge.
2. Trace how the scalar path surfaces a worker exception, to use as the reference behavior the list path should match.
3. Reproduce the two snippets above locally against a debug build (`maturin develop`) and set breakpoints / add temporary `eprintln!` diagnostics around the list dtype's result-assembly path to confirm exactly where the concat call fires relative to the exception check.
4. Once maintainer feedback comes back on the two options I raised in my issue comment (propagate-before-concat vs. handle empty-list-of-series in `Series::concat` itself), implement the agreed approach.
5. Add a regression test (Python-level, using the two minimal repro snippets from the issue) asserting that both scalar and list UDF paths surface the same underlying exception type/message, so this can't silently regress.

**Review:**
Daft uses `pre-commit` (rustfmt/clippy for Rust, ruff for Python) — I'll run `pre-commit run --all-files` before pushing anything. Since this is a bug in core UDF execution rather than an isolated utility, the fix will likely need a Rust-side test in addition to a Python integration test, consistent with how other executor-level fixes are covered in the repo.

**Evaluate:**
- Both repro snippets (scalar and list-return UDF) raise the *same* underlying exception after the fix
- No behavioral change to the already-correct scalar UDF error path
- A regression test is added covering both cases
- Maintainer (`everettVT`) confirms the chosen fix approach (propagation-order fix vs. `Series::concat` empty-case handling) before/while the PR is opened

---

## Phase III: Implementation

- [ ] Confirmed with maintainer (`everettVT`) which fix approach is preferred — **currently pending reply on the issue thread**
- [ ] Local build working via `maturin develop`; both repro snippets run and reproduce the reported (buggy) and control (correct) behavior
- [ ] Divergence point between scalar and list UDF error-propagation paths located in the Rust source
- [ ] Fix implemented per agreed approach (propagate worker exception before concat, and/or handle empty-list-of-series in `Series::concat`)
- [ ] Regression test added covering both scalar and list UDF exception propagation
- [ ] `pre-commit` passing (rustfmt, clippy, ruff)

---

## Phase IV: Pull Request

- **PR Link:** *(pending — blocked on maintainer confirmation of fix approach)*
- **PR Summary:** *(pending)*
- **Maintainer Feedback Log:** *(pending)*
