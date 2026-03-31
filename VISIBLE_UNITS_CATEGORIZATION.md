# visible_units.h -- Function Categorization

Every function grouped by where it belongs after this cleanup.

---

## Legend

| Destination | Meaning |
|---|---|
| `position_units.h` -- relocate | Already takes `Position`/`PositionWithAffinity`; move as-is |
| `position_units.h` -- new overload | VP-only today; add a `Position`-based overload |
| `visible_units.h` -- keep | VP-returning navigation; stays |
| `visible_units.h` -- keep (layout utility) | Not a navigation function; stays regardless of type |

---

## Caret / Position Utilities

| Function | Signature | Destination | CL |
|---|---|---|---|
| `CaretMinOffset` | `const Node*` -> `int` | `visible_units.h` -- keep (layout utility) | |
| `CaretMaxOffset` | `const Node*` -> `int` | `visible_units.h` -- keep (layout utility) | |
| `MostBackwardCaretPosition` | `const Position&` -> `Position` | `visible_units.h` -- keep (layout utility) | |
| `MostForwardCaretPosition` | `const Position&` -> `Position` | `visible_units.h` -- keep (layout utility) | |
| `IsVisuallyEquivalentCandidate` | `const Position&` -> `bool` | `visible_units.h` -- keep (layout utility) | |
| `EndsOfNodeAreVisuallyDistinctPositions` | `const Node*` -> `bool` | `visible_units.h` -- keep (layout utility) | |
| `CanonicalPositionOf` | `const Position&` -> `Position` | `visible_units.h` -- keep (layout utility) | |
| `RendersInDifferentPosition` | `const Position&, const Position&` -> `bool` | `visible_units.h` -- keep (layout utility) | |
| `SkipWhitespace` | `const Position&` -> `Position` | `visible_units.h` -- keep (layout utility) | |

These all require clean layout state or answer visual/rendering questions, not navigation-unit boundary questions.

---

## Character

| Function | Signature | Destination | CL |
|---|---|---|---|
| `CharacterAfter` | `const VisiblePosition&` -> `UChar32` | `visible_units.h` -- keep | |
| `CharacterAfter` | `const VisiblePositionInFlatTree&` -> `UChar32` | `visible_units.h` -- keep | |
| `CharacterBefore` | `const VisiblePosition&` -> `UChar32` | `visible_units.h` -- keep | |
| `CharacterBefore` | `const VisiblePositionInFlatTree&` -> `UChar32` | `visible_units.h` -- keep | |
| `CharacterAfter` | `const Position&` -> `UChar32` | `position_units.h` -- **new overload** | CL 1-B |

---

## Position Traversal

| Function | Signature | Destination | CL |
|---|---|---|---|
| `NextPositionOf` | `const Position&` -> `VisiblePosition` | `visible_units.h` -- keep | |
| `NextPositionOf` | `const VisiblePosition&` -> `VisiblePosition` | `visible_units.h` -- keep | |
| `NextPositionOf` | `const VisiblePositionInFlatTree&` -> `VisiblePositionInFlatTree` | `visible_units.h` -- keep | |
| `PreviousPositionOf` | `const VisiblePosition&` -> `VisiblePosition` | `visible_units.h` -- keep | |
| `PreviousPositionOf` | `const VisiblePositionInFlatTree&` -> `VisiblePositionInFlatTree` | `visible_units.h` -- keep | |
| `NextPositionOf` | `const Position&` -> `Position` | `position_units.h` -- **new overload** | CL 1-B |
| `PreviousPositionOf` | `const Position&` -> `Position` | `position_units.h` -- **new overload** | CL 1-B |

---

## Words

