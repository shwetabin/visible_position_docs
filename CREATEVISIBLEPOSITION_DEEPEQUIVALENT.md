# CreateVisiblePosition → Position Migration

## Background

`CreateVisiblePosition(pos).DeepEquivalent()` is the dominant anti-pattern in
`editing/commands/`. It creates a `VisiblePosition` from a raw `Position`,
unconditionally forces `Document::UpdateStyleAndLayout()`, then immediately discards
the VP to extract a `Position` back out. The canonicalized `Position` could be
obtained without a layout pass in almost every case.

There are **126 `CreateVisiblePosition` calls across 16 files** in `commands/*.cc`.
Approximately **110 (~87%)** of them do not need a full layout update — the VP is
used only as a type adapter or an equivalence-check vehicle.

### What `CreateVisiblePosition` actually does

```
CreateVisiblePosition(pos)
  -> Document::UpdateStyleAndLayout()   // ← forced layout pass
  -> CanonicalPositionOf(pos)           // snaps pos to a caret candidate
  -> stores result + document_version
```

### Cheaper alternatives already in the codebase

| Primitive | What it does | Layout cost |
|-----------|-------------|-------------|
| `MostForwardCaretPosition(pos)` | Walks forward to deepest equivalent caret position | Reads `GetLayoutObject()` — no forced update |
| `MostBackwardCaretPosition(pos)` | Walks backward to shallowest equivalent caret position | Same — reads existing layout, no forced update |
| `CanonicalPositionOf(pos)` | Returns `MostForwardCaretPosition(pos)` — canonical name for the same operation | No forced update |
| `CreateVisiblePosition(pos)` | Forces full layout, then snaps | Forces `UpdateStyleAndLayout` |

`MostForwardCaretPosition` and `MostBackwardCaretPosition` read the existing layout
object graph without forcing a recompute. They are the right tool whenever the
question is "what is the canonical `Position` for this caret location?" — which is
what `CreateVisiblePosition(pos).DeepEquivalent()` is answering in most of these
sites.

---

## Group A — Equivalence checks: `CreateVisiblePosition(x).DeepEquivalent() == CreateVisiblePosition(y).DeepEquivalent()`

**What is being asked:** "Do two raw positions represent the same visual caret?"

**Why VP is wrong here:** Both positions are canonicalized to compare them, then
both VPs are discarded. The comparison does not need a rendered line model — it
needs canonical `Position` values.

**Replacement:** `MostForwardCaretPosition(x) == MostForwardCaretPosition(y)`

Both sides are brought to the same canonical form (deepest equivalent caret) without
forcing layout. This is semantically equivalent for all cases where the position is
not at a line-wrap boundary with affinity (see Group D for those).

**Sites:**

| File | Lines | What is being compared | Replacement |
|------|-------|----------------------|-------------|
| `delete_selection_command.cc` | 149–153 | Adjusted `start`/`end` vs `selection_to_delete_` endpoints — detect if special-container walk drifted the selection | `MostForwardCaretPosition(start) == MostForwardCaretPosition(selection_to_delete_.Start())` |
| `insert_list_command.cc` | 139, 142 | `should_be_former` vs `should_be_later` — DCHECK that positions are visually ordered after selection adjustment | `DCHECK_EQ(should_be_former, MostForwardCaretPosition(should_be_former))` — positions are asserted canonical on input; DCHECK can be removed |
| `insert_list_command.cc` | 408–415 | `PositionBeforeNode(*list_element)` vs `current_selection.StartPosition()` / `PositionAfterNode` vs `EndPosition()` — check if list element boundary aligns with selection boundary | `MostForwardCaretPosition(PositionBeforeNode(*list_element)) == MostForwardCaretPosition(current_selection.StartPosition())` |
| `insert_paragraph_separator_command.cc` | 621 | `insertion_position` vs `VisiblePosition::BeforeNode(*block_to_insert)` — verify position did not move after node insertion | `MostForwardCaretPosition(insertion_position) != MostForwardCaretPosition(PositionBeforeNode(*block_to_insert))` |
| `replace_selection_command.cc` | 170–171 | `pos` vs `next_position` — detect visually distinct positions in line-break detection loop | `MostForwardCaretPosition(pos) != MostForwardCaretPosition(next_position)` |
| `editing_commands_utilities.cc` | 442–443 | `first` vs `MostBackwardCaretPosition(second)` in `IsVisiblyAdjacent` | `MostForwardCaretPosition(first) == MostBackwardCaretPosition(second)` — directly, no VP |
| `composite_edit_command.cc` | 2130–2131, 2138–2139 | `PositionBeforeNode(node)` / `PositionAfterNode(node)` vs selection range boundary | `MostForwardCaretPosition(PositionBeforeNode(node)) == MostForwardCaretPosition(selected_range.StartPosition())` |

