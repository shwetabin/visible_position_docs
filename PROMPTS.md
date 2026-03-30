# AI Agent Prompts: VisiblePosition Cleanup

Each section is a self-contained prompt for one CL. Paste the entire section into
an AI coding agent. The agent must be able to complete the CL without further
clarification.

---

## UNIVERSAL HARD RULES — apply to every CL in every phase

1. **Never use `VisiblePosition`, `CreateVisiblePosition`, `CanonicalPositionOf`,
   `.DeepEquivalent()`, or any method that constructs or canonicalizes a VP.**
   `CanonicalPositionOf` calls `UpdateStyleAndLayout` exactly like `CreateVisiblePosition`.
   The replacement for `CreateVisiblePosition(pos).DeepEquivalent()` is `pos` — drop
   the call entirely. No normalization, no substitution.

2. **Use `EndingSelection().Start()` / `.End()` / `.Anchor()` instead of
   `EndingVisibleSelection().VisibleStart().DeepEquivalent()` etc.** `EndingSelection()`
   returns a `SelectionForUndoStep` that stores raw `Position` values — zero cost, no
   layout. `EndingVisibleSelection()` unconditionally calls `UpdateStyleAndLayout`.

3. **Do not introduce `VisiblePosition` anywhere** — not in helpers, not in local
   variables, not as temporary bridges.

4. Each new overload (Phase 1) must have a unit test in the corresponding `*_test.cc`.

---

## LEGITIMATE VP — DO NOT TOUCH IN ANY CL

1. **Entry boundary** — `CompositeEditCommand` constructor calling
   `frame->Selection().ComputeVisibleSelectionInDOMTreeDeprecated()`.
2. **Exit boundary** — `EndingVisibleSelection()` in `Apply()` for the
   `IsRichlyEditablePosition` check.
3. **`IndexForVisiblePosition` / `VisiblePositionForIndex` / `MakeRange(VP, VP)`
   in `editing_utilities.cc`** — public API. Out of scope.

---

## PHASE 1 — Add Position-based Navigation Overloads

**Nature:** Purely additive. No existing code is changed (except relocating existing
`Position` overloads from `visible_units.h` into the new file in CL 1-A).

Working directory: `third_party/blink/renderer/core/editing/`

---

### CL 1-A — New file + trivial wrappers

**Create:** `visible_units_position.h`, `visible_units_position.cc`

This CL establishes the new home for all `Position`-based navigation. No
`VisiblePosition` types appear anywhere in these two files.

**Step 1 — Relocate existing `Position` overloads from `visible_units.h`:**

Move the declarations and definitions of the following from `visible_units.h` /
`visible_units.cc` into `visible_units_position.h` / `visible_units_position.cc`,
and remove them from the originals:

- `StartOfWordPosition(const Position&)`
- `EndOfWordPosition(const Position&)`
- `StartOfLine(const PositionWithAffinity&)` → `PositionWithAffinity`
- `EndOfLine(const PositionWithAffinity&)` → `PositionWithAffinity`
- `InSameLine(const PositionWithAffinity&, const PositionWithAffinity&)`
- `StartOfDocument(const Position&)` → `Position`

Update every file that included `visible_units.h` for these functions to include
`visible_units_position.h` instead (or in addition, if they also use VP functions).

**Step 2 — Add new trivial `Position` overloads in `visible_units_position.cc`:**

```cpp
Position StartOfParagraph(const Position&,
                          EditingBoundaryCrossingRule = kCannotCrossEditingBoundary);
Position EndOfParagraph(const Position&,
                        EditingBoundaryCrossingRule = kCannotCrossEditingBoundary);
bool IsStartOfParagraph(const Position&,
                        EditingBoundaryCrossingRule = kCannotCrossEditingBoundary);
bool IsEndOfParagraph(const Position&,
                      EditingBoundaryCrossingRule = kCannotCrossEditingBoundary);
bool InSameParagraph(const Position&, const Position&,
                     EditingBoundaryCrossingRule = kCannotCrossEditingBoundary);
Position StartOfNextParagraph(const Position&);
Position EndOfDocument(const Position&);
bool IsStartOfDocument(const Position&);
bool IsEndOfDocument(const Position&);
```

