[← Chapter 6: When VisiblePosition Is Triggered](06_when_visible_position_is_triggered.md) | [Home](README.md) | [Chapter 8: Special Handling →](08_special_handling_elements.md)

---

# Appendix A: Complete Function Reference

This appendix documents **every** function involved in the VisiblePosition computation pipeline, including private and utility functions. Organized by source file.

---

## A.1 visible_position.h / visible_position.cc

### `VisiblePositionTemplate<Strategy>::VisiblePositionTemplate()`
- **Args**: None
- **Returns**: Default (null) VisiblePosition
- **Does**: Initializes with null position, zero version numbers (DCHECK mode)

### `VisiblePositionTemplate<Strategy>::VisiblePositionTemplate(const PositionWithAffinityTemplate<Strategy>&)`
- **Args**: `position_with_affinity` — the canonical position + affinity
- **Returns**: VisiblePosition wrapping the given position
- **Does**: Private constructor. Stores the position_with_affinity and captures DOM/style versions for validity checking
- **Visibility**: Private — only called from `Create()`

### `VisiblePositionTemplate<Strategy>::Create(const PositionWithAffinityTemplate<Strategy>&)` 
- **Args**: `position_with_affinity` — raw position with affinity
- **Returns**: `VisiblePositionTemplate<Strategy>` — the canonical visible position
- **Does**: The main factory method. Canonicalizes the position, resolves affinity. See Chapter 2 for full algorithm. DCHECKs: connected, valid, no layout update needed. Uses `DocumentLifecycle::DisallowTransitionScope`.

### `VisiblePositionTemplate<Strategy>::DeepEquivalent() const`
- **Args**: None
- **Returns**: `PositionTemplate<Strategy>` — the canonical position
- **Does**: Returns the position from the stored position_with_affinity

### `VisiblePositionTemplate<Strategy>::ToParentAnchoredPosition() const`
- **Args**: None
- **Returns**: `PositionTemplate<Strategy>` — parent-anchored equivalent
- **Does**: Calls `DeepEquivalent().ParentAnchoredEquivalent()`

### `VisiblePositionTemplate<Strategy>::ToPositionWithAffinity() const`
- **Args**: None
- **Returns**: `PositionWithAffinityTemplate<Strategy>`
- **Does**: Returns the stored position_with_affinity

### `VisiblePositionTemplate<Strategy>::Affinity() const`
- **Args**: None
- **Returns**: `TextAffinity` — UPSTREAM or DOWNSTREAM
- **Does**: Returns the affinity from the stored position_with_affinity

### `VisiblePositionTemplate<Strategy>::IsValid() const`
- **Args**: None
- **Returns**: `bool`
- **Does**: In DCHECK builds, checks DOM tree version and style version haven't changed since creation, and no layout update is needed. Always returns true in release.

### `VisiblePositionTemplate<Strategy>::IsValidFor(const Document&) const`
- **Args**: `document` — the document to check against
- **Returns**: `bool`
- **Does**: Delegates to `position_with_affinity_.IsValidFor(document)`

### `VisiblePositionTemplate<Strategy>::IsNull() const`
- **Args**: None
- **Returns**: `bool`
- **Does**: Returns true if position_with_affinity is null

### `VisiblePositionTemplate<Strategy>::IsNotNull() const`
- **Args**: None
- **Returns**: `bool`
- **Does**: Returns true if position_with_affinity is not null

### `VisiblePositionTemplate<Strategy>::IsOrphan() const`
- **Args**: None
- **Returns**: `bool`
- **Does**: Returns `DeepEquivalent().IsOrphan()` — non-null but disconnected

### Static Factory: `AfterNode(const Node&)`, `BeforeNode(const Node&)`, `FirstPositionInNode(const Node&)`, `LastPositionInNode(const Node&)`, `InParentAfterNode(const Node&)`, `InParentBeforeNode(const Node&)`
- **Args**: A DOM node
- **Returns**: `VisiblePositionTemplate<Strategy>`
- **Does**: Creates a `PositionWithAffinity` from the corresponding `Position::XxxNode()` factory and calls `Create()`

