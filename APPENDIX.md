# Appendix: Methods to Change Per Phase

Exhaustive per-file, per-method listing of every change needed. Cross-referenced
with line numbers in the current tree so owners can verify each site before and after.

---

## Phase 1 — Add Position-based Overloads

These are **new functions only** — no existing code is modified.

### `visible_units_paragraph.cc` / `visible_units.h`

| Function to add | Algorithm template already exists? |
|---|---|
| `Position StartOfParagraph(const Position&, EditingBoundaryCrossingRule)` | Yes — `StartOfParagraphAlgorithm<EditingStrategy>(Position, rule)` |
| `Position EndOfParagraph(const Position&, EditingBoundaryCrossingRule)` | Yes — `EndOfParagraphAlgorithm<EditingStrategy>` |
| `bool IsStartOfParagraph(const Position&, EditingBoundaryCrossingRule)` | Yes — delegate to `StartOfParagraphAlgorithm` |
| `bool IsEndOfParagraph(const Position&, EditingBoundaryCrossingRule)` | Yes — delegate to `EndOfParagraphAlgorithm` |
| `Position StartOfNextParagraph(const Position&)` | Check — likely wraps `StartOfParagraph(NextPositionOf(...))` |
| `bool InSameParagraph(const Position&, const Position&, EditingBoundaryCrossingRule)` | Trivial — `StartOfParagraph(a) == StartOfParagraph(b)` |

### `visible_units.cc` / `visible_units.h`

| Function to add | Algorithm template already exists? |
|---|---|
| `Position EndOfDocument(const Position&)` | Check — `EndOfDocumentAlgorithm` likely exists |
| `bool IsStartOfDocument(const Position&)` | Trivial — `pos == StartOfDocument(pos)` |
| `bool IsEndOfDocument(const Position&)` | Trivial — `pos == EndOfDocument(pos)` |
| `UChar32 CharacterAfter(const Position&)` | No VP-free algorithm — write one or bridge internally with TODO |

### `editing_commands_utilities.cc` / `editing_commands_utilities.h`

| Function to add | Notes |
|---|---|
| `Position StartOfBlock(const Position&, EditingBoundaryCrossingRule)` | Check for existing `StartOfBlockAlgorithm` |
| `Position EndOfBlock(const Position&, EditingBoundaryCrossingRule)` | Check for existing `EndOfBlockAlgorithm` |
| `bool IsStartOfBlock(const Position&)` | Trivial — `pos == StartOfBlock(pos)` |
| `bool IsEndOfBlock(const Position&)` | Trivial — `pos == EndOfBlock(pos)` |
| `Node* EnclosingEmptyListItem(const Position&)` | Currently takes `VisiblePosition` — add `Position` overload |

---

## Phase 2 — Simple Command Files

### `undo_step.cc` (2 refs — comments only, no code changes needed)

Both VP references at lines 51 and 97 are inside comments explaining why
`UpdateStyleAndLayout` is called. Verify no actual VP types are constructed; if
confirmed, this file requires no changes.

---

### `editor_command.cc` (5 refs)

| Location | Current code | Change to |
|---|---|---|
| Line 265 | `const VisiblePosition& caret = selection.VisibleStart()` | `Position caret = selection.Start()` |
| Line 266 | `const VisiblePosition& next = NextPositionOf(...)` | `Position next = NextPositionOf(caret, ...)` using `Position` overload |
| Line 268 | `const VisiblePosition& previous = PreviousPositionOf(next)` | `Position previous = PreviousPositionOf(next, ...)` |
| Line 269 | `next.DeepEquivalent() == previous.DeepEquivalent()` | `next == previous` |
| Line 272 | `const VisiblePosition& previous_of_previous = PreviousPositionOf(previous)` | `Position previous_of_previous = PreviousPositionOf(previous, ...)` |
| Line 1228 | `CreateVisiblePosition(selection.Anchor()).DeepEquivalent()` | `selection.Anchor()` |

---