**Implementation:**
`StartOfParagraphAlgorithm<EditingStrategy>` and `EndOfParagraphAlgorithm<EditingStrategy>`
already exist in `visible_units_paragraph.cc` taking `PositionTemplate<Strategy>`.
Call them directly:

```cpp
Position StartOfParagraph(const Position& pos,
                          EditingBoundaryCrossingRule rule) {
  return StartOfParagraphAlgorithm<EditingStrategy>(pos, rule);
}
bool IsStartOfParagraph(const Position& pos,
                        EditingBoundaryCrossingRule rule) {
  return pos == StartOfParagraphAlgorithm<EditingStrategy>(pos, rule);
}
```

Apply the same pattern for `EndOfParagraph`, `IsEndOfParagraph`, `InSameParagraph`,
`StartOfNextParagraph`.

`EndOfDocument(Position)`: `return Position::LastPositionInNode(*pos.GetDocument());`
`IsStartOfDocument(Position)`: `return pos == StartOfDocument(pos);`
`IsEndOfDocument(Position)`: `return pos == EndOfDocument(pos);`

No `CreateVisiblePosition` anywhere.

**Verification:**
```
autoninja -C out/Default blink_unittests
./out/Default/blink_unittests --gtest_filter="*Paragraph*:*Document*"
```

---

### CL 1-B — Non-trivial overloads

**Files:** `visible_units_position.cc`, `visible_units_position.h`
(do NOT add to `visible_units.cc` / `visible_units.h`)

**Functions to add:**

```cpp
// Do NOT change existing VP-returning signatures in visible_units.h.
// Add new Position-returning overloads in visible_units_position.h only.
Position PreviousPositionOf(const Position&,
                            EditingBoundaryCrossingRule = kCanCrossEditingBoundary);
Position NextPositionOf(const Position&,
                        EditingBoundaryCrossingRule = kCanCrossEditingBoundary);
UChar32 CharacterAfter(const Position&);

// Line boundary — no Position overloads exist today.
bool IsStartOfLine(const PositionWithAffinity&);
bool IsStartOfLine(const Position&);
bool IsEndOfLine(const PositionWithAffinity&);
bool IsEndOfLine(const Position&);
```

**Implementation for `PreviousPositionOf` / `NextPositionOf`:**
`PreviousVisuallyDistinctCandidateAlgorithm` and `NextVisuallyDistinctCandidateAlgorithm`
already take and return `Position` with no layout forcing. The only layout cost in the
existing VP overloads is the `CreateVisiblePosition` call used to re-canonicalize —
that must not appear here.

```cpp
Position PreviousPositionOf(const Position& pos,
                            EditingBoundaryCrossingRule rule) {
  const Position prev = PreviousVisuallyDistinctCandidate(pos);
  if (prev.IsNull())
    return Position();
  if (rule == kCannotCrossEditingBoundary) {
    return AdjustBackwardPositionToAvoidCrossingEditingBoundaries(
               PositionWithAffinity(prev), pos)
        .GetPosition();
  }
  return prev;
}
```

Apply the same pattern for `NextPositionOf`. No `CreateVisiblePosition` anywhere.

**Implementation for `CharacterAfter(Position)`:**
Replicate the character-reading logic from the VP overload. Call
`MostForwardCaretPosition(pos)` directly to get the forward position, then read the
character — no VP construction.

**Implementation for `IsStartOfLine` / `IsEndOfLine`:**
`StartOfLine(PositionWithAffinity)` and `EndOfLine(PositionWithAffinity)` already
exist (relocated from `visible_units.h` in CL 1-A). Delegate to them:

