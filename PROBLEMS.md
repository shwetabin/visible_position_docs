# Why We Are Removing VisiblePosition from Editing Commands

This document explains all the concrete problems caused by pervasive `VisiblePosition`
usage in editing commands. The root cause is architectural: VP is a rendering type
and does not belong inside command execution logic. Every other problem in this list
is a downstream consequence of that one mistake.

---

## Problem 1: VP Is the Wrong Type for Command Logic (Architecture)

`VisiblePosition` is a rendering concept. It answers the question: *where does the
caret visually appear on screen, given the current layout?* It requires a live,
up-to-date `LayoutObject` graph to be meaningful. It is correct at the
renderer↔command boundary — where a user gesture is translated into a command, and
where a command result is handed back to the renderer to update the cursor.

Editing commands operate on DOM structure. `DoApply()` asks questions like:
- Is this position at the end of its paragraph?
- What node delimits the current block?
- Where does the next paragraph start?

These are DOM-structural questions. They have answers expressible as `Position`
values — node plus offset. They do not require knowing where the caret *looks* on a
rendered line. The fact that the commands answer them using `VisiblePosition` is not
a design choice — it is a historical accident. `IsEndOfParagraph`, `StartOfParagraph`,
`MoveParagraph`, and similar functions simply did not have `Position` overloads when
the commands were written, so VP was used as a type adapter throughout.

**What this looks like in the code:**

```cpp
// The code wants: is this DOM position at a paragraph boundary?
if (IsEndOfParagraph(CreateVisiblePosition(pos))) { ... }

// The code wants: what Position is at the start of this paragraph?
Position result = StartOfParagraph(CreateVisiblePosition(pos)).DeepEquivalent();

// The code wants: pos
VisiblePosition vp = CreateVisiblePosition(pos);
SomeMethod(vp.DeepEquivalent());
```

In every case a `Position` is wrapped into a VP, used to call a function, then
`.DeepEquivalent()` is called to unwrap it back to a `Position`. The VP contributes
nothing except `UpdateStyleAndLayout()` as a side effect.

The correct abstraction boundary is:

```
[Boundary-IN]   renderer hands VisibleSelection to the command system
[Command body]  works entirely with Position / PositionWithAffinity / EphemeralRange
[Boundary-OUT]  command hands SelectionForUndoStep back to the renderer
```

VP inside `DoApply()` is a type leak across that boundary. All other problems below
are direct consequences of that leak.

---

## Problem 2: Stale VP After DOM Mutation (Correctness / Crashes)

Commands mutate the DOM. `VisiblePosition` records `dom_tree_version_` and
`style_version_` at construction (`visible_position.cc:64-65`). A VP captured before
a mutation is **technically invalid** after it — the document version changed, so the
snapped canonical position may no longer refer to the same location in the tree.

`Position` has no such validity window. It is a raw node+offset reference that
remains valid through mutations (subject only to the node itself being removed, which
`RelocatablePosition` handles).

The codebase acknowledges the VP staleness problem explicitly with three separate
TODO comments:

```cpp
// TODO(editing-dev): Stop storing VisiblePositions through mutations.
// See crbug.com/648949 for details.
```

This appears at:
- `insert_line_break_command.cc:89`
- `insert_list_command.cc:763`
- `replace_selection_command.cc:1077`

In all three cases a VP is captured, then DOM is mutated, then the VP is used again.
In debug builds this triggers `DCHECK(IsValid())` failures. In release builds it
silently operates on a position that may point into a detached or restructured subtree.

**What the developer sees:**
- DCHECK failures in debug builds, tracked as `crbug.com/648949`.
- `InsertListCommand::MoveParagraphOverPositionIntoEmptyListItem` takes a VP
  parameter that is stale the moment `AppendNode` fires inside the method.
- `IsValid()` is gated on `#if DCHECK_IS_ON()` — in release builds it unconditionally
  returns `true`. Stale VP bugs are invisible in shipped Chrome.

**What the user sees:**
- Rare but real crashes, especially on Android with list-heavy pages.
- Silent correctness bugs: wrong paragraph moved, wrong range deleted, caret lands
  in the wrong place after paste or indent.

---

## Problem 3: Forced Layout Mid-Command (Performance)

`CreateVisiblePosition()` calls `CanonicalPositionOf()`, which calls
`Document::UpdateStyleAndLayout()`. A composite editing command — paste, indent, list
insertion — constructs dozens of VPs interleaved with DOM mutations. Each VP
construction forces a full style+layout recompute against a partially-mutated tree.

This cost exists entirely because of Problem 1: the wrong type is being used, and
that type happens to require layout. If the commands used `Position` throughout,
no mid-command layout would be forced.

**What the developer sees:**
- `apply_block_element_command.cc` calls `GetDocument().UpdateStyleAndLayout()` at
  lines 127, 248, 316, 353, 410 — five times inside a single command execution.
- `composite_edit_command.cc` has `DCHECK(!GetDocument().NeedsLayoutTreeUpdate())`
  gates at lines 972, 1079, 1452, 1470, 1604, 1753 — each is a point where the code
  requires layout to be clean before proceeding.
