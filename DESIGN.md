# Design: Reducing VisiblePosition Usage in Editing Commands

**Status:** Draft for review
**Audience:** Chromium editing-dev owners
**Authors:** TBD
**Last updated:** 2026-03-17

---

## Background & Motivation

`VisiblePosition` is Blink's "canonical position" type — a `PositionWithAffinity`
whose inner position has been snapped to a visually-renderable caret location via
`CanonicalPositionOf()`. That canonicalization walk requires an up-to-date layout tree.

Today, editing commands create `VisiblePosition` objects pervasively during command
execution — not just at the entry/exit boundary where they are needed for user-visible
selection handling. This causes several concrete problems:

1. **Forced layout mid-command.** Every `CreateVisiblePosition()` call triggers
   `Document::UpdateStyleAndLayout()`. A composite command that makes dozens of DOM
   mutations therefore computes layout many more times than necessary.

2. **Stale VP after DOM mutation.** `VisiblePosition` validates against
   `dom_tree_version_` / `style_version_` at construction. A VP captured before a
   mutation is technically invalid after it. The codebase acknowledges this explicitly:
   `// TODO(editing-dev): Stop storing VisiblePositions through mutations.`
   (`insert_line_break_command.cc:89`).

3. **Redundant round-trips.** The pattern `CreateVisiblePosition(pos).DeepEquivalent()`
   appears throughout the commands — creating a VP only to immediately extract the raw
   `Position` back. The only effect is the `CanonicalPositionOf()` snap, which is often
   unnecessary when the input position is already canonical.

4. **VP as a parameter convention.** `MoveParagraph`, `MoveParagraphWithClones`,
   `CleanupAfterDeletion`, and `ReplaceCollapsibleWhitespaceWithNonBreakingSpaceIfNeeded`
   take `VisiblePosition` parameters. This forces every caller to create VPs even when
   working with raw positions, cascading VP creation through the heavy composite
   commands.

### Scale

In `editing/commands/*.cc` (source files only):

| Metric | Count |
|--------|-------|
| `VisiblePosition` type usage | ~391 across 18 files |
| `CreateVisiblePosition()` calls | ~126 across 16 files |
| `.DeepEquivalent()` calls | ~288 across 18 files |

Top offenders: `composite_edit_command.cc` (112), `replace_selection_command.cc` (98),
`insert_list_command.cc` (91), `delete_selection_command.cc` (64),
`indent_outdent_command.cc` (61).

### The Correct Abstraction Boundary

```
User Interaction
    |
    v
[Boundary-IN]   VisibleSelection → SelectionInDOMTree   (renderer → command)
    |
    v
[Command body]  Works entirely with Position / PositionWithAffinity / EphemeralRange
    |
    v
[Boundary-OUT]  SelectionInDOMTree → SetEndingSelection(...)  (command → renderer)
```

`VisiblePosition` belongs only at the two boundary points. `EndingVisibleSelection()`
in `CompositeEditCommand` is the correct boundary-out location. The problem is that
commands call it — and create VPs — deep inside `DoApply()`.

---

## Anti-Patterns to Eliminate

### P1: Create-then-extract

```cpp
// insert_line_break_command.cc:116
if (IsEndOfParagraph(CreateVisiblePosition(caret.ToPositionWithAffinity())) && ...)
```

A VP is created from a `PositionWithAffinity` and immediately passed to `IsEndOfParagraph`.
The VP serves only as a type adapter; `caret` was already obtained from
`EndingVisibleSelection().VisibleStart()` and is already canonical.

### P2: VP as a type requirement for navigation tests

```cpp
// composite_edit_command.cc:812-834
VisiblePosition visible_upstream_pos = CreateVisiblePosition(Position(text_node, upstream));
VisiblePosition visible_downstream_pos = CreateVisiblePosition(Position(text_node, downstream));
if (IsEndOfParagraph(visible_downstream_pos) || ...)
    should_emit_nbsp_before_end = ...;
```

