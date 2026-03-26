# `CreateVisiblePosition(pos).DeepEquivalent()` — All Usages in `editing/commands/`

This document catalogs every site in `third_party/blink/renderer/core/editing/commands/`
where `CreateVisiblePosition(pos).DeepEquivalent()` appears, explains what each site
is actually doing, and classifies it against the anti-patterns in DESIGN.md.

---

## What the call sequence means

```cpp
CreateVisiblePosition(pos).DeepEquivalent()
```

1. `CreateVisiblePosition(pos)` — calls `Document::UpdateStyleAndLayout()`, then
   snaps `pos` to the nearest visually valid caret position (canonicalization).
2. `.DeepEquivalent()` — discards the `VisiblePosition` wrapper and returns the
   underlying `Position`.

The only reason to do both together is **canonicalization**: converting an
arbitrary raw `Position` into the canonical position that a caret would visually
occupy. The `VisiblePosition` wrapper itself is thrown away immediately; it carries
no information that is used.

---

## Sites

### 1. `replace_selection_command.cc` lines 170–171

**Function:** `PositionAvoidingPrecedingNodes(Position pos)` (file-local static)

```cpp
if (next_position == pos ||
    EnclosingBlock(next_position.ComputeContainerNode()) !=
        enclosing_block_element ||
    CreateVisiblePosition(pos).DeepEquivalent() !=
        CreateVisiblePosition(next_position).DeepEquivalent())
  break;
```

**What it does:**
Walks `pos` forward through ancestor nodes (staying inside the same block) to find
the last position that is visually identical to the starting position. The loop skips
nodes that are visually co-located with `pos` — i.e., invisible whitespace, `<br>`
placeholders, collapsed elements. The `CreateVisiblePosition(...).DeepEquivalent()`
comparison checks whether two structurally different `Position` values land on the
same canonical caret location. If they differ visually, the walk stops.

**Why VP is used:**
`Position` equality (`==`) is structural — it compares node + offset exactly. Two
positions that look identical on screen (same caret location) can differ structurally
(e.g., before-node vs. inside-node at offset 0). The VP round-trip is the only
existing way to ask "do these two positions map to the same visual caret?" without a
`Position`-typed overload for that query.

**Anti-pattern classification:** P1 — create-then-extract used as a caret-equivalence
test. The VP serves only to canonicalize; no VP-specific state (affinity, layout
object) is used.

**Replacement:** `CanonicalPositionOf(pos) != CanonicalPositionOf(next_position)`.
`CanonicalPositionOf` calls the same algorithm without constructing a `VisiblePosition`.
Note: `UpdateStyleAndLayout` is already called by the outer loop's context (the
function is only called from `DoApply` after an explicit layout gate); however, the
`CreateVisiblePosition` calls here each trigger a redundant second layout update per
iteration.

---

### 2. `delete_selection_command.cc` lines 149–153

**Function:** `DeleteSelectionCommand::InitializePositionData` (expanding selection
around "special elements" like `<table>`, `<select>`, `<object>`)

```cpp
if (CreateVisiblePosition(start).DeepEquivalent() !=
        CreateVisiblePosition(selection_to_delete_.Start()).DeepEquivalent() ||
    CreateVisiblePosition(end).DeepEquivalent() !=
        CreateVisiblePosition(selection_to_delete_.End()).DeepEquivalent())
  break;
```

**What it does:**
Before expanding the selection boundaries outward to fully encompass a "special
element", this checks whether the proposed expanded positions (`start`, `end`) are
still visually equivalent to the original selection boundaries
(`selection_to_delete_.Start()` / `.End()`). If expanding would visually move the
selection, the expansion is abandoned. This prevents deleting more content than the
user selected.

**Why VP is used:**
Same reason as site 1: checking whether two structurally different positions
represent the same visual caret location. `start` and `end` have been adjusted by
`PositionBeforeContainingSpecialElement` / `PositionAfterContainingSpecialElement`,
which return positions anchored to different nodes than the originals. Structural
equality would always differ; only visual/canonical equality matters here.

**Anti-pattern classification:** P1 — caret-equivalence check via VP round-trip.
Four `CreateVisiblePosition` calls (two pairs), each triggering `UpdateStyleAndLayout`,
in a loop that may iterate multiple times.

**Replacement:** Four `CanonicalPositionOf(...)` calls, same logic.

---

### 3. `insert_text_command.cc` line 299

**Function:** `InsertTextCommand::InsertTab`

```cpp
GetDocument().UpdateStyleAndLayout(DocumentUpdateReason::kEditing);

Position insert_pos = CreateVisiblePosition(pos).DeepEquivalent();
if (insert_pos.IsNull())
  return pos;

Node* node = insert_pos.ComputeContainerNode();
auto* text_node = DynamicTo<Text>(node);
unsigned offset = text_node ? insert_pos.OffsetInContainerNode() : 0;
```