**~9 sites. Standalone CL, no prerequisites.**

---

## Group B — Re-canonicalization after tree manipulation: `pos = CreateVisiblePosition(pos).DeepEquivalent()`

**What is being asked:** "Snap this position to a clean caret after a DOM mutation
or before/after-node boundary encoding."

**Why VP is wrong here:** The position needs to be normalized to the deepest
canonical caret location. `MostForwardCaretPosition` does exactly this without a
layout pass.

**Replacement:** `pos = MostForwardCaretPosition(pos)`

**Sites:**

| File | Lines | Context | Replacement |
|------|-------|---------|-------------|
| `insert_paragraph_separator_command.cc` | 279 | `canonical_pos = CreateVisiblePosition(insertion_position).DeepEquivalent()` — resolve before/after-node ambiguity before block boundary checks | `MostForwardCaretPosition(insertion_position)` |
| `composite_edit_command.cc` | 974 | `NextPositionOf(CreateVisiblePosition(pos)).DeepEquivalent()` — advance past a boundary and re-canonicalize | Requires `NextPositionOf(Position)` overload (Step 1), then `MostForwardCaretPosition` |

**~2 sites. `composite_edit_command.cc:974` unblocked after Step 1 overloads.**

---

## Group C — Type adapter: `CreateVisiblePosition(pos)` passed directly to VP-only API

**What is being asked:** Nothing about canonicalization — the position is already
correct. `CreateVisiblePosition` is called purely to satisfy a function signature
(`IsEndOfParagraph(VP)`, `StartOfParagraph(VP)`, `MoveParagraph(VP, VP, VP, ...)`,
etc.) that has no `Position` overload today.

**Why VP is wrong here:** The layout update is purely incidental to the type system.
The VP is passed to the function and never used for anything else.

**Replacement:** Pass `pos` directly once Step 1 adds `Position` overloads for the
downstream functions.

```cpp
// Before
IsEndOfParagraph(CreateVisiblePosition(pos))

// After (Step 1 landed)
IsEndOfParagraph(pos)
```

**Sites (selected — ~55 total):**

| File | Lines | Downstream callee | Step 1 overload needed |
|------|-------|------------------|----------------------|
| `apply_block_element_command.cc` | 197, 249, 444 | `MoveParagraph(VP, VP, VP, ...)` | `MoveParagraph(Position, Position, Position, ...)` |
| `apply_block_element_command.cc` | 305 | `StartOfParagraph(VP)` | `StartOfParagraph(Position)` |
| `format_block_command.cc` | 97–100 | `StartOfBlock(VP)` / `EndOfBlock(VP)` | `StartOfBlock(Position)` / `EndOfBlock(Position)` |
| `format_block_command.cc` | 106, 126, 129–130, 155–156 | `IsEndOfParagraph(VP)` / `IsStartOfParagraph(VP)` / `MoveParagraph` | Paragraph boundary overloads |
| `indent_outdent_command.cc` | 121, 126, 207–208, 243, 272, 361, 377, 542, 547 | `MoveParagraph` / `MoveParagraphs` / `IsStartOfParagraph` | `MoveParagraph(Position, ...)` |
| `insert_line_break_command.cc` | 116 | `IsEndOfParagraph(VP)` | `IsEndOfParagraph(Position)` |
| `insert_list_command.cc` | 115, 120, 241, 286, 326, 335, 584, 586, 605, 606 | `StartOfParagraph(VP)` / `MoveParagraph` / `IsStartOfParagraph` / `IsEndOfParagraph` | Paragraph + move overloads |
| `replace_selection_command.cc` | 1099, 1745–1746, 2171–2172 | `MoveParagraph` / `IsStartOfParagraph` / `IsEndOfParagraph` | Same |
| `composite_edit_command.cc` | 813, 815 | `IsEndOfParagraph(VP)` in whitespace helpers | `IsEndOfParagraph(Position)` |
| `editor_command.cc` | 1228 | `VisiblePositionForIndex` — public VP API | Remains VP (public boundary) |