`IsEndOfParagraph` / `IsStartOfParagraph` have no `Position` overload today, so VP
creation is forced purely to satisfy the type signature.

### P3: VP stored across a DOM mutation

```cpp
// insert_line_break_command.cc:84-98
VisibleSelection selection = EndingVisibleSelection();
VisiblePosition caret(selection.VisibleStart());  // VP captured here
// Comment at :89: "TODO: Stop storing VisiblePositions through mutations."
Position pos(caret.DeepEquivalent());
```

### P4: VP parameter forces caller canonicalization

```cpp
// composite_edit_command.h:200-206
void MoveParagraph(const VisiblePosition& start,
                   const VisiblePosition& end,
                   const VisiblePosition& destination, ...);
```

Every caller in `indent_outdent_command.cc`, `insert_list_command.cc`,
`delete_selection_command.cc`, etc. must wrap raw positions into VPs before calling.
Much of the VP count in these files traces back to this single cascade.

### P5: `EndingVisibleSelection()` called mid-command for raw position access

```cpp
// composite_edit_command.cc:890-897
void CompositeEditCommand::RebalanceWhitespace() {
  VisibleSelection selection = EndingVisibleSelection();  // triggers UpdateStyleAndLayout
  if (selection.IsNone()) return;
  RebalanceWhitespaceAt(selection.Start());  // Start() is just a Position
  if (selection.IsRange())
    RebalanceWhitespaceAt(selection.End());
}
```

`EndingSelection().Anchor()` / `.Start()` / `.End()` are available on
`SelectionForUndoStep` without a layout update.

---

## Proposed Approach: Hybrid (bottom-up foundation, top-down infrastructure)

The approach combines two complementary moves in a specific order that keeps every
intermediate state compile-clean and independently testable:

1. **Add missing `Position`-based overloads** for navigation functions in `visible_units`
   and `editing_commands_utilities` (bottom-up, purely additive).
2. **Change `CompositeEditCommand` infrastructure signatures** to take `Position`
   instead of `VisiblePosition` (top-down, uses the new overloads internally).
3. **Migrate command files** using the new overloads and the new infrastructure
   signatures, in ascending order of VP count.

### Why this ordering

Doing the signature changes (step 2) without the overloads (step 1) is blocked: the
bodies of `MoveParagraphs` and related methods still need navigation functions that
only have VP overloads. Adding the overloads first makes step 2 possible.

Doing only the overloads (step 1) without the signature changes leaves the P4 cascade
intact: callers of `MoveParagraph` are still forced to create VPs regardless. Step 2
eliminates this entire class of forced VP creation in one CL per method.

---

## Phase 1: Add Position-based Navigation Overloads

**Nature:** Purely additive. No existing code is changed. Each overload is its own CL
with tests. Zero regression risk.

### Gaps to fill in `visible_units.h` / `visible_units.cc`

```cpp
// paragraph
bool IsStartOfParagraph(const Position&,
                        EditingBoundaryCrossingRule = kCannotCrossEditingBoundary);
bool IsEndOfParagraph(const Position&,
                      EditingBoundaryCrossingRule = kCannotCrossEditingBoundary);
Position StartOfParagraph(const Position&,
                          EditingBoundaryCrossingRule = kCannotCrossEditingBoundary);
Position EndOfParagraph(const Position&,
                        EditingBoundaryCrossingRule = kCannotCrossEditingBoundary);
Position StartOfNextParagraph(const Position&);
bool InSameParagraph(const Position&, const Position&,
                     EditingBoundaryCrossingRule = kCannotCrossEditingBoundary);

// document
Position EndOfDocument(const Position&);
bool IsStartOfDocument(const Position&);
bool IsEndOfDocument(const Position&);

// character inspection
UChar32 CharacterAfter(const Position&);
```

### Gaps to fill in `editing_commands_utilities.h` / `.cc`