### `insert_line_break_command.cc` (5 refs)

| Location | Current code | Change to |
|---|---|---|
| Line 84 | `VisibleSelection selection = EndingVisibleSelection()` | `SelectionInDOMTree selection = EndingSelection()` |
| Line 91 | `VisiblePosition caret(selection.VisibleStart())` | `Position caret = selection.Start()` |
| Line 98 | `Position pos(caret.DeepEquivalent())` | `Position pos = caret` (already a `Position`) |
| Line 116 | `IsEndOfParagraph(CreateVisiblePosition(caret.ToPositionWithAffinity()))` | `IsEndOfParagraph(caret)` (Phase 1 overload) |
| Line 117 | `LineBreakExistsAtVisiblePosition(caret)` | `LineBreakExistsAtPosition(caret)` |
| Line 154 | `IsStartOfParagraph(VisiblePosition::BeforeNode(*node_to_insert))` | `IsStartOfParagraph(Position::BeforeNode(*node_to_insert))` |
| Line 240 | `.Collapse(EndingVisibleSelection().End())` | `.Collapse(EndingSelection().End())` |

---

### `insert_text_command.cc` (7 refs)

| Location | Current code | Change to |
|---|---|---|
| Line 116 | `EndingVisibleSelection().Start()` | `EndingSelection().Start()` |
| Line 119–122 | `VisiblePosition first_in_anchor = VisiblePosition::FirstPositionInNode(*enclosing_anchor)` / `last_in_anchor` | `Position::FirstPositionInNode(*enclosing_anchor)` / `Position::LastPositionInNode(...)` |
| Line 123 | `EndingVisibleSelection().End()` | `EndingSelection().End()` |
| Line 124–125 | `first_in_anchor.DeepEquivalent() == start && last_in_anchor.DeepEquivalent() == end` | `first_in_anchor == start && last_in_anchor == end` |
| Line 140 | `.Collapse(EndingVisibleSelection().End())` | `.Collapse(EndingSelection().End())` |
| Line 150 | `const VisibleSelection& visible_selection = EndingVisibleSelection()` | `const SelectionInDOMTree& selection = EndingSelection()` |
| Line 162 | `IsStartOfBlock(EndingVisibleSelection().VisibleEnd())` | `IsStartOfBlock(EndingSelection().End())` (Phase 1 overload) |
| Line 191/194/283 | `EndingVisibleSelection().*` | `EndingSelection().*` |
| Line 299 | `CreateVisiblePosition(pos).DeepEquivalent()` | `pos` |

---

## Phase 3 — Infrastructure Signature Changes

All changes are in `composite_edit_command.h` and `composite_edit_command.cc`.

### Methods whose signatures change

| Method | Old parameter types | New parameter types |
|---|---|---|
| `MoveParagraph` | `const VisiblePosition& start, end, destination` | `const Position& start, end, destination` |
| `MoveParagraphs` | `const VisiblePosition& start, end, destination` | `const Position& start, end, destination` |
| `MoveParagraphWithClones` | `const VisiblePosition& start, end` | `const Position& start, end` |
| `CleanupAfterDeletion` | `VisiblePosition destination` | `Position destination = Position()` |
| `ReplaceCollapsibleWhitespaceWithNonBreakingSpaceIfNeeded` | `const VisiblePosition&` | `const Position&` |

---

### `composite_edit_command.cc` — body changes per method

#### `ReplaceCollapsibleWhitespaceWithNonBreakingSpaceIfNeeded` (line 878)

| Location | Current code | Change to |
|---|---|---|
| Line 878 | `const VisiblePosition& visible_position` param | `const Position& position` |
| Line 881 | `MostForwardCaretPosition(visible_position.DeepEquivalent())` | `MostForwardCaretPosition(position)` |
| Implicit | `CharacterAfter(visible_position)` | `CharacterAfter(position)` (Phase 1 overload) |

#### `RebalanceWhitespaceOnTextSubstring` (lines 812–876)

