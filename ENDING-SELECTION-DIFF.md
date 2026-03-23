# `EndingSelection()` vs `EndingVisibleSelection()` — What Is Actually Different

---

## The Underlying Storage

Both methods expose the same underlying field:

```cpp
// composite_edit_command.h:247
SelectionForUndoStep ending_selection_;
```

`SelectionForUndoStep` stores:
- `anchor_` — raw `Position` (can be disconnected from document)
- `focus_` — raw `Position` (can be disconnected from document)
- `affinity_` — `TextAffinity` (upstream/downstream)
- `is_anchor_first_` — computed at construction by comparing anchor ≤ focus
- `root_editable_element_` — captured at construction time, not recomputed

No layout object references. No canonicalization. Survives DOM mutations.

---

## `EndingSelection()` — What It Does

```cpp
// composite_edit_command.h:59-60
const SelectionForUndoStep& EndingSelection() const {
  return ending_selection_;
}
```

**Returns the field directly. No side effects. No layout. O(1).**

Accessors available on the returned object — all return raw `Position` or primitive
values, no layout required:

| Method | Returns | Cost |
|--------|---------|------|
| `.Start()` | `Position` — the logically earlier endpoint | O(1) |
| `.End()` | `Position` — the logically later endpoint | O(1) |
| `.Anchor()` | `Position` — the anchor (base) of selection | O(1) |
| `.Focus()` | `Position` — the focus (extent) of selection | O(1) |
| `.Affinity()` | `TextAffinity` | O(1) |
| `.IsAnchorFirst()` | `bool` | O(1) |
| `.IsCaret()` | `bool` — `anchor == focus` | O(1) |
| `.IsNone()` | `bool` — `anchor.IsNull()` | O(1) |
| `.IsRange()` | `bool` — `anchor != focus` | O(1) |
| `.RootEditableElement()` | `Element*` — captured at construction | O(1) |

---

## `EndingVisibleSelection()` — What It Does

```cpp
// composite_edit_command.cc:132-138
VisibleSelection CompositeEditCommand::EndingVisibleSelection() const {
  // TODO(editing-dev): The use of UpdateStyleAndLayout()
  // needs to be audited. See http://crbug.com/590369 for more details.
  GetDocument().UpdateStyleAndLayout(DocumentUpdateReason::kEditing);
  return CreateVisibleSelection(ending_selection_);
}
```

**Two operations happen, in order:**

### Step 1 — `UpdateStyleAndLayout()`

Forces a full style and layout recalculation on the document. This is expensive:
- Resolves all dirty CSS
- Recomputes layout boxes, line boxes, inline formatting contexts
- Required before any `LayoutObject` geometry or position-snapping is valid

### Step 2 — `CreateVisibleSelection(ending_selection_)`

Calls `CreateVisibleSelection(selection_in_undo_step.AsSelection())` which:
1. Calls `AsSelection()` to produce a `SelectionInDOMTree` from the raw positions
2. Calls `CreateVisibleSelection(SelectionInDOMTree)` which for each endpoint calls
   `CreateVisiblePosition(position, affinity)` which calls `CanonicalPositionOf(pos)` —
   a DOM walk that snaps the position to the nearest visually renderable caret location

**Net result:** returns a `VisibleSelection` whose `.VisibleStart()` / `.VisibleEnd()`
are snapped canonical positions, valid as of the just-forced layout.

---

## Side-by-Side Comparison

| Property | `EndingSelection()` | `EndingVisibleSelection()` |
|----------|--------------------|-----------------------------|
| Return type | `const SelectionForUndoStep&` | `VisibleSelection` (by value) |
| Forces layout | **No** | **Yes** — `UpdateStyleAndLayout()` every call |
| Canonicalizes positions | **No** — raw stored positions | **Yes** — `CanonicalPositionOf()` per endpoint |
| Valid after DOM mutation | **Yes** — stores raw positions | **Only after** calling `UpdateStyleAndLayout` first |
| Cost | O(1) | O(layout) — proportional to document dirty state |
| Invalidated by mutation | Never | Immediately (positions become stale until next layout) |
| Endpoint access | `.Start()`, `.End()`, `.Anchor()`, `.Focus()` | `.VisibleStart()`, `.VisibleEnd()` returning `VisiblePosition` |

