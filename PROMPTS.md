# AI Agent Prompts: VisiblePosition Cleanup

Each section below is a self-contained prompt for one phase. Paste the entire
section into an AI coding agent. The agent should be able to complete the phase
without further clarification.

---

## PHASE 1 PROMPT — Add Position-based Navigation Overloads

### Context

You are working in the Chromium source tree at
`third_party/blink/renderer/core/editing/`.

`VisiblePosition` is Blink's canonicalized position type. Creating one forces
`Document::UpdateStyleAndLayout()`. Editing commands should not force layout
mid-execution; they should work with raw `Position` objects.

The goal of this phase is to add public `Position`-typed overloads for navigation
functions that today only accept `VisiblePosition`. These overloads will be used in
later phases to remove VP from editing commands entirely.

### Hard rules — read before writing any code

1. **Never use `VisiblePosition`, `CreateVisiblePosition`, `CanonicalPositionOf`,
   `.DeepEquivalent()`, or any other method that constructs or canonicalizes a VP
   in the new overload bodies or in any algorithm template you write for them.**
   `CanonicalPositionOf` is not a safe substitute — it calls `UpdateStyleAndLayout`
   just like `CreateVisiblePosition` does. The replacement for
   `CreateVisiblePosition(pos).DeepEquivalent()` is `pos`. No normalization at all.

2. **Check for an existing `Position`-typed algorithm template first.** Files like
   `visible_units_paragraph.cc` already have private templates such as
   `StartOfParagraphAlgorithm<Strategy>(const PositionTemplate<Strategy>&, ...)`.
   The VP public overload already delegates to this template. Your new `Position`
   overload must call the same template directly.

3. **If no `Position`-typed algorithm template exists**, extract the logic from the
   VP implementation into a new `Algorithm<Strategy>` template that takes
   `PositionTemplate<Strategy>` and contains zero VP usage. Then have both the
   existing VP overload and your new `Position` overload delegate to it.

4. **Never introduce VP inside an algorithm template.** If the existing VP
   implementation calls other VP-returning functions internally, those calls must be
   replaced with their `Position`-returning equivalents before you can use the
   template. Add those equivalents to the Phase 1 list and do them first.

5. Each new overload must have a unit test in the corresponding `*_test.cc` file.

### Functions to add

**In `visible_units.h` / `visible_units_paragraph.cc`:**

```cpp
bool IsStartOfParagraph(const Position&,
                        EditingBoundaryCrossingRule = kCannotCrossEditingBoundary);
bool IsEndOfParagraph(const Position&,
                      EditingBoundaryCrossingRule = kCannotCrossEditingBoundary);
Position StartOfParagraph(const Position&,
                          EditingBoundaryCrossingRule = kCannotCrossEditingBoundary);
Position EndOfParagraph(const Position&,
                        EditingBoundaryCrossingRule = kCannotCrossEditingBoundary);
Position StartOfNextParagraph(const Position&);
bool InSameParagraph(const Position&, const Position&,
                     EditingBoundaryCrossingRule = kCannotCrossEditingBoundary);
```

**In `visible_units.h` / `visible_units.cc`:**

```cpp
Position EndOfDocument(const Position&);
bool IsStartOfDocument(const Position&);
bool IsEndOfDocument(const Position&);
UChar32 CharacterAfter(const Position&);
```

**In `editing_commands_utilities.h` / `editing_commands_utilities.cc`:**

```cpp
Position StartOfBlock(const Position&,
                      EditingBoundaryCrossingRule = kCannotCrossEditingBoundary);
Position EndOfBlock(const Position&,
                    EditingBoundaryCrossingRule = kCannotCrossEditingBoundary);
bool IsStartOfBlock(const Position&);
bool IsEndOfBlock(const Position&);
Node* EnclosingEmptyListItem(const Position&);
```

### Already exist — do not duplicate