**What it does:**
Canonicalizes the insertion position before inserting a tab character. The result
`insert_pos` is used to find the container node and offset within that node where
the `"\t"` text will be inserted. The null-check guards against `pos` being in an
invisible or non-editable location.

**Why VP is used:**
The incoming `pos` may be a before-node or after-node position. Inserting text
requires an offset-in-text-node position. `CreateVisiblePosition` snaps to a
text-node offset position that is actually editable. The null return from
`DeepEquivalent()` on a null VP signals that the position has no valid visual
caret — used as the null guard.

**Anti-pattern classification:** P1 — create-then-extract for canonicalization.
The explicit `UpdateStyleAndLayout` one line above already ensures layout is clean;
`CreateVisiblePosition` triggers a second redundant layout update.

**Replacement:** `CanonicalPositionOf(pos)`. The null-check remains valid because
`CanonicalPositionOf` returns a null `Position` when the input has no valid
canonical form.

---

### 4. `insert_paragraph_separator_command.cc` line 279

**Function:** `InsertParagraphSeparatorCommand::DoApply`

```cpp
GetDocument().UpdateStyleAndLayout(DocumentUpdateReason::kEditing);

// ...
Position canonical_pos =
    CreateVisiblePosition(insertion_position).DeepEquivalent();

if (!start_block || ... ||
    (!canonical_pos.IsNull() &&
     IsDisplayInsideTable(canonical_pos.AnchorNode())) ||
    (!canonical_pos.IsNull() &&
     IsA<HTMLHRElement>(*canonical_pos.AnchorNode()))) {
  ApplyCommandToComposite(
      MakeGarbageCollected<InsertLineBreakCommand>(GetDocument()), ...);
```

**What it does:**
After deleting any existing selection, the insertion position may be a structural
boundary position (e.g., before a `<table>` or `<hr>`). `canonical_pos` is the
visually canonical form of that position. It is then used to ask: does the canonical
position sit inside a table, or on an `<hr>` element? If so, insert a `<br>` instead
of a new paragraph block.

**Why VP is used:**
`insertion_position` may be a raw before-node / after-node boundary position.
`IsDisplayInsideTable` and `IsA<HTMLHRElement>` need the *anchor node* of the
position, and that anchor node must be the node that the caret visually sits *on*,
not a parent or boundary anchor. The VP round-trip resolves the position to the
node that actually owns the rendered caret location. A comment in the source even
notes this: "If the node is hidden, we don't have a canonical position so we will do
the wrong thing for tables and `<hr>`."

**Anti-pattern classification:** P1 — create-then-extract. The explicit
`UpdateStyleAndLayout` above makes the second layout update inside
`CreateVisiblePosition` redundant.

**Replacement:** `CanonicalPositionOf(insertion_position)`.

---

### 5. `insert_paragraph_separator_command.cc` line 621

**Function:** `InsertParagraphSeparatorCommand::DoApply` (later in the same function)

```cpp
if (CreateVisiblePosition(insertion_position).DeepEquivalent() !=
    VisiblePosition::BeforeNode(*block_to_insert).DeepEquivalent()) {
  // move start node and its siblings into the new block
```

**What it does:**
Checks whether the insertion position is visually equivalent to the position
immediately before the newly inserted block element (`block_to_insert`). If they are
the same visual position, there is nothing to move into the new block — the content
is already correctly positioned. If they differ, nodes need to be moved.

**Why VP is used:**
Again a caret-equivalence check between two structurally different positions:
`insertion_position` (a position inside the existing tree) vs.
`VisiblePosition::BeforeNode(*block_to_insert)` (a position anchored before a newly
inserted element). Structural equality would never hold. The check requires visual
equivalence.

The right-hand side also uses `VisiblePosition::BeforeNode(...)` — a static factory
that creates a VP anchored before a node, which is then also discarded with
`.DeepEquivalent()`. Both VPs are pure type adapters.

**Anti-pattern classification:** P1 — dual create-then-extract for a visual equality
test. Also note the right-hand side uses `VisiblePosition::BeforeNode` which can be
replaced with `Position::BeforeNode`.

**Replacement:**
```cpp
if (CanonicalPositionOf(insertion_position) !=
    CanonicalPositionOf(Position::BeforeNode(*block_to_insert)))
```

---

### 6. `insert_list_command.cc` lines 138–143 (DCHECK only)

**Function:** `InSameTreeAndOrdered` (file-local static helper)

