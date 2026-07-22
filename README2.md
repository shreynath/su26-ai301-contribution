# AI301 Open Source Contribution Log
**Student ID:** 153102
**Program:** CodePath AI301 — AI Open Source Capstone, Summer 2026
**Section:** 1B | Wednesdays 4PM–6PM PT

---

## Phase I: Issue Selection

### Issue

🔗 **https://github.com/rizinorg/rizin/issues/6425**

**Repository:** rizinorg/rizin
**Organization:** rizinorg (Rizin — UNIX-like reverse engineering framework and command-line toolset, a fork/successor of radare2)
**Languages:** C, C++, Meson
**Labels:** `good first issue`, `refactor`, `RzUtil`, `performance`

---

### Problem Summary

Rizin is a mature, widely-used reverse-engineering framework (3.7k+ stars) with a large internal utility library, `RzUtil`, that includes a generic `RzIterator` type used throughout the codebase for iterating over collections. Issue #6425, opened by core maintainer `Rot127`, points out that every function currently returning an `RzIterator` allocates it on the heap — even though the struct is small and its lifetime is almost always trivial (allocated, returned, immediately consumed by the caller, then freed). Heap allocation for something this short-lived is unnecessary overhead: it costs a `malloc`/`free` round-trip for a value that could just as easily be returned by value on the stack.

The maintainer's ask is a refactor: change the `RzIterator`-returning functions across the codebase so the iterator is passed/returned over the stack instead of the heap, eliminating the redundant allocation. This is a performance and code-quality cleanup rather than a bug fix — the current heap-based version is functionally correct, just wasteful.

---

### Why I Chose This Issue

This is one of the two most recently opened "good first issue"-labeled tickets in the entire `rizin` issue tracker (opened May 29, 2026, by a core maintainer), and it's genuinely unclaimed — no assignee, and the issue page's "Development" section shows no linked branches or pull requests. I specifically checked the other issue opened the same day by the same maintainer, `#6426` ("Implement a hash table iterator over key/value pairs"), because it looked like an equally good candidate — but it already has an open pull request (`#6483`) referencing it, so someone is actively working it. I ruled that one out rather than risk duplicating effort, which left `#6425` as the more recent, truly available option.

The fact that `Rot127` personally opened both issues (and appears to be the author of nearly every recently-opened issue in the tracker) is a strong signal this is an actively, deliberately curated backlog rather than stale auto-labeled cruft — similar to how the Daft issue I picked last time came directly from a maintainer rather than a bot.

It's also a clean fit for local development on my Mac: `rizin` is pure C/C++ built with Meson, with no CUDA, no GPU dependency, and no platform-gated code paths — the same category of "buildable anywhere" issue I prioritized last time, which ruled out a lot of otherwise-interesting CUDA-heavy candidates elsewhere in the spreadsheet (e.g. `cudf`/rapidsai). Rizin also isn't a new build environment for me conceptually — it's the same `meson`/`ninja` toolchain I already have working locally from unrelated build experience, so setup risk should be low.

Scope-wise, this is well-bounded for a first C-touching contribution to a large systems codebase: it's a single, precisely-stated API convention change (heap-returned `RzIterator` → stack-returned `RzIterator`), not a new feature or an open-ended design question, which makes "done" reasonably well-defined — get the call sites migrated, make sure nothing double-frees or dangles a stack reference, and confirm the existing test suite still passes.

Finally, this one lines up well with my own background — `rizin` is a binary analysis / reverse-engineering framework, which is squarely in the same space as the binary security work I did at SEFCOM, so I have a head start on the domain vocabulary (disassembly, analysis passes, `RzUtil` conventions) even before touching the specific data-structure code.

Before writing any code, my plan is to comment on the issue thread the same way I did with Daft: confirm scope with `Rot127` — specifically, whether the expected fix is a full sweep of every `RzIterator`-returning function across the codebase, or whether they want it scoped to a particular subsystem first as a proof of concept before a larger sweep — and ask whether there's a preferred pattern already in use elsewhere in `RzUtil` for stack-returned structs that I should mirror for consistency.

