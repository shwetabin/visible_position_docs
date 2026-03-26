# Canonicalization Usage in Editing Commands

## Background

Editing commands in Blink operate on `Position` — a raw DOM node + offset reference
that points into the live tree. Before acting on a position, code often
*canonicalizes* it: resolves ambiguity about exactly which caret location a position
represents, or normalizes it to a form a downstream API can accept.

There are two fundamentally different ways to canonicalize a `Position` in this
codebase:

- **`MostForwardCaretPosition` / `MostBackwardCaretPosition`** — walk forward or
  backward to the deepest or shallowest equivalent caret position. They read
  `GetLayoutObject()` on the existing layout tree but do **not** call
  `UpdateStyleAndLayout`. Cost: a few pointer traversals.

- **`CreateVisiblePosition(pos).DeepEquivalent()`** — full canonicalization via
  `VisiblePosition`. This unconditionally calls `Document::UpdateStyleAndLayout()`,
  which forces Blink to recompute style and layout for the entire document before
  returning, then immediately discards the VP to get a `Position` back out. Cost: a
  full layout pass — potentially milliseconds on a large document.

The problem is that `CreateVisiblePosition` is used in **126 call sites across 16
source files** in `editing/commands/`, but most of those sites do not need a full
layout update. They call `CreateVisiblePosition` only because a downstream function
requires `VisiblePosition`, or because two positions need to be compared for caret
equivalence — neither of which requires `UpdateStyleAndLayout`.

This document classifies all 126 `CreateVisiblePosition` call sites by their actual
purpose, to understand what canonicalization is genuinely needed versus what is
incidental to the `VisiblePosition` type system.

### Scale

| Metric | Count |
|--------|-------|
| `CreateVisiblePosition` calls in `commands/*.cc` | **126** across 16 files |
| Sites that force `UpdateStyleAndLayout` unnecessarily | **~110** (~87%) |
| Sites where the canonicalization is genuinely load-bearing | **~16** (~13%) |

### Canonicalization primitives in this codebase

| Primitive | What it does | Layout cost |
|-----------|-------------|-------------|
| `MostForwardCaretPosition(pos)` | Walks forward to the deepest equivalent caret position | Reads `GetLayoutObject()` — no forced update |
| `MostBackwardCaretPosition(pos)` | Walks backward to the shallowest equivalent caret position | Same — reads existing layout, no forced update |
| `CreateVisiblePosition(pos)` | Forces `UpdateStyleAndLayout`, then snaps to canonical caret | Forces `UpdateStyleAndLayout` |
| `ToParentAnchoredPosition()` | Converts offset-in-anchor to child-index in parent | Pure DOM arithmetic, no layout |
| `ToOffsetInAnchor()` | Converts child-index position to character offset in parent | Pure DOM arithmetic, no layout |

---

## Category 1: Selection boundary normalization

**Purpose:** At the start of a command, push the selection's start position
forward (into editable content, past collapsed whitespace) and its end position
backward to establish the canonical working range for the operation. The raw
positions from `EndingSelection()` may sit at a before/after-node boundary that
would make DOM operations ambiguous.

**How it is done:** `MostForwardCaretPosition` / `MostBackwardCaretPosition` —
**no `CreateVisiblePosition` involved**.

**Sites:**

| File | Lines | Call |
|------|-------|------|
| `apply_style_command.cc` | 133–134, 160–161, 176–177 | `start_(MostForwardCaretPosition(EndingSelection().Start()))` / `end_(MostBackwardCaretPosition(EndingSelection().End()))` — all three constructors |
| `apply_style_command.cc` | 425 | `start = MostBackwardCaretPosition(start)` |
| `apply_style_command.cc` | 762 | `remove_start = MostBackwardCaretPosition(start)` |
| `apply_style_command.cc` | 795 | `embedding_remove_end = MostForwardCaretPosition(...)` |
| `apply_style_command.cc` | 1494, 1512 | `push_down_start = MostForwardCaretPosition(start)` / `push_down_end = MostBackwardCaretPosition(end)` |
| `delete_selection_command.cc` | 268–271 | `upstream_start_ = MostBackwardCaretPosition(start)` / `downstream_start_ = MostForwardCaretPosition(start)` / `upstream_end_ = MostBackwardCaretPosition(end)` / `downstream_end_ = MostForwardCaretPosition(end)` |
| `delete_selection_command.cc` | 367–368, 386–387 | Re-canonicalize after `PreviousPositionOf` / `NextPositionOf` adjustment |
| `insert_text_command.cc` | 199, 208, 217 | Normalize insertion point |
| `insert_paragraph_separator_command.cc` | 300, 302, 459, 523, 540–542 | Normalize `insertion_position` at various stages |
| `style_commands.cc` | 469, 477 | `MostForwardCaretPosition(selection.Start())` / `MostBackwardCaretPosition(selection.End())` |
| `typing_command.cc` | 1103, 1114, 1145 | Normalize deletion endpoint |
| `break_blockquote_command.cc` | 99, 134 | `MostForwardCaretPosition(start)` |
| `composite_edit_command.cc` | 859, 862, 865, 1558–1559 | Whitespace deletion and rebalance range |
| `replace_selection_command.cc` | 1760, 1763, 1911, 1954 | Normalize inserted content start/end after paste |