```cpp
Position StartOfBlock(const Position&,
                      EditingBoundaryCrossingRule = kCannotCrossEditingBoundary);
Position EndOfBlock(const Position&,
                    EditingBoundaryCrossingRule = kCannotCrossEditingBoundary);
bool IsStartOfBlock(const Position&);
bool IsEndOfBlock(const Position&);
Node* EnclosingEmptyListItem(const Position&);
// LineBreakExistsAtPosition(Position) already exists — verify all callers can use it.
```

### Already exist (reference)

No work needed for these — they are already in `visible_units.h`:

```cpp
Position StartOfWordPosition(const Position&, WordSide);
Position EndOfWordPosition(const Position&, WordSide);
PositionWithAffinity StartOfLine(const PositionWithAffinity&);
PositionWithAffinity EndOfLine(const PositionWithAffinity&);
bool InSameLine(const PositionWithAffinity&, const PositionWithAffinity&);
Position StartOfDocument(const Position&);
```

### Implementation strategy for the new overloads

The `Position`-typed algorithm already exists as a private template in the
implementation file. `visible_units_paragraph.cc` already has
`StartOfParagraphAlgorithm<Strategy>(const PositionTemplate<Strategy>&, ...)` which
operates purely on DOM structure — no `UpdateStyleAndLayout`, no VP construction.
The existing VP public overload already calls into it by extracting `.DeepEquivalent()`
first.

The new public `Position` overloads call the algorithm directly:

```cpp
// visible_units_paragraph.cc
Position StartOfParagraph(const Position& pos,
                           EditingBoundaryCrossingRule rule) {
  return StartOfParagraphAlgorithm<EditingStrategy>(pos, rule);
}

bool IsStartOfParagraph(const Position& pos,
                         EditingBoundaryCrossingRule rule) {
  return pos == StartOfParagraphAlgorithm<EditingStrategy>(pos, rule);
}
```

The pattern holds for all Phase 1 additions: check whether a `Position`-typed
algorithm template already exists before writing the overload body. For
`visible_units_paragraph.cc` and `visible_units.cc` it does. For any function
that has no such algorithm, write the algorithm first and have both the VP and
Position overloads call into it.

### CL priority order

By usage frequency across commands:
1. `IsStartOfParagraph(Position)` / `IsEndOfParagraph(Position)` — used in 9 files
2. `StartOfParagraph(Position)` / `EndOfParagraph(Position)` — used in 7 files
3. `StartOfBlock(Position)` / `EndOfBlock(Position)` / `IsStartOfBlock(Position)` / `IsEndOfBlock(Position)`
4. `InSameParagraph(Position, Position)`
5. `StartOfNextParagraph(Position)`
6. `EnclosingEmptyListItem(Position)` → `Node*`

---

## Phase 2: Migrate Simple Commands

Migrate the 5 lowest-VP command files using the Phase 1 overloads. These serve as
proof-of-concept and validate that the new overloads are drop-in replacements against
the WPT editing test suite.

| File | VP refs | Notes |
|------|---------|-------|
| `style_commands.cc` | 1 | XS — one VP reference |
| `undo_step.cc` | 2 | XS |
| `editor_command.cc` | 6 | S |
| `insert_line_break_command.cc` | 6 | S — P3 and P1 examples live here |
| `insert_text_command.cc` | 7 | S — uses `FirstPositionInNode`, `LastPositionInNode`, `CreateVisiblePosition` |

### Migration patterns for call sites

| Pattern (before) | Replacement (after) |
|---|---|
| `CreateVisiblePosition(pos).DeepEquivalent()` | `pos` — drop the call entirely; do not substitute `CanonicalPositionOf` |
| `IsEndOfParagraph(CreateVisiblePosition(pos))` | `IsEndOfParagraph(pos)` (Phase 1 overload) |
| `StartOfParagraph(vp).DeepEquivalent()` | `StartOfParagraph(pos)` (Phase 1 overload) |
| `VP::FirstPositionInNode(*node).DeepEquivalent()` | `Position::FirstPositionInNode(*node)` |
| `VP::LastPositionInNode(*node).DeepEquivalent()` | `Position::LastPositionInNode(*node)` |
| `LineBreakExistsAtVisiblePosition(vp)` | `LineBreakExistsAtPosition(vp.DeepEquivalent())` |
| `VisiblePosition::BeforeNode(*node)` (as position source) | `Position::BeforeNode(*node)` |
| `EndingVisibleSelection().VisibleStart().DeepEquivalent()` | `EndingSelection().Start()` |