```cpp
static bool InSameTreeAndOrdered(const Position& should_be_former,
                                 const Position& should_be_later) {
  // Input positions must be canonical positions.
  DCHECK_EQ(should_be_former,
            CreateVisiblePosition(should_be_former).DeepEquivalent())
      << should_be_former;
  DCHECK_EQ(should_be_later,
            CreateVisiblePosition(should_be_later).DeepEquivalent())
      << should_be_later;
  return Position::CommonAncestorTreeScope(should_be_former, should_be_later) &&
         ComparePositions(should_be_former, should_be_later) <= 0;
}
```

**What it does:**
This is a **debug assertion only** (`DCHECK_EQ`). It verifies that the two incoming
positions are already canonical — i.e., they are equal to their own VP-canonicalized
forms. This is a precondition check: callers are expected to pass only canonical
positions. The `CreateVisiblePosition` call (and its `UpdateStyleAndLayout`) does not
execute in release builds.

**Why VP is used:**
A DCHECK asserting canonicality. There is no release-build logic here.

**Anti-pattern classification:** Not an anti-pattern in terms of release-build cost
(DCHECKs compile to nothing). However, the assertion pattern itself can be replaced
with `CanonicalPositionOf` in a cleaner way that does not imply a layout update even
in debug builds.

**Replacement:** `DCHECK_EQ(should_be_former, CanonicalPositionOf(should_be_former))`
— same intent, no VP construction.

---

### 7. `format_block_command.cc` lines 97–100

**Function:** `FormatBlockCommand::FormatRange`

```cpp
GetDocument().UpdateStyleAndLayout(DocumentUpdateReason::kEditing);
if (IsElementForFormatBlock(ref_element->TagQName()) &&
    CreateVisiblePosition(start).DeepEquivalent() ==
        StartOfBlock(CreateVisiblePosition(start)).DeepEquivalent() &&
    (CreateVisiblePosition(end).DeepEquivalent() ==
         EndOfBlock(CreateVisiblePosition(end)).DeepEquivalent() ||
     IsNodeVisiblyContainedWithin(*ref_element, range)) &&
    ref_element != root && !root->IsDescendantOf(ref_element)) {
```

**What it does:**
After an explicit `UpdateStyleAndLayout`, this checks whether `start` is at the
visual start of its block and `end` is at the visual end of its block (or the
`ref_element` fully contains the range). If both conditions hold, the selection
already exactly covers the block element — so instead of inserting a new block
wrapper, the existing `ref_element` can be reused or replaced in-place.

`StartOfBlock(vp)` and `EndOfBlock(vp)` take a `VisiblePosition`. The `start` and
`end` positions are wrapped into VPs purely to satisfy these function signatures,
then immediately unwrapped.

**Why VP is used:**
`StartOfBlock` and `EndOfBlock` have no `Position` overloads (they take
`const VisiblePosition&`). This is the P2 anti-pattern: VP creation forced by a
missing `Position` overload on a navigation function.

Note: `start` and `end` are each passed to `CreateVisiblePosition` **twice** in this
expression — once for the `DeepEquivalent()` on the left of `==`, and once as the
argument to `StartOfBlock`/`EndOfBlock`. This means 4 `CreateVisiblePosition` calls
(4 layout updates) where 0 are needed once the overloads exist.

**Anti-pattern classification:** P2 — VP creation forced by missing `Position`
overloads on `StartOfBlock` / `EndOfBlock`. The explicit `UpdateStyleAndLayout`
above makes all 4 internal layout updates redundant.

**Replacement:** Once Step 1 (`IsStartOfBlock(pos)` / `IsEndOfBlock(pos)` overloads)
is done:
```cpp
if (IsElementForFormatBlock(ref_element->TagQName()) &&
    IsStartOfBlock(start) &&
    (IsEndOfBlock(end) || IsNodeVisiblyContainedWithin(*ref_element, range)) &&
    ref_element != root && !root->IsDescendantOf(ref_element))
```
Zero `CreateVisiblePosition` calls.

---

### 8. `editing_commands_utilities.cc` line 442

**Function:** `IsVisiblyAdjacent(const Position& first, const Position& second)` (file-local static)

```cpp
static bool IsVisiblyAdjacent(const Position& first, const Position& second) {
  return CreateVisiblePosition(first).DeepEquivalent() ==
         CreateVisiblePosition(MostBackwardCaretPosition(second))
             .DeepEquivalent();
}
```

**What it does:**
Returns true if two positions are visibly adjacent — i.e., there is no visible
content between them when whitespace and non-rendered nodes are ignored. Used by
`CanMergeLists` to decide whether two adjacent list elements can be merged into one.