These sites are already correct — `MostForward/BackwardCaretPosition` is the right
tool and does not force layout.

---

## Category 2: Type adapter — `CreateVisiblePosition(pos)` to satisfy a VP-only signature

**Purpose:** The position is already valid and correct. `CreateVisiblePosition` is
called only because the downstream function (`IsEndOfParagraph`, `StartOfParagraph`,
`MoveParagraph`, etc.) accepts only `VisiblePosition` today. The VP is passed
immediately and never stored or reused. The `.DeepEquivalent()` call that often
follows immediately discards the VP right back to a `Position`.

This is the largest category — approximately **55 of 126** sites.

**Sites:**

| File | Lines | Call | Why VP needed |
|------|-------|------|--------------|
| `apply_block_element_command.cc` | 197, 249 | `CreateVisiblePosition(end)` | Loop variable assigned to `VisiblePosition end_of_current_paragraph`, passed to `MoveParagraph` |
| `apply_block_element_command.cc` | 305 | `CreateVisiblePosition(PreviousPositionOf(...))` | `StartOfParagraph(VP)` — no `Position` overload |
| `format_block_command.cc` | 97–100 | `CreateVisiblePosition(start/end)` | `StartOfBlock(VP)` / `EndOfBlock(VP)` — no `Position` overload |
| `format_block_command.cc` | 106, 126, 155–156 | `CreateVisiblePosition(pos)` | `IsEndOfParagraph(VP)` / `IsStartOfParagraph(VP)` — no `Position` overload |
| `format_block_command.cc` | 129–130 | `CreateVisiblePosition(start/end)` | `MoveParagraph(VP, VP, VP, ...)` — no `Position` overload |
| `indent_outdent_command.cc` | 121, 126 | `CreateVisiblePosition(start/end)` | `MoveParagraph` / `MoveParagraphs` — no `Position` overload |
| `indent_outdent_command.cc` | 207–208 | `CreateVisiblePosition(end)` → `NextPositionOf(VP)` | Then `IsStartOfParagraph(VP)` |
| `indent_outdent_command.cc` | 243, 272 | `CreateVisiblePosition(start/end)` | Loop `VisiblePosition` variables for `MoveParagraph` |
| `indent_outdent_command.cc` | 361, 377 | `CreateVisiblePosition(Position::BeforeNode/AfterNode(...))` | `MoveParagraph` — no `Position` overload |
| `indent_outdent_command.cc` | 542, 547 | `CreateVisiblePosition(...)` | Loop iteration for `MoveParagraph` |
| `insert_line_break_command.cc` | 116 | `CreateVisiblePosition(caret.ToPositionWithAffinity())` | `IsEndOfParagraph(VP)` — no `Position` overload |
| `insert_list_command.cc` | 115, 120 | `CreateVisiblePosition(selection_start/end)` | `StartOfParagraph(VP)` — no `Position` overload |
| `insert_list_command.cc` | 241, 286, 326, 335 | `CreateVisiblePosition(pos)` | Loop variables for `MoveParagraph` |
| `insert_list_command.cc` | 584, 586, 605, 606 | `CreateVisiblePosition(start/end)` | `IsStartOfParagraph(VP)` / `IsEndOfParagraph(VP)` and `MoveParagraph` |
| `replace_selection_command.cc` | 1099, 1745–1746 | `CreateVisiblePosition(pos)` | `MoveParagraph` — no `Position` overload |
| `replace_selection_command.cc` | 2171–2172 | `CreateVisiblePosition(insert_pos)` | `IsStartOfParagraph(VP)` / `IsEndOfParagraph(VP)` |
| `composite_edit_command.cc` | 813, 815 | `CreateVisiblePosition(Position(text_node, upstream/downstream))` | `IsEndOfParagraph(VP)` in whitespace helpers |
| `editor_command.cc` | 1228 | `CreateVisiblePosition(selection.Anchor())` | `VisiblePositionForIndex` — public VP API |

---

## Category 3: Canonical equivalence check — `CreateVisiblePosition(x).DeepEquivalent() == CreateVisiblePosition(y).DeepEquivalent()`

