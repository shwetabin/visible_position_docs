# VisiblePosition in Blink — Complete Documentation

## Table of Contents

| # | Chapter | Description |
|---|---------|-------------|
| 1 | [Overview: VisiblePosition vs DOM Position](01_overview_visible_position_vs_dom_position.md) | What is VisiblePosition, how it differs from DOM Position, key terminology (DeepEquivalent, ParentAnchoredEquivalent, IsVisuallyEquivalentCandidate, RootEditableElement, HighestEditableRoot), two tree strategies, class relationships |
| 2 | [How VisiblePosition Is Computed](02_how_visible_position_is_computed.md) | The complete computation flow from raw DOM position to VisiblePosition. Detailed algorithms for `VisiblePosition::Create()`, `CanonicalPositionOf()`, `MostBackwardCaretPosition()`, `MostForwardCaretPosition()`, `IsVisuallyEquivalentCandidate()`, with mermaid flowcharts and sequence diagrams |
| 3 | [Position and PositionTemplate Classes](03_position_and_position_template_classes.md) | `PositionTemplate<Strategy>`, `PositionAnchorType`, constructors, static factories, key methods, `ParentAnchoredEquivalent` deep dive, `PositionWithAffinityTemplate`, tree conversion functions |
| 4 | [Selection and SelectionTemplate Classes](04_selection_and_selection_template_classes.md) | `SelectionTemplate<Strategy>` with Builder pattern, `VisibleSelectionTemplate<Strategy>`, the 5-step VisibleSelection creation pipeline, `SelectionAdjuster` methods |
| 5 | [Files Starting with `visible_`](05_visible_star_files_reference.md) | Complete reference for `visible_position.*`, `visible_selection.*`, `visible_units.*`, `visible_units_line.cc`, `visible_units_word.cc`, `visible_units_sentence.cc`, `visible_units_paragraph.cc` |
| 6 | [When VisiblePosition Is Triggered](06_when_visible_position_is_triggered.md) | User interactions (mouse, keyboard, touch), JS APIs, editing commands, spell check, IME, accessibility, find-in-page. Sequence diagrams showing the trigger chain |
| 7 | [Appendix: Complete Function Reference](07_appendix_function_reference.md) | Every function involved in VisiblePosition computation — name, args, return type, behavior, bugs, TODOs |
| 8 | [Special Handling: Non-Editable, Tables, Hidden, Text, SVG](08_special_handling_elements.md) | How each element type is treated differently in position computation. Non-editable boundaries, table position adjustment, display:none/visibility:hidden, rendered vs collapsed text, SVG restrictions, BR elements, empty inlines, writing mode boundaries |

## Key Source Files

| File | Purpose |
|------|---------|
| `visible_position.h` / `.cc` | VisiblePosition class and Create() |
| `visible_units.h` / `.cc` | CanonicalPositionOf, MostForward/BackwardCaretPosition, IsVisuallyEquivalentCandidate, Next/PreviousPositionOf |
| `position.h` / `.cc` | PositionTemplate class, ParentAnchoredEquivalent, ToPositionInFlatTree |
| `position_with_affinity.h` / `.cc` | PositionWithAffinityTemplate class |
| `selection_template.h` / `.cc` | SelectionTemplate class and Builder |
| `visible_selection.h` / `.cc` | VisibleSelectionTemplate, CanonicalizeSelection, creation pipeline |
| `editing_utilities.h` / `.cc` | IsEditable, EditingIgnoresContent, IsAtomicNode, NextCandidate, etc. |
| `selection_adjuster.h` / `.cc` | Granularity, shadow, editing boundary adjustments |
| `editing_boundary.h` | EditingBoundaryCrossingRule enum |
| `ng_flat_tree_shorthands.h` / `.cc` | NGInlineFormattingContextOf, InSameNGLineBox |
| `local_caret_rect.h` / `.cc` | LocalCaretRect, AbsoluteCaretBoundsOf |
| `visible_units_line.cc` | InSameLine, StartOfLine, EndOfLine |
| `visible_units_word.cc` | Word boundary functions |
| `visible_units_sentence.cc` | Sentence boundary functions |
| `visible_units_paragraph.cc` | Paragraph boundary functions |

## Master Bug Reference

