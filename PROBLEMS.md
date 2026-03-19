# Why We Are Removing VisiblePosition from Editing Commands

This document explains all the concrete problems caused by pervasive `VisiblePosition`
usage in editing commands — what breaks, who feels it, and why the abstraction is
misused in this context. It is not just about layout performance.

---

## Problem 1: Forced Layout Mid-Command (Performance)

Every `CreateVisiblePosition()` call triggers `Document::UpdateStyleAndLayout()`.
A composite editing command — paste, indent, list insertion — makes dozens of DOM
mutations and dozens of VP constructions interleaved. Each VP construction forces a
full style+layout recompute against a partially-mutated tree.

**What the developer sees:**
- `apply_block_element_command.cc` calls `GetDocument().UpdateStyleAndLayout()` at
  lines 127, 248, 316, 353, 410 — five times inside a single command execution.
- `composite_edit_command.cc` has `DCHECK(!GetDocument().NeedsLayoutTreeUpdate())`
  gates at lines 972, 1079, 1452, 1470, 1604, 1753 — every one of these is a point
  where the code *requires* layout to be clean before it can proceed, meaning
  something before it forced a layout and now the code must assert it stayed clean.

**What the user sees:**
- Typing lag in large or complex documents (contenteditable with tables, large lists,
  heavy CSS).
- Jank during paste of rich content — the more complex the pasted content, the more
  VP constructions fire, the more layout passes run mid-paste.
- Slow indent/outdent on multi-paragraph selections because `IndentOutdentCommand`
  loops over paragraphs and calls `MoveParagraph` per iteration, each of which
  constructs multiple VPs.

---

## Problem 2: Stale VP After DOM Mutation (Correctness / Crashes)

`VisiblePosition` records `dom_tree_version_` and `style_version_` at construction
(`visible_position.cc:64-65`). `IsValid()` checks these at runtime
(`visible_position.cc:244-246`). A VP captured before a DOM mutation is **technically
invalid** after it — the document version changed, so the snapped canonical position
may no longer refer to the same location in the tree.

The codebase acknowledges this explicitly with three separate TODO comments:

```
// TODO(editing-dev): Stop storing VisiblePositions through mutations.
// See crbug.com/648949 for details.
```

This comment appears at:
- `insert_line_break_command.cc:89`
- `insert_list_command.cc:763`
- `replace_selection_command.cc:1077`

In all three cases a VP is captured, then DOM is mutated, then the VP is used again.
In debug builds this triggers `DCHECK(IsValid())` failures. In release builds it
silently operates on a position that may point into a detached or restructured
subtree.

**What the developer sees:**
- DCHECK failures in debug builds during editing operations, filed as `crbug.com/648949`.
- Hard-to-reproduce crashes in release builds when editing complex content.
- `InsertListCommand::MoveParagraphOverPositionIntoEmptyListItem` takes a
  `VisiblePosition` parameter that was constructed before the `AppendNode` call
  inside the method — the VP is stale the moment `AppendNode` fires.

**What the user sees:**
- Rare but real crashes (especially on Android, where editing of complex web pages
  with lists is common).
- Silent correctness bugs: the wrong paragraph gets moved, the wrong range gets
  deleted, the caret lands in the wrong place after an operation.

---

## Problem 3: The VP Parameter Cascade (Code Complexity)

`MoveParagraph`, `MoveParagraphs`, `MoveParagraphWithClones`, `CleanupAfterDeletion`,
and `ReplaceCollapsibleWhitespaceWithNonBreakingSpaceIfNeeded` take `VisiblePosition`
parameters. This forces every caller to construct VPs even when the caller is already
working with raw `Position` values.

The cascade: `IndentOutdentCommand` loops over paragraphs, constructs VPs from
`Position`s, calls `MoveParagraph`, which internally calls `StartOfParagraph(VP)`,
which calls `CanonicalPositionOf` — another `UpdateStyleAndLayout`. A single
`IndentOutdentCommand::IndentBlock` call triggers layout multiple times per
paragraph in the selection.

**What the developer sees:**
- You cannot call `MoveParagraph` without constructing 3 VPs, even if you have
  perfectly good `Position` values. The type signature forces the cost.
- Tracing the flow of a VP through callers is non-trivial — by the time it arrives
  at `MoveParagraphs`, the VP may have been constructed 3 call frames up, with DOM
  mutations in between.
- `indent_outdent_command.cc` has 38 VP references, the majority of which exist
  solely to satisfy `MoveParagraph`'s signature.

**What the user sees:**
- Same jank as Problem 1, but multiplied: a Shift+Tab on a 20-item list triggers
  the cascade 20 times.

---

## Problem 4: VP Validity is Only Checked in Debug Builds