**Purpose:** Two raw positions may be distinct DOM node+offset pairs yet represent
the same visual caret location. `CreateVisiblePosition` is called on both sides and
`.DeepEquivalent()` is immediately extracted to compare whether they resolve to the
same canonical position. The VP is created and discarded in the same expression.

The underlying question being answered is: "do these two positions land on the same
caret?" The VP is never used for anything else.

**Sites:**

| File | Lines | Purpose |
|------|-------|---------|
| `delete_selection_command.cc` | 149–153 | Check whether adjusted `start`/`end` has drifted from `selection_to_delete_` endpoints |
| `insert_list_command.cc` | 139, 142 | Verify `should_be_former` is visually before `should_be_later` after selection adjustment |
| `insert_list_command.cc` | 408–415 | Check whether list element boundary equals selection boundary — `PositionBeforeNode`/`PositionAfterNode` vs `current_selection.StartPosition()`/`EndPosition()` |
| `insert_paragraph_separator_command.cc` | 279 | Re-canonicalize `insertion_position` to resolve before/after-node ambiguity after tree manipulation |
| `insert_paragraph_separator_command.cc` | 621 | Verify `insertion_position` did not change after node insertion |
| `insert_paragraph_separator_command.cc` | 640 | `ComparePositions(CreateVisiblePosition(insertion_position), ...)` — relative order check post-insertion |
| `replace_selection_command.cc` | 170–171 | Check `pos` vs `next_position` are visually distinct in line-break detection loop |
| `editing_commands_utilities.cc` | 442–443 | `RendersInDifferentPosition` — `CreateVisiblePosition(first).DeepEquivalent() == CreateVisiblePosition(MostBackwardCaretPosition(second)).DeepEquivalent()` |
| `composite_edit_command.cc` | 2130–2131, 2138–2139 | Check `PositionBeforeNode(node)` / `PositionAfterNode(node)` equals selection range boundary |

---

## Category 4: Node boundary snap — `CreateVisiblePosition(PositionBeforeNode/AfterNode(...))`

**Purpose:** `PositionBeforeNode` / `PositionAfterNode` / `LastPositionInOrAfterNode`
/ `FirstPositionInOrBeforeNode` produce positions at the before/after slot of an
element. These are often non-canonical — the actual caret renders inside the element,
not at the boundary. `CreateVisiblePosition` is used to snap to the first editable
leaf inside the element.

This category breaks into two sub-purposes:

**Sub-purpose A — snap exists only to satisfy a VP-typed callee:** The position is
correct; the snap is incidental to the function signature. The VP is passed through
immediately.

**Sub-purpose B — snap is used for position identity:** The snapped position is
stored or compared, and the before/after-node encoding is genuinely ambiguous for
that use.

**Sites:**

| File | Lines | Position source | Sub-purpose |
|------|-------|----------------|-------------|
| `apply_block_element_command.cc` | 444 | `PositionAfterNode(...)` | A — loop end passed to `MoveParagraph` |
| `editing_commands_utilities.cc` | 144, 146 | `FirstPositionInOrBeforeNode` / `LastPositionInOrAfterNode` | A — passed to `IsStartOfParagraph` / `IsEndOfParagraph` |
| `editing_commands_utilities.cc` | 199, 201, 222, 224 | `FirstPositionInOrBeforeNode` / `LastPositionInOrAfterNode` | A — `IsFirstVisiblePositionInNode` / `IsLastVisiblePositionInNode` |
| `replace_selection_command.cc` | 874, 946 | `LastPositionInOrAfterNode(*element)` | B — stored as return value of `EndOfInsertedContent()` / `PositionAfterInsertedContent()` |
| `replace_selection_command.cc` | 950, 959 | `end_of_inserted_content_` / `start_of_inserted_content_` | B — public VP return type of `EndOfInsertedContent()` / `StartOfInsertedContent()` |
| `replace_selection_command.cc` | 1435 | `insertion_pos` | B — snap insertion point before DOM operations |
| `replace_selection_command.cc` | 1601 | `start_of_inserted_content_` via `RelocatablePosition` | B — post-mutation snap |
| `insert_list_command.cc` | 744, 750 | `start_pos` / `relocatable_original_start->GetPosition()` | A — passed to `ListifyParagraph(VP)` |
| `insert_list_command.cc` | 779 | `pos.ToPositionWithAffinity()` | A — used as VP for `PreviousPositionOf(VP)` |
| `composite_edit_command.cc` | 1777–1779 | `PositionAfterNode(*block_enclosing_list/list_node)` | B — list boundary equality check |

---

## Category 5: Re-canonicalize after `RelocatablePosition` retrieval

**Purpose:** `RelocatablePosition::GetPosition()` returns a raw `Position` that
survived a DOM mutation. After the mutation, the position's node+offset may sit at
an awkward boundary (e.g., between two text nodes that were split). `CreateVisiblePosition`
is called to snap it to a clean caret position before use.