- `EndingVisibleSelection()` calls `UpdateStyleAndLayout()` unconditionally, even
  when the caller already did so one line earlier (double-layout on a clean tree).

**What the user sees:**
- Typing lag in large or complex documents (contenteditable with tables, lists,
  heavy CSS).
- Jank during paste of rich content.
- Slow indent/outdent on multi-paragraph selections.

---

## Problem 4: The VP Parameter Cascade (Code Complexity)

Because `MoveParagraph`, `MoveParagraphs`, `MoveParagraphWithClones`,
`CleanupAfterDeletion`, and `ReplaceCollapsibleWhitespaceWithNonBreakingSpaceIfNeeded`
take `VisiblePosition` parameters, every caller must construct VPs even when working
with perfectly good `Position` values.

`indent_outdent_command.cc` has 38 VP references. The majority exist solely to
satisfy `MoveParagraph`'s signature. The cascade: construct VP from `Position`, pass
to `MoveParagraph`, which internally calls `StartOfParagraph(VP)`, which calls
`CanonicalPositionOf` — another `UpdateStyleAndLayout`. One `IndentBlock` call
triggers layout multiple times per paragraph in the selection.

**What the developer sees:**
- You cannot call `MoveParagraph` without constructing 3 VPs, even with perfectly
  good `Position` values. The type signature forces the cost.
- Tracing a VP through callers is non-trivial — by the time it arrives at
  `MoveParagraphs`, it may have been constructed 3 call frames up, with DOM mutations
  in between.

**What the user sees:**
- The cascade cost multiplied: Shift+Tab on a 20-item list triggers it 20 times.

---

## Problem 5: VP Makes Command Logic Harder to Read and Review

Because VP is a rendering type, reading command code requires simultaneously tracking
two concerns: the DOM-structural algorithm and the rendering side-effects. These do
not belong together.

**What the developer sees:**
- `composite_edit_command.cc` algorithms are written in terms of VP even when purely
  structural. The visual/layout machinery obscures the DOM intent.
- New contributors cargo-cult the VP pattern because it is everywhere — they learn
  the wrong idiom and introduce new VP usage in new commands.
- Code review of editing patches requires mentally tracking VP lifetimes across
  mutations to reason about correctness. This is non-trivial and routinely misses
  the staleness hazard (Problem 2).

**What the user sees:**
- Indirect: more bugs shipped by contributors who follow the existing pattern without
  understanding the mutation-validity hazard.

---

## Problem 6: VP Requires a Full Rendering Pipeline for Unit Tests

`CreateVisiblePosition` requires a clean, up-to-date layout tree
(`visible_position.cc:107`: `DCHECK(!document.NeedsLayoutTreeUpdate())`). Any command
that uses VP internally cannot be tested with a lightweight DOM fixture — it requires
a full rendering pipeline.

**What the developer sees:**
- Editing command tests require `RenderingTest` or `EditingTestBase` with full layout
  infrastructure. A fast, lightweight unit test for `MoveParagraphs` logic in
  isolation is not possible today.
- The test suite for editing commands is slow, degrading the development feedback loop.

---

## Problem 7: VP Pins Commands to the Main Thread Layout Tree

`VisiblePosition` cannot exist without a live, up-to-date `LayoutObject` graph on the
main thread. As long as editing commands are written in terms of VP, they cannot be
factored into code that runs without layout.

Blink's direction for `contenteditable` and IME is to reduce main-thread layout
dependencies. VP usage inside `DoApply()` is a structural blocker for any such work:
every VP construction is an implicit main-thread layout dependency pinned into
command logic.

**What the developer sees:**
- Any attempt to operate on a detached document fragment (e.g., speculative editing,
  undo batching, or off-thread IME) immediately hits VP as a hard blocker.

---

## Summary

| Problem | Root cause | Developer impact | User impact |
|---------|-----------|-----------------|-------------|
| Wrong abstraction | VP is a rendering type used for DOM-structural logic | Code harder to read, wrong idiom taught, VP leaks across boundary | All other problems below |
| Stale VP after mutation | VP validity window does not survive DOM mutations | DCHECK crashes (debug), silent bugs (release), crbug.com/648949 | Rare crashes, wrong caret position |
| Forced layout mid-command | `CreateVisiblePosition` calls `UpdateStyleAndLayout` | Extra layout passes per command, double-layout patterns | Typing/paste jank on complex pages |
| VP parameter cascade | Infrastructure signatures take VP, forcing callers | VP construction forced at every call site, non-traceable lifetimes | Multiplied jank on multi-paragraph operations |
| Obscures DOM intent | VP mixes rendering semantics into structural algorithms | Harder code review, mutation-validity hazard invisible to reviewers | Bugs from new contributors following existing pattern |
| Requires full rendering for tests | `CreateVisiblePosition` requires live layout tree | No lightweight unit tests for command logic, slow test suite | Slower iteration, bugs shipped |
| Blocks off-thread work | VP tied to main-thread `LayoutObject` graph | Structural blocker for future architecture work | Editing stays on main thread longer |