| Location | Current code | Change to |
|---|---|---|
| Line 812–813 | `VisiblePosition visible_upstream_pos = CreateVisiblePosition(Position(text_node, upstream))` | `Position visible_upstream_pos(text_node, upstream)` |
| Line 814–815 | `VisiblePosition visible_downstream_pos = CreateVisiblePosition(Position(text_node, downstream))` | `Position visible_downstream_pos(text_node, downstream)` |
| Line 866–867 | `VisiblePosition visible_pos = CreateVisiblePosition(position)` / `VisiblePosition previous_visible_pos = PreviousPositionOf(visible_pos)` | `Position visible_pos = position` / `Position previous = PreviousPositionOf(position, ...)` |
| Line 873 | `CreateVisiblePosition(position)` passed to some function | Use `Position` overload |

#### `CleanupAfterDeletion` (lines 1307–1345)

| Location | Current code | Change to |
|---|---|---|
| Line 1307 | `CleanupAfterDeletion(editing_state, VisiblePosition())` call site | `CleanupAfterDeletion(editing_state)` or `CleanupAfterDeletion(editing_state, Position())` |
| Line 1311 | `VisiblePosition destination` param | `Position destination` |
| Line 1314 | `VisiblePosition caret_after_delete = EndingVisibleSelection().VisibleStart()` | `Position caret_after_delete = EndingSelection().Start()` |
| Line 1315 | `destination.DeepEquivalent().AnchorNode()` | `destination.AnchorNode()` |
| Line 1316 | `caret_after_delete.DeepEquivalent() != destination.DeepEquivalent()` | `caret_after_delete != destination` |
| Line 1321 | `MostForwardCaretPosition(caret_after_delete.DeepEquivalent())` | `MostForwardCaretPosition(caret_after_delete)` |
| Line 1341 | `RendersInDifferentPosition(position, destination.DeepEquivalent())` | `RendersInDifferentPosition(position, destination)` |
| Implicit | `IsStartOfParagraph(caret_after_delete)` / `IsEndOfParagraph(...)` | Use Phase 1 `Position` overloads |

#### `MoveParagraphWithClones` (lines 1367–1441)

| Location | Current code | Change to |
|---|---|---|
| Line 1367–1368 | `const VisiblePosition& start_of_paragraph_to_move, end_of_paragraph_to_move` | `const Position& start, const Position& end` |
| Line 1380 | `PreviousPositionOf(start_of_paragraph_to_move).DeepEquivalent()` | `PreviousPositionOf(start, kCannotCrossEditingBoundary)` — Phase 1 `Position`-returning overload |
| Line 1383 | `NextPositionOf(end_of_paragraph_to_move).DeepEquivalent()` | `NextPositionOf(end, kCannotCrossEditingBoundary)` — Phase 1 `Position`-returning overload |
| Line 1389–1394 | `start_of_paragraph_to_move.DeepEquivalent()` / `end_of_paragraph_to_move.DeepEquivalent()` | `start` / `end` directly |
| Lines 1427–1436 | `CreateVisiblePosition(relocatable_before_paragraph->GetPosition())` / same for after | `relocatable_before_paragraph->GetPosition()` — use `Position` comparisons directly |

#### `MoveParagraphs` (lines 1463–1665)