```cpp
bool IsStartOfLine(const PositionWithAffinity& pos) {
  return pos == StartOfLine(pos);
}
bool IsStartOfLine(const Position& pos) {
  return IsStartOfLine(PositionWithAffinity(pos));
}
bool IsEndOfLine(const PositionWithAffinity& pos) {
  return pos == EndOfLine(pos);
}
bool IsEndOfLine(const Position& pos) {
  return IsEndOfLine(PositionWithAffinity(pos));
}
```

The one existing VP call site this fixes is `editing_commands_utilities.cc:301`:
```cpp
// Before:
IsStartOfLine(CreateVisiblePosition(position, affinity))
// After:
IsStartOfLine(PositionWithAffinity(position, affinity))
```

`LogicalStartOfLine`, `LogicalEndOfLine`, `IsLogicalEndOfLine` — no `Position`
overloads in this CL. Zero VP call sites in `commands/`; defer to follow-on work.

**Verification:**
```
autoninja -C out/Default blink_unittests
./out/Default/blink_unittests --gtest_filter="*Edit*:*Character*:*Line*"
```

---

### CL 1-C — Block boundary overloads

**Files:** `editing_commands_utilities.cc`, `editing_commands_utilities.h`

**Functions to add:**

```cpp
Position StartOfBlock(const Position&,
                      EditingBoundaryCrossingRule = kCannotCrossEditingBoundary);
Position EndOfBlock(const Position&,
                    EditingBoundaryCrossingRule = kCannotCrossEditingBoundary);
bool IsStartOfBlock(const Position&);
bool IsEndOfBlock(const Position&);
Node* EnclosingEmptyListItem(const Position&);
```

**Implementation:**
Check for existing algorithm templates in `editing_commands_utilities.cc` first. If
they exist, call directly. If not, read the VP overload body — if it uses only
structural DOM queries (no layout), replicate it with `Position` parameters. If it
internally calls VP-returning functions, replace those with the CL 1-A/1-B overloads.

`IsStartOfBlock(Position)`: `return pos == StartOfBlock(pos);`
`IsEndOfBlock(Position)`: `return pos == EndOfBlock(pos);`

**Verification:**
```
autoninja -C out/Default blink_unittests
./out/Default/blink_unittests --gtest_filter="*Block*:*Edit*"
```

**Phase 1 gate:** All three CLs compile, all new unit tests pass:
```
./out/Default/blink_unittests --gtest_filter="*Paragraph*:*Document*:*Block*:*Edit*:*Line*"
```

**Include rule after Phase 1:** Command files (and any other callers) must include
`visible_units_position.h` for `Position`-typed queries and `visible_units.h` only
when VP-returning functions are still needed. A file that has completed VP cleanup
should include `visible_units_position.h` only and have no `visible_units.h` include.

---

## PHASE 2 — Migrate Simple Command Files

**Phase 1 must be complete before starting Phase 2.**

Working directory: `third_party/blink/renderer/core/editing/commands/`

### Replacement patterns (applies to all CL 2-x)

| Before | After |
|--------|-------|
| `CreateVisiblePosition(pos).DeepEquivalent()` | `pos` — drop entirely |
| `IsEndOfParagraph(CreateVisiblePosition(pos))` | `IsEndOfParagraph(pos)` |
| `IsStartOfParagraph(CreateVisiblePosition(pos))` | `IsStartOfParagraph(pos)` |
| `StartOfParagraph(vp).DeepEquivalent()` | `StartOfParagraph(pos)` |
| `EndOfParagraph(vp).DeepEquivalent()` | `EndOfParagraph(pos)` |
| `VisiblePosition::FirstPositionInNode(*n).DeepEquivalent()` | `Position::FirstPositionInNode(*n)` |
| `VisiblePosition::LastPositionInNode(*n).DeepEquivalent()` | `Position::LastPositionInNode(*n)` |
| `VisiblePosition::BeforeNode(*n)` | `Position::BeforeNode(*n)` |
| `LineBreakExistsAtVisiblePosition(vp)` | `LineBreakExistsAtPosition(pos)` |
| `EndingVisibleSelection().VisibleStart().DeepEquivalent()` | `EndingSelection().Start()` |
| `EndingVisibleSelection().Start()` | `EndingSelection().Start()` |
| `EndingVisibleSelection().VisibleEnd().DeepEquivalent()` | `EndingSelection().End()` |

