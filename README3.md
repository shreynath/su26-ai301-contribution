# AI301 Open Source Contribution Log
**Student ID:** 153102
**Program:** CodePath AI301 — AI Open Source Capstone, Summer 2026
**Section:** 1B | Wednesdays 4PM–6PM PT

---

## Phase I: Issue Selection

### Issue

🔗 **https://github.com/Eventual-Inc/Daft/issues/7308**

**Repository:** Eventual-Inc/Daft
**Organization:** Eventual (Daft — high-performance data engine for AI/multimodal workloads)
**Languages:** Rust, Python
**Labels:** `enhancement`, `help wanted`

---

### Problem Summary

Daft supports a large number of IO backends (S3, Azure, GCS, HTTP, Unity, Gravitino, Hugging Face, TOS, COS, GooseFS, HDFS, and OpenDAL). Issue #7308, opened by maintainer `srilman`, reports that `df.explain()` and related plan displays dump the full default config for *every* backend into the plan output — even when the user never configured those backends and is reading a simple local parquet file.

That produces multi-dozen-line walls of noise in explain output, and the same dump is embedded into expression Display strings for paths like `url_download(...)`, which makes physical-plan snapshots and debugging much harder than they need to be.

The maintainer's requested fix has two parts:
1. Only print options that differ from their defaults.
2. When the read scheme is known (e.g. S3), only print that backend's config section.

---

### Why I Chose This Issue

This was opened on July 24, 2026 — five days before I started — by a Daft maintainer with `help wanted`, and it was fully unclaimed (no assignee, no comments, no linked PR). That combination is rare after how quickly my earlier Daft candidate (#7196) got scooped by another contributor's PR.

It is also a natural third contribution for me because I already have Daft built locally from the first contribution attempt (`~/Programming/Daft`, `make build` / maturin), so setup cost is near zero. The bug is pure display/plan-formatting logic in Rust — no CUDA, no cloud credentials required to reproduce — and "done" is well-defined: default explain output should stop dumping every backend's defaults, and a non-default field (e.g. `max_connections=32` on `url_download`) should still surface.

Scope-wise it matches how Daft already treats GooseFS display (sparse / omit when fully default), so the fix is an extension of an existing convention rather than a speculative redesign.

---

## Phase II: Reproduce & Plan

### Environment Setup

**OS:** macOS (Apple Silicon)
**Python:** 3.13 (venv via Daft Makefile)
**Rust:** nightly toolchain pinned by the repo
**Build:** `make build` (maturin)

**Actual local setup (already completed from prior Daft work):**
```bash
cd ~/Programming/Daft
git fetch origin main && git checkout main && git pull
git checkout -b fix/truncate-ioconfig-explain-7308
make build
```

**Working branch:** `fix/truncate-ioconfig-explain-7308` (pushed to `shreynath/Daft`)

---

### Steps to Reproduce

1. Build Daft from source and open a Python REPL with the local wheel.
2. Write a tiny local parquet and call explain:
   ```python
   import daft, tempfile, pyarrow as pa, pyarrow.parquet as pq
   path = tempfile.mktemp(suffix=".parquet")
   pq.write_table(pa.table({"x": [1, 2, 3]}), path)
   print(daft.read_parquet(path).explain())
   ```
3. **Before fix:** `IO config =` prints every backend's full default config (S3 Max connections / SSL / retries, Azure, GCS, HTTP user-agent, TOS, COS, …).
4. **After fix:** `IO config = <default>` (or only non-default fields when configured).

Also exercised the `url_download` Display path via `tests/dataframe/test_morsels.py::test_batch_size_from_udf_propagated_through_ops_to_scan`, which previously embedded the full multi-backend dump in the physical plan.

---

### Solution Approach

1. Extend the existing GooseFS sparse-display pattern to every backend's `multiline_display` (S3, Azure, GCS, HTTP, HuggingFace, TOS, COS): only emit fields that differ from `Default`.
2. Make `IOConfig::multiline_display` skip empty backend sections; only emit `Disable suffix range` when true.
3. Make `Display for IOConfig` reuse the sparse multiline view (`IOConfig {}` when fully default) so expression Display / `url_download` plans shrink too.
4. Add `IoBackendKind` + `multiline_display_for_backends` for scheme-aware filtering; GlobScanOperator explain uses URI schemes from glob paths so `s3://…` only prints S3 config.

---

## Phase III: Implementation

- [x] Confirmed issue still open / unclaimed; commented on #7308 that I was taking it
- [x] Branch `fix/truncate-ioconfig-explain-7308` created from latest `main`
- [x] Sparse `multiline_display` for S3 / Azure / GCS / HTTP / HuggingFace / TOS / COS
- [x] `IOConfig::multiline_display` skips empty backends; `Display` uses sparse view
- [x] Scheme-aware filtering via `IoBackendKind` + GlobScanOperator / StorageConfig hooks
- [x] Unit tests in `common-io-config` for default/sparse/scheme-aware behavior
- [x] Updated `test_morsels.py` expected explain snapshot for truncated IOConfig
- [x] `cargo test -p common-io-config --lib` (43 passed)
- [x] `cargo check -p daft-scan`; `cargo fmt -p common-io-config -p daft-scan`
- [x] Local reproduce: `read_parquet(...).explain()` shows `IO config = <default>`
- [x] `DAFT_RUNNER=native` morsels explain regression test passed

---

## Phase IV: Pull Request

- **PR Link:** https://github.com/Eventual-Inc/Daft/pull/7327
- **PR Summary:** Sparse/diff-from-default IOConfig display for explain() and expression Display, plus scheme-aware filtering for GlobScan plans. Closes #7308.
- **Maintainer Feedback Log:**
  - `srilman` opened #7308 with the two-part truncate request (diff-from-default + scheme-aware).
  - I commented on the issue confirming I was taking it and outlining the approach.
  - PR #7327 opened against `Eventual-Inc/Daft` `main` from `shreynath:fix/truncate-ioconfig-explain-7308`.
  - Awaiting maintainer review / CI.