### `CreateVisiblePosition(const Position&, TextAffinity)` — Free function
- **Args**: `position` — raw DOM position, `affinity` — defaults to `kDefault`
- **Returns**: `VisiblePosition`
- **Does**: Wraps position in `PositionWithAffinity` and calls `VisiblePosition::Create()`

### `CreateVisiblePosition(const PositionWithAffinity&)` — Free function
- **Args**: `position_with_affinity`
- **Returns**: `VisiblePosition`
- **Does**: Directly calls `VisiblePosition::Create()`

### `InDifferentLinesOfSameInlineFormattingContext(position1, position2)` — Static helper
- **Args**: Two `PositionWithAffinityTemplate<Strategy>`
- **Returns**: `bool`
- **Does**: Returns true if positions are on different lines but in the same inline formatting context. Used in `Create()` to detect when canonicalization jumped to a previous line.

---

## A.2 visible_units.cc

### `CanonicalPositionOf(const Position&)` / `CanonicalPositionOf(const PositionInFlatTree&)`
- **Args**: A raw position
- **Returns**: The canonical position (leftmost valid candidate)
- **Does**: Delegates to `CanonicalPosition<Strategy>()`. See Chapter 2.3.

### `CanonicalPosition(const PositionType& position)` — Static template
- **Args**: A raw position
- **Returns**: Canonical position
- **Does**: 
  1. Try `MostBackwardCaretPosition` → check if `IsVisuallyEquivalentCandidate`
  2. Try `MostForwardCaretPosition` → check if candidate
  3. Search wider with `NextCandidate`/`PreviousCandidate`
  4. Apply editable root constraints
  5. Apply same-block preference