One CL per command file. Each CL must pass `blink_unittests --gtest_filter="*Edit*"`
and the relevant WPT tests before landing.

---

## Phase 3: Infrastructure Signature Changes

Change the `CompositeEditCommand` protected methods that take `VisiblePosition`
parameters to take `Position`. With Phase 1 overloads in place, the method bodies
can be migrated simultaneously.

### Target signatures

```cpp
// composite_edit_command.h — proposed changes

// Before:
void MoveParagraph(const VisiblePosition& start,
                   const VisiblePosition& end,
                   const VisiblePosition& destination,
                   EditingState*,
                   ShouldPreserveSelection = kDoNotPreserveSelection,
                   ShouldPreserveStyle = kPreserveStyle,
                   Node* constraining_ancestor = nullptr);

void MoveParagraphs(const VisiblePosition& start,
                    const VisiblePosition& end,
                    const VisiblePosition& destination,
                    EditingState*,
                    ShouldPreserveSelection = kDoNotPreserveSelection,
                    ShouldPreserveStyle = kPreserveStyle,
                    Node* constraining_ancestor = nullptr);

void MoveParagraphWithClones(const VisiblePosition& start,
                             const VisiblePosition& end,
                             HTMLElement* block_element,
                             Node* outer_node,
                             EditingState*);

void CleanupAfterDeletion(EditingState*, VisiblePosition destination);

void ReplaceCollapsibleWhitespaceWithNonBreakingSpaceIfNeeded(
    const VisiblePosition&);

// After:
void MoveParagraph(const Position& start,
                   const Position& end,
                   const Position& destination,
                   EditingState*,
                   ShouldPreserveSelection = kDoNotPreserveSelection,
                   ShouldPreserveStyle = kPreserveStyle,
                   Node* constraining_ancestor = nullptr);

void MoveParagraphs(const Position& start,
                    const Position& end,
                    const Position& destination,
                    EditingState*,
                    ShouldPreserveSelection = kDoNotPreserveSelection,
                    ShouldPreserveStyle = kPreserveStyle,
                    Node* constraining_ancestor = nullptr);

void MoveParagraphWithClones(const Position& start,
                             const Position& end,
                             HTMLElement* block_element,
                             Node* outer_node,
                             EditingState*);

void CleanupAfterDeletion(EditingState*, Position destination = Position());

void ReplaceCollapsibleWhitespaceWithNonBreakingSpaceIfNeeded(const Position&);
```

### Internal body changes for `MoveParagraphs`

The body of `MoveParagraphs` (`composite_edit_command.cc:1462`) already uses
`Position` internally for most operations — `TextIterator::RangeLength` takes
`ToParentAnchoredPosition()`, `CloneParagraphUnderNewElement` already takes
`Position`, and `RelocatablePosition` stores `Position`. The VP parameters are used
in three places:

**VP comparison operators** (lines 1481-1484):
```cpp
// Before:
if (destination.DeepEquivalent() >= start_of_paragraph_to_move.DeepEquivalent() &&
    destination.DeepEquivalent() <= end_of_paragraph_to_move.DeepEquivalent())

// After:
if (ComparePositions(destination, start) >= 0 &&
    ComparePositions(destination, end) <= 0)
```

**`PreviousPositionOf` for `RelocatablePosition` setup** (lines 1541-1545):

`PreviousPositionOf` has a `Position`-typed overload (`PreviousPositionOf(const Position&, EditingBoundaryCrossingRule)`) in `visible_units.h`. Use it directly:

```cpp
// Before:
RelocatablePosition* before_paragraph_position =
    MakeGarbageCollected<RelocatablePosition>(
        PreviousPositionOf(start_of_paragraph_to_move,
                           kCannotCrossEditingBoundary).DeepEquivalent());

// After:
RelocatablePosition* before_paragraph_position =
    MakeGarbageCollected<RelocatablePosition>(
        PreviousPositionOf(start, kCannotCrossEditingBoundary));
```

**`kPreserveSelection` path** (lines 1494-1539): calls
`EndingVisibleSelection().VisibleStart()` / `.VisibleEnd()`. These become:
```cpp
// Before:
VisiblePosition visible_start = EndingVisibleSelection().VisibleStart();
VisiblePosition visible_end   = EndingVisibleSelection().VisibleEnd();

// After:
Position visible_start = EndingSelection().Start();
Position visible_end   = EndingSelection().End();
```

`ComparePositions(visible_start, end_of_paragraph_to_move)` already accepts `Position`
on both sides after the parameter type change. The `TextIterator::RangeLength` calls
use `ToParentAnchoredPosition()` which is available on `Position` directly.

### Internal body changes for `CleanupAfterDeletion`

`CleanupAfterDeletion` (`composite_edit_command.cc:1310`) currently takes
`VisiblePosition destination` and uses it as:
- `destination.DeepEquivalent().AnchorNode()` → becomes `destination.AnchorNode()`
- `caret_after_delete.DeepEquivalent() != destination.DeepEquivalent()` → becomes
  position comparison directly
- `RendersInDifferentPosition(position, destination.DeepEquivalent())` → takes
  `Position` already

`caret_after_delete` is currently `EndingVisibleSelection().VisibleStart()`. With
Position parameters this becomes `EndingSelection().Start()`, which avoids the
`UpdateStyleAndLayout()` inside `EndingVisibleSelection()`. The `IsStartOfParagraph`
/ `IsEndOfParagraph` checks on `caret_after_delete` use the Phase 1 overloads.

### Internal body changes for `ReplaceCollapsibleWhitespaceWithNonBreakingSpaceIfNeeded`

`CharacterAfter` currently has VP-only overloads (`visible_units.h:113-114`). Add a
`Position` overload to Phase 1 so no VP is needed here:

```cpp
// Phase 1 addition in visible_units.h / visible_units.cc:
CORE_EXPORT UChar32 CharacterAfter(const Position&);

// Then the method body becomes:
void CompositeEditCommand::
    ReplaceCollapsibleWhitespaceWithNonBreakingSpaceIfNeeded(
        const Position& position) {
  if (!IsCollapsibleWhitespace(CharacterAfter(position)))
    return;
  Position pos = MostForwardCaretPosition(position);
  ...
}
```

No VP construction anywhere in the method. The `CharacterAfter(Position)` overload
should be implemented via the existing `CharacterAfterAlgorithm` template if one
exists, or by calling `CharacterAfter(CreateVisiblePosition(pos))` internally only
as a temporary bridge — marked with a TODO for follow-up removal once the algorithm
is ported.

### One CL per method

Each signature change is a separate CL: `MoveParagraph`/`MoveParagraphs` together
(they share the same body), then `MoveParagraphWithClones`, then `CleanupAfterDeletion`,
then `ReplaceCollapsibleWhitespaceWithNonBreakingSpaceIfNeeded`. This keeps each CL
independently reviewable and revertable.

---

## Phase 4: Heavy Command Migration

With Phase 1 overloads and Phase 3 infrastructure in place, the 6 heavy command files
can be migrated. The majority of their VP usage falls into two categories already
addressed:

- **Category A** (fixed by Phase 3): VP created solely to pass to `MoveParagraph` /
  `MoveParagraphWithClones` / `CleanupAfterDeletion`.
- **Category B** (fixed by Phase 1): VP created solely to call `IsEndOfParagraph`,
  `IsStartOfParagraph`, `StartOfParagraph`, `EndOfParagraph`, `StartOfBlock`, etc.