These already have `Position` overloads. Verify callers can use them but do not
re-add:

```cpp
Position StartOfWordPosition(const Position&, WordSide);
Position EndOfWordPosition(const Position&, WordSide);
PositionWithAffinity StartOfLine(const PositionWithAffinity&);
PositionWithAffinity EndOfLine(const PositionWithAffinity&);
bool InSameLine(const PositionWithAffinity&, const PositionWithAffinity&);
Position StartOfDocument(const Position&);
bool LineBreakExistsAtPosition(const Position&);
```

### Correct implementation pattern

`StartOfParagraphAlgorithm<Strategy>(const PositionTemplate<Strategy>&, ...)` already
exists in `visible_units_paragraph.cc` and operates purely on DOM structure. Add only
the public overload:

```cpp
// visible_units_paragraph.cc
Position StartOfParagraph(const Position& pos,
                           EditingBoundaryCrossingRule rule) {
  return StartOfParagraphAlgorithm<EditingStrategy>(pos, rule);
}

bool IsStartOfParagraph(const Position& pos,
                         EditingBoundaryCrossingRule rule) {
  return pos == StartOfParagraphAlgorithm<EditingStrategy>(pos, rule);
}
```

Apply the same pattern for every function: find the existing algorithm template,
call it directly. If it does not exist, write it — with zero VP inside.

### Verification after each addition

```
autoninja -C out/Default blink_unittests
./out/Default/blink_unittests --gtest_filter="*Paragraph*:*Document*:*Block*:*Edit*"
```

---

## PHASE 2 PROMPT — Migrate Simple Command Files

### Context

Phase 1 is complete. All `Position`-typed navigation overloads now exist. You are
migrating the 5 lowest-VP editing command files to remove all `VisiblePosition`
usage.

Working directory: `third_party/blink/renderer/core/editing/commands/`

### Hard rules

1. **Remove all `VisiblePosition` usage from each file listed below.** After your
   changes the file must not include `visible_position.h` and must not use
   `VisiblePosition`, `CreateVisiblePosition`, or `.DeepEquivalent()`.

2. **Do not introduce `VisiblePosition` anywhere** — not in helpers, not in local
   variables, not in temporary bridges.

3. **Never use `CanonicalPositionOf` as a replacement for `CreateVisiblePosition`.**
   Both call `UpdateStyleAndLayout`. The replacement for
   `CreateVisiblePosition(pos).DeepEquivalent()` is `pos`. Drop the call entirely.

4. Use `EndingSelection().Start()` / `.End()` / `.Anchor()` instead of
   `EndingVisibleSelection().VisibleStart().DeepEquivalent()` etc.

### Files to migrate (one CL per file)

1. `style_commands.cc`
2. `undo_step.cc`
3. `editor_command.cc`
4. `insert_line_break_command.cc`
5. `insert_text_command.cc`

### Replacement patterns

| Before | After |
|--------|-------|
| `CreateVisiblePosition(pos).DeepEquivalent()` | `pos` — drop the call entirely |
| `IsEndOfParagraph(CreateVisiblePosition(pos))` | `IsEndOfParagraph(pos)` |
| `IsStartOfParagraph(CreateVisiblePosition(pos))` | `IsStartOfParagraph(pos)` |
| `StartOfParagraph(vp).DeepEquivalent()` | `StartOfParagraph(pos)` |
| `EndOfParagraph(vp).DeepEquivalent()` | `EndOfParagraph(pos)` |
| `VisiblePosition::FirstPositionInNode(*node).DeepEquivalent()` | `Position::FirstPositionInNode(*node)` |
| `VisiblePosition::LastPositionInNode(*node).DeepEquivalent()` | `Position::LastPositionInNode(*node)` |
| `LineBreakExistsAtVisiblePosition(vp)` | `LineBreakExistsAtPosition(pos)` |
| `VisiblePosition::BeforeNode(*node)` | `Position::BeforeNode(*node)` |
| `EndingVisibleSelection().VisibleStart().DeepEquivalent()` | `EndingSelection().Start()` |
| `EndingVisibleSelection().Start()` | `EndingSelection().Start()` |