### Goal per CL

After your changes, the file must not include `visible_position.h` and must not use
`VisiblePosition`, `CreateVisiblePosition`, `CanonicalPositionOf`, or `.DeepEquivalent()`.

### Verification per CL (run before moving to next file)

```
autoninja -C out/Default blink_unittests
./out/Default/blink_unittests --gtest_filter="*Edit*"
python3 third_party/blink/tools/run_web_tests.py editing/
```

---

### CL 2-A — `undo_step.cc`

**File:** `third_party/blink/renderer/core/editing/commands/undo_step.cc`

VP references are in comments only (lines 51, 97). Verify no VP types are actually
constructed anywhere in the file. If confirmed: no code change required, close as NOP.

---

### CL 2-B — `style_commands.cc`

**File:** `third_party/blink/renderer/core/editing/commands/style_commands.cc`

Approximately 1 VP reference. Apply the replacement table above mechanically.
Remove `#include "visible_position.h"` after all VP usage is gone.

---

### CL 2-C — `editor_command.cc`

**File:** `third_party/blink/renderer/core/editing/commands/editor_command.cc`

Approximately 5 VP references. Replace `VisiblePosition` locals with `Position`.
Use `PreviousPositionOf(pos)` / `NextPositionOf(pos)` from CL 1-B. See APPENDIX
§editor_command.cc for line-by-line table.

---

### CL 2-D — `insert_line_break_command.cc`

**File:** `third_party/blink/renderer/core/editing/commands/insert_line_break_command.cc`

Approximately 5 VP references. Replace `EndingVisibleSelection()` with
`EndingSelection()`. Use `IsEndOfParagraph(Position)` from CL 1-A. Remove the TODO
comment about VP across mutations (the bug it describes no longer applies once VP
is gone). See APPENDIX §insert_line_break_command.cc for line-by-line table.

---

### CL 2-E — `insert_text_command.cc`

**File:** `third_party/blink/renderer/core/editing/commands/insert_text_command.cc`

Approximately 7 VP references. Replace `VisiblePosition::FirstPositionInNode` /
`LastPositionInNode` with `Position::` variants. Replace `EndingVisibleSelection()`
with `EndingSelection()`. See APPENDIX §insert_text_command.cc for line-by-line table.

**Phase 2 gate before Phase 3:** All 5 files VP-free, WPT editing suite clean.

---

## PHASE 3 — Infrastructure Signature Changes

**Phases 1 and 2 must be complete before starting Phase 3.**

**Files:**
- `third_party/blink/renderer/core/editing/commands/composite_edit_command.h`
- `third_party/blink/renderer/core/editing/commands/composite_edit_command.cc`

**Rule: Fix every caller in the same CL.** After changing a method signature,
callers will not compile. All callers must be updated in the same CL. Do not leave
broken callers.

**Rule: No VP bridges.** If an internal call inside the method body requires VP
today, use the Phase 1 `Position` overload instead. If a `Position` overload is
missing, add it to Phase 1 first.

---

### CL 3-A — `MoveParagraph` + `MoveParagraphs`

**New signatures:**

```cpp
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
```

**Body migration for `MoveParagraphs`:**
- VP comparison: `destination >= start && destination <= end` →
  `ComparePositions(destination, start) >= 0 && ComparePositions(destination, end) <= 0`