The remaining VP usage after removing Category A and B will be small and can be
addressed file-by-file.

| File | VP refs | Primary VP category | Assigned |
|------|---------|-------------------|---------|
| `insert_paragraph_separator_command.cc` | 19 | B | Person A |
| `apply_block_element_command.cc` | 37 | B + A | Person B |
| `indent_outdent_command.cc` | 61 | A (MoveParagraph callers) | Person A |
| `delete_selection_command.cc` | 64 | A + B | Person B |
| `insert_list_command.cc` | 91 | A + B | Person A |
| `replace_selection_command.cc` | 98 | B | Person B |

**Note on `composite_edit_command.cc` (112 refs):** Phase 3 addresses the method
signatures. The remaining VP in the private methods (`RebalanceWhitespaceOnTextSubstring`,
`MoveParagraphContentsToNewBlockIfNecessary`, `BreakOutOfEmptyListItem`,
`BreakOutOfEmptyMailBlockquotedParagraph`) are addressed as part of Phase 4 alongside
the heavy commands, since they are typically called from those files.

### Estimated VP reduction

| Phase | Estimated VPs removed |
|-------|-----------------------|
| 2 — simple commands | ~25 |
| 3 — infrastructure signatures | ~80 (cascades to all callers) |
| 4 — heavy commands + composite private methods | ~220 |
| **Total** | **~325 of ~391 (83%)** |

---

## What is Legitimately VP (Do Not Remove)

Not all VP usage should be removed. The following are correct uses:

1. **Entry boundary.** The `CompositeEditCommand` constructor reads
   `frame->Selection().ComputeVisibleSelectionInDOMTreeDeprecated()` to set
   `starting_selection_`. This is correct — it is the boundary-in point where the
   renderer hands a visible selection to the command system.

2. **Exit boundary.** `EndingVisibleSelection()` in `Apply()` checks
   `IsRichlyEditablePosition(EndingVisibleSelection().Anchor())`. This is correct —
   it is the boundary-out validation point.

3. **Affinity-sensitive navigation.** `PreviousPositionOf(VisiblePosition)` /
   `NextPositionOf(VisiblePosition)` return VP because the result depends on line-wrap
   affinity. In `MoveParagraphWithClones` / `MoveParagraphs`, the before/after
   paragraph positions are computed this way and immediately stored in
   `RelocatablePosition`. This is correct; the VP is used once and immediately
   converted to a stable `Position`.

4. **`VisiblePosition::FirstPositionInNode` / `LastPositionInNode` as equality
   tests.** `delete_selection_command.cc:66-67` checks:
   ```cpp
   VisiblePosition::FirstPositionInNode(*cell).DeepEquivalent() ==
   VisiblePosition::LastPositionInNode(*cell).DeepEquivalent();
   ```
   This may be replaceable with `Position::FirstPositionInNode` / `LastPositionInNode`,
   but requires per-site analysis to confirm the cell's positions are canonical. Flag
   these for careful review rather than mechanical replacement.

---

## Testing Strategy

**Per-CL (required before landing):**
- `blink_unittests --gtest_filter="*Edit*"` — all pass
- `third_party/blink/web_tests/editing/` WPT suite — no new failures
- `third_party/blink/web_tests/editing/contenteditable/` — no new failures

**Phase gate (before starting the next phase):**
- Full `blink_unittests` suite
- `content_browsertests` editing subset
- Verify `visible_position.h` include count in `commands/` is decreasing

**Manual smoke test (per phase):**
Basic typing, delete/backspace, paste (Ctrl+V), undo/redo, list creation,
indent/outdent, Enter key inside list items, table editing.

**Regression monitoring:**
Watch CQ for editing test failures for 2 weeks after each phase lands.

---

## Open Questions for Owners

**1. `EndingVisibleSelection()` migration scope.**
The method is called ~40 times across commands and always triggers
`UpdateStyleAndLayout()`. Should we deprecate it and migrate all call sites in
Phase 3, or limit scope to the P5 cases where it demonstrably forces unnecessary layout?