`VisiblePosition::IsValid()` is gated on `#if DCHECK_IS_ON()` — the actual version
check only runs in debug builds (`visible_position.cc:240-248`). In release builds
`IsValid()` unconditionally returns `true`. This means:

- Stale VPs (Problem 2) are silently tolerated in production.
- There is no runtime protection against using a VP that was constructed before a
  mutation and is now pointing at a dangling or restructured position.
- The code *looks* safe because of the DCHECK infrastructure, but the safety only
  exists in the builds developers run locally — not in shipped Chrome.

**What the developer sees:**
- Tests pass in debug builds (DCHECKs fire and catch issues). Flaky crashes in
  release/field builds go undiagnosed because there is no signal.
- The `TODO: Stop storing VisiblePositions through mutations` comments are years old
  (crbug.com/648949 is a long-running bug) — the fact that they haven't been fixed
  is partly because the DCHECK infrastructure makes the issue intermittent.

---

## Problem 5: VP Obscures What the Code Actually Needs

Most VP usage in commands falls into one of these shapes:

```cpp
if (IsEndOfParagraph(CreateVisiblePosition(pos))) { ... }
// Only needs: is pos at the end of its paragraph?

Position result = StartOfParagraph(CreateVisiblePosition(pos)).DeepEquivalent();
// Only needs: what Position is at the paragraph start?

VisiblePosition vp = CreateVisiblePosition(pos);
// ... 3 lines later ...
SomeMethod(vp.DeepEquivalent());
// Only needs: pos
```

In every case the code wants a DOM-structural answer (is this position at a paragraph
boundary? what node starts this block?). VP provides a *visual* answer that requires
layout. The code is using a heavier tool than the question requires.

**What the developer sees:**
- Reading `composite_edit_command.cc` is harder than it should be because every
  algorithm is written in terms of VP even when the algorithm is purely structural.
  The visual/layout machinery obscures the DOM intent.
- Adding a new editing command means cargo-culting the VP pattern because it is
  everywhere — new contributors learn the wrong idiom.
- Code review of editing patches is harder: reviewers must mentally track VP
  lifetimes across mutations to reason about correctness.

**What the user sees:**
- Indirect: more bugs introduced by new contributors who follow the existing
  pattern without understanding the mutation-validity hazard.

---

## Problem 6: VP Makes Commands Impossible to Unit Test Without a Layout Tree

`CreateVisiblePosition` requires a clean, up-to-date layout tree
(`visible_position.cc:107`: `DCHECK(!document.NeedsLayoutTreeUpdate())`). This means
any command that uses VP internally cannot be unit tested with a lightweight DOM —
it requires a full rendering pipeline.

**What the developer sees:**
- Editing command tests live in `editing_commands_*_test.cc` and require
  `RenderingTest` or `EditingTestBase` with full layout infrastructure.
- You cannot write a fast, lightweight unit test that exercises `MoveParagraphs`
  logic in isolation — the test must set up and run layout.
- Test suite for editing commands is slow as a result, reducing the feedback loop
  during development.
- Mocking or stubbing VP behavior is impractical — the type requires real layout
  state, not a fake position.

---

## Problem 7: VP Blocks the Path to Off-Thread Editing

Blink's longer-term direction is to move editing operations (especially for
`contenteditable` and IME) further from the main thread layout. `VisiblePosition`
is fundamentally tied to the main thread layout tree — it cannot exist without a
live, up-to-date `LayoutObject` graph.

As long as editing commands are written in terms of VP, they cannot be factored
into code that runs without layout. Every VP call is an implicit main-thread
layout dependency pinned into the command logic.

**What the developer sees:**
- Any attempt to refactor editing commands toward a data model separate from the
  rendering tree immediately hits VP as a blocker.
- `CompositeEditCommand` cannot be made to operate on a detached document fragment
  (e.g., for speculative editing or undo batching) because VP requires a live
  connected document with layout.

---

## Summary: Problems by Stakeholder

| Problem | Developer impact | User impact |
|---------|-----------------|-------------|
| Forced layout mid-command | Extra `UpdateStyleAndLayout` calls per command | Typing/paste jank, especially on complex pages |
| Stale VP after mutation | DCHECK crashes in debug, silent bugs in release | Rare crashes, caret landing in wrong place |
| VP parameter cascade | Forced VP construction at every call site | Multiplied jank on multi-paragraph operations |
| Debug-only validity checks | Release bugs invisible until they crash in the field | Intermittent crashes with no repro path |
| VP obscures DOM intent | Harder code review, wrong idiom taught to new contributors | Indirect: more bugs shipped |
| Requires full layout for tests | Slow test suite, no lightweight unit tests for command logic | Indirect: slower iteration, bugs shipped |
| Blocks off-thread editing | Structural blocker for future architecture work | Indirect: editing stays on main thread longer |