| Function | Signature | Destination | CL |
|---|---|---|---|
| `StartOfWordPosition` | `const Position&` -> `Position` | `position_units.h` -- **relocate** | CL 1-A |
| `StartOfWordPosition` | `const PositionInFlatTree&` -> `PositionInFlatTree` | `position_units.h` -- **relocate** | CL 1-A |
| `EndOfWordPosition` | `const Position&` -> `Position` | `position_units.h` -- **relocate** | CL 1-A |
| `EndOfWordPosition` | `const PositionInFlatTree&` -> `PositionInFlatTree` | `position_units.h` -- **relocate** | CL 1-A |
| `MiddleOfWordPosition` | `const Position&, const Position&` -> `Position` | `position_units.h` -- **relocate** | CL 1-A |
| `MiddleOfWordPosition` | `const PositionInFlatTree&, const PositionInFlatTree&` -> `PositionInFlatTree` | `position_units.h` -- **relocate** | CL 1-A |
| `PreviousWordPosition` | `const Position&` -> `PositionWithAffinity` | `position_units.h` -- **relocate** | CL 1-A |
| `PreviousWordPosition` | `const PositionInFlatTree&` -> `PositionInFlatTreeWithAffinity` | `position_units.h` -- **relocate** | CL 1-A |
| `NextWordPosition` | `const Position&` -> `PositionWithAffinity` | `position_units.h` -- **relocate** | CL 1-A |
| `NextWordPosition` | `const PositionInFlatTree&` -> `PositionInFlatTreeWithAffinity` | `position_units.h` -- **relocate** | CL 1-A |
| `IsWordBreak` | `UChar` -> `bool` | `visible_units.h` -- keep (character classifier) | |
| `IsWordBoundary` | `UChar` -> `bool` | `visible_units.h` -- keep (character classifier) | |

---

## Sentences

| Function | Signature | Destination | CL |
|---|---|---|---|
| `StartOfSentencePosition` | `const Position&` -> `Position` | `position_units.h` -- **relocate** | CL 1-A |
| `StartOfSentencePosition` | `const PositionInFlatTree&` -> `PositionInFlatTree` | `position_units.h` -- **relocate** | CL 1-A |
| `EndOfSentence` | `const Position&` -> `PositionWithAffinity` | `position_units.h` -- **relocate** | CL 1-A |
| `EndOfSentence` | `const PositionInFlatTree&` -> `PositionInFlatTreeWithAffinity` | `position_units.h` -- **relocate** | CL 1-A |
| `EndOfSentence` | `const VisiblePosition&` -> `VisiblePosition` | `visible_units.h` -- keep | |
| `EndOfSentence` | `const VisiblePositionInFlatTree&` -> `VisiblePositionInFlatTree` | `visible_units.h` -- keep | |
| `PreviousSentencePosition` | `const PositionInFlatTree&` -> `PositionInFlatTree` | `position_units.h` -- **relocate** | CL 1-A |
| `NextSentencePosition` | `const PositionInFlatTree&` -> `PositionInFlatTree` | `position_units.h` -- **relocate** | CL 1-A |
| `ExpandEndToSentenceBoundary` | `const EphemeralRange&` -> `EphemeralRange` | `position_units.h` -- **relocate** | CL 1-A |
| `ExpandRangeToSentenceBoundary` | `const EphemeralRange&` -> `EphemeralRange` | `position_units.h` -- **relocate** | CL 1-A |

---

## Lines

| Function | Signature | Destination | CL |
|---|---|---|---|
| `StartOfLine` | `const VisiblePosition&` -> `VisiblePosition` | `visible_units.h` -- keep | |
| `StartOfLine` | `const VisiblePositionInFlatTree&` -> `VisiblePositionInFlatTree` | `visible_units.h` -- keep | |
| `StartOfLine` | `const PositionWithAffinity&` -> `PositionWithAffinity` | `position_units.h` -- **relocate** | CL 1-A |
| `StartOfLine` | `const PositionInFlatTreeWithAffinity&` -> `PositionInFlatTreeWithAffinity` | `position_units.h` -- **relocate** | CL 1-A |
| `EndOfLine` | `const PositionWithAffinity&` -> `PositionWithAffinity` | `position_units.h` -- **relocate** | CL 1-A |
| `EndOfLine` | `const PositionInFlatTreeWithAffinity&` -> `PositionInFlatTreeWithAffinity` | `position_units.h` -- **relocate** | CL 1-A |
| `InSameLine` | `const VisiblePosition&, const VisiblePosition&` -> `bool` | `visible_units.h` -- keep | |
| `InSameLine` | `const VisiblePositionInFlatTree&, ...` -> `bool` | `visible_units.h` -- keep | |
| `InSameLine` | `const PositionWithAffinity&, const PositionWithAffinity&` -> `bool` | `position_units.h` -- **relocate** | CL 1-A |
| `InSameLine` | `const PositionInFlatTreeWithAffinity&, ...` -> `bool` | `position_units.h` -- **relocate** | CL 1-A |
| `IsStartOfLine` | `const VisiblePosition&` -> `bool` | `visible_units.h` -- keep | |
| `IsStartOfLine` | `const VisiblePositionInFlatTree&` -> `bool` | `visible_units.h` -- keep | |
| `IsStartOfLine` | `const PositionWithAffinity&` -> `bool` | `position_units.h` -- **new overload** | CL 1-B |
| `IsStartOfLine` | `const Position&` -> `bool` | `position_units.h` -- **new overload** | CL 1-B |
| `IsEndOfLine` | `const VisiblePosition&` -> `bool` | `visible_units.h` -- keep | |
| `IsEndOfLine` | `const VisiblePositionInFlatTree&` -> `bool` | `visible_units.h` -- keep | |
| `IsEndOfLine` | `const PositionWithAffinity&` -> `bool` | `position_units.h` -- **new overload** | CL 1-B |
| `IsEndOfLine` | `const Position&` -> `bool` | `position_units.h` -- **new overload** | CL 1-B |
| `LogicalStartOfLine` | `const VisiblePosition&` -> `VisiblePosition` | `visible_units.h` -- keep | |
| `LogicalStartOfLine` | `const VisiblePositionInFlatTree&` -> `VisiblePositionInFlatTree` | `visible_units.h` -- keep | |
| `LogicalEndOfLine` | `const VisiblePosition&` -> `VisiblePosition` | `visible_units.h` -- keep | |
| `LogicalEndOfLine` | `const VisiblePositionInFlatTree&` -> `VisiblePositionInFlatTree` | `visible_units.h` -- keep | |
| `IsLogicalEndOfLine` | `const VisiblePosition&` -> `bool` | `visible_units.h` -- keep | |
| `IsLogicalEndOfLine` | `const VisiblePositionInFlatTree&` -> `bool` | `visible_units.h` -- keep | |

