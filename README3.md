# AI301 Open Source Contribution Log
**Student ID:** 153102
**Program:** CodePath AI301 — AI Open Source Capstone, Summer 2026
**Section:** 1B | Wednesdays 4PM–6PM PT

**Contribution:** Daft #7308 — Truncate IOConfig prints in `df.explain()`
**Upstream:** https://github.com/Eventual-Inc/Daft
**Fork:** https://github.com/shreynath/Daft
**Working branch:** [`fix/truncate-ioconfig-explain-7308`](https://github.com/shreynath/Daft/tree/fix/truncate-ioconfig-explain-7308)
**PR (merged):** https://github.com/Eventual-Inc/Daft/pull/7327
**Status:** ✅ Merged into `main` on 2026-07-30 by maintainer `@srilman` (merge commit `318e788`)

---

## Phase I: Issue Selection

### Issue

🔗 **https://github.com/Eventual-Inc/Daft/issues/7308**

**Repository:** Eventual-Inc/Daft
**Organization:** Eventual (Daft — high-performance data engine for AI/multimodal workloads)
**Languages:** Rust, Python
**Labels:** `enhancement`, `help wanted`
**Opened:** 2026-07-24 by maintainer `@srilman`
**At selection time:** open, unassigned, 0 comments, 0 linked PRs (fully claimable)

**Fork under my account:** https://github.com/shreynath/Daft (fork of Eventual-Inc/Daft)

---

### Problem Summary

Daft supports many IO backends (S3, Azure, GCS, HTTP, Unity, Gravitino, Hugging Face, TOS, COS, GooseFS, HDFS, and OpenDAL). Issue #7308 reports that `df.explain()` and related plan displays dump the full default config for *every* backend — even when reading a simple local parquet file with no custom IO settings.

That produces multi-dozen-line walls of noise in explain output, and the same dump is embedded into expression `Display` strings for paths like `url_download(...)`, which makes physical-plan snapshots and debugging much harder than they need to be.

I chose it because it was freshly opened by a maintainer with `help wanted`, fully unclaimed after my earlier Daft candidate (#7196) got scooped, and “done” was well-defined: stop dumping defaults; keep non-default fields visible.

---

### Why I Chose This Issue

**Skill match.** I already had Daft built locally from my first contribution attempt (`~/Programming/Daft`, `make build` / maturin, Rust nightly + Python 3.13). The fix lives in Rust display/plan-formatting code (`common-io-config`, `daft-scan`) — the same stack I’d already compiled — with no CUDA or cloud credentials required to reproduce.

**Learning goal.** I wanted a contribution that was *display/UX of plans*, not another executor bug: specifically, how a production data engine decides what to show in `explain()`, how `Display` for config types feeds into plan snapshots, and how to extend an existing sparse-display convention (GooseFS) across many backends without breaking snapshot tests.

**Understanding.** The maintainer’s ask was explicit and two-part:
1. Only print options that differ from their defaults.
2. When the read scheme is known (e.g. S3), only print that backend’s config section.

That made acceptance criteria concrete before I wrote code (see below).

---

### Understanding the Issue

**What’s broken:** `IOConfig` / per-backend `multiline_display` and `Display` emit every field (and every backend section), including defaults. GlobScan explain and `url_download` expression Display both surface that wall of text.

**Why it matters:** Explain output is the primary debugging surface for physical plans. Noise from unused backends (Azure/GCS/TOS/… when reading local parquet) makes plans hard to read and inflates snapshot tests (see `tests/dataframe/test_morsels.py`).

**Files/modules likely involved (identified before coding):**
- `src/common/io-config/src/config.rs` — `IOConfig::multiline_display`, `Display for IOConfig`
- Per-backend configs: `s3.rs`, `azure.rs`, `gcs.rs`, `http.rs`, `huggingface.rs`, `tos.rs`, `cos.rs`, `goosefs.rs` (already sparse)
- `src/daft-scan/src/storage_config.rs` — StorageConfig explain lines
- `src/daft-scan/src/glob.rs` — GlobScanOperator explain
- Snapshot: `tests/dataframe/test_morsels.py::test_batch_size_from_udf_propagated_through_ops_to_scan`

**Maintainer framing:** [@srilman in #7308](https://github.com/Eventual-Inc/Daft/issues/7308) asked for diff-from-default + scheme-aware printing; alternative considered was removing config prints entirely (worse for plans).

**Concrete acceptance criteria (“fixed” looks like):**
1. Default local `read_parquet(...).explain()` shows `IO config = <default>` (or equivalent compact form), not a multi-backend default dump.
2. Non-default fields still appear (e.g. `url_download` with `max_connections=32` → `S3 config = { Max connections = 32 }`).
3. When URIs are scheme-known (`s3://…`), explain prefers that backend’s section only.
4. Existing explain/snapshot tests updated and passing; new unit tests cover sparse/default/scheme-aware display.

---

## Phase II: Reproduce & Plan

### Environment Setup

**OS:** macOS (Apple Silicon)
**Python:** 3.13 (venv via Daft Makefile)
**Rust:** nightly toolchain pinned by the repo (`rust-toolchain.toml`)
**Build:** `make build` (maturin) — **setup path used:** project README / Makefile instructions (not a dev container; confirmed build flags against CI-style `cargo test` / `cargo fmt` targets)

**Working branch in my fork (required link):**
https://github.com/shreynath/Daft/tree/fix/truncate-ioconfig-explain-7308

Branch name mirrors the issue: `fix/truncate-ioconfig-explain-7308`.

**Actual local setup:**
```bash
cd ~/Programming/Daft
git remote add fork git@github.com:shreynath/Daft.git   # if missing
git fetch origin main && git checkout main && git pull
# Stashed prior #7196 WIP as wip-7196-before-7308 so this branch stays scoped
git checkout -b fix/truncate-ioconfig-explain-7308
make build
```

**Challenges encountered + how resolved:**
1. **Prior WIP on the same clone** — uncommitted #7196 work would have polluted this branch. Resolved by stashing as `wip-7196-before-7308` before branching from updated `main`.
2. **Python version mismatch** — Makefile defaults assume 3.11; only 3.13/3.14 available locally. Resolved with `make .venv PYTHON_VERSION=python3.13` (same fix from the earlier Daft setup).
3. **`gh` CLI missing / auth** — needed for fork + PR. Resolved by installing `gh` via Homebrew and authenticating with the existing osxkeychain GitHub token + SSH remote `fork` → `git@github.com:shreynath/Daft.git`.
4. **Fork not yet under my account for this cycle** — created `shreynath/Daft` from upstream, pushed the feature branch with `-u fork HEAD`, then opened the PR against `Eventual-Inc/Daft` `main` (not against the fork alone).

---

### Reproduction Process

Numbered steps another person can follow:

1. Clone/fork Daft, check out a commit *before* the fix (or temporarily revert the IOConfig display changes), and build with `make build`.
2. Open a Python REPL with the local wheel and write a tiny local parquet:
   ```python
   import daft, tempfile, pyarrow as pa, pyarrow.parquet as pq
   path = tempfile.mktemp(suffix=".parquet")
   pq.write_table(pa.table({"x": [1, 2, 3]}), path)
   print(daft.read_parquet(path).explain())
   ```
3. Observe the GlobScanOperator `IO config =` line.
4. Optionally run `tests/dataframe/test_morsels.py::test_batch_size_from_udf_propagated_through_ops_to_scan` under `DAFT_RUNNER=native` to see the `url_download` plan embed the full multi-backend `IOConfig` dump.

**Expected:**
- Local parquet explain shows a compact IO config (defaults omitted / `<default>`).
- Non-default fields (e.g. `max_connections=32`) still appear.
- Scheme-known remote reads only show the relevant backend section.

**Actual (before fix):**
- Local parquet explain printed every backend’s full default config (S3 Max connections / SSL / retries, Azure, GCS, HTTP user-agent, TOS, COS, …) — matching the wall of text in [#7308](https://github.com/Eventual-Inc/Daft/issues/7308).
- `url_download` Display embedded the entire multi-backend dump in the physical plan snapshot.

**After fix (verified locally):**
```
* GlobScanOperator
|   Glob paths = [...]
|   IO config = <default>
|   Use multithreading = true
```
and for non-default S3 max connections:
```
IOConfig:
S3 config = { Max connections = 32 }
```

---

### Solution Approach (UMPIRE)

**Understand — root cause (not just symptom):**
The symptom is noisy explain output. The root cause is that each backend’s `multiline_display` prints *all* configured fields (including values equal to `Default`), and `IOConfig::multiline_display` concatenates *every* backend section. Expression `Display for IOConfig` uses the same verbose path, so `url_download(...)` plans inherit the wall of text. Scheme-aware filtering did not exist for GlobScan explain.

**Match — analogous pattern already in-tree:**
`GooseFSConfig::multiline_display` already implements sparse / omit-when-default behavior. The fix is to extend that convention to S3/Azure/GCS/HTTP/HF/TOS/COS and teach `IOConfig` aggregation to skip empty sections — not invent a new display system.

**Plan — specific files to modify:**
1. Per-backend sparse `multiline_display`: `s3.rs`, `azure.rs`, `gcs.rs`, `http.rs`, `huggingface.rs`, `tos.rs`, `cos.rs`.
2. `config.rs`: sparse aggregation; add `IoBackendKind` + `from_uris` + `multiline_display_for_backends`.
3. `storage_config.rs`: explain hook that can filter backends; empty → `IO config = <default>`.
4. `glob.rs`: pass URI-inferred backends into StorageConfig display.
5. Tests: unit tests in `common-io-config`; update `test_morsels.py` snapshot.

**Implement:** see Phase III (commits `1c984e4`, `91cb8b5`).

**Review:**
- Edge case (proactive): unrecognized / protocol-alias schemes in a mixed glob must not filter to an incomplete backend set (would hide active non-default settings). Greptile later flagged this (P2); fixed by returning `None` from `from_uris` when any scheme is unrecognized (`91cb8b5`).
- Style: clippy `collapsible_if` failed required CI style check; collapsed nested `if`s in `config.rs` / `s3.rs` in the same follow-up.

**Evaluate:**
- Local reproduce + unit tests (43 passed in `common-io-config`) + morsels snapshot under native runner.
- Upstream CI went green after the style fix; maintainer `@srilman` reviewed positively and merged.

---

## Phase III: Implementation

### Implementation Progress

**Branch:** https://github.com/shreynath/Daft/tree/fix/truncate-ioconfig-explain-7308

**Key commits (on the PR branch):**
| Date | Hash | Message |
|------|------|---------|
| 2026-07-29 | [`1c984e47e`](https://github.com/Eventual-Inc/Daft/commit/1c984e47e3c23973639a772e990b71c954af6042) | `fix: truncate default IOConfig fields in explain output` |
| 2026-07-30 | [`bed1739f3`](https://github.com/Eventual-Inc/Daft/commit/bed1739f3) | `Merge branch 'main' into fix/truncate-ioconfig-explain-7308` |
| 2026-07-30 | [`91cb8b58f`](https://github.com/Eventual-Inc/Daft/commit/91cb8b58f1384591f5513d529cbb4abb5c4e2a26) | `fix: address clippy collapsible_if and unresolved URI filtering` |

Commit cadence: feature commit on 2026-07-29, merge-from-main + review/CI follow-up on 2026-07-30 (no multi-day gaps during implementation). Messages describe *why* (sparse display / clippy + unrecognized URI filtering), not `wip`/`fix`/`asdf`.

**Files modified (scoped to the issue):**
- `src/common/io-config/src/config.rs` — sparse `IOConfig` display, `IoBackendKind`, `from_uris`, scheme-aware aggregation
- `src/common/io-config/src/{s3,azure,gcs,http,huggingface,tos,cos}.rs` — diff-from-default `multiline_display`
- `src/common/io-config/src/lib.rs` — export `IoBackendKind`
- `src/daft-scan/src/storage_config.rs` — `multiline_display_with_backends`; `<default>` when empty
- `src/daft-scan/src/glob.rs` — GlobScan explain filters by URI schemes
- `tests/dataframe/test_morsels.py` — expected plan snapshot for compact IOConfig

No unrelated formatting sweeps or commented-out code.

**Checklist:**
- [x] Confirmed issue still open / unclaimed; commented on #7308 (@mentioned `@srilman`)
- [x] Branch created from latest `main` and pushed to fork
- [x] Sparse `multiline_display` for S3 / Azure / GCS / HTTP / HuggingFace / TOS / COS
- [x] `IOConfig::multiline_display` skips empty backends; `Display` uses sparse view
- [x] Scheme-aware filtering via `IoBackendKind` + GlobScanOperator / StorageConfig hooks
- [x] Unit tests in `common-io-config` for default/sparse/scheme-aware (+ alias/`oss://` fallback)
- [x] Updated `test_morsels.py` expected explain snapshot
- [x] Follow-up for Greptile P2 + clippy style CI

### Challenges Faced

1. **Many backends, one convention** — Had to mirror GooseFS’s sparse pattern field-by-field across S3/Azure/GCS/HTTP/HF/TOS/COS without accidentally dropping credential redaction (`***`) or optional fields. Resolved by comparing each field to `Self::default()` and only pushing diffs.
2. **Display vs explain two surfaces** — Fixing GlobScan alone would leave `url_download` noisy. Resolved by making `Display for IOConfig` reuse the sparse multiline view so both surfaces stay consistent.
3. **Unrecognized URI schemes (edge case)** — Filtering only recognized schemes would hide S3 settings used via aliases in mixed globs. Greptile P2 called this out; fixed in `91cb8b58f` by returning `None` (show all non-defaults) when any scheme is unrecognized.
4. **Required style CI** — Nested `if` triggered `clippy::collapsible_if` and failed `PR test suite / style`. Fixed in the same follow-up commit by collapsing the nested conditions.

### Testing Strategy / Testing Notes

**New automated tests (exercise the fix):**
- Unit tests in `src/common/io-config/src/config.rs` for:
  - fully default `IOConfig` → empty / `IOConfig {}`
  - sparse non-default field emission
  - `IoBackendKind::from_uris` for known schemes, local/`file://`, and unrecognized aliases (`my-s3://…`, `oss://…` → `None`)
- Pattern: same `#[cfg(test)]` / `#[test]` style as existing crate tests; assert on `multiline_display` / `from_uris` return values (no new test framework).

**Existing suite:**
- `cargo test -p common-io-config --lib` — **43 passed**
- `cargo check -p daft-scan`
- `cargo fmt -p common-io-config -p daft-scan`
- `DAFT_RUNNER=native pytest tests/dataframe/test_morsels.py::test_batch_size_from_udf_propagated_through_ops_to_scan` — passed after snapshot update
- Upstream PR CI: rust-tests, unit-tests, integration-io, etc. succeeded; style succeeded after `91cb8b58f`

**Manual:**
- Reproduced local `read_parquet(...).explain()` against a `make build` wheel — before: multi-backend dump; after: `IO config = <default>`.

---

## Phase IV: Pull Request

### Pull Request

- **PR Link:** https://github.com/Eventual-Inc/Daft/pull/7327
- **Against:** `Eventual-Inc/Daft` `main` ← `shreynath:fix/truncate-ioconfig-explain-7308` (upstream, not draft-only on fork)
- **PR Summary:** Sparse/diff-from-default IOConfig display for `explain()` and expression Display, plus scheme-aware filtering for GlobScan plans. Uses Daft’s PR template sections (`## Changes Made`, `## Related Issues`) with **why** (noisy plans / unusable snapshots) before **what** (sparse backends + scheme filter). Includes before/after explain snippets and test commands. Closes #7308.
- **Current status:** ✅ **Merged** 2026-07-30 by `@srilman` — “Love to see this @shreynath, thanks!” Merge commit: `318e788`. Issue #7308 closed as completed.

### Maintainer Feedback Log

| Date | Source | Feedback | My response | Commit / ref |
|------|--------|----------|-------------|--------------|
| 2026-07-24 | `@srilman` on #7308 | Requested (1) diff-from-default printing (2) scheme-aware backend sections | Used as acceptance criteria for the PR | Issue #7308 |
| 2026-07-29 | Me on #7308 | Claimed the issue and outlined sparse + scheme-aware plan; `@srilman` mentioned | — | Issue comment |
| 2026-07-29 | Me | Opened PR #7327 with `Closes #7308`, before/after, testing notes | — | `1c984e47e` |
| 2026-07-29 | `@greptile-apps` (P2) | `from_uris` can drop unresolved/alias schemes and hide active non-default config on mixed globs | Agreed; changed `from_uris` to return `None` (no filter) when any scheme is unrecognized; added unit tests for aliases/`oss://` | `91cb8b58f` |
| 2026-07-30 | CI `style` check | Failed on `clippy::collapsible_if` in `config.rs` / `s3.rs` | Collapsed nested `if`s; re-pushed | `91cb8b58f` |
| 2026-07-30 | `@srilman` on PR | “Love to see this @shreynath, thanks!” and merged | No further code changes needed | Merge `318e788` |

### Learnings & Reflections

**Technical gains.**
- Learned how Daft wires IO config into scan explain (`StorageConfig` → GlobScanOperator) and into expression Display (`url_download` embeds `IOConfig`’s `Display`).
- Practiced extending an existing in-repo convention (GooseFS sparse display) rather than inventing a parallel API — the “Match” step of UMPIRE paid off in reviewability.
- Got concrete experience with required Rust style CI (`clippy::collapsible_if`) and bot review (Greptile) as first-class feedback, not afterthoughts.

**Process gains.**
- Claiming + linking the PR on the issue the same day, `@mentioning` the maintainer, and keeping the branch scoped (stash unrelated WIP) made the review path short: opened 2026-07-29, merged 2026-07-30.
- Full open-source loop completed: claim → implement → PR → address bot/CI feedback → maintainer approval → merge → issue closed.

**What I’d do differently.**
- I would have written the unrecognized-scheme fallback *before* opening the PR (I thought about aliases while planning but only encoded the happy-path filter first). Greptile’s P2 was fair; baking that edge case into the first commit would have avoided a follow-up.
- For course logging, I would have filled Phase III “files + commit hashes” and the feedback table *as I pushed*, instead of backfilling after merge — easier to keep dates/refs accurate.

**Teachable insight for future cohorts.**
When you add “smart filtering” to diagnostic output (here: filter IO backends by URI scheme), always define the **failure mode of inference**. Incomplete inference that *hides* real config is worse than showing slightly more noise. Prefer “if unsure, show all non-defaults” over “if partially sure, show only what we recognized.” That single policy (`from_uris` → `None` on any unrecognized scheme) is what made the Greptile finding a one-commit fix instead of a design debate.

---

## Consistency check (earlier phases ↔ final PR)

| Phase I/II claim | Final PR outcome |
|------------------|------------------|
| Diff-from-default display | Shipped in `1c984e47e` across backends |
| Scheme-aware when URI known | GlobScan + `IoBackendKind::from_uris` |
| GooseFS as the Match pattern | Explicitly extended that convention |
| Acceptance: local explain compact; non-defaults visible | Verified manually + `test_morsels.py` + unit tests |
| Edge: aliases / unknown schemes | Addressed in `91cb8b58f` after Greptile P2 |
| PR closes #7308 against upstream main | Merged; issue closed |