---

## Phase II: Reproduce & Plan

### Confirming the scope of the problem

Before touching any code, I used Claude to do a full sweep of every `RzIterator`-returning function in the codebase to understand the real blast radius of the issue as literally stated. This confirmed my instinct from Phase I: a "convert every `RzIterator`-returning function" sweep touches dozens of call sites across `agraph.c`, `canalysis.c`, `graph_impl.c`, `ht_inc.c`, and multiple test files. That's too large and too risky to land as a first PR without maintainer buy-in on the approach, which is exactly why my Phase I plan was to propose scoping this to a subsystem first.

**Decision:** rather than attempt the full sweep, I scoped the proof-of-concept to the core `RzGraph` node/edge iterator API in `librz/util/graph_impl.c`. This is a self-contained subsystem with a clear public API surface (`rz_graph_get_nodes`, `rz_graph_out_edges`, `rz_graph_in_edges`, etc.), which makes it a good candidate to demonstrate the pattern before asking `Rot127` to sign off on rolling it out further.

### Design snag found during planning

While planning the conversion, I found a real wrinkle worth flagging to the maintainer rather than papering over: some iterators are *composed* — for example, the neighbour iterator wraps an edge iterator, and the inner iterator's state needs to outlive the function that created it. A naive conversion (just changing the inner iterator's return type to stack-returned) would leave a dangling reference once the creating function's stack frame is gone.

**Planned fix:** embed the inner `RzIterator` *by value* inside the outer iterator's heap-allocated context struct, instead of storing a pointer to a separately heap-allocated inner iterator. Net effect: the composed iterator's state still requires one heap allocation (for the outer context), but it's one allocation fewer than the original design (no separate wrapper allocation for the inner iterator). This preserves correctness while still delivering on the spirit of the issue — removing gratuitous heap churn.

### Migration plan (in order)

