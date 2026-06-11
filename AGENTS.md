# AGENTS.md

## Repository Role

This repository is the user's fork of Alibaba CicadaPlayer. Treat the upstream
source as reference SDK code, and treat this fork primarily as a player and C++
study workspace.

The local checkout usually works on `release/0.4.4`. `origin` should point to
the user's GitHub fork. This fork is not expected to keep syncing from the
Alibaba upstream repository.

## Main Editing Surface

Prefer edits under `study/`, especially `study/cpp-learning/`, unless the user
explicitly asks to modify source code.

Keep study documents self-contained and learning-friendly. Explain the control
flow, ownership model, threading model, and source anchors in one readable
document before adding broad source indexes.

## Study Material Privacy Boundary

Public study notes in this repository should focus on technical understanding:
source reading, architecture, C++ ownership and lifetime, threading, design
patterns, and transferable engineering ideas.

Do not add new repository documents whose main purpose is interview scripting,
resume packaging, job-search strategy, company-experience mapping, or personal
narrative. If a note contains keywords such as `面试`, `interview`, `简历`,
`resume`, `求职`, `job search`, `公司经历`, `个人经历`, or `话术`, ask before
writing it into the repository and prefer storing it under:

```text
/home/sxf/docs/private/cicada-player-study/
```

Existing repository notes that already contain interview-oriented wording are
legacy material. Do not expand them with more private content. Future public
notes should use neutral technical names such as design transfer, learning
notes, or transfer candidates.

Unless the user explicitly asks for interview, resume, job-search, or personal
presentation material, generate development-oriented learning documents by
default. The primary focus is player principles and engineering design:
playback flow, control plane, data path, buffering, seek, state handling, QoE,
and module boundaries. C++ engineering practice is the secondary focus and
should be added when it clarifies ownership, lifetime, threading,
synchronization, interfaces, or error propagation. Do not force every topic
into a C++17 syntax recommendation.

## Source Code Boundary

Do not casually modify `mediaPlayer/`, `framework/`, `platform/`, `plugin/`,
`cmdline/`, `external/`, or build files. Those directories are upstream player
SDK code and should be treated as reference material unless the user asks for a
specific source change.

If source edits are requested, inspect the nearby README, CMake files, build
scripts, and surrounding implementation style first. Keep changes surgical and
avoid broad modernization passes.

## C++ Guidance

CicadaPlayer source is mainly C++11-era mixed C/C++/FFmpeg-style code. Preserve
the existing language baseline and local style when changing source.

For study docs and personal design examples, prefer "boring C++17":

- clear ownership and lifetime boundaries
- RAII for resource management
- value members when the object is always present
- `std::unique_ptr` for exclusive dynamic ownership
- explicit error/result handling instead of hidden control flow
- simple standard-library types before custom abstractions

Do not copy weak upstream habits into new examples: header-level
`using namespace`, raw owning pointers, broad `catch (...)`, heavy macros,
unclear `void *` ownership, or magic integer error flows.

## Media Guidance

When analyzing playback code, pay attention to packet and frame ownership,
FFmpeg refcount/release rules, data path versus control path, seek/flush/stale
data handling, clocks, buffering, A/V sync, and thread shutdown behavior.

## Git Guidance

Keep doc-only commits separate from source changes. Push normal work to
`origin`.

GitHub is the main remote. Gitee is intended to be a mirror maintained by
`.github/workflows/mirror-to-gitee.yml`, not by local dual-push configuration.

### Change Review Process

Repository writes require a reviewed plan and explicit user approval in the
current conversation before the write is performed.

The plan should state:

- the change goal
- the files expected to change
- the intended edits
- the expected impact on source code, documentation, build configuration, CI,
  and the GitHub-to-Gitee mirror workflow
- the narrowest useful verification plan
- the proposed commit message

Read-only inspection, source browsing, code search, and analysis do not require
a separate approval step.

After an approved write is completed, report the resulting commit SHA, the
actual files changed, and the verification status.

Before finalizing work, run:

```bash
git status --short --untracked-files=all
```

For documentation-only changes, no build is required. For source changes, run
the narrowest useful build or test command discovered from local project files,
and state clearly if verification cannot be run.
