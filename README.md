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

**Update (post-investigation):** the hypothesis was directionally correct but one layer off — the divergence isn't in the executor/`daft-dsl`, it's in `series_from_literals_iter` (`src/daft-core/src/series/from_lit.rs`), which both scalar and list UDFs funnel per-row results through. See Solution Approach and Phase III below for the confirmed root cause.

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

**Actual local setup (completed):** cloned into `~/Programming/Daft`, installed `rustup` (repo auto-selects its pinned `nightly-2025-09-03`), installed `uv` via Homebrew, and built with `make build` (wraps `maturin`) rather than a bare `maturin develop`. Only Python 3.13/3.14 were available locally (not the Makefile's default 3.11), so the venv was created with `make .venv PYTHON_VERSION=python3.13`. `daft.abi3.so` built and imported successfully. Note: `gh` (GitHub CLI) is not installed, so forking/pushing/opening the PR is still a manual step.

**Working branch:**
`fix/list-udf-error-propagation-7196` (local; not yet pushed — changes are currently uncommitted in the working tree, pending maintainer confirmation before committing)

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

**Understand (confirmed, post-investigation):**
The divergence is not in the UDF executor itself — it's one layer down, in how per-row results are assembled into a `Series`. Both scalar and list UDFs funnel their per-row outputs through `series_from_literals_iter` in `src/daft-core/src/series/from_lit.rs`. That function already accumulates per-row errors into an `errs` map (via the `unwrap_inner!` macro) and is meant to surface them at the end — but the ordering is wrong for container dtypes:
- **Scalar dtypes** (`Int64`, `Utf8`, …) build their array with a null in each errored slot, never fail during construction, and reach the final `errs` check — so the real exception is correctly surfaced.
- **Container dtypes** (`List`, and structurally the same for `Map`/`Tensor`/`SparseTensor`/`Embedding`) build their output by concatenating the surviving per-row child series via `ListArray::from_series` → `Series::concat`. When every row raises, there are zero surviving child series, so `Series::concat(&[])` fails immediately with `Need at least 1 series to perform concat` and returns via `?` — **before** the `errs` check ever runs, masking the true error.

**Plan (as executed):**
1. Located the UDF result-assembly code path in the Rust source: `series_from_literals_iter` in `src/daft-core/src/series/from_lit.rs` — one level below where the issue's own hypothesis pointed (the executor/`daft-dsl`), but the same root idea (concat-before-propagate).
2. Confirmed the scalar path already surfaces worker exceptions correctly, and used it as the reference behavior.
3. Reproduced both minimal repro snippets against a debug build (`make build`) and traced exactly where the list dtype path calls `Series::concat` relative to the `errs` check.
4. Chose the **error-propagation fix** over the alternative (making `Series::concat` tolerate an empty slice): wrapped the series construction in an immediately-invoked closure and moved the `errs` check ahead of returning any construction result, so accumulated per-row errors always win over a downstream failure they caused. This fixes all container dtypes in one place (not just `list`), and keeps `Series::concat`'s existing contract intact — there's an existing test (`tests/series/test_concat.py:250`) asserting `Series.concat([])` should raise, so changing that primitive would have weakened it for no real gain.
5. Added a parametrized regression test (`list[int]`, `list[struct]`, scalar `int64`) to `tests/udf/test_row_wise_udf.py` asserting scalar and list UDF paths surface the same underlying exception.

**Review:**
Ran `rustfmt --check` on the changed Rust file (clean) and `ruff check` / `ruff format --check` on the changed test file. One pre-existing `ruff` finding remains in `test_row_wise_udf.py` (a blind `except Exception` in unrelated `ray`-GPU-detection test code around line 101–103) — confirmed via `git stash` that it exists on the unmodified baseline too, so it's not something my change introduced.

**Evaluate (confirmed via full re-verification against a fresh rebuild):**
- ✅ Scalar UDF (control) and `list[int]` UDF (minimal repro from the issue) now raise the **identical** underlying `ImportError` — no more concat error
- ✅ `list[struct]` and `list[image]` return dtypes also fixed (confirms the bug affects container dtypes generally, as the issue speculated)
- ✅ Real user-facing path verified end-to-end: `video_frames()` on an actual sample video (`tests/assets/sample_video.mp4`) with `pillow` mocked unavailable now raises `ImportError: The 'pillow' module is required for frame decoding. Install it with pip install daft[video].` — while `video_metadata()` continues to work, exactly matching the issue's stated "Expected behavior"
- ✅ Full `tests/udf/test_row_wise_udf.py` (39 passed), `test_concat.py` + `test_audio.py` (162 passed), and the Rust `from_lit` unit tests (34 passed) all pass on the fresh build
- ✅ `rustfmt` clean; no new `clippy`/`ruff` findings introduced

---

## Phase III: Implementation

- [ ] Confirmed with maintainer (`everettVT`) which fix approach is preferred — **still pending reply on the issue thread.** I prototyped the propagation-order fix ahead of confirmation rather than waiting idle; the draft comment in Phase IV is written to be transparent about that and explicitly ask for sign-off before opening a PR.
- [x] Local build working via `make build` (`maturin`-based); both repro snippets run and reproduce the reported (buggy, pre-fix) and control behavior
- [x] Divergence point between scalar and list UDF error-propagation paths located in the Rust source — confirmed as `series_from_literals_iter` in `src/daft-core/src/series/from_lit.rs` (one layer below the executor, refining the issue's original hypothesis)
- [x] Fix implemented — error-propagation approach: accumulated per-row `errs` are now surfaced before any downstream construction failure (e.g. empty-slice `concat`) they caused
- [x] Regression test added — parametrized `list[int]`, `list[struct]`, and scalar `int64` cases in `tests/udf/test_row_wise_udf.py`
- [x] `pre-commit`-equivalent checks passing — `rustfmt --check` clean; `ruff check`/`ruff format --check` show zero new findings (one pre-existing, unrelated finding confirmed present on baseline)

**Additional verification beyond the original checklist**, done via a fresh rebuild to guarantee the binary matched source:
- [x] `list[image]` return dtype also confirmed fixed (not just `list[int]`/`list[struct]`)
- [x] Real user-facing path exercised end-to-end: `video_frames()` on an actual sample video with `pillow` mocked unavailable now raises the correct `ImportError`, while `video_metadata()` continues to work
- [x] Full existing test suites re-run and passing: `test_row_wise_udf.py` (39), `test_concat.py` + `test_audio.py` (162), Rust `from_lit` unit tests (34)

**Not yet done (intentionally):** nothing has been committed or pushed. The fix exists only in the local working tree on branch `fix/list-udf-error-propagation-7196`, per the plan to get maintainer sign-off before committing/opening a PR.

---

## Phase IV: Pull Request

- **PR Link:** *(pending — blocked on maintainer confirmation of fix approach; nothing committed or pushed yet)*
- **PR Summary:** *(pending — will summarize the `series_from_literals_iter` propagation-order fix once opened)*
- **Maintainer Feedback Log:**
  - `everettVT` opened the issue and added `bug`/`help wanted`/`p1`/`fix` labels.
  - I posted an initial comment asking whether the fix should live in error-propagation logic vs. `Series::concat` itself, before doing any implementation work.
  - **Drafted follow-up comment (not yet posted)** reporting the confirmed root cause (`series_from_literals_iter`, not the executor), the chosen fix rationale (propagation-order over changing `concat`'s contract), and full verification results — while explicitly asking for sign-off on the approach and desired test coverage bar before a PR is opened. Draft:

    > Following up on my earlier question — while waiting to hear back on the preferred approach, I went ahead and built Daft from source (macOS, arm64) and traced the two paths so I could compare them concretely. Here's what I found, and a fix I've got working locally.
    >
    > **Root cause**
    >
    > It's actually not in the UDF executor itself — the divergence is one layer down, in how per-row results are assembled into a `Series`. Both scalar and list UDFs funnel their per-row outputs through `series_from_literals_iter` in [`src/daft-core/src/series/from_lit.rs`](https://github.com/Eventual-Inc/Daft/blob/main/src/daft-core/src/series/from_lit.rs).
    >
    > That function already captures per-row errors into an `errs` map (via the `unwrap_inner!` macro) and surfaces them at the very end. The problem is ordering:
    > - **Scalar dtypes** (`Int64`, `Utf8`, …) build their array with a null in each errored slot, never fail, and reach the final `errs` check — so the real `ImportError` is surfaced. ✅
    > - **Container dtypes** (`List`, and structurally the same for `Map`/`Tensor`/`SparseTensor`/`Embedding`) build their output by concatenating the surviving per-row child series. When every row raised, there are zero surviving child series, so `ListArray::from_series` → `Series::concat(&[])` fails first with `Need at least 1 series to perform concat` and returns via `?` **before** the `errs` check ever runs — masking the true error. ❌
    >
    > So `list[int]`, `list[struct]`, `list[image]` all reproduce it (confirmed below), which matches what you noted in the issue.
    >
    > **On the two options I raised**
    >
    > I ended up going with the **error-propagation option** rather than changing `Series::concat`. Making `concat` tolerate an empty slice would weaken a general-purpose primitive (and there's an existing test asserting `Series.concat([])` *should* raise), and it wouldn't fix the underlying ordering issue — a real per-row error should take priority over a downstream failure it caused, regardless of dtype. So the fix keeps `concat`'s contract intact and instead makes `series_from_literals_iter` surface the accumulated per-row `errs` before returning any construction failure they induced. This fixes all container dtypes in one place, not just `list`.
    >
    > **Verification**
    >
    > Against a source build with the fix, all of these now surface the original `ImportError` (and no longer the concat error):
    > - the minimal `scalar` (control) and `list[int]` repro from the issue — identical error text now
    > - `list[struct]` and `list[image]` return dtypes
    > - the real user-facing path: `video_frames()` on an actual mp4 with `pillow` unavailable now raises `ImportError: The 'pillow' module is required for frame decoding. Install it with pip install daft[video].` — while `video_metadata()` continues to work — exactly the expected behavior from the issue.
    >
    > I also added a parametrized regression test (`list[int]`, `list[struct]`, scalar `int64`) to `tests/udf/test_row_wise_udf.py`, and the existing `daft-core` `from_lit` unit tests, the full `test_row_wise_udf.py` suite, and the `test_concat`/`test_audio` suites all still pass. `rustfmt` is clean and no new `clippy`/`ruff` findings.
    >
    > Happy to open a PR with this — but since this is my first contribution here, I wanted to check in first: does the propagation-order fix in `series_from_literals_iter` sound like the right layer to you, or would you prefer it live elsewhere? And is there a coverage bar (e.g. an explicit Rust-level test in `from_lit.rs` in addition to the Python one) you'd like before I put the PR up?
