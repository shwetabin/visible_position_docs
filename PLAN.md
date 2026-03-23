# Plan: Remove VisiblePosition from Editing Commands

## Goal

Remove all unnecessary `VisiblePosition` usage from `editing/commands/` so that
command bodies operate entirely on raw `Position` objects. VP is reserved for the
two boundary points: entry (renderer → command) and exit (command → renderer).

**Reference:** `DESIGN.md` for full rationale, anti-patterns, and body-change details.
**Reference:** `PROBLEMS.md` for the seven concrete problems this fixes.
**Reference:** `APPENDIX.md` for exhaustive per-file, per-line change tables.

---

## Phase 1 — Add Position-based Navigation Overloads

**Nature:** Purely additive. No existing code is changed. Each CL is independent.

### CL 1-A — Trivial wrappers (`visible_units_paragraph.cc`, `visible_units.cc`)

All functions below delegate to an existing `Algorithm<EditingStrategy>(Position)`
template. The public overload is a one-liner calling that template directly.

- [ ] `Position StartOfParagraph(const Position&, EditingBoundaryCrossingRule)`
- [ ] `Position EndOfParagraph(const Position&, EditingBoundaryCrossingRule)`
- [ ] `bool IsStartOfParagraph(const Position&, EditingBoundaryCrossingRule)`
- [ ] `bool IsEndOfParagraph(const Position&, EditingBoundaryCrossingRule)`
- [ ] `bool InSameParagraph(const Position&, const Position&, EditingBoundaryCrossingRule)`
- [ ] `Position StartOfNextParagraph(const Position&)`
- [ ] `Position EndOfDocument(const Position&)`
- [ ] `bool IsStartOfDocument(const Position&)`
- [ ] `bool IsEndOfDocument(const Position&)`

Implementation: `StartOfParagraphAlgorithm<EditingStrategy>` /
`EndOfParagraphAlgorithm<EditingStrategy>` already exist and take `Position`. Call
them directly. Document boundary functions are one-liners using
`Position::LastPositionInNode` / `StartOfDocument`. No `CreateVisiblePosition` anywhere.

### CL 1-B — Non-trivial overloads (`visible_units.cc`)

These require real implementation logic, not just delegation to an existing template.

- [ ] `Position PreviousPositionOf(const Position&, EditingBoundaryCrossingRule)`
- [ ] `Position NextPositionOf(const Position&, EditingBoundaryCrossingRule)` (new
  `Position`-returning overload alongside the existing VP-returning one)
- [ ] `UChar32 CharacterAfter(const Position&)`

Implementation for `PreviousPositionOf`/`NextPositionOf`: call
`PreviousVisuallyDistinctCandidate` / `NextVisuallyDistinctCandidate` directly.
Handle boundary rules via
`AdjustBackward/ForwardPositionToAvoidCrossingEditingBoundaries(...).GetPosition()`.
No `CreateVisiblePosition`.

Implementation for `CharacterAfter`: replicate the character-reading logic from the
VP overload, calling `MostForwardCaretPosition(pos)` directly — no VP construction.

### CL 1-C — Block boundary overloads (`editing_commands_utilities.cc`)

Separate file and separate reviewer context from CL 1-A/1-B.

- [ ] `Position StartOfBlock(const Position&, EditingBoundaryCrossingRule)`
- [ ] `Position EndOfBlock(const Position&, EditingBoundaryCrossingRule)`
- [ ] `bool IsStartOfBlock(const Position&)`
- [ ] `bool IsEndOfBlock(const Position&)`
- [ ] `Node* EnclosingEmptyListItem(const Position&)`

Implementation: check for existing algorithm templates first. If they exist, call
directly. If not, extract DOM traversal logic from VP overload body into a new
template taking `Position` — zero VP inside.

**Phase 1 gate before Phase 2:** All overloads compile, all new unit tests pass.

---

## Phase 2 — Migrate Simple Command Files

One CL per file. Each CL must pass `blink_unittests --gtest_filter="*Edit*"` and
the WPT editing suite before landing.

### CL 2-A — `undo_step.cc`

VP references are in comments only (lines 51, 97). Verify no actual VP types are
constructed. If confirmed: no code change required, close as NOP.

### CL 2-B — `style_commands.cc`

~1 VP ref. Mechanical replacement.

### CL 2-C — `editor_command.cc`

~5 VP refs. Replace `VisiblePosition` locals with `Position`; use `PreviousPositionOf`
/ `NextPositionOf` Phase 1 overloads. See APPENDIX §editor_command.cc.

### CL 2-D — `insert_line_break_command.cc`

~5 VP refs (P1 + P3 examples). Replace `EndingVisibleSelection()` with
`EndingSelection()`; use `IsEndOfParagraph(Position)` overload; remove TODO comment
about VP across mutations. See APPENDIX §insert_line_break_command.cc.

### CL 2-E — `insert_text_command.cc`

~7 VP refs. Replace `VisiblePosition::FirstPositionInNode` / `LastPositionInNode`
with `Position::` variants; replace `EndingVisibleSelection()` with `EndingSelection()`.
See APPENDIX §insert_text_command.cc.

**Phase 2 gate before Phase 3:** All 5 files VP-free, WPT editing suite clean.

---

## Phase 3 — Infrastructure Signature Changes

One CL per method. Each CL changes the signature, migrates the body, and fixes every
caller in the same CL.

### CL 3-A — `MoveParagraph` + `MoveParagraphs`

