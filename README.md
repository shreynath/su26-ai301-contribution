# AI301 Open Source Contribution Log
**Student ID:** 153102  
**Program:** CodePath AI301 — AI Open Source Capstone, Summer 2026  
**Section:** 1B | Wednesdays 4PM–6PM PT  

## Phase I: Issue Selection

### Issue

> 🔗 **https://docs.google.com/spreadsheets/d/1_MuOCiQmaMo6MmDOjawojZ8pS5DLzwjFTVYepca2deM/edit?gid=0#gid=0**  

**Repository:** pwndbg/pwndbg  
**Organization:** pwndbg  
**Languages:** Python, Shell, C  
**Labels:** `bug`, `help wanted`, `good first issue`

### Problem Summary

pwndbg is a GDB/LLDB plugin used widely in exploit development and binary reverse engineering. When a debugger stops at a `syscall` instruction on x86_64 Linux, pwndbg is supposed to display the syscall name and its argument registers in the context panel to help the analyst understand what system call is being made. However, certain x86_64 syscall invocations are not being annotated — pwndbg either silently skips the annotation or displays incomplete argument information — leaving analysts without the call context they need at a critical debugging step. A fix would ensure full, accurate syscall argument annotation is displayed consistently for x86_64 targets.

### Why I Chose This Issue

I chose this issue because it sits at the exact intersection of my background and what I want to build on. I've done research at ASU's SEFCOM Lab, which is one of the leading binary security research groups in the US, where I worked with NLP pipelines for policy analysis, and I regularly encountered tools like pwndbg in that environment. Knowing how pwndbg is used in practice makes this contribution feel meaningful rather than cosmetic.

The fix is Python-based, which is my strongest language, and it lives in the `arguments.py` / `arch.py` layer of pwndbg's codebase: the part responsible for interpreting ABI calling conventions and mapping register values to syscall parameters. This requires understanding x86_64 Linux syscall ABI (rax = syscall number, rdi/rsi/rdx/r10/r8/r9 = args), which is knowledge I can acquire quickly and that directly reinforces my Cybersecurity minor coursework.

"Done" for this issue means: when pwndbg stops at any x86_64 `syscall` instruction, the context panel correctly displays the syscall name and all relevant argument registers, consistently, without missing cases. There is no ambiguity about the acceptance criteria.

I also chose this because pwndbg is actively maintained, has a clear contributing guide, and has maintainers who leave substantive comments on issues. The community signal is strong, and I'd rather make a real contribution to a tool used by security professionals globally than tick a box on a low-stakes project.

## Phase II: Reproduce & Plan

### Environment Setup

**OS:** Ubuntu 24.04 (WSL2 / native Linux)  
**GDB version:** 14.2  
**Python:** 3.12  
**pwndbg commit:** `dev` branch (latest)

**Setup steps:**
```bash
# 1. Clone fork
git clone https://github.com/shreynath/pwndbg.git
cd pwndbg

# 2. Run the official setup script (installs into a virtualenv automatically)
./setup.sh

# 3. Verify pwndbg loads correctly
gdb -q -ex "python import pwndbg; print('loaded')" -ex quit
```

The setup script handled all dependencies automatically on Ubuntu 24.04 with
no errors. GDB 14.2 was already present; if it isn't on your system,
`sudo apt install gdb` resolves it. Total setup time: ~25 minutes including
the virtualenv build.

**Working branch:**  
`https://github.com/shreynath/pwndbg/tree/fix-color-param-validation`

---

### Steps to Reproduce

1. Start GDB with pwndbg loaded:
```bash
   gdb -q
```
2. In the GDB prompt, attempt to set a color parameter to an invalid value: pwndbg> set context-code-color notacolor
3. **Expected behavior:** pwndbg raises an error immediately, e.g.: Invalid color value 'notacolor'. Valid values are: none, bold, red, green,
blue, yellow, cyan, magenta, ...
4. **Actual behavior:** The command completes silently with no output. The
   invalid value is accepted and stored.
5. Confirm the bad value is stored: pwndbg> pwndbg context-code-color
6. The invalid value is shown as the active setting.Run a program to trigger the context display and observe either a corrupted render or a Python traceback originating in `pwndbg/color/__init__.py`.

*Reproduced consistently across two fresh sessions. The silent acceptance is
the core bug — downstream crashes are a secondary symptom.*

---

### Solution Approach

**Understand:**  
pwndbg's color configuration uses `ColorParamSpec` objects (defined in the
color submodule) backed by `Parameter` instances in `pwndbg/lib/config.py`.
When GDB's `set` command writes a new value to a color parameter, the setter
does not check whether the incoming string is a valid terminal color identifier
before storing it. Any string is accepted. The valid set of values is the
union of named terminal colors (`bold`, `red`, `green`, etc.) and the special
value `none`, as defined by pwndbg's own ANSI helpers in
`pwndbg/color/__init__.py`.

**Match:**  
Other pwndbg parameter types that have constrained value sets use a
`PARAM_ENUM` style pattern where validation is enforced at assignment time.
The `syntax-highlight-style` parameter in `pwndbg/color/syntax_highlight.py`
already performs a similar check — it calls `get_all_styles()` and validates
against that list on change. This is the direct pattern to replicate for color
params.

**Plan:**
1. Identify all color parameters registered via `ColorParamSpec` — they are
   spread across files like `pwndbg/color/context.py`, `pwndbg/color/disasm.py`,
   `pwndbg/color/memory.py`, etc.
2. Locate where `ColorParamSpec` resolves the user-supplied string to an ANSI
   code — this is in `pwndbg/color/__init__.py`, likely a function like
   `generateColorFunction` or the string-to-ANSI mapping.
3. Extract that valid-value set into a shared constant or helper function
   (e.g., `VALID_COLOR_NAMES`).
4. Add a validator in the `ColorParamSpec` setter (or its parent `Parameter`
   subclass) that checks the incoming value against `VALID_COLOR_NAMES` and
   raises a `ValueError` / prints a pwndbg-style warning with the valid options
   if it doesn't match.
5. Ensure the error message is user-friendly and lists valid values.
6. Write tests in `tests/test_commands.py` (or an appropriate test file) that:
   - Assert setting a valid color value succeeds silently.
   - Assert setting an invalid color value produces the expected error output.
7. Run the full test suite (`./tests.sh` or `make tests`) to confirm no
   regressions.

**Review:**  
Will self-review against `CONTRIBUTING.md` before opening a PR. Key
conventions to follow: all new code must pass `ruff` linting
(`ruff check .`), type hints are expected on new functions, and commit
messages should be in the imperative mood (e.g., `Add validation for color
config parameters`).

**Evaluate:**  
- Manual: `set context-code-color notacolor` in GDB → clear error message
  appears.  
- Manual: `set context-code-color bold` → continues to work, context displays
  correctly with bold formatting.  
- Automated: new test in the test suite asserts both the valid and invalid
  cases.  
- Regression: existing tests all pass (`./tests.sh`).

---

## Phase III: Implementation

- [ ] Tests written
- [ ] Fix implemented
- [ ] Tests passing locally

## Phase IV: Pull Request

- **PR Link:** *(pending)*
- **PR Summary:** *(pending)*
- **Maintainer Feedback Log:** *(pending)*