**~54 sites. Unblocked after Step 1 overloads.**

---

## Group D — Affinity-aware placement: `CreateVisiblePosition(pos, affinity)`

**What is being asked:** "Given this position and this affinity, which visual line
does the caret sit on?"

**Why VP is genuinely needed here:** `TextAffinity` distinguishes whether the caret
at a line-wrap point is at the end of the upper line or the start of the lower line.
This is a visual-line question that requires layout. These are the **only** two sites
where `CreateVisiblePosition` is doing work that cannot be replaced without layout.

**Sites:**

| File | Line | Affinity | Purpose |
|------|------|---------|---------|
| `delete_selection_command.cc` | 343 | `selection_to_delete_.Affinity()` | `IsStartOfLine` / `HasSoftLineBreak` check — determines whether deletion endpoint is visually at the start of the next line |
| `insert_paragraph_separator_command.cc` | 345 | `affinity` (local var) | Determines whether caret is at end of current line or start of next, controlling which paragraph the separator is inserted into |

**2 sites. These are legitimately VP — no replacement.**

---

## Group E — Node boundary snap: `CreateVisiblePosition(PositionBeforeNode/AfterNode(...))`

**What is being asked:** "What is the first editable caret inside this element?"
`PositionBeforeNode` / `PositionAfterNode` / `FirstPositionInOrBeforeNode` /
`LastPositionInOrAfterNode` produce before/after-slot positions that are
non-canonical. The VP snaps to the first caret inside.

This breaks into two sub-groups:

**E-A: Snap exists only to satisfy a VP-typed callee.** The snapped position is
passed directly to `IsStartOfParagraph`, `IsEndOfParagraph`, `MoveParagraph`, etc.
Once Step 1 adds `Position` overloads, `PositionBeforeNode(x)` can be passed
directly — `MostForwardCaretPosition` is applied inside the overload.

**E-B: Snapped position is stored or compared.** The canonical position is the
actual value being computed — it is stored as a member, returned from a function, or
used in an equality check. Here `MostForwardCaretPosition(PositionBeforeNode(x))`
is the direct replacement.

**Sites:**

| File | Lines | Source position | Sub-group | Replacement |
|------|-------|----------------|-----------|-------------|
| `apply_block_element_command.cc` | 444 | `PositionAfterNode(...)` | E-A | Pass directly after Step 1 |
| `editing_commands_utilities.cc` | 144, 146 | `FirstPositionInOrBeforeNode` / `LastPositionInOrAfterNode` | E-A | Pass directly after Step 1 |
| `editing_commands_utilities.cc` | 199, 201, 222, 224 | `FirstPositionInOrBeforeNode` / `LastPositionInOrAfterNode` | E-A | `IsFirstVisiblePositionInNode` / `IsLastVisiblePositionInNode` need Position overloads |
| `replace_selection_command.cc` | 874, 946 | `LastPositionInOrAfterNode(*element)` | E-B | `MostForwardCaretPosition(LastPositionInOrAfterNode(*element))` |
| `replace_selection_command.cc` | 950, 959 | `end_of_inserted_content_` / `start_of_inserted_content_` | E-B | Member type changes to `Position` in Step 3 |
| `replace_selection_command.cc` | 1435 | `insertion_pos` | E-B | `MostForwardCaretPosition(insertion_pos)` |
| `replace_selection_command.cc` | 1601 | `start_of_inserted_content_` via `RelocatablePosition` | E-B | Covered by Step 3 member change |
| `insert_list_command.cc` | 744, 750 | `start_pos` / `relocatable_original_start->GetPosition()` | E-A | Pass directly after Step 1 |
| `insert_list_command.cc` | 779 | `pos.ToPositionWithAffinity()` | E-A | `PreviousPositionOf(Position)` overload (Step 1) |
| `composite_edit_command.cc` | 1777–1779 | `PositionAfterNode(*block_enclosing_list/list_node)` | E-B | `MostForwardCaretPosition(PositionAfterNode(...))` |