- **Bugs**: [crbug.com/472258](https://crbug.com/472258), [issues.chromium.org/40890187](https://issues.chromium.org/40890187)
- **FIXME**: FIXME (9535) — leftmost bias causes caret painting issues

### `CanonicalizeCandidate(const PositionType& candidate)` — Static template
- **Args**: A candidate position (must be `IsVisuallyEquivalentCandidate`)
- **Returns**: The upstream equivalent if also a candidate, otherwise the original
- **Does**: Tries `MostBackwardCaretPosition` on the candidate; returns whichever is valid

### `InSameBlock(const Node*, const Node*)` — Static
- **Args**: `original_node`, `new_position_node`
- **Returns**: `bool`
- **Does**: Checks if two nodes are in the same block flow element. Special handling: if both editable, checks DOM hierarchy. If `new_position_node` is descendant of `original_node`, returns true.
- **Referenced tests**: `editing/execCommand/indent-pre-list.html`, `editing/execCommand/indent-pre.html`

### `MostBackwardCaretPosition(const Position&, EditingBoundaryCrossingRule, SnapToClient)` → `Position`
- **Args**: `position`, `rule` (default: `kCannotCrossEditingBoundary`), `client` (default: `kOthers`)
- **Returns**: The most upstream visually equivalent caret position
- **Does**: Wrapper that converts to flat tree, calls the flat tree algorithm, then adjusts for shadow boundaries. See Chapter 5.8.
- **FIXME**: PositionIterator should respect Before/After positions
- **Bug**: [crbug.com/1248744](https://crbug.com/1248744)

### `MostBackwardCaretPosition<Strategy>(position, rule, client)` — Static template
- **Args**: Same as above
- **Returns**: Flat tree position
- **Does**: The core backward iteration algorithm. See Chapter 2.6 for full flowchart.

### `MostForwardCaretPosition(const Position&, EditingBoundaryCrossingRule, SnapToClient)` → `Position`
- **Args**: Same as backward
- **Returns**: The most downstream visually equivalent caret position
- **Does**: Wrapper + flat tree algorithm, mirror of backward

### `MostForwardCaretPosition<Strategy>(position, rule, client)` — Static template
- **Args**: Same
- **Returns**: Flat tree position
- **Does**: Core forward iteration algorithm

### `MostBackwardOrForwardCaretPosition(position, rule, client, AlgorithmInFlatTree)` — Static
- **Args**: DOM position + flat tree algorithm function pointer
- **Returns**: DOM position
- **Does**: Converts to flat tree, calls the algorithm, converts back, adjusts for shadow boundaries
- **Special handling**: Fast path when no shadow DOM involved

### `IsVisuallyEquivalentCandidate(const Position&)` / `(const PositionInFlatTree&)` → `bool`
- **Args**: A position
- **Returns**: Whether this is a valid caret position
- **Does**: Delegates to `IsVisuallyEquivalentCandidateAlgorithm`. See Chapter 2.8.

### `IsVisuallyEquivalentCandidateAlgorithm<Strategy>(position)` — Static template
- **Args**: A position
- **Returns**: `bool`
- **Does**: Full candidate check: layout object, visibility, display lock, BR, text, SVG, table, editing boundary.
- **TODO(leviw)**: Condition for BR should be `kBeforeAnchor`

### `EndsOfNodeAreVisuallyDistinctPositions(const Node*)` → `bool`
- **Args**: A DOM node
- **Returns**: Whether start and end of node are visually distinct
- **Does**: True for non-inline, `<marquee>`, empty inline-blocks. False for inline tables, null.
- — `<marquee>` elements are moving so assume always distinct

### `EnclosingVisualBoundary<Strategy>(Node*)` — Static template
- **Args**: A node
- **Returns**: The nearest ancestor where ends are visually distinct
- **Does**: Walks up `Strategy::Parent` until `EndsOfNodeAreVisuallyDistinctPositions` returns true

### `IsStreamer<Strategy>(const PositionIteratorAlgorithm<Strategy>&)` — Static template
- **Args**: A position iterator
- **Returns**: `bool`
- **Does**: True if at null node, atomic node, or start of node

### `AdjustPositionForBackwardIteration<Strategy>(position)` — Static template
- **Args**: A position
- **Returns**: Adjusted position for backward iteration
- **Does**: For `kAfterAnchor` + `user-select:contain` → `ToOffsetInAnchor()`. Otherwise → `EditingPositionOf(node, CaretMaxOffset)`.

### `CanHaveCaretPosition(const Node&)` — Static
- **Args**: A node
- **Returns**: `bool`
- **Does**: True for non-SVG, SVG `<text>` ([crbug.com/891908](https://crbug.com/891908)), SVG `<foreignObject>` ([crbug.com/1348816](https://crbug.com/1348816)). False for other SVG elements.

### `HasInvisibleFirstLetter(const Node*)` — Static (namespace)
- **Args**: A node
- **Returns**: `bool`
- **Does**: True if the node is a text node with a first-letter pseudo-element that is invisible

### `InRenderedText<Strategy>(position)` — Static template
- **Args**: A position
- **Returns**: `bool`
- **Does**: True if position is in rendered (non-collapsed) text. Checks: text node, valid layout object, `ContainsCaretOffset`, grapheme boundary.
- **TODO(editing-dev)**: Should use `offset_in_node` instead of `text_offset`

### `AtEditingBoundary<Strategy>(position)` — Static template
- **Args**: A position
- **Returns**: `bool`
- **Does**: True if position is at an editing boundary (adjacent to non-editable content)

### `HasRenderedNonAnonymousDescendantsWithHeight(const LayoutObject*)` → `bool`
- **Args**: A layout object
- **Returns**: Whether it has non-anonymous descendants with height
- **Does**: Skips display-locked, anonymous, pseudo elements. Checks text, boxes, inline objects.
- **TODO(editing-dev)**: Semantics wrong for one-letter first-letter blocks

### `NextPositionOfAlgorithm<Strategy>(position, rule)` — Static template
- **Args**: `PositionWithAffinity`, `EditingBoundaryCrossingRule`
- **Returns**: `VisiblePositionTemplate<Strategy>`
- **Does**: Calls `NextVisuallyDistinctCandidate`, creates VisiblePosition, applies boundary rule

### `PreviousPositionOfAlgorithm<Strategy>(position, rule)` — Static template
- **Args**: `Position`, `EditingBoundaryCrossingRule`
- **Returns**: `VisiblePositionTemplate<Strategy>`
- **Does**: Calls `PreviousVisuallyDistinctCandidate`, creates VisiblePosition, applies boundary rule

### `NextPositionOf(const VisiblePosition&, EditingBoundaryCrossingRule)` → `VisiblePosition`
### `PreviousPositionOf(const VisiblePosition&, EditingBoundaryCrossingRule)` → `VisiblePosition`
- Public API wrappers calling the Algorithm versions

### `CharacterAfterAlgorithm<Strategy>(visible_position)` — Static template
- **Args**: A visible position
- **Returns**: `UChar32`
- **Does**: Gets `MostForwardCaretPosition`, reads character from text node at that offset

### `CharacterBeforeAlgorithm<Strategy>(visible_position)` — Static template
- **Args**: A visible position
- **Returns**: `UChar32`
- **Does**: `CharacterAfter(PreviousPositionOf(visible_position))`

### `SkipToEndOfEditingBoundary<Strategy>(pos, anchor)` — Static template
- **Args**: Candidate position, original anchor
- **Returns**: Adjusted position
- **Does**: If different editable roots, skips to end of the editable boundary

### `SkipToStartOfEditingBoundary<Strategy>(pos, anchor)` — Static template
- **Args**: Candidate position, original anchor
- **Returns**: Adjusted position
- **Does**: Mirror — skips to start of editing boundary

### `AdjustBackwardPositionToAvoidCrossingEditingBoundariesTemplate<Strategy>(pos, anchor)` — Static template
- **Args**: `PositionWithAffinity`, `Position` anchor
- **Returns**: `PositionWithAffinity`
- **Does**: Returns pos if same editable root as anchor. Returns empty if pos is outside root. Returns `LastEditablePositionBeforePositionInRoot()` otherwise.

### `AdjustForwardPositionToAvoidCrossingEditingBoundariesTemplate<Strategy>(pos, anchor)` — Static template
- **Args**: `PositionWithAffinity`, `Position` anchor
- **Returns**: `PositionWithAffinity`
- **Does**: Mirror of backward. Returns `FirstEditablePositionAfterPositionInRoot()`.

### `ParentEditingBoundary<Strategy>(position)` — Static template
- **Args**: A position
- **Returns**: `Node*`
- **Does**: Walks up ancestors until editability changes (crossing the editing boundary)

### `StartOfDocumentAlgorithm<Strategy>(position)` — Static template
- **Args**: A position
- **Returns**: Position at `FirstPositionInNode(*documentElement)`

### `EndOfDocumentAlgorithm<Strategy>(visible_position)` — Static template
- **Args**: A visible position
- **Returns**: VisiblePosition at `LastPositionInNode(*documentElement)`

### `RendersInDifferentPosition(position1, position2)` → `bool`
- **Args**: Two positions
- **Returns**: Whether they render at different visual locations
- **Does**: Compares `LocalCaretRectOfPosition` → `LocalToAbsoluteQuadOf`

### `PositionForContentsPointRespectingEditingBoundary(point, frame)` → `PositionWithAffinity`
- **Args**: A point in contents coordinates, a frame
- **Returns**: Hit-tested position respecting editing boundaries
- **Does**: `HitTestResult` with move/readonly/active/ignore-clipping flags

### `SkipWhitespaceAlgorithm<Strategy>(position)` — Static template
- **Args**: A position
- **Returns**: Position after whitespace
- **Does**: Iterates forward skipping spaces (but not newlines)
- **TODO(editing-dev)**: Should consider U+20E3 (COMBINING ENCLOSING KEYCAP)

### `MakeSearchRange<Strategy>(pos)` — Static template
- **Args**: A position
- **Returns**: `EphemeralRange` from pos to end of enclosing block
- **Does**: Helper for whitespace skipping

---

## A.3 editing_utilities.cc / editing_utilities.h

### `EditingIgnoresContent(const Node&)` → `bool`
- **Args**: A node
- **Returns**: True if node has no content for editing purposes
- **Does**: `!node.CanContainRangeEndPoint() || IsEmptyNonEditableNodeInEditable(node)`
- **TODO(yosin)**: Should not use `IsEmptyNonEditableNodeInEditable()` since it requires clean layout

### `IsAtomicNode(const Node*)` → `bool`
- **Args**: A node
- **Returns**: True if no children or content ignored for editing
- **Does**: `!node->hasChildren() || EditingIgnoresContent(*node)`

### `IsAtomicNodeInFlatTree(const Node*)` → `bool`
- **Args**: A node
- **Returns**: Same as IsAtomicNode but uses `FlatTreeTraversal::HasChildren()`

### `IsEditable(const Node&)` → `bool`
- **Args**: A node
- **Returns**: True if node is content-editable
- **Does**: Checks `ComputedStyle::UsedUserModify()` up the ancestor chain
- **TODO**: [crbug.com/667681](https://crbug.com/667681) — shouldn't check in inactive documents

### `IsRichlyEditable(const Node&)` → `bool`
- **Args**: A node
- **Returns**: True if node is richly editable (not plaintext-only)
- **Does**: Same as IsEditable but excludes `read-write-plaintext-only`

### `IsEditablePosition(const Position&)` → `bool`
- **Args**: A position
- **Returns**: True if position is editable
- **Does**: Gets container node, adjusts for tables, calls `IsEditable()`

### `IsRichlyEditablePosition(const Position&)` → `bool`
- **Args**: A position
- **Returns**: True if position is richly editable
- **Does**: Same as IsEditablePosition but calls `IsRichlyEditable()`

### `RootEditableElement(const Node&)` → `Element*`
- **Args**: A node
- **Returns**: Highest editable ancestor Element, stopping at body
- **Does**: Walks ancestors while editable, tracks last Element

### `RootEditableElementOf(const Position&)` → `Element*`
- **Args**: A position
- **Returns**: Root editable element for the position
- **Does**: Gets container, adjusts for tables, calls `RootEditableElement()`

### `HighestEditableRoot(const Position&)` → `ContainerNode*`
- **Args**: A position
- **Returns**: Highest editable root (may be Document for designMode)
- **Does**: Searches ancestors for highest editable, ignoring editing boundaries

### `IsRootEditableElement(const Node&)` → `bool`
- **Args**: A node
- **Returns**: True if node is a root editable element
- **Does**: Is editable, is Element, parent not editable or is body

### `IsEmptyNonEditableNodeInEditable(const Node&)` → `bool`
- **Args**: A node
- **Returns**: True if empty, non-editable, within editable parent
- **Does**: No children + !IsEditable + parent IsEditable
- **Bug**: [crbug.com/428986](https://crbug.com/428986)

### `NextCandidate(const Position&)` → `Position`
- **Args**: A position
- **Returns**: Next visually equivalent candidate
- **Does**: Iterates forward with PositionIterator, returns first `IsVisuallyEquivalentCandidate`

### `PreviousCandidate(const Position&)` → `Position`
- **Args**: A position
- **Returns**: Previous visually equivalent candidate
- **Does**: Iterates backward

### `NextVisuallyDistinctCandidate(const Position&, EditingBoundaryCrossingRule)` → `Position`
- **Args**: Position + boundary rule
- **Returns**: Next candidate whose forward/backward caret positions differ from the start's
- **Does**: Skips candidates that are visually equivalent to the starting position
- **Flag**: `SkipNonEditableInAtomicMoveEnabled` — can skip non-editable regions

### `PreviousVisuallyDistinctCandidate(const Position&, EditingBoundaryCrossingRule)` → `Position`
- **Args**: Position + boundary rule
- **Returns**: Mirror of NextVisuallyDistinctCandidate

### `FirstEditablePositionAfterPositionInRoot(const Position&, const Node&)` → `Position`
- **Args**: Position + highest root node
- **Returns**: First editable position after the given position within root
- **Does**: Handles shadow trees, uses NextCandidate/NextVisuallyDistinctCandidate
- **Bugs**: [crbug.com/571420](https://crbug.com/571420), [crbug.com/1334557](https://crbug.com/1334557), [crbug.com/1406207](https://crbug.com/1406207)

### `LastEditablePositionBeforePositionInRoot(const Position&, const Node&)` → `Position`
- **Args**: Position + highest root node
- **Returns**: Last editable position before the given position within root
- **Does**: Mirror of First...

### `PreviousPositionOf(const Position&, PositionMoveType)` → `Position`
- **Args**: Position + move type (CodeUnit/BackwardDeletion/GraphemeCluster)
- **Returns**: Previous position
- **Does**: Handles EditingIgnoresContent, text decrements, parent walking

### `NextPositionOf(const Position&, PositionMoveType)` → `Position`
- **Args**: Position + move type
- **Returns**: Next position
- **Does**: Mirror of PreviousPositionOf

### `EnclosingBlockFlowElement(const Node&)` → `Element*`
- **Args**: A node
- **Returns**: Enclosing block flow element (deprecated — use EnclosingBlock)
- **Does**: Walks ancestors looking for IsBlockFlowElement or body

### `EnclosingBlock(const Node*, EditingBoundaryCrossingRule)` → `Element*`
- **Args**: Node + boundary rule
- **Returns**: Enclosing block element
- **Does**: Uses `EnclosingNodeOfType` with `IsEnclosingBlock`

### `IsDisplayInsideTable(const Node*)` → `bool`
- **Args**: A node
- **Returns**: True if node is an `<table>` with layout object

### `IsTableCell(const Node*)` → `bool`
- **Args**: A node
- **Returns**: True if layout object is table cell

### `CanHaveChildrenForEditing(const Node*)` → `bool`
- **Args**: A node
- **Returns**: True if not text and can contain range endpoints

### `IsUserSelectContain(const Node&)` → `bool`
- **Args**: A node
- **Returns**: True for textarea, input, select
- **TODO**: Should check CSS user-select property

### `AssociatedLayoutObjectOf(const Node&, int, LayoutObjectSide)` → `LayoutObject*`
- **Args**: Node + offset + side (for first-letter boundary)
- **Returns**: The layout object associated with the position
- **Does**: Handles first-letter pseudo-element boundaries

### `AdjustForEditingBoundary(const PositionWithAffinity&)` → `PositionWithAffinity`
- **Args**: A position
- **Returns**: Adjusted position that respects editing boundaries
- **Does**: Moves positions out of non-editable into adjacent editable

### `PositionRespectingEditingBoundary(const Position&, const HitTestResult&)` → `PositionWithAffinity`
- **Args**: Selection start + hit test result
- **Returns**: Position adjusted for editing boundary and user-select:contain
- **Bugs**: [crbug.com/185089](https://crbug.com/185089), [crbug.com/658129](https://crbug.com/658129)

### `ComputePositionForNodeRemoval(const Position&, const Node&)` → `Position`
- **Args**: A position + node being removed
- **Returns**: Adjusted position after the node removal
- **Does**: Handles all PositionAnchorType variants

### `NeedsLayoutTreeUpdate(const Position&)` / `NeedsLayoutTreeUpdate(const Node&)` → `bool`
- **Args**: Position or Node
- **Returns**: Whether layout tree needs update
- **Does**: Checks document needsLayoutTreeUpdate and view needsLayout
- **TODO(yosin)**: document::needsLayoutTreeUpdate should check LayoutView::needsLayout

---

## A.4 ng_flat_tree_shorthands.h / ng_flat_tree_shorthands.cc

### `NGInlineFormattingContextOf(const Position&)` → `const LayoutBlockFlow*`
- **Args**: A DOM position
- **Returns**: The NG inline formatting context (LayoutBlockFlow) containing the position
- **Does**: Looks up the offset mapping for the position

### `NGInlineFormattingContextOf(const PositionInFlatTree&)` → `const LayoutBlockFlow*`
- **Args**: A flat tree position
- **Returns**: Same, converts to DOM tree first

### `InSameNGLineBox(PositionInFlatTreeWithAffinity, PositionInFlatTreeWithAffinity)` → `bool`
- **Args**: Two flat tree positions with affinity
- **Returns**: Whether they're in the same NG line box
- **Does**: Converts to DOM tree, calls DOM version

---

## A.5 local_caret_rect.h / local_caret_rect.cc

### `struct LocalCaretRect`
- **Fields**: `layout_object` (LayoutObject*), `rect` (PhysicalRect), `root_box_fragment`
- **Does**: Represents a caret rectangle in local coordinates

### `LocalCaretRectOfPosition(const PositionWithAffinity&, CaretShape, EditingBoundaryCrossingRule)` → `LocalCaretRect`
- **Args**: Position with affinity + caret shape + boundary rule
- **Returns**: Local-coordinates caret rect

### `AbsoluteCaretBoundsOf(const PositionWithAffinity&, CaretShape, EditingBoundaryCrossingRule)` → `gfx::Rect`
- **Args**: Same
- **Returns**: Absolute-coordinates bounding rect
- **Does**: Gets LocalCaretRect, converts via LocalToAbsoluteQuadOf

### `LocalToAbsoluteQuadOf(const LocalCaretRect&)` → `gfx::QuadF`
- **Args**: A local caret rect
- **Returns**: Absolute quad
- **Does**: Converts via layout_object's local-to-absolute mapping

---

## A.6 selection_adjuster.h

### `SelectionAdjuster::AdjustSelectionRespectingGranularity`
- **Args**: `SelectionTemplate`, `TextGranularity`, `WordInclusion`
- **Returns**: Adjusted `SelectionTemplate`
- **Does**: Expands to word/sentence/line/paragraph/document boundaries

### `SelectionAdjuster::AdjustSelectionToAvoidCrossingShadowBoundaries`
- **Args**: `SelectionTemplate`
- **Returns**: Adjusted `SelectionTemplate`
- **Does**: Prevents selection spanning shadow DOM boundaries

### `SelectionAdjuster::AdjustSelectionToAvoidCrossingEditingBoundaries`
- **Args**: `SelectionTemplate`
- **Returns**: Adjusted `SelectionTemplate`
- **Does**: Prevents selection spanning editable/non-editable boundaries

### `SelectionAdjuster::AdjustSelectionType`
- **Args**: `SelectionTemplate`
- **Returns**: Type-adjusted `SelectionTemplate`
- **Does**: If positions equal → caret; otherwise → range

---

## A.7 editing_boundary.h

### `enum EditingBoundaryCrossingRule`
| Value | Description |
|-------|-------------|
| `kCanCrossEditingBoundary` | Allow crossing between editable and non-editable |
| `kCannotCrossEditingBoundary` | Stop at editing boundary |
| `kCanSkipOverEditingBoundary` | Skip over non-editable regions entirely |

---

## A.8 text_affinity.h

### `enum class TextAffinity`
| Value | Description |
|-------|-------------|
| `kDefault` | Same as `kDownstream` |
| `kDownstream` | Caret at start of next line (after wrap) |
| `kUpstream` | Caret at end of current line (before wrap) |

---

## A.9 editing_strategy.h

### `EditingStrategy`
- Tree traversal for DOM tree. Uses `NodeTraversal` for parent/child/sibling operations.

### `EditingInFlatTreeStrategy`
- Tree traversal for flat tree. Uses `FlatTreeTraversal` for parent/child/sibling operations, resolving shadow DOM.

---

## A.10 position_iterator.h

### `PositionIteratorAlgorithm<Strategy>`
- **Does**: Iterates through all positions in document order
- **Key methods**: `Increment()`, `Decrement()`, `GetNode()`, `OffsetInTextNode()`, `AtStart()`, `AtEnd()`, `AtStartOfNode()`, `AtEndOfNode()`, `ComputePosition()`, `DeprecatedComputePosition()`
- Used by `MostBackwardCaretPosition` and `MostForwardCaretPosition` for iteration

---

## A.11 visible_selection.cc

### `CanonicalizeSelection<Strategy>(const SelectionTemplate<Strategy>&)` — Static template
- **Args**: A selection
- **Returns**: Canonicalized selection with visible positions
- **Does**: Calls `CreateVisiblePosition()` on anchor and focus, builds new selection from their `DeepEquivalent()` positions

### `VisibleSelectionTemplate<Strategy>::Creator::ComputeVisibleSelection(...)` — Static
- **Args**: `SelectionTemplate`, `TextGranularity`, `WordInclusion`
- **Returns**: Adjusted `SelectionTemplate`
- **Does**: The 5-step pipeline: canonicalize → granularity → shadow → editing → type

### `NormalizeRangeAlgorithm<Strategy>(selection)` — Static template
- **Args**: A selection
- **Returns**: `EphemeralRange`
- **Does**: For carets: `MostBackwardCaretPosition(start).ParentAnchoredEquivalent()`. For ranges: normalized min range.