The right-hand side first calls `MostBackwardCaretPosition(second)` (which moves
`second` to its most-backward canonical caret position, skipping invisible content),
then canonicalizes the result through a VP. The left-hand side canonicalizes `first`.
Equality means the two positions render at the same caret location.

**Why VP is used:**
Caret-equivalence check. `MostBackwardCaretPosition` already returns a `Position`,
but both sides still need canonicalization via VP to ensure they are in comparable
forms.

**Anti-pattern classification:** P1 — create-then-extract used as a caret-equivalence
check. `CanMergeLists` is called with a `DCHECK(!NeedsLayoutTreeUpdate(...))` guard
on both arguments, so layout is already clean; the VP calls trigger two unnecessary
`UpdateStyleAndLayout` calls.

**Replacement:**
```cpp
static bool IsVisiblyAdjacent(const Position& first, const Position& second) {
  return CanonicalPositionOf(first) ==
         CanonicalPositionOf(MostBackwardCaretPosition(second));
}
```

---

### 9. `insert_list_command_test.cc` lines 234–235 (test file)

```cpp
ASSERT_EQ(CreateVisiblePosition(base).DeepEquivalent(), base);
ASSERT_EQ(CreateVisiblePosition(extent).DeepEquivalent(), extent);
```

**What it does:**
Test assertions confirming that the positions passed into `InSameTreeAndOrdered`
satisfy its canonicality precondition. Mirrors the DCHECKs in site 6.

**Anti-pattern classification:** Test-only. No production cost. Can be updated to
`CanonicalPositionOf` for consistency when site 6 is updated.

---

## Summary table

| # | File | Line | Function | Anti-pattern | Layout updates fired | Replacement |
|---|------|------|----------|--------------|----------------------|-------------|
| 1 | `replace_selection_command.cc` | 170–171 | `PositionAvoidingPrecedingNodes` | P1 caret-eq | 2× per loop iter | `CanonicalPositionOf(pos) != CanonicalPositionOf(next_position)` |
| 2 | `delete_selection_command.cc` | 149–153 | `InitializePositionData` | P1 caret-eq | 4× per loop iter | 4× `CanonicalPositionOf(...)` |
| 3 | `insert_text_command.cc` | 299 | `InsertTab` | P1 create-then-extract | 2× (one redundant) | `CanonicalPositionOf(pos)` |
| 4 | `insert_paragraph_separator_command.cc` | 279 | `DoApply` | P1 create-then-extract | 2× (one redundant) | `CanonicalPositionOf(insertion_position)` |
| 5 | `insert_paragraph_separator_command.cc` | 621 | `DoApply` | P1 caret-eq + `VP::BeforeNode` | 2× | `CanonicalPositionOf(x) != CanonicalPositionOf(Position::BeforeNode(y))` |
| 6 | `insert_list_command.cc` | 138–143 | `InSameTreeAndOrdered` | DCHECK only (no release cost) | 0 in release | `DCHECK_EQ(pos, CanonicalPositionOf(pos))` |
| 7 | `format_block_command.cc` | 97–100 | `FormatRange` | P2 missing overloads | 4× (all redundant) | `IsStartOfBlock(pos)` / `IsEndOfBlock(pos)` (Step 1 overloads) |
| 8 | `editing_commands_utilities.cc` | 442 | `IsVisiblyAdjacent` | P1 caret-eq | 2× (layout already clean) | `CanonicalPositionOf(first) == CanonicalPositionOf(MostBackwardCaretPosition(second))` |
| 9 | `insert_list_command_test.cc` | 234–235 | test | test-only | — | `CanonicalPositionOf` for consistency |

---

## Key observations

**All sites are P1 or P2.** None use the `VisiblePosition` for affinity, layout
object access, or any VP-specific state. The VP wrapper is discarded at every site.

**`CanonicalPositionOf` is the common replacement for P1.** It calls
`MostForwardCaretPosition(MostBackwardCaretPosition(pos))` and returns a `Position`
directly, with a single layout-read (not a full `UpdateStyleAndLayout` trigger on
each call the way `CreateVisiblePosition` does when layout is stale). Where an
explicit `UpdateStyleAndLayout` already precedes the call (sites 3, 4, 7), layout is
already clean and `CanonicalPositionOf` reads the existing layout state at zero
additional cost.

**Site 7 is the highest-value fix.** Four `CreateVisiblePosition` calls in a single
`if` expression, all redundant after the explicit layout gate above, all removable
with Step 1 `Position` overloads for `IsStartOfBlock` / `IsEndOfBlock`.

**Site 2 is the most dangerous.** The four VP calls are inside a `while` loop in
`InitializePositionData`, meaning they fire multiple times per delete operation. Each
loop iteration triggers 4 full `UpdateStyleAndLayout` calls; the replacements trigger
zero.