### Verification per file

```
autoninja -C out/Default blink_unittests
./out/Default/blink_unittests --gtest_filter="*Edit*"
python3 third_party/blink/tools/run_web_tests.py editing/
```

All must pass before moving to the next file.

---

## PHASE 3 PROMPT — Infrastructure Signature Changes

### Context

Phase 1 overloads are in place. You are changing `CompositeEditCommand` protected
method signatures from `VisiblePosition` to `Position` and migrating their bodies.

This eliminates the P4 cascade: every caller of these methods is forced to wrap raw
positions into VP just to satisfy the type signature. Changing the signatures removes
that forced creation for all callers at once.

Files to modify:
- `third_party/blink/renderer/core/editing/commands/composite_edit_command.h`
- `third_party/blink/renderer/core/editing/commands/composite_edit_command.cc`

### Hard rules

1. **No `VisiblePosition`, `CreateVisiblePosition`, or `.DeepEquivalent()` in any
   method body you touch.** If a VP is pre-existing in a part of the method you are
   not modifying, leave it — but do not add any new ones.

2. **Do not create temporary VP bridges.** If an internal call inside the method body
   requires VP today, use the Phase 1 `Position` overload instead. If no `Position`
   overload exists, it is a gap that belongs in Phase 1 — add it there first, then
   continue here.

3. **Fix every caller in the same CL.** After changing a method signature the callers
   will not compile. All callers must be updated in the same CL (or the same batch if
   doing one CL per method). Do not leave broken callers.

### Target signature changes

**`MoveParagraph` and `MoveParagraphs`** (change together):

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

Body migration for `MoveParagraphs`:
- VP comparison operators → `ComparePositions(destination, start) >= 0 && ComparePositions(destination, end) <= 0`
- `PreviousPositionOf(start_of_paragraph_to_move, kCannotCrossEditingBoundary).DeepEquivalent()` → `PreviousPositionOf(start, kCannotCrossEditingBoundary)` using the `Position` overload of `PreviousPositionOf`
- `EndingVisibleSelection().VisibleStart()` / `.VisibleEnd()` → `EndingSelection().Start()` / `.End()`

**`MoveParagraphWithClones`:**

```cpp
void MoveParagraphWithClones(const Position& start,
                             const Position& end,
                             HTMLElement* block_element,
                             Node* outer_node,
                             EditingState*);
```

**`CleanupAfterDeletion`:**

```cpp
void CleanupAfterDeletion(EditingState*, Position destination = Position());
```

Body migration: `destination.DeepEquivalent().AnchorNode()` → `destination.AnchorNode()`; `EndingVisibleSelection().VisibleStart()` → `EndingSelection().Start()`; `IsStartOfParagraph` / `IsEndOfParagraph` use Phase 1 overloads.

**`ReplaceCollapsibleWhitespaceWithNonBreakingSpaceIfNeeded`:**

```cpp
void ReplaceCollapsibleWhitespaceWithNonBreakingSpaceIfNeeded(const Position&);
```

Body migration: `CharacterAfter(position)` using the Phase 1 `Position` overload; `MostForwardCaretPosition(position)` already takes `Position`.

### Verification after each method

```
autoninja -C out/Default blink_unittests
./out/Default/blink_unittests --gtest_filter="*Edit*"
python3 third_party/blink/tools/run_web_tests.py editing/
```

---

## PHASE 4 PROMPT — Heavy Command Migration

### Context

Phases 1–3 are complete:
- Phase 1 `Position` overloads exist for all navigation functions.
- Phase 3 infrastructure methods now take `Position` parameters.

You are migrating the 6 heavy command files plus remaining private methods in
`composite_edit_command.cc`.