| Bug | Description | Referenced In |
|-----|-------------|---------------|
| [crbug.com/472258](https://crbug.com/472258) | Expensive selection position updates | CanonicalPosition TRACE_EVENT |
| [crbug.com/648949](https://crbug.com/648949) | Clients store VisiblePosition and inspect after mutation | VisiblePosition.h TODO |
| [crbug.com/735327](https://crbug.com/735327) | anchor_node_ should be const | Position constructors |
| [crbug.com/761173](https://crbug.com/761173) | IsConnected() expensive for flat tree | PositionTemplate |
| [crbug.com/889737](https://crbug.com/889737) | Before/After anchor parent requirement | ComputeContainerNode |
| [crbug.com/891908](https://crbug.com/891908) | SVG `<text>` caret support | CanHaveCaretPosition |
| [crbug.com/1348816](https://crbug.com/1348816) | SVG `<foreignObject>` caret support | CanHaveCaretPosition |
| [crbug.com/1248744](https://crbug.com/1248744) | Null adjusted_position debug | MostBackwardCaretPosition |
| [crbug.com/428986](https://crbug.com/428986) | Empty non-editable in editable | IsEmptyNonEditableNodeInEditable |
| [crbug.com/571420](https://crbug.com/571420) | InsertListCommand loop | FirstEditablePositionAfterPositionInRoot |
| [crbug.com/1334557](https://crbug.com/1334557) | Bypass next editable sibling | FirstEditablePositionAfterPositionInRoot |
| [crbug.com/1406207](https://crbug.com/1406207) | NextCandidate vs NextVisuallyDistinct | FirstEditablePositionAfterPositionInRoot |
| [crbug.com/667681](https://crbug.com/667681) | Editable check in inactive documents | IsEditable |
| [crbug.com/185089](https://crbug.com/185089) | `<input readonly>` boundary | PositionRespectingEditingBoundary |
| [crbug.com/658129](https://crbug.com/658129) | user-select:contain handling | PositionRespectingEditingBoundary |
| [crbug.com/590369](https://crbug.com/590369) | UpdateStyleAndLayout audit | editing_utilities |
| [crbug.com/453042](https://crbug.com/453042) | No-break space in space runs | editing_utilities |
| [issues.chromium.org/40890187](https://issues.chromium.org/40890187) | Return position when valid candidate | CanonicalPosition |
| [issues.chromium.org/392725745](https://issues.chromium.org/392725745) | Shadow host style editability | HasEditableLevel |
| [issues.chromium.org/41490809](https://issues.chromium.org/41490809) | Inert attribute non-editable | HasEditableLevel |
| [wkb.ug/63040](http://wkb.ug/63040) | Position(Node*, int) should go away | PositionTemplate constructor |

## Spec References

| Spec | Context |
|------|---------|
| [Unicode Standard Annex #29](http://www.unicode.org/reports/tr29/) | Grapheme cluster segmentation for PositionMoveType::kGraphemeCluster |
| [HTML Spec: inert attribute](https://html.spec.whatwg.org/multipage/interaction.html#the-inert-attribute) | Inert subtree non-editability |
| [W3C Input Events](http://w3c.github.io/editing/input-events.html#dom-inputevent-inputtype) | InputEvent inputType ranges |

## Master FIXME/TODO Reference

| Location | Type | Text |
|----------|------|------|
| CanonicalPosition | FIXME | (9535) Canonicalizing to leftmost candidate causes caret painting issues at line wraps |
| Position operator== | FIXME | `[div, 0] != [img, 0]` even though treated as identical |
| AtFirstEditingPositionForNode | FIXME | Before anchor shouldn't be considered at first editing position |
| ParentAnchoredEquivalent | FIXME | Should only be for legacy positions, also needed for tables |
| InParentBeforeNode | FIXME | Should DCHECK parent — delete-ligature-001.html crashes |
| PositionIterator | FIXME | Should respect Before and After positions |
| MostBackward/Forward | TODO | Should work for positions other than kOffsetInAnchor |
| VisiblePosition.h | TODO | Should have DCHECK(isValid()) in accessors ([crbug.com/648949](https://crbug.com/648949)) |
| VisiblePosition.h | TODO | Will have equals() when use cases arise |
| IsVisuallyEquivalentCandidate (BR) | TODO | Condition should be kBeforeAnchor |
| IsEndOfEditableOrNonEditableContent | TODO | Rename to isLastVisiblePositionOrEndOfInnerEditor |
| HasRenderedNonAnonymousDescendants | TODO | Wrong semantics for one-letter first-letter blocks |
| HasRenderedNonAnonymousDescendants | TODO | Avoid single-character parameter names |
| IsUserSelectContain | TODO | Should check CSS user-select property |
| EditingIgnoresContent | TODO | Should not use IsEmptyNonEditableNodeInEditable (needs clean layout) |
| NeedsLayoutTreeUpdate | TODO | document::needsLayoutTreeUpdate should check needsLayout |
| IsEditable | TODO | Shouldn't check in inactive documents |
| EnclosingBlock | TODO | Deploy everywhere enclosingBlockFlow is used |
| SkipWhitespace | TODO | Consider U+20E3 COMBINING ENCLOSING KEYCAP |
| Various builder | TODO | SetBaseAndExtentDeprecated should be removed |
| Position constructor | TODO | Should not pass nullptr, should be const Node |
