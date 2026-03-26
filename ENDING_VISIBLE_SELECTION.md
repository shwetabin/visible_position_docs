# EndingVisibleSelection → EndingSelection Migration

## Background

`EndingSelection()` returns `const SelectionForUndoStep&` — raw `Position` objects
(`anchor_`, `focus_`, `Start()`, `End()`, `Affinity()`, `RootEditableElement()`,
`IsNone()`, `IsCaret()`, `IsRange()`). Zero cost; no layout.

`EndingVisibleSelection()` calls `Document::UpdateStyleAndLayout()` then wraps the
result in a `VisibleSelection`. Defined in `composite_edit_command.cc:132`:

```
VisibleSelection CompositeEditCommand::EndingVisibleSelection() const {
  GetDocument().UpdateStyleAndLayout(DocumentUpdateReason::kEditing);
  return CreateVisibleSelection(ending_selection_);
}
```

Every call to `EndingVisibleSelection()` inside `DoApply()` forces a full layout
update against a potentially partially-mutated tree. All 60+ call sites across 13
source files pay this cost even when only a raw `Position` is needed.

`SelectionForUndoStep` exposes every raw accessor that callers need:

| Accessor | Returns |
|----------|---------|
| `Start()` | `Position` — tree-order start of selection |
| `End()` | `Position` — tree-order end of selection |
| `Anchor()` | `Position` — anchor of selection |
| `Focus()` | `Position` — focus of selection |
| `Affinity()` | `TextAffinity` |
| `RootEditableElement()` | `Element*` |
| `IsNone()` | `bool` |
| `IsCaret()` | `bool` |
| `IsRange()` | `bool` |

---

## Group A — Directly replaceable today

These sites call `EndingVisibleSelection()` then immediately access a member that
`SelectionForUndoStep` also exposes. Direct substitution; no other change needed.

| File | Line | Access pattern | Replacement |
|------|------|----------------|-------------|
| `break_blockquote_command.cc` | 108 | `.Start()` | `EndingSelection().Start()` |
| `composite_edit_command.cc` | 142 | `.Anchor()` | `EndingSelection().Anchor()` |
| `composite_edit_command.cc` | 1624 | `.RootEditableElement()` | `EndingSelection().RootEditableElement()` |
| `create_link_command.cc` | 53 | `.Start()` | `EndingSelection().Start()` |
| `delete_selection_command.cc` | 1125 | `.Start()` | `EndingSelection().Start()` |
| `delete_selection_command.cc` | 1433 | `.Start()` | `EndingSelection().Start()` |
| `indent_outdent_command.cc` | 499 | `.End()` | `EndingSelection().End()` |
| `indent_outdent_command.cc` | 552 | `.Start().AnchorNode()` | `EndingSelection().Start().AnchorNode()` |
| `insert_line_break_command.cc` | 240 | `.End()` | `EndingSelection().End()` |
| `insert_list_command.cc` | 188 | `.IsRange()` | `EndingSelection().IsRange()` |
| `insert_list_command.cc` | 361 | `.Start().AnchorNode()` | `EndingSelection().Start().AnchorNode()` |
| `insert_paragraph_separator_command.cc` | 186 | `.Start()` | `EndingSelection().Start()` |
| `insert_text_command.cc` | 116 | `.Start()` | `EndingSelection().Start()` |
| `insert_text_command.cc` | 123 | `.End()` | `EndingSelection().End()` |
| `insert_text_command.cc` | 140 | `.End()` | `EndingSelection().End()` |
| `insert_text_command.cc` | 191 | `.IsNone()` | `EndingSelection().IsNone()` |
| `insert_text_command.cc` | 194 | `.Start()` | `EndingSelection().Start()` |
| `replace_selection_command.cc` | 1360 | `.Start()` | `EndingSelection().Start()` |
| `replace_selection_command.cc` | 1376 | `.Start().AnchorNode()` | `EndingSelection().Start().AnchorNode()` |
| `replace_selection_command.cc` | 2252 | `.Start()` | `EndingSelection().Start()` |
| `replace_selection_command.cc` | 2265 | `.Start()` | `EndingSelection().Start()` |

**~21 sites. Standalone CL, no prerequisites.**

---

## Group B — Replaceable after Step 1 overloads

These sites call `.VisibleStart()` or `.VisibleEnd()` on the result, then pass the
`VisiblePosition` to a function that has no `Position` overload today
(`IsEndOfParagraph`, `IsStartOfParagraph`, `IsEndOfBlock`, `IsStartOfBlock`,
`StartOfParagraph`, `StartOfNextParagraph`, `EnclosingEmptyListItem`,
`PreviousPositionOf`, `ListifyParagraph`).

Once Step 1 adds `Position` overloads for those functions, these sites become:

```cpp
// Before
IsEndOfParagraph(EndingVisibleSelection().VisibleStart())
// After
IsEndOfParagraph(EndingSelection().Start())
```