Change both signatures to take `const Position&`. Migrate body: VP comparisons →
`ComparePositions(Position, Position)`; `PreviousPositionOf` / `NextPositionOf` use
Phase 1 overloads; `EndingVisibleSelection()` → `EndingSelection()`.
Callers: `indent_outdent_command.cc`, `insert_list_command.cc`,
`delete_selection_command.cc`, `apply_block_element_command.cc`,
`format_block_command.cc`. See APPENDIX §MoveParagraphs body changes.

### CL 3-B — `MoveParagraphWithClones`

Change signature to take `const Position&`. Migrate body: `PreviousPositionOf` /
`NextPositionOf` use Phase 1 overloads; `RelocatablePosition` wraps `Position`
directly. See APPENDIX §MoveParagraphWithClones body changes.

### CL 3-C — `CleanupAfterDeletion`

Change `VisiblePosition destination` → `Position destination = Position()`.
Migrate body: `destination.AnchorNode()` directly; `EndingSelection().Start()`;
`IsStartOfParagraph` / `IsEndOfParagraph` use Phase 1 overloads.

### CL 3-D — `ReplaceCollapsibleWhitespaceWithNonBreakingSpaceIfNeeded`

Change param to `const Position&`. Use `CharacterAfter(Position)` from CL 1-C.

**Phase 3 gate before Phase 4:** All infrastructure methods take `Position`. No
caller of any of these methods constructs a VP anymore.

---

## Phase 4 — Heavy Command Migration

One CL per file, ascending VP count order.

### CL 4-A — `insert_paragraph_separator_command.cc` (~19 VP refs)

Category B dominant. Phase 1 overloads cover most sites. Local `static InSameBlock`
helper signature changes to `Position`. See APPENDIX §insert_paragraph_separator.

### CL 4-B — `apply_block_element_command.cc` (~37 VP refs)

Category B + A. Phase 1 overloads + Phase 3 `MoveParagraph` callers. Includes
`SelectionForParagraphIteration` — change to take/return `SelectionInDOMTree`.

### CL 4-C — `indent_outdent_command.cc` (~61 VP refs)

Category A dominant. Phase 3 signature changes already remove the bulk. Remaining:
local VP variables wrapping `Position`s for `MoveParagraph` calls.

### CL 4-D — `delete_selection_command.cc` (~64 VP refs)

Category A + B. Phase 3 + Phase 1 cover most. Remaining: `SetBaseAndExtentDeprecated`
affinity handling (see DESIGN.md Open Questions).

### CL 4-E — `insert_list_command.cc` (~91 VP refs)

Category A + B. Large volume of `MoveParagraph` callers (Phase 3) + paragraph
boundary tests (Phase 1).

### CL 4-F — `replace_selection_command.cc` (~98 VP refs)

Category B dominant. `EndingVisibleSelection()` throughout — replace with
`EndingSelection()` + explicit `UpdateStyleAndLayout` gates already present.
See DESIGN.md P5 section.

### CL 4-G — `composite_edit_command.cc` remaining private methods

`RebalanceWhitespaceOnTextSubstring`, `MoveParagraphContentsToNewBlockIfNecessary`,
`BreakOutOfEmptyListItem`, `BreakOutOfEmptyMailBlockquotedParagraph`.
See APPENDIX for per-method breakdown.

**Phase 4 gate (done):**
```
grep -r "visible_position.h" \
  third_party/blink/renderer/core/editing/commands/*.cc
```
Should return no results (or only files with documented legitimate boundary VP).

---

## Estimated VP Reduction

| Phase | VPs removed |
|-------|------------|
| 2 — simple commands | ~25 |
| 3 — infrastructure signatures | ~80 (cascades to all callers) |
| 4 — heavy commands + composite private methods | ~220 |
| **Total** | **~325 of ~391 (83%)** |

Remaining ~66 are legitimate boundary-in/out VP or the confirmed `CreateVisibleSelection`
at `MoveParagraphs` line 1741 (pending owner answer on canonicality).

---

## Legitimate VP — Do Not Remove

1. `CompositeEditCommand` constructor: `ComputeVisibleSelectionInDOMTreeDeprecated()` — boundary-in.
2. `Apply()`: `IsRichlyEditablePosition(EndingVisibleSelection().Anchor())` — boundary-out.
3. `IndexForVisiblePosition` / `VisiblePositionForIndex` in `editing_utilities.cc` — public API, out of scope.

---

## Open Questions (from DESIGN.md)

1. **`EndingVisibleSelection()` sites with no existing layout gate** — are there any
   in the heavy commands where the VP snap is genuinely load-bearing (not just a type
   adapter)? Owner review needed before CL 4-D/4-E/4-F.

2. **`MoveParagraphs` line 1741** — is `PlainTextRange::CreateRangeForSelection`
   output already canonical? If yes, drop `CreateVisibleSelection`. If no, keep it as
   legitimate boundary-out VP.

3. **`SelectionForParagraphIteration`** — change to `SelectionInDOMTree` in CL 4-B,
   or defer to Phase 4? Affects `apply_block_element_command.cc` and
   `format_block_command.cc`.

---

## Next Steps: Beyond Commands

After Phase 4, migrate these files using the same Phase 1 overloads (no new
foundation work needed):

- `selection_adjuster.cc` — 28 VP refs, P1/P2
- `editing_utilities.cc` — ~12 VP refs in internal helpers
- `serializers/styled_markup_serializer.cc` — 7 VP refs, P1/P2
- `editor.cc` — ~5 VP refs in internal helpers