`LogicalStartOfLine`, `LogicalEndOfLine`, `IsLogicalEndOfLine` have no `Position` overloads and no VP call sites in `commands/`. Out of scope for this effort; revisit when extending to `editor.cc` / `selection_adjuster.cc`.

---

## Paragraphs

| Function | Signature | Destination | CL |
|---|---|---|---|
| `StartOfParagraph` | `const VisiblePosition&` -> `VisiblePosition` | `visible_units.h` -- keep | |
| `StartOfParagraph` | `const VisiblePositionInFlatTree&` -> `VisiblePositionInFlatTree` | `visible_units.h` -- keep | |
| `StartOfParagraphInFlatTree` | `const VisiblePosition&` -> `VisiblePosition` | `visible_units.h` -- keep | |
| `EndOfParagraph` | `const VisiblePosition&` -> `VisiblePosition` | `visible_units.h` -- keep | |
| `EndOfParagraph` | `const VisiblePositionInFlatTree&` -> `VisiblePositionInFlatTree` | `visible_units.h` -- keep | |
| `EndOfParagraphInFlatTree` | `const VisiblePosition&` -> `VisiblePosition` | `visible_units.h` -- keep | |
| `StartOfNextParagraph` | `const VisiblePosition&` -> `VisiblePosition` | `visible_units.h` -- keep | |
| `IsStartOfParagraph` | `const VisiblePosition&` -> `bool` | `visible_units.h` -- keep | |
| `IsStartOfParagraph` | `const VisiblePositionInFlatTree&` -> `bool` | `visible_units.h` -- keep | |
| `IsEndOfParagraph` | `const VisiblePosition&` -> `bool` | `visible_units.h` -- keep | |
| `IsEndOfParagraph` | `const VisiblePositionInFlatTree&` -> `bool` | `visible_units.h` -- keep | |
| `InSameParagraph` | `const VisiblePosition&, const VisiblePosition&` -> `bool` | `visible_units.h` -- keep | |
| `ExpandToParagraphBoundary` | `const EphemeralRange&` -> `EphemeralRange` | `visible_units.h` -- keep | |
| `StartOfParagraph` | `const Position&` -> `Position` | `position_units.h` -- **new overload** | CL 1-A |
| `EndOfParagraph` | `const Position&` -> `Position` | `position_units.h` -- **new overload** (already added to `visible_units_paragraph.cc`; needs relocation) | CL 1-A |
| `IsStartOfParagraph` | `const Position&` -> `bool` | `position_units.h` -- **new overload** | CL 1-A |
| `IsEndOfParagraph` | `const Position&` -> `bool` | `position_units.h` -- **new overload** | CL 1-A |
| `StartOfNextParagraph` | `const Position&` -> `Position` | `position_units.h` -- **new overload** | CL 1-A |
| `InSameParagraph` | `const Position&, const Position&` -> `bool` | `position_units.h` -- **new overload** | CL 1-A |