| File | Lines | VP used for |
|------|-------|-------------|
| `apply_block_element_command.cc` | 69–70, 93–94 | `MoveParagraph` loop variables |
| `composite_edit_command.cc` | 1314 | `VisiblePosition caret_after_delete` — navigation after delete |
| `composite_edit_command.cc` | 1496–1497 | `VisiblePosition visible_start/end` — `MoveParagraphWithClones` |
| `composite_edit_command.cc` | 1711–1712 | `IsStartOfParagraph` / `IsEndOfParagraph` |
| `composite_edit_command.cc` | 1755 | `EnclosingEmptyListItem` |
| `composite_edit_command.cc` | 1886 | `VisiblePosition caret` — whitespace rebalance |
| `indent_outdent_command.cc` | 281 | `StartOfParagraph(VisibleStart())` |
| `indent_outdent_command.cc` | 542 | `CreateVisiblePosition(EndingVisibleSelection().End())` — loop bound |
| `insert_list_command.cc` | 290 | `StartOfNextParagraph(VisibleStart())` |
| `insert_list_command.cc` | 487 | `ListifyParagraph(..., VisibleStart(), ...)` |
| `insert_paragraph_separator_command.cc` | 199 | `IsEndOfBlock(VisibleStart())` |
| `insert_text_command.cc` | 162 | `IsStartOfBlock(VisibleEnd())` |
| `replace_selection_command.cc` | 1245, 1298 | `VisibleStart()` stored, passed to whitespace helpers |
| `replace_selection_command.cc` | 1315 | `PreviousPositionOf(VisibleStart())` |
| `replace_selection_command.cc` | 1791, 1829 | `VisibleStart().DeepEquivalent()` — position identity check |
| `typing_command.cc` | 924 | `VisiblePosition visible_start` — deletion endpoint |
| `typing_command.cc` | 1104 | `VisiblePosition visible_end` — deletion endpoint |

**~20 sites. Unblocked after Step 1.**

---

## Group C — Replaceable after Step 1 + Step 3 signature changes

These sites assign `EndingVisibleSelection()` to a `VisibleSelection` local variable
or pass the full `VisibleSelection` to a VP-typed callee
(`SelectionForParagraphIteration`, `FirstEphemeralRangeOf`, `ToggleSelectedListItem`,
`ListifyParagraph`). They require either the callee signature to be widened to accept
`SelectionInDOMTree` / `SelectionForUndoStep`, or the call site to be rewritten to
not pass a `VisibleSelection` at all.

| File | Lines | Callee / local use | Blocker |
|------|-------|-------------------|---------|
| `apply_block_element_command.cc` | 98 | `SelectionForParagraphIteration(const VisibleSelection&)` | Callee signature |
| `break_blockquote_command.cc` | 128 | `const VisibleSelection& visible_selection` stored, used throughout function | Multiple VP methods |
| `composite_edit_command.cc` | 891 | `VisibleSelection selection` — `IsStartOfParagraph` / navigation | Step 1 overloads |
| `insert_incremental_text_command.cc` | 154 | `const VisibleSelection& visible_selection` — multiple VP uses | Step 1 overloads |
| `insert_line_break_command.cc` | 84 | `VisibleSelection selection` — multiple VP uses | Step 1 overloads |
| `insert_list_command.cc` | 153 | `const VisibleSelection& visible_selection` — `.Start()` and VP methods | Step 1 overloads |
| `insert_list_command.cc` | 190, 218 | `FirstEphemeralRangeOf(const VisibleSelection&)` | Callee signature |
| `insert_list_command.cc` | 197 | `SelectionForParagraphIteration(const VisibleSelection&)` | Callee signature |
| `insert_list_command.cc` | 474 | `ToggleSelectedListItem(const VisibleSelection&, ...)` | Callee signature (Step 3) |
| `insert_paragraph_separator_command.cc` | 248, 264 | `const VisibleSelection& visible_selection` — passed to VP-typed helpers | Step 1 overloads |
| `replace_selection_command.cc` | 1119, 1324, 1759 | `const VisibleSelection& ...` — multiple VP-typed downstream calls | Step 1 + Step 3 |

**~15 sites. Unblocked after Step 1 overloads + Step 3 signature changes.**

---

## Summary

| Group | Sites | Prerequisite |
|-------|-------|-------------|
| A — raw `Start()`/`End()`/`IsRange()` etc. | ~21 | None — standalone CL |
| B — `.VisibleStart()`/`.VisibleEnd()` to VP-only API | ~20 | Step 1 overloads |
| C — whole `VisibleSelection` to VP-only callee | ~15 | Step 1 + Step 3 |
| **Total** | **~56** | |

Eliminating all Group A sites saves ~21 forced `UpdateStyleAndLayout` calls with no
prerequisite work. The remaining ~35 sites are naturally eliminated as the Step 1 and
Step 3 work lands.