**3. `MoveParagraphs` `kPreserveSelection` path semantics.**
The selection-preservation path (lines 1494-1539 in `composite_edit_command.cc`)
currently uses VP comparison operators between the ending selection's visible
start/end and the paragraph boundaries. Replacing these with `ComparePositions` on
raw positions is semantically equivalent only if the positions are canonical. This
path needs a targeted test case that exercises `kPreserveSelection` to verify
correctness before and after the change.

**4. `SelectionForParagraphIteration` signature.**
This utility takes and returns `VisibleSelection`. Its callers in
`apply_block_element_command.cc` and `format_block_command.cc` are Phase 4 targets.
Should this be changed to take/return `SelectionInDOMTree` as part of Phase 2 to
unblock those files, or deferred to Phase 4?

---

## Scope: Why Commands First

This document focuses on `editing/commands/` because it contains the highest
concentration of *unnecessary* VP usage — VP used purely as an internal computation
type with no connection to visual rendering. The anti-patterns described above exist
because the code predates `Position`-based navigation overloads; there is no
architectural reason for VP to be present inside `DoApply()` bodies at all.

The broader `editing/` directory has ~925 additional VP references outside of
`commands/`. These fall into two categories:

**Legitimate VP usage — out of scope for this effort:**

| File | VP refs | Why VP is correct |
|------|---------|-------------------|
| `selection_modifier.cc` | 93 | User-driven caret/selection movement (arrow keys, Shift+Click). Affinity and canonicalization are genuinely required to know where the caret lands on a wrapped line. |
| `granularity_strategy.cc` | 16 | Touch selection expansion by word/paragraph. Same rationale as `selection_modifier`. |
| `frame_selection.cc` | 15 | The renderer boundary itself. VP is correct at the entry/exit point of the editing stack. |
| `editing_utilities.cc` (partial) | ~10 | `IndexForVisiblePosition`, `VisiblePositionForIndex`, `MakeRange(VP, VP)` — explicit public API contracts. Changing these is a separate, larger undertaking. |

**Files with the same anti-patterns as commands — natural follow-on targets:**

| File | VP refs | Anti-patterns present |
|------|---------|----------------------|
| `selection_adjuster.cc` | 28 | P1, P2 — VP created solely to call `StartOfLine`, `StartOfParagraph`, `EndOfParagraph`, `NextPositionOf` for granularity expansion. Phase 1 overloads unblock this directly. |
| `editing_utilities.cc` (partial) | ~12 | P1, P2 — VP in `AdjustSelectionToAvoidCrossingShadowBoundaries` and `InsertTabSpanBefore` internals. |
| `serializers/styled_markup_serializer.cc` | 7 | P1, P2 — VP for paragraph boundary detection during markup serialization. Phase 1 overloads apply directly. |
| `editor.cc` | 10 | Mixed — some P2 boundary test calls, some legitimate boundary use. Needs per-site classification. |

The Phase 1 overloads added in this effort (`IsStartOfParagraph(Position)`,
`StartOfParagraph(Position)`, etc.) are immediately usable by all of the follow-on
files above. No additional foundation work is needed to address them.

---

## Next Steps: Beyond Commands

Once the four phases in this document are complete, the following files in the broader
`editing/` tree should be migrated using the same Phase 1 overloads:

- `core/editing/selection_adjuster.cc` — 28 VP refs, primarily P1/P2
- `core/editing/editing_utilities.cc` — ~12 VP refs in internal helpers (excluding the public VP API functions)
- `core/editing/serializers/styled_markup_serializer.cc` — 7 VP refs, P1/P2
- `core/editing/editor.cc` — ~5 VP refs in internal helpers

These are explicitly out of scope for the current effort to keep the initial CL
surface manageable and to allow the Phase 1 overloads to prove out against the
commands test suite first. They should be tracked as follow-on work once Phase 4 lands.