---

## Document

| Function | Signature | Destination | CL |
|---|---|---|---|
| `StartOfDocument` | `const Position&` -> `Position` | `position_units.h` -- **relocate** | CL 1-A |
| `StartOfDocument` | `const PositionInFlatTree&` -> `PositionInFlatTree` | `position_units.h` -- **relocate** | CL 1-A |
| `EndOfDocument` | `const VisiblePosition&` -> `VisiblePosition` | `visible_units.h` -- keep | |
| `EndOfDocument` | `const VisiblePositionInFlatTree&` -> `VisiblePositionInFlatTree` | `visible_units.h` -- keep | |
| `IsStartOfDocument` | `const VisiblePosition&` -> `bool` | `visible_units.h` -- keep | |
| `IsEndOfDocument` | `const VisiblePosition&` -> `bool` | `visible_units.h` -- keep | |
| `EndOfDocument` | `const Position&` -> `Position` | `position_units.h` -- **new overload** | CL 1-A |
| `IsStartOfDocument` | `const Position&` -> `bool` | `position_units.h` -- **new overload** | CL 1-A |
| `IsEndOfDocument` | `const Position&` -> `bool` | `position_units.h` -- **new overload** | CL 1-A |

---

## Editable Content

| Function | Signature | Destination | CL |
|---|---|---|---|
| `StartOfEditableContent` | `const PositionInFlatTree&` -> `PositionInFlatTree` | `position_units.h` -- **relocate** | CL 1-A |
| `EndOfEditableContent` | `const PositionInFlatTree&` -> `PositionInFlatTree` | `position_units.h` -- **relocate** | CL 1-A |
| `IsEndOfEditableOrNonEditableContent` | `const VisiblePosition&` -> `bool` | `visible_units.h` -- keep | |
| `IsEndOfEditableOrNonEditableContent` | `const VisiblePositionInFlatTree&` -> `bool` | `visible_units.h` -- keep | |

---

## Layout / Hit-test Utilities

Stay in `visible_units.h` regardless of parameter types. These answer rendering or
layout questions, not navigation-unit boundary questions.

| Function | Signature |
|---|---|
| `HasRenderedNonAnonymousDescendantsWithHeight` | `const LayoutObject*` -> `bool` |
| `PositionForContentsPointRespectingEditingBoundary` | `const gfx::Point&, LocalFrame*` -> `PositionWithAffinity` |
| `AdjustForwardPositionToAvoidCrossingEditingBoundaries` | `const PositionWithAffinity&, const Position&` -> `PositionWithAffinity` |
| `AdjustForwardPositionToAvoidCrossingEditingBoundaries` | `const PositionInFlatTreeWithAffinity&, const PositionInFlatTree&` -> `PositionInFlatTreeWithAffinity` |
| `AdjustBackwardPositionToAvoidCrossingEditingBoundaries` | `const PositionWithAffinity&, const Position&` -> `PositionWithAffinity` |
| `AdjustBackwardPositionToAvoidCrossingEditingBoundaries` | `const PositionInFlatTreeWithAffinity&, const PositionInFlatTree&` -> `PositionInFlatTreeWithAffinity` |

---

## Geometry / Bounds

Stay in `visible_units.h`. Require layout and return geometric types.

| Function | Signature |
|---|---|
| `ComputeTextBounds` | `const EphemeralRangeTemplate<Strategy>&` -> `Vector<gfx::QuadF>` |
| `ComputeTextRect` | `const EphemeralRange&` -> `gfx::Rect` |
| `ComputeTextRect` | `const EphemeralRangeInFlatTree&` -> `gfx::Rect` |
| `ComputeTextRectF` | `const EphemeralRange&` -> `gfx::RectF` |
| `FirstRectForRange` | `const EphemeralRange&` -> `gfx::Rect` |

---

## Summary

| Destination | Count | CL |
|---|---|---|
| `position_units.h` -- relocate | 22 | CL 1-A |
| `position_units.h` -- new overload (trivial) | 10 | CL 1-A |
| `position_units.h` -- new overload (non-trivial) | 7 | CL 1-B |
| `visible_units.h` -- keep (VP navigation) | 28 | |
| `visible_units.h` -- keep (layout / geometry / character classifier utility) | 15 | |
| **Total** | **82** | |