- `PreviousPositionOf(start_of_paragraph_to_move, kCannotCrossEditingBoundary).DeepEquivalent()` →
  `PreviousPositionOf(start_of_paragraph_to_move, kCannotCrossEditingBoundary)` (CL 1-B overload)
- `EndingVisibleSelection().VisibleStart()` / `.VisibleEnd()` →
  `EndingSelection().Start()` / `.End()`

**Callers to fix:** `indent_outdent_command.cc`, `insert_list_command.cc`,
`delete_selection_command.cc`, `apply_block_element_command.cc`,
`format_block_command.cc`. Each caller wraps raw `Position` values in
`CreateVisiblePosition` solely to satisfy the old signature — delete those wrappers
and pass the `Position` directly. See APPENDIX §MoveParagraphs body changes.

**Verification:**
```
autoninja -C out/Default blink_unittests
./out/Default/blink_unittests --gtest_filter="*Edit*"
python3 third_party/blink/tools/run_web_tests.py editing/
```

---

### CL 3-B — `MoveParagraphWithClones`

**New signature:**

```cpp
void MoveParagraphWithClones(const Position& start,
                             const Position& end,
                             HTMLElement* block_element,
                             Node* outer_node,
                             EditingState*);
```

**Body migration:**
- `PreviousPositionOf` / `NextPositionOf` calls: use CL 1-B `Position` overloads.
- `RelocatablePosition` already wraps `Position` — drop the VP re-wrap step. Positions
  extracted via `RelocatablePosition::GetPosition()` are used directly.

See APPENDIX §MoveParagraphWithClones body changes.

**Verification:** same as CL 3-A.

---

### CL 3-C — `CleanupAfterDeletion`

**New signature:**

```cpp
void CleanupAfterDeletion(EditingState*, Position destination = Position());
```

**Body migration:**
- `destination.DeepEquivalent().AnchorNode()` → `destination.AnchorNode()`
- `EndingVisibleSelection().VisibleStart()` → `EndingSelection().Start()`
- `IsStartOfParagraph` / `IsEndOfParagraph` → CL 1-A `Position` overloads

**Verification:** same as CL 3-A.

---

### CL 3-D — `ReplaceCollapsibleWhitespaceWithNonBreakingSpaceIfNeeded`

**New signature:**

```cpp
void ReplaceCollapsibleWhitespaceWithNonBreakingSpaceIfNeeded(const Position&);
```

**Body migration:**
- `CharacterAfter(position)` → CL 1-B `Position` overload
- `MostForwardCaretPosition(position)` already takes `Position` — no change needed

**Verification:** same as CL 3-A.

**Phase 3 gate before Phase 4:** All infrastructure methods take `Position`. No caller constructs VP
to call any of these methods.

---

## PHASE 4 — Heavy Command Migration

**Phases 1–3 must be complete before starting Phase 4.**

### VP category reference (applies to all CL 4-x)

- **Category A** — VP constructed only to pass to `MoveParagraph` /
  `MoveParagraphWithClones` / `CleanupAfterDeletion`. Phase 3 signature changes mean:
  delete the `CreateVisiblePosition` wrapper, pass the raw `Position` directly.

- **Category B** — VP constructed only to call `IsEndOfParagraph`, `StartOfParagraph`,
  etc. Replace with Phase 1 `Position` overloads.

- **`VisiblePosition::FirstPositionInNode` / `LastPositionInNode` equality tests:**
  replace with `Position::FirstPositionInNode` / `Position::LastPositionInNode` only
  after confirming positions are already canonical at that site. If uncertain, add
  `// TODO(vp-cleanup): verify canonicality`.

### Goal per CL

After your changes the file must not include `visible_position.h` and must not use
`VisiblePosition`, `CreateVisiblePosition`, `CanonicalPositionOf`, or `.DeepEquivalent()`.

### Verification per CL (run before moving to next file)