| Location | Current code | Change to |
|---|---|---|
| Line 1463–1465 | `const VisiblePosition& start_of_paragraph_to_move, end_of_paragraph_to_move, destination` | `const Position& start, end, destination` |
| Lines 1475–1476 | `start_of_paragraph_to_move.DeepEquivalent() == destination.DeepEquivalent()` | `start == destination` |
| Lines 1481–1484 | `destination.DeepEquivalent() >= start_of_paragraph_to_move.DeepEquivalent() && ...` | `ComparePositions(destination, start) >= 0 && ComparePositions(destination, end) <= 0` |
| Lines 1496–1497 | `VisiblePosition visible_start = EndingVisibleSelection().VisibleStart()` / `visible_end` | `Position visible_start = EndingSelection().Start()` / `.End()` |
| Lines 1541–1549 | `PreviousPositionOf(start_of_paragraph_to_move, kCannotCrossEditingBoundary).DeepEquivalent()` / `NextPositionOf(end...).DeepEquivalent()` | `PreviousPositionOf(start, kCannotCrossEditingBoundary)` / `NextPositionOf(end, kCannotCrossEditingBoundary)` — Phase 1 `Position`-returning overloads; no VP, no layout forced |
| Lines 1551–1552 | `start_of_paragraph_to_move.DeepEquivalent()` / `end...` | `start` / `end` |
| Lines 1573–1574 | `start_of_paragraph_to_move.DeepEquivalent() != end_of_paragraph_to_move.DeepEquivalent()` | `start != end` |
| Lines 1590–1591 | same pattern | same |
| Lines 1651–1662 | `CreateVisiblePosition(before_paragraph_position->GetPosition())` / after | Use `Position` values directly from `RelocatablePosition::GetPosition()`; drop VP re-wrap entirely |
| Line 1711–1712 | `IsStartOfParagraph(EndingVisibleSelection().VisibleStart())` / `IsEndOfParagraph(...)` | `IsStartOfParagraph(EndingSelection().Start())` — Phase 1 overloads |
| Line 1755 | `EnclosingEmptyListItem(EndingVisibleSelection().VisibleStart())` | `EnclosingEmptyListItem(EndingSelection().Start())` — Phase 1 overload |
| Lines 1777–1780 | `CreateVisiblePosition(PositionAfterNode(*block_enclosing_list)).DeepEquivalent() == CreateVisiblePosition(PositionAfterNode(*list_node)).DeepEquivalent()` | `PositionAfterNode(*block_enclosing_list) == PositionAfterNode(*list_node)` |
| Lines 2130–2131 / 2138–2139 | `CreateVisiblePosition(...).DeepEquivalent() == CreateVisiblePosition(...).DeepEquivalent()` | Direct `Position` equality |

#### `BreakOutOfEmptyMailBlockquotedParagraph` (lines 1886–1935)

| Location | Current code | Change to |
|---|---|---|
| Line 1886 | `VisiblePosition caret = EndingVisibleSelection().VisibleStart()` | `Position caret = EndingSelection().Start()` |
| Line 1888 | `caret.DeepEquivalent()` in `EnclosingNodeOfType(...)` | `caret` directly |
| Lines 1895–1898 | `VisiblePosition previous = PreviousPositionOf(...)` | `Position previous = PreviousPositionOf(caret, ...)` |
| Line 1899 | `EnclosingNodeOfType(previous.DeepEquivalent(), ...)` | `EnclosingNodeOfType(previous, ...)` |
| Line 1912 | `VisiblePosition at_br = VisiblePosition::BeforeNode(*br)` | `Position at_br = Position::BeforeNode(*br)` |
| Line 1929 | `LineBreakExistsAtVisiblePosition(caret)` | `LineBreakExistsAtPosition(caret)` |
| Line 1932 | `MostForwardCaretPosition(caret.DeepEquivalent())` | `MostForwardCaretPosition(caret)` |

#### `NextPositionOf` local in `composite_edit_command.cc` (line 974)

| Location | Current code | Change to |
|---|---|---|
| Line 974 | `NextPositionOf(CreateVisiblePosition(pos)).DeepEquivalent()` | `NextPositionOf(pos, ...)` using `Position` overload |

#### `MoveParagraphContentsToNewBlockIfNecessary` (lines 1085–1168)

