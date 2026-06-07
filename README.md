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

- [ ] Local environment set up
- [ ] Issue reproduced locally
- [ ] Root cause identified
- [ ] Solution approach drafted

## Phase III: Implementation

- [ ] Tests written
- [ ] Fix implemented
- [ ] Tests passing locally

## Phase IV: Pull Request

- **PR Link:** *(pending)*
- **PR Summary:** *(pending)*
- **Maintainer Feedback Log:** *(pending)*