### Hard rules

1. **Remove all `VisiblePosition` usage from each file.** After migration the file
   must not include `visible_position.h` and must not use `VisiblePosition`,
   `CreateVisiblePosition`, or `.DeepEquivalent()`.

2. **Never introduce VP to bridge a type mismatch.** Use the Phase 1 `Position`
   overload. If one is missing, it is a Phase 1 gap — add it there first.

3. **Never use `CanonicalPositionOf`.** It calls `UpdateStyleAndLayout` the same as
   `CreateVisiblePosition`. `CreateVisiblePosition(pos).DeepEquivalent()` becomes
   `pos` — drop the call entirely, no substitution.

4. **Category A sites** — VP created only to pass to `MoveParagraph` /
   `MoveParagraphWithClones` / `CleanupAfterDeletion`: the Phase 3 signature changes
   mean you simply delete the `CreateVisiblePosition` wrapper and pass the raw
   `Position` directly.

5. **Category B sites** — VP created only to call `IsEndOfParagraph`,
   `StartOfParagraph`, etc.: replace with Phase 1 overloads directly.

6. **`VisiblePosition::FirstPositionInNode(*cell).DeepEquivalent() ==
   VisiblePosition::LastPositionInNode(*cell).DeepEquivalent()`** style equality
   tests: replace with `Position::FirstPositionInNode(*cell) ==
   Position::LastPositionInNode(*cell)` only after confirming the positions are
   already canonical at that site. If uncertain, leave a
   `// TODO(vp-cleanup): verify canonicality` comment.

### Files to migrate (one CL per file, in this order)

1. `insert_paragraph_separator_command.cc` — ~19 VP refs (Category B dominant)
2. `apply_block_element_command.cc` — ~37 VP refs (Category B + A)
3. `indent_outdent_command.cc` — ~61 VP refs (Category A dominant)
4. `delete_selection_command.cc` — ~64 VP refs (Category A + B)
5. `insert_list_command.cc` — ~91 VP refs (Category A + B)
6. `replace_selection_command.cc` — ~98 VP refs (Category B dominant)
7. `composite_edit_command.cc` remaining private methods:
   `RebalanceWhitespaceOnTextSubstring`, `MoveParagraphContentsToNewBlockIfNecessary`,
   `BreakOutOfEmptyListItem`, `BreakOutOfEmptyMailBlockquotedParagraph`

### Verification per file

```
autoninja -C out/Default blink_unittests
./out/Default/blink_unittests --gtest_filter="*Edit*"
python3 third_party/blink/tools/run_web_tests.py editing/
python3 third_party/blink/tools/run_web_tests.py editing/contenteditable/
```

After all CLs land, confirm cleanup is complete:

```
grep -r "visible_position.h" \
  third_party/blink/renderer/core/editing/commands/*.cc
```

Should return no results (or only files with documented legitimate boundary VP usage).

---

## WHAT IS LEGITIMATE VP — DO NOT REMOVE

The following VP usage is correct. Do not touch it in any phase.

1. **Entry boundary** — `CompositeEditCommand` constructor reading
   `frame->Selection().ComputeVisibleSelectionInDOMTreeDeprecated()`. This is the
   renderer → command boundary-in point where VP is the correct type.

2. **Exit boundary** — `EndingVisibleSelection()` in `Apply()` for the
   `IsRichlyEditablePosition` check. This is the command → renderer boundary-out.

3. **Affinity-sensitive navigation** where the result depends on line-wrap affinity
   and the VP is immediately stored in `RelocatablePosition`. If you encounter
   this, leave it with a comment:
   `// VP required: affinity-sensitive, immediately stored as RelocatablePosition.`

4. **`IndexForVisiblePosition` / `VisiblePositionForIndex` / `MakeRange(VP, VP)`
   in `editing_utilities.cc`** — explicit public API contracts. Out of scope.