| Location | Current code | Change to |
|---|---|---|
| Line 1085 | `VisiblePosition visible_pos = CreateVisiblePosition(pos)` | `Position visible_pos = pos` (rename to `pos` for clarity) |
| Lines 1090–1093 | `VisiblePosition visible_paragraph_start = StartOfParagraph(visible_pos)` / `end` / `next` / `visible_end` | `Position paragraph_start = StartOfParagraph(pos)` / `EndOfParagraph(pos)` / `NextPositionOf(paragraph_end)` — Phase 1 overloads |
| Lines 1096–1098 | `MostBackwardCaretPosition(visible_paragraph_start.DeepEquivalent())` / `visible_end.DeepEquivalent()` | `MostBackwardCaretPosition(paragraph_start)` / `most_backward_end` |
| Line 1146 | `visible_paragraph_end.DeepEquivalent().AnchorNode()` | `paragraph_end.AnchorNode()` |
| Lines 1151–1152 | `VisiblePosition::FirstPositionInNode(*new_block)` used as destination | `Position::FirstPositionInNode(*new_block)` |
| Lines 1167–1168 | `DCHECK_LE(visible_paragraph_start.DeepEquivalent(), visible_paragraph_end.DeepEquivalent())` | `DCHECK_LE(paragraph_start, paragraph_end)` |

---

## Phase 4 — Heavy Command Files

### `insert_paragraph_separator_command.cc` (15 refs — Category B dominant)

| Location | Current code | Change to |
|---|---|---|
| Line 130–135 | `static bool InSameBlock(const VisiblePosition& a, const VisiblePosition& b)` with `a.DeepEquivalent()` / `b.DeepEquivalent()` | Change signature to `const Position& a, const Position& b`; use `a` / `b` directly |
| Line 161 | `VisiblePosition visible_pos = CreateVisiblePosition(pos)` | `Position visible_pos = pos` |
| Line 186 | `EndingVisibleSelection().Start()` | `EndingSelection().Start()` |
| Line 199 | `IsEndOfBlock(EndingVisibleSelection().VisibleStart())` | `IsEndOfBlock(EndingSelection().Start())` — Phase 1 overload |
| Line 248 | `const VisibleSelection& visible_selection = EndingVisibleSelection()` | `const SelectionInDOMTree& selection = EndingSelection()` |
| Line 264 | `EndingVisibleSelection()` | `EndingSelection()` |
| Line 279 | `CreateVisiblePosition(insertion_position).DeepEquivalent()` | `insertion_position` |
| Lines 344–345 | `VisiblePosition visible_pos = CreateVisiblePosition(insertion_position, affinity)` | `PositionWithAffinity visible_pos(insertion_position, affinity)` |
| Line 354 | `LineBreakExistsAtVisiblePosition(visible_pos)` | `LineBreakExistsAtPosition(visible_pos.GetPosition())` |
| Lines 508–512 | `visible_pos = CreateVisiblePosition(insertion_position)` / `visible_pos.DeepEquivalent().AnchorNode()` | `Position visible_pos = insertion_position` / `visible_pos.AnchorNode()` |
| Lines 529–534 | `VisiblePosition visible_insertion_position = CreateVisiblePosition(...)` / `PositionOutsideTabSpan(visible_insertion_position.DeepEquivalent())` | `PositionOutsideTabSpan(insertion_position)` directly |
| Lines 605–612 | `visible_pos = CreateVisiblePosition(insertion_position)` / `LineBreakExistsAtVisiblePosition(visible_pos)` | `LineBreakExistsAtPosition(insertion_position)` |
| Lines 621–622 | `CreateVisiblePosition(insertion_position).DeepEquivalent() != VisiblePosition::BeforeNode(*block_to_insert).DeepEquivalent()` | `insertion_position != Position::BeforeNode(*block_to_insert)` |
| Lines 638–640 | `VisiblePosition before_node_position = VisiblePosition::BeforeNode(*n)` / `ComparePositions(CreateVisiblePosition(insertion_position), ...)` | `Position::BeforeNode(*n)` / `ComparePositions(insertion_position, ...)` |

---

### `apply_block_element_command.cc` (21 refs — Category B + A)