```
autoninja -C out/Default blink_unittests
./out/Default/blink_unittests --gtest_filter="*Edit*"
python3 third_party/blink/tools/run_web_tests.py editing/
python3 third_party/blink/tools/run_web_tests.py editing/contenteditable/
```

---

### CL 4-A — `insert_paragraph_separator_command.cc`

**~19 VP refs. Category B dominant.**

File: `third_party/blink/renderer/core/editing/commands/insert_paragraph_separator_command.cc`

All paragraph-boundary tests use VP as a type adapter for `IsEndOfParagraph` /
`StartOfParagraph`. Replace with CL 1-A `Position` overloads. The local
`static InSameBlock` helper should change its parameter to `Position`. See APPENDIX
§insert_paragraph_separator.

---

### CL 4-B — `apply_block_element_command.cc`

**~37 VP refs. Category B + A.**

File: `third_party/blink/renderer/core/editing/commands/apply_block_element_command.cc`

Category B: paragraph boundary tests → CL 1-A overloads.
Category A: `MoveParagraph` callers → delete VP wrappers per CL 3-A.
`SelectionForParagraphIteration`: change to take/return `SelectionInDOMTree` (no VP).
See APPENDIX §apply_block_element_command.

---

### CL 4-C — `indent_outdent_command.cc`

**~61 VP refs. Category A dominant.**

File: `third_party/blink/renderer/core/editing/commands/indent_outdent_command.cc`

The bulk of VP usage is local variables constructed only to pass to `MoveParagraph`.
CL 3-A signature changes make these trivial: delete each `CreateVisiblePosition`
wrapper. Remaining Category B sites: CL 1-A overloads.
See APPENDIX §indent_outdent_command.

---

### CL 4-D — `delete_selection_command.cc`

**~64 VP refs. Category A + B.**

File: `third_party/blink/renderer/core/editing/commands/delete_selection_command.cc`

Category A: `MoveParagraph` / `CleanupAfterDeletion` callers → CL 3-A / 3-C.
Category B: paragraph/block boundary tests → CL 1-A / 1-C overloads.
`SetBaseAndExtentDeprecated` affinity handling: check DESIGN.md Open Questions before
touching. See APPENDIX §delete_selection_command.

---

### CL 4-E — `insert_list_command.cc`

**~91 VP refs. Category A + B.**

File: `third_party/blink/renderer/core/editing/commands/insert_list_command.cc`

Large volume of `MoveParagraph` callers (CL 3-A) and paragraph boundary tests (CL 1-A).
Remove the TODO comment about VP across mutations at line 763 — the bug it describes
no longer applies once VP is gone. See APPENDIX §insert_list_command.

---

### CL 4-F — `replace_selection_command.cc`

**~98 VP refs. Category B dominant.**

File: `third_party/blink/renderer/core/editing/commands/replace_selection_command.cc`

`EndingVisibleSelection()` appears throughout — replace with `EndingSelection()`.
Explicit `UpdateStyleAndLayout` gates are already present in this file as structural
query guards; they remain. Only the VP type adapters are removed. Remove the TODO
comment about VP across mutations at line 1077. See APPENDIX §replace_selection_command.

---

### CL 4-G — `composite_edit_command.cc` remaining private methods

**File:** `third_party/blink/renderer/core/editing/commands/composite_edit_command.cc`

**Methods to migrate:**
- `RebalanceWhitespaceOnTextSubstring`
- `MoveParagraphContentsToNewBlockIfNecessary`
- `BreakOutOfEmptyListItem`
- `BreakOutOfEmptyMailBlockquotedParagraph`

For each method: apply the same Category A / B patterns. Use CL 1-A through 1-C
overloads for all navigation calls. See APPENDIX §composite_edit_command private methods.

**Phase 4 gate (done):**
```
grep -r "visible_position.h" \
  third_party/blink/renderer/core/editing/commands/*.cc
```
Must return no results (or only files with documented legitimate boundary VP).