---

## What `VisibleStart()` Adds Over `Start()`

```
EndingSelection().Start()
  → raw Position stored at construction; no snapping

EndingVisibleSelection().VisibleStart()
  → UpdateStyleAndLayout()
  → CanonicalPositionOf(Start())  ← snaps to nearest renderable caret position
  → wraps in VisiblePosition with affinity
```

The canonical snap matters when:
- The stored `Position` is between two characters that have different visual
  positions (e.g., at a line-wrap boundary — the same DOM offset can be at the
  end of line 1 or start of line 2 depending on affinity)
- The stored `Position` is inside a node that has no rendered box (e.g., a
  collapsed element) and needs to be walked forward/backward to a renderable point

The canonical snap **does not matter** when:
- The position is already canonical (which it usually is — positions stored in
  `ending_selection_` come from `SetEndingSelection` calls that were themselves
  constructed from canonical positions)
- Only the node or offset is needed, not the visual location
- The code is asking a structural question (`AnchorNode()`, `IsNull()`, comparing
  positions) rather than a visual one

---

## The Double-Layout Problem

In `replace_selection_command.cc`, a common pattern is:

```cpp
GetDocument().UpdateStyleAndLayout(DocumentUpdateReason::kEditing);  // explicit gate
VisiblePosition start_after_delete =
    EndingVisibleSelection().VisibleStart();                         // forces layout AGAIN
if (IsEndOfParagraph(start_after_delete) && ...)
```

`EndingVisibleSelection()` calls `UpdateStyleAndLayout` unconditionally inside
itself — even when the caller already did so one line above. The layout is recomputed
twice on an already-clean tree.

**The fix:**

```cpp
GetDocument().UpdateStyleAndLayout(DocumentUpdateReason::kEditing);  // explicit gate stays
Position start_after_delete = EndingSelection().Start();             // no layout, raw position
if (IsEndOfParagraph(start_after_delete) && ...)                     // Phase 1 Position overload
```

The explicit `UpdateStyleAndLayout` gate already in the caller is the correct
synchronization point. `EndingVisibleSelection()` adds nothing on top of it.

---

## Decision Guide: Which to Use

```
Need the position for arithmetic, comparison, or passing to a
structural function (AnchorNode, IsNull, EnclosingBlock, etc.)?
  → EndingSelection().Start() / .End() / .Anchor()

Need to know if the position is at a visual boundary
(IsEndOfParagraph, IsStartOfBlock, etc.) after a DOM mutation?
  → UpdateStyleAndLayout() explicitly, then EndingSelection().Start()
     with a Phase 1 Position overload for the boundary test.
     Do NOT use EndingVisibleSelection() — it adds a redundant second layout.

Need a VisiblePosition to pass to a function that has no Position overload yet?
  → Add the Position overload first (Phase 1).
     EndingVisibleSelection() is not the right answer.

Are you at the entry boundary of Apply() checking IsRichlyEditablePosition?
  → EndingVisibleSelection() is correct here. This is the one legitimate
     use inside commands — the boundary-out validation point.
```

---

## Why `ending_selection_` Positions Are Usually Already Canonical

`SetEndingSelection` is called with a `SelectionForUndoStep::From(SelectionInDOMTree)`
where the `SelectionInDOMTree` was built from positions that were valid at call time.
In most cases those positions came from:
- `EndingVisibleSelection()` itself in an earlier step (already canonical)
- `InsertNode`, `AppendNode`, etc. which return canonical positions
- `StartOfParagraph`, `EndOfParagraph` etc. which return canonical positions

So by the time code reads `EndingSelection().Start()`, the position is nearly always
already at a canonical caret location. The `CanonicalPositionOf` snap inside
`EndingVisibleSelection()` is re-doing work that was already done when the position
was stored.