**~10 sites. E-B replaceable now; E-A unblocked after Step 1.**

---

## Group F — Post-`RelocatablePosition` re-canonicalization

**What is being asked:** After a DOM mutation, `RelocatablePosition::GetPosition()`
returns a `Position` that may sit at an awkward boundary (e.g. between two text
nodes that were split). `CreateVisiblePosition` snaps it to a clean caret before use.

**Why VP is wrong here:** The snap is needed but layout is not. `MostForwardCaretPosition`
snaps to the same canonical caret without a layout pass.

In practice, all five of these sites pass the VP directly to a VP-typed callee
(`MoveParagraph`, `MergeBlocksAfterDelete`, `ListifyParagraph`). They are therefore
also in Group C. Once Step 1 lands, the full expression becomes:

```cpp
// Before
CreateVisiblePosition(relocatable->GetPosition())  // passed to MoveParagraph(VP, ...)

// After (Step 1 landed)
relocatable->GetPosition()  // passed to MoveParagraph(Position, ...)
```

**Sites:**

| File | Lines | Pattern |
|------|-------|---------|
| `composite_edit_command.cc` | 1428, 1430 | `CreateVisiblePosition(relocatable_before/after_paragraph->GetPosition())` → `MoveParagraph` |
| `composite_edit_command.cc` | 1652, 1654 | Same in `MoveParagraphContentsToNewBlockIfNecessary` |
| `indent_outdent_command.cc` | 448–450, 475–477 | `CreateVisiblePosition(start/end_of_paragraph->GetPosition())` → `MoveParagraph` |
| `insert_list_command.cc` | 750 | `CreateVisiblePosition(relocatable_original_start->GetPosition())` → `ListifyParagraph` |
| `delete_selection_command.cc` | 1039 | `CreateVisiblePosition(relocatable_start->GetPosition())` → `MergeBlocksAfterDelete` |

**~10 sites. All unblocked after Step 1 overloads.**

---

## Summary

| Group | Sites | Replacement | Prerequisite |
|-------|-------|-------------|-------------|
| A — Equivalence check | ~9 | `MostForwardCaretPosition(x) == MostForwardCaretPosition(y)` | None — standalone CL |
| B — Re-canonicalize after mutation | ~2 | `MostForwardCaretPosition(pos)` | One site needs Step 1 |
| C — Type adapter for VP-only API | ~54 | Pass `Position` directly | Step 1 overloads |
| D — Affinity-aware placement | 2 | **Keep as VP** — genuinely load-bearing | N/A |
| E-B — Node boundary snap, stored/compared | ~5 | `MostForwardCaretPosition(PositionBeforeNode/AfterNode(...))` | None — standalone CL |
| E-A — Node boundary snap, passed to VP callee | ~5 | Pass directly | Step 1 overloads |
| F — Post-`RelocatablePosition` snap | ~10 | Pass raw `Position` directly | Step 1 overloads |
| **Total removable** | **~110** | | |
| **Legitimate VP** | **~16** | | |

Groups A, B (partial), and E-B can be cleaned up in a standalone CL before any Step 1
work. The remaining ~80 sites are naturally eliminated as Step 1 overloads land.