1. Add the new stack-friendly primitives to `rz_iterator.h` / `iterator.c`: `rz_iterator_init()`, `rz_iterator_fini()`, `rz_iterator_empty()` (needed because a by-value return can't use `NULL` to represent "nothing to iterate").
2. Convert the core graph API function signatures (`rz_graph_get_nodes`, `rz_graph_out_edges`, `rz_graph_in_edges`, `rz_graph_out_edges_by_id`, `rz_graph_in_edges_by_id`) and the `impl_ops` vtable (`get_out_edges`/`get_in_edges`) from `RzIterator *` to `RzIterator`.
3. Convert both backend implementations (list-impl and matrix-impl) to match.
4. Convert the neighbour-iterator composition using the embed-by-value fix above.
5. Work outward from `graph_impl.c` to its direct consumers (`graph_drawable.c`), then to the wider core call sites (`agraph.c`, `cagraph.c`, `cmd_debug.c`, `canalysis.c`), converting each `RzIterator *it = fn(...)` / `rz_iterator_free(it)` pair to `RzIterator it = fn(...)` / `rz_iterator_fini(&it)`.
6. Rebuild after each file/module to catch mismatches early rather than letting errors compound.
7. Once clean, run the existing test suite to confirm no regressions, then open the PR referencing #6425 and describe the composed-iterator design decision explicitly so `Rot127` can weigh in before I propose extending the pattern further.

---

## Phase III: Implementation

**Status: in progress — core subsystem converted and building clean; several downstream call sites still outstanding.**

### Completed and verified (compiled against a real meson/ninja build of `librz/util`)

- Added `rz_iterator_init()`, `rz_iterator_fini()`, and `rz_iterator_empty()` to `rz_iterator.h` / `iterator.c`.
- Converted the core graph API to return `RzIterator` by value instead of `RzIterator *`:
  - `rz_graph_get_nodes`, `rz_graph_out_edges`, `rz_graph_in_edges`, `rz_graph_out_edges_by_id`, `rz_graph_in_edges_by_id`
  - the `impl_ops` vtable (`get_out_edges` / `get_in_edges`)
- Converted both graph backends (list-impl and matrix-impl) to the new signatures.
- Converted the neighbour-iterator composition using the embed-by-value approach described in Phase II — one fewer heap allocation than the original, with no dangling-reference risk.
- Fixed remaining in-file consumers of the old API within `graph_impl.c`: `rz_graph_out_degree`, `rz_graph_in_degree`, `rz_graph_as_dot_str`, and `rz_graph_find_sccs`.
- **Result:** `graph_impl.c` compiles clean on its own, and a full rebuild of `librz_util` completes with no errors.
- Converted `graph_drawable.c`'s consumers of `rz_graph_get_nodes` (`to_json`, `to_cmd`, `to_gml`), including removing a null-check code path that was only needed for the old heap-pointer API and is redundant now that "empty" is represented by `rz_iterator_empty()` instead of `NULL`.
- Converted `agraph.c`:
  - `agraph_get_nodes` — since it has roughly 45 existing callers expecting `RzIterator *`, rather than changing its signature (which would ripple far beyond this PR's scope) I boxed the new by-value result: allocate one `RzIterator` on the heap and copy the by-value result into it. This keeps `agraph_get_nodes`'s public contract stable for this PR while still using the new stack-based API internally.
  - `agraph_collect_edges` — converted directly to the by-value API, since it consumes the iterator immediately into a vector and doesn't need to hand it further down the call stack.
  - Direct call sites in `rz_core_create_agraph_from_graph_at` (~line 5900–5970) — converted to stack-allocated iterators, replacing `NULL`-based failure checks with `rz_iterator_empty()` checks and removing the now-unnecessary heap cleanup from the `goto fail` paths.

### Not yet complete

- `cmd_debug.c`: turned out not to be its own compilation unit — it appears to be amalgamated into a larger `cmd.c` build target, which made it harder to isolate for a quick incremental build. Still need to track down and convert any call sites here.
- `cagraph.c`: not yet confirmed clean; needs the same call-site sweep applied to `agraph.c`.
- `canalysis.c`: compiled without errors during the object-file-level build, which suggests it may not actually touch the converted API — needs a final check to confirm rather than assume.
- Full `librz` (core included) has not yet been rebuilt end-to-end; only targeted object files have been compiled so far to keep iteration fast and avoid pulling in heavy unrelated dependencies (e.g., capstone) on every check.
- Existing test suite has not yet been run against the changes.
- Have not yet posted the scope-confirmation comment on the issue thread that I outlined in Phase I — I got pulled into implementation first since the proof-of-concept subsystem was well-defined enough to start on, but I still owe `Rot127` that comment, ideally posted alongside (or just before) the PR so the composed-iterator design decision gets reviewed before I propose extending the pattern to the rest of the codebase.

### Next steps

1. Finish `cmd_debug.c` / `cagraph.c` call sites.
2. Full clean build of `librz` core.
3. Run the test suite.
4. Post the scope-confirmation comment to `Rot127` on #6425.
5. Open the PR (Phase IV).

---

## Phase IV: Pull Request

**Status: pending — will open once Phase III's outstanding build/test items above are resolved.** Opening a PR before the build is fully clean and tests pass isn't something I want in my log as "done" when it isn't yet — so this section will be filled in with the actual PR link, description, and any maintainer discussion once that's true.

### Planned PR description (draft, to be finalized on submission)

**Title:** `util/graph: return RzIterator by value instead of heap-allocating (proof of concept for #6425)`

**Body outline:**
- References #6425.
- States explicitly that this is scoped to the core `RzGraph` node/edge iterator API as a proof of concept, not the full codebase sweep, and asks `Rot127` to confirm this is the right first step before a broader rollout.
- Explains the composed-iterator design decision (neighbour iterator embedding the edge iterator by value inside its heap-allocated context) and why the naive stack-return approach would have introduced a dangling reference.
- Lists the converted functions and files.
- Notes any build/test verification performed (to be filled in with final results).
- Explicitly invites feedback on the pattern before I take on further subsystems, per the Phase I plan to confirm preferred conventions with the maintainer.