Requires audit of all VP sites. Primary patterns:
- `StartOfParagraph(vp)` / `EndOfParagraph(vp)` → Phase 1 overloads
- `MoveParagraph(vp, vp, vp, ...)` / `MoveParagraphWithClones(vp, vp, ...)` → Phase 3 signatures — just pass `Position` directly
- `EndingVisibleSelection().VisibleStart()` → `EndingSelection().Start()`
- `SelectionForParagraphIteration` takes/returns `VisibleSelection` — change to `SelectionInDOMTree` as part of this file or as a prerequisite

---

### `indent_outdent_command.cc` (38 refs — Category A dominant)

| Location | Current code | Change to |
|---|---|---|
| Lines 77/119–127 | `VisiblePosition& out_end_of_next_of_paragraph_to_move` param; `CreateVisiblePosition(start/end)` / `VisiblePosition::BeforeNode/AfterNode` | `Position&` param; `start` / `end` / `Position::BeforeNode/AfterNode` |
| Line 207 | `NextPositionOf(CreateVisiblePosition(end)).DeepEquivalent()` | `NextPositionOf(end, ...)` |
| Line 208 | `IsStartOfParagraph(CreateVisiblePosition(next_position))` | `IsStartOfParagraph(next_position)` |
| Lines 243–272 | `VisiblePosition start_of_contents = CreateVisiblePosition(start)` / `end_of_contents` / `VisiblePosition::InParentAfterNode(*target_blockquote)` | `Position start_of_contents = start` / `end` / `Position::InParentAfterNode(...)` |
| Lines 280–282 | `VisiblePosition visible_start_of_paragraph = StartOfParagraph(EndingVisibleSelection().VisibleStart())` | `Position start_of_paragraph = StartOfParagraph(EndingSelection().Start())` |
| Lines 308–330 | `VisiblePosition position_in_enclosing_block = VisiblePosition::FirstPositionInNode(...)` and related; VP comparison operators | `Position::FirstPositionInNode(...)` / `ComparePositions(...)` |
| Lines 361–465 | Multiple `CreateVisiblePosition(...)` for `MoveParagraph` / `MoveParagraphWithClones` callers | Just pass `Position` directly — Phase 3 signatures accept `Position` |
| Lines 487–517 | Method params `const VisiblePosition& start_of_selection, end_of_selection`; `EndOfParagraph(vp)`; `.DeepEquivalent()` comparisons | `const Position&` params; `EndOfParagraph(pos)`; direct `Position` comparisons |
| Lines 563–578 | Same pattern in second method | Same |

---

### `delete_selection_command.cc` (42 refs — Category A + B)

Requires full audit. Primary patterns:
- All `MoveParagraph(CreateVisiblePosition(...), ...)` calls → pass raw `Position` (Phase 3)
- `IsStartOfParagraph` / `IsEndOfParagraph` with VP → Phase 1 overloads
- `EndingVisibleSelection().*` → `EndingSelection().*`
- `VisiblePosition::FirstPositionInNode(*cell).DeepEquivalent() == VisiblePosition::LastPositionInNode(*cell).DeepEquivalent()` — replace with `Position::FirstPositionInNode(*cell) == Position::LastPositionInNode(*cell)`; if there is any doubt whether the tree is in a state where these differ from a VP-snapped result, add a code comment explaining why the raw positions are sufficient here

---

### `insert_list_command.cc` (50 refs — Category A + B)

Requires full audit. Primary patterns:
- Large volume of `MoveParagraph(CreateVisiblePosition(...), ...)` calls → pass raw `Position`
- `StartOfParagraph(vp)` / `EndOfParagraph(vp)` → Phase 1 overloads
- Local VP variables used only to call navigation functions → replace with `Position`

---

### `replace_selection_command.cc` (58 refs — Category B dominant)

Requires full audit. Primary patterns:
- `IsStartOfParagraph` / `IsEndOfParagraph` / `StartOfParagraph` / `EndOfParagraph` with VP → Phase 1 overloads
- `EndingVisibleSelection().*` → `EndingSelection().*`
- `CreateVisiblePosition(pos).DeepEquivalent()` round-trips → `pos`