**Sites:**

| File | Lines | Pattern |
|------|-------|---------|
| `composite_edit_command.cc` | 1428, 1430 | `CreateVisiblePosition(relocatable_before/after_paragraph->GetPosition())` — passed to `MoveParagraph` |
| `composite_edit_command.cc` | 1652, 1654 | Same in `MoveParagraphContentsToNewBlockIfNecessary` |
| `indent_outdent_command.cc` | 448–450, 475–477 | `CreateVisiblePosition(start/end_of_paragraph->GetPosition())` — passed to `MoveParagraph` |
| `insert_list_command.cc` | 750 | `CreateVisiblePosition(relocatable_original_start->GetPosition())` — passed to `ListifyParagraph` |
| `delete_selection_command.cc` | 1039 | `CreateVisiblePosition(relocatable_start->GetPosition())` — passed to `MergeBlocksAfterDelete` |

In all five cases the VP is passed directly to a function that requires
`VisiblePosition` — there is no separate snap logic; this overlaps with Category 2.
The `RelocatablePosition` origin is noted separately because it signals a
post-mutation context.

---

## Category 6: Affinity-aware caret placement — `CreateVisiblePosition(pos, affinity)`

**Purpose:** A `Position` is paired with an explicit `TextAffinity`
(upstream/downstream) to determine which visual line the caret sits on at a
line-wrap point. This is the only `CreateVisiblePosition` call form where affinity
is not immediately discarded — it is used to drive a visual line-boundary query
before `.DeepEquivalent()` is called.

**Sites:**

| File | Lines | Affinity used | Why |
|------|-------|--------------|-----|
| `delete_selection_command.cc` | 343 | `CreateVisiblePosition(upstream_start_, selection_to_delete_.Affinity())` | Preserves the original selection's affinity for an `IsStartOfLine` / `HasSoftLineBreak` check — determines whether the deletion endpoint is visually at the start of the next line |
| `insert_paragraph_separator_command.cc` | 345 | `CreateVisiblePosition(insertion_position, affinity)` | Determines whether the caret is at the end of the current line or start of the next, controlling which paragraph the separator is inserted into |

These are the only two sites where the `VisiblePosition` is doing genuine
affinity-sensitive work rather than acting as a type adapter.

---

## Category 7: `ToParentAnchoredPosition` / `ToOffsetInAnchor`

**Purpose:** Representation conversions between two encodings of the same
`Position` — child-index form vs. character-offset form — required by specific
APIs. Not canonicalization in the VP sense; no layout involved.

**Sites:**

| File | Lines | Purpose |
|------|-------|---------|
| `composite_edit_command.cc` | 1527–1528, 1534–1535 | `TextIterator::RangeLength` requires `ToParentAnchoredPosition()` for text length computation in `kPreserveSelection` path |
| `composite_edit_command.cc` | 1672, 1678 | Same — `TextIterator::RangeLength` in `MoveParagraphWithClones` |
| `composite_edit_command.cc` | 2073 | `Position::BeforeNode(*node).ToOffsetInAnchor()` — `AddedNodes` range tracking |
| `apply_style_command.cc` | 1581 | `Position::BeforeNode(*next).ToOffsetInAnchor()` — style split point |

---

## Summary

| Category | Approx. sites | `CreateVisiblePosition` involved? | Layout forced? |
|----------|--------------|----------------------------------|---------------|
| 1 — Selection boundary normalization | ~25 | No | No |
| 2 — Type adapter for VP-only API | ~55 | Yes | Yes |
| 3 — Canonical equivalence check | ~10 | Yes | Yes |
| 4A — Node boundary snap, passed to VP API | ~8 | Yes | Yes |
| 4B — Node boundary snap, genuine identity | ~7 | Yes | Yes |
| 5 — Post-`RelocatablePosition` re-canonicalize | ~10 | Yes (overlaps Cat. 2) | Yes |
| 6 — Affinity-aware caret placement | 2 | Yes | Yes |
| 7 — `ToParentAnchoredPosition` / `ToOffsetInAnchor` | ~6 | No | No |

Of the 126 `CreateVisiblePosition` calls:

- **~110 (~87%)** are in categories 2, 3, 4A, 4B, and 5 — the VP is a type adapter
  or an equivalence-check vehicle; it forces a full layout update for work that does
  not semantically require it.
- **~14 (~11%)** are in category 1 (already correct, `MostForward/BackwardCaretPosition`)
  and category 7 (pure encoding conversions) — no issue.
- **2 (~2%)** are in category 6 — the only sites doing genuinely affinity-sensitive
  work that `VisiblePosition` is designed for.
