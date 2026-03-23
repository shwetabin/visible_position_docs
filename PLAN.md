# Plan: Remove Unnecessary VisiblePosition Usage from Editing Commands

## Goal

Decouple editing command execution from VisiblePosition computation so that commands operate on raw `Position`/`PositionWithAffinity` wherever possible, reserving VisiblePosition only for the user-facing selection boundary (input from renderer and output back to renderer).

## Problem Statement

Currently, Blink's editing commands pervasively use `VisiblePosition` and its `DeepEquivalent()` accessor internally during command execution. `VisiblePosition` computes "canonical position" which requires up-to-date layout. This creates several problems:

1. **Unnecessary layout dependency**: Commands force layout updates mid-execution to create VisiblePositions when the raw `Position` would suffice.
2. **Performance**: `CanonicalPositionOf()` is expensive - it walks the DOM to find visually equivalent positions. Many call sites immediately call `.DeepEquivalent()` to extract the raw Position anyway.
3. **Fragility**: Storing VisiblePositions across DOM mutations invalidates them, leading to subtle bugs.
4. **Circular logic**: Commands create a VisiblePosition from a Position, mutate the DOM, then extract the Position back - the round-trip is wasteful and error-prone.

### Correct Flow (Target Architecture)

```
User Interaction
    |
    v
[1] Take visible selection from renderer (VisiblePosition -> Position)
    |
    v
[2] Execute editing command using Position (NOT VisiblePosition)
    |
    v
[3] Set visible selection for user (Position -> VisiblePosition for rendering)
```

VisiblePosition should only exist at the boundary: **input** (step 1) and **output** (step 3). The command execution (step 2) should work entirely with `Position`/`PositionWithAffinity`.

---

## Scope & Scale

### Quantitative Analysis (from codebase scan)

**Total occurrences in `editing/commands/`:** ~693 references across 26 files (source + headers + tests)

**Key metrics:**
- `VisiblePosition` type usage: ~391 occurrences across 18 `.cc` files
- `.DeepEquivalent()` calls: ~288 occurrences across 18 `.cc` files
- `CreateVisiblePosition()` calls: ~126 occurrences across 16 `.cc` files
- `CanonicalPositionOf()` calls: ~3 occurrences (only in tests, the implicit calls via `CreateVisiblePosition` are the real issue)

### Files Ranked by VisiblePosition Usage (source files only)

| File | VP Refs | Category |
|------|---------|----------|
| `composite_edit_command.cc` | 112 | Composite (shared infrastructure) |
| `replace_selection_command.cc` | 98 | Composite command |
| `insert_list_command.cc` | 91 | Composite command |
| `delete_selection_command.cc` | 64 | Composite command |
| `indent_outdent_command.cc` | 61 | Composite command |
| `editing_commands_utilities.cc` | 49 | Shared utilities |
| `apply_block_element_command.cc` | 37 | Composite command |
| `insert_paragraph_separator_command.cc` | 19 | Composite command |
| `typing_command.cc` | 17 | Composite command |
| `format_block_command.cc` | 16 | Composite command |
| `apply_style_command.cc` | 15 | Composite command |
| `break_blockquote_command.cc` | 13 | Composite command |
| `insert_text_command.cc` | 7 | Simple-ish command |
| `insert_line_break_command.cc` | 6 | Simple command |
| `editor_command.cc` | 6 | Command dispatch |
| `undo_step.cc` | 2 | Undo infrastructure |
| `style_commands.cc` | 1 | Style utilities |

### Methods That Compute VisiblePosition (Task 1 Inventory)

These are the primary functions that produce or consume VisiblePositions:

**Core creation:**
- `CreateVisiblePosition(Position, TextAffinity)` - Main factory, calls `CanonicalPositionOf()`
- `VisiblePosition::Create(PositionWithAffinity)` - Template factory
- `VisiblePosition::FirstPositionInNode()`, `LastPositionInNode()`, `BeforeNode()`, `AfterNode()`, `InParentBeforeNode()`, `InParentAfterNode()` - Static constructors

**Position extraction:**
- `VisiblePosition::DeepEquivalent()` - Extracts the underlying `Position`
- `VisiblePosition::ToPositionWithAffinity()` - Extracts `PositionWithAffinity`
- `VisiblePosition::ToParentAnchoredPosition()` - Extracts parent-anchored position

**VisiblePosition-based navigation (in `visible_units.h`):**
- `StartOfParagraph(VisiblePosition)`, `EndOfParagraph(VisiblePosition)` - Paragraph boundaries
- `StartOfLine(VisiblePosition)`, `LogicalStartOfLine(VisiblePosition)` - Line boundaries
- `EndOfDocument(VisiblePosition)` - Document boundary
- `NextPositionOf(VisiblePosition)`, `PreviousPositionOf(VisiblePosition)` - Adjacent positions
- `StartOfNextParagraph(VisiblePosition)` - Next paragraph
- `IsStartOfParagraph(VisiblePosition)`, `IsEndOfParagraph(VisiblePosition)` - Boundary tests
- `IsStartOfLine(VisiblePosition)`, `IsEndOfLine(VisiblePosition)` - Line boundary tests
- `InSameLine(VisiblePosition, VisiblePosition)` - Same-line test
- `InSameParagraph(VisiblePosition, VisiblePosition)` - Same-paragraph test
- `CharacterAfter(VisiblePosition)`, `CharacterBefore(VisiblePosition)` - Character inspection

**VisibleSelection methods:**
- `VisibleSelection::VisibleStart()`, `VisibleEnd()`, `VisibleAnchor()`, `VisibleFocus()` - Selection endpoints as VP
- `EndingVisibleSelection().VisibleStart()` / `.VisibleEnd()` - Used heavily in commands

**Editing command utilities (in `editing_commands_utilities.h`):**
- `StartOfBlock(VisiblePosition)`, `EndOfBlock(VisiblePosition)` - Block boundaries
- `IsStartOfBlock(VisiblePosition)`, `IsEndOfBlock(VisiblePosition)` - Block boundary tests
- `LineBreakExistsAtVisiblePosition(VisiblePosition)` - Line break detection
- `EnclosingEmptyListItem(VisiblePosition)` - List item detection
- `SelectionForParagraphIteration(VisibleSelection)` - Paragraph iteration

**CompositeEditCommand methods that use VP:**
- `EndingVisibleSelection()` - Returns VisibleSelection (triggers VP creation)
- `MoveParagraph(VP, VP, VP, ...)` - Takes 3 VisiblePosition params
- `MoveParagraphs(VP, VP, VP, ...)` - Takes 3 VisiblePosition params
- `MoveParagraphWithClones(VP, VP, ...)` - Takes 2 VisiblePosition params
- `CleanupAfterDeletion(EditingState*, VisiblePosition)` - Takes VP param
- `ReplaceCollapsibleWhitespaceWithNonBreakingSpaceIfNeeded(VP)` - Takes VP param

### Common Anti-Patterns to Fix

1. **Create-then-extract**: `CreateVisiblePosition(pos).DeepEquivalent()` - Creates VP only to immediately extract the Position back. Replace with `pos` directly. Do not substitute `CanonicalPositionOf` — it calls `UpdateStyleAndLayout` the same as `CreateVisiblePosition`.

2. **VP for comparison**: Using `VP::FirstPositionInNode(*node).DeepEquivalent() == pos` - Can often be replaced with direct Position comparison using `MostForwardCaretPosition`/`MostBackwardCaretPosition`.

3. **VP for navigation boundary**: `StartOfParagraph(CreateVisiblePosition(pos)).DeepEquivalent()` - Need Position-based overloads of boundary functions.

4. **VP stored across mutations**: `VisiblePosition caret = ...; /* mutate DOM */; use(caret.DeepEquivalent())` - Should use `RelocatablePosition` instead.

---

## Team & Timeline

**Team:** 2 engineers (Person A, Person B)
**Duration:** 8 weeks
**Cadence:** Weekly sync to review progress, unblock issues, coordinate shared file changes

---

## Phase 1: Discovery & Design (Weeks 1-2)

### Task 1.1: Complete Inventory of VisiblePosition Methods [Person A - Week 1]
- [ ] Catalog all methods in `visible_units.h` that take/return VisiblePosition
- [ ] Identify which already have `Position`/`PositionWithAffinity` overloads
- [ ] Identify which need new Position-based overloads
- [ ] Document in a spreadsheet: `method | VP overload exists | Position overload exists | used by commands`

### Task 1.2: Classify All Editing Command VP Usage [Person B - Week 1]
- [ ] For each of the 18 source files, classify every VP usage as:
  - **BOUNDARY** (legitimate - converting user selection to/from VP at entry/exit)
  - **NAVIGATION** (uses VP for paragraph/line/word boundary computation)
  - **COMPARISON** (uses VP only to compare positions)
  - **UNNECESSARY** (create-then-extract or redundant canonicalization)
- [ ] Produce a per-file breakdown with line numbers and classification
- [ ] Estimate effort per file (S/M/L)

### Task 1.3: Design Doc / Explainer [Both - Week 2]
- [ ] Write design doc covering:
  - **Problem statement** with concrete examples of bugs caused by current approach
  - **Approach A: Bottom-up** - Add Position overloads for visible_units functions, then migrate callers one-by-one
    - Pro: Low risk per CL, easy to review, gradual migration
    - Con: Long tail, both APIs coexist for extended period
  - **Approach B: Top-down** - Change CompositeEditCommand infrastructure first (EndingVisibleSelection, MoveParagraph signatures), then fix leaf commands
    - Pro: Forces consistent pattern, reduces total work
    - Con: Large initial CLs, higher risk of regressions
  - **Approach C: Hybrid** (recommended) - Add Position overloads bottom-up (visible_units), then migrate commands top-down by complexity
    - Pro: Balances risk and velocity
    - Con: Requires coordination
  - **Migration pattern** for each anti-pattern type
  - **Testing strategy** - reliance on existing WPT editing tests + targeted regression tests
- [ ] Share with Chromium editing-dev owners for review
- [ ] Incorporate feedback

### Task 1.4: Bug Triage [Person A - Week 2]
- [ ] Search crbug.com for bugs related to VisiblePosition in editing commands
- [ ] Search for bugs mentioning `CanonicalPositionOf`, `DeepEquivalent`, layout-during-editing
- [ ] Categorize bugs as: blocked-by-this-change, related, or will-be-fixed-by-cleanup
- [ ] Create tracking bug if one doesn't exist
- [ ] Link dependent bugs

**Deliverables:**
- Complete VP usage inventory spreadsheet
- Design doc shared with editing-dev@
- Bug triage with tracking bug

---

## Phase 2: Foundation - Position-Based Overloads (Weeks 3-4)

### Task 2.1: Add Position Overloads for visible_units Functions [Person A - Weeks 3-4]

Add `Position`/`PositionWithAffinity` overloads for the most-used navigation functions. Many already exist (e.g., `StartOfLine(PositionWithAffinity)` exists alongside `StartOfLine(VisiblePosition)`). Fill the gaps:

- [ ] `StartOfParagraph(Position)` / `EndOfParagraph(Position)` returning `Position`
- [ ] `IsStartOfParagraph(Position)` / `IsEndOfParagraph(Position)` returning `bool`
- [ ] `StartOfNextParagraph(Position)` returning `Position`
- [ ] `IsStartOfLine(Position)` / `IsEndOfLine(Position)` returning `bool`
- [ ] `InSameParagraph(Position, Position)` returning `bool`
- [ ] `CharacterAfter(Position)` / `CharacterBefore(Position)` (if needed)
- [ ] `EndOfDocument(Position)` returning `Position`
- [ ] `IsStartOfDocument(Position)` / `IsEndOfDocument(Position)` returning `bool`
- [ ] `PreviousPositionOf(Position, EditingBoundaryCrossingRule)` returning `Position` — implement via `PreviousVisuallyDistinctCandidate` directly, no `CreateVisiblePosition`
- [ ] `NextPositionOf(Position, EditingBoundaryCrossingRule)` returning `Position` — existing `NextPositionOf(Position)` returns VP; add a `Position`-returning overload alongside it

Each overload should be a separate CL with tests. Priority order by usage frequency
across commands:
1. `IsStartOfParagraph(Position)` / `IsEndOfParagraph(Position)` — used in 9 files
2. `StartOfParagraph(Position)` / `EndOfParagraph(Position)` — used in 7 files
3. `PreviousPositionOf(Position)` / `NextPositionOf(Position)` — unblocks Phase 3 infrastructure
4. `StartOfBlock(Position)` / `EndOfBlock(Position)` / `IsStartOfBlock(Position)` / `IsEndOfBlock(Position)`
5. `InSameParagraph(Position, Position)`
6. `StartOfNextParagraph(Position)`
7. `EnclosingEmptyListItem(Position)` → `Node*`
8. `CharacterAfter(Position)` / `EndOfDocument(Position)` / document boundary tests

### Task 2.2: Add Position Overloads for editing_commands_utilities [Person B - Weeks 3-4]

- [ ] `StartOfBlock(Position)` / `EndOfBlock(Position)` returning `Position`
- [ ] `IsStartOfBlock(Position)` / `IsEndOfBlock(Position)` returning `bool`
- [ ] `LineBreakExistsAtPosition(Position)` (already exists, verify usability)
- [ ] `EnclosingEmptyListItem(Position)` returning `Node*`
- [ ] `SelectionForParagraphIteration` - evaluate if it can take `SelectionInDOMTree` instead of `VisibleSelection`

Each overload should be a separate CL with tests.

### Task 2.3: CompositeEditCommand Helper Alternatives [Person B - Week 4]

Create Position-based alternatives for the most-used CompositeEditCommand helpers:

- [ ] `CleanupAfterDeletion(EditingState*, Position destination)` overload
- [ ] `ReplaceCollapsibleWhitespaceWithNonBreakingSpaceIfNeeded(Position)` overload
- [ ] Evaluate `MoveParagraph` / `MoveParagraphs` - these are complex; design Position-based signature

**Deliverables:**
- ~10-15 CLs adding Position-based overloads
- All with tests, reviewed and landed

---

## Phase 3: Simple Command Cleanup (Weeks 5-6)

Migrate commands with low VP usage first. These are lower risk and build confidence.

### Task 3.1: Cleanup Simple Editing Commands [Person A - Week 5]

| Command | VP Refs | Estimated Effort |
|---------|---------|-----------------|
| `style_commands.cc` | 1 | XS |
| `undo_step.cc` | 2 | XS |
| `editor_command.cc` | 6 | S |
| `insert_line_break_command.cc` | 6 | S |
| `insert_text_command.cc` | 7 | S |

- [ ] `style_commands.cc` - Remove 1 VP reference
- [ ] `undo_step.cc` - Remove 2 VP references (may need SelectionForUndoStep changes)
- [ ] `editor_command.cc` - Remove 6 VP references
- [ ] `insert_line_break_command.cc` - Remove 6 VP references (uses `VisibleStart()`, `IsEndOfParagraph`, `IsStartOfParagraph`, `LineBreakExistsAtVisiblePosition`)
- [ ] `insert_text_command.cc` - Remove 7 VP references (uses `FirstPositionInNode`, `LastPositionInNode`, `CreateVisiblePosition`)

### Task 3.2: Cleanup Medium Editing Commands [Person B - Week 5]

| Command | VP Refs | Estimated Effort |
|---------|---------|-----------------|
| `break_blockquote_command.cc` | 13 | M |
| `apply_style_command.cc` | 15 | M |
| `format_block_command.cc` | 16 | M |
| `typing_command.cc` | 17 | M |

- [ ] `break_blockquote_command.cc` - Remove VP from `IsFirstVisiblePositionInNode`/`IsLastVisiblePositionInNode` helpers and main `DoApply`
- [ ] `apply_style_command.cc` - Migrate to Position-based comparisons
- [ ] `format_block_command.cc` - Heavy `CreateVisiblePosition` usage for paragraph iteration
- [ ] `typing_command.cc` - Uses `EndingVisibleSelection().VisibleStart()` pattern, `EndOfParagraph`, table cell checks

### Task 3.3: Cleanup Shared Utilities [Both - Week 6]

- [ ] `editing_commands_utilities.cc` (49 refs) - Migrate helper functions to use Position overloads created in Phase 2
  - `EnclosingEmptyListItem` - Replace VP comparisons with Position comparisons
  - `PositionBeforeContainingSpecialElement` / `PositionAfterContainingSpecialElement` - Already use some Position, reduce VP
  - `SelectionForParagraphIteration` - Consider VisibleSelection -> SelectionInDOMTree
  - `StartOfBlock` / `EndOfBlock` - Already addressed by Position overloads, mark old VP versions as deprecated

**Deliverables:**
- ~12-15 CLs migrating simple and medium commands
- Each CL should be independently reviewable and testable

---

## Phase 4: Private Methods & Composite Command Cleanup (Weeks 6-8)

### Task 4.1: CompositeEditCommand Private Methods [Person A - Weeks 6-7]

`composite_edit_command.cc` has 112 VP references. Target the private helper methods first:

- [ ] `RebalanceWhitespace()` / `RebalanceWhitespaceAt()` - Migrate to Position
- [ ] `PrepareWhitespaceAtPositionForSplit()` - Already takes Position, reduce internal VP usage
- [ ] `DeleteInsignificantText()` / `DeleteInsignificantTextDownstream()` - Evaluate VP necessity
- [ ] `MoveParagraphContentsToNewBlockIfNecessary()` - Heavy VP user (paragraph boundaries)
- [ ] `CleanupAfterDeletion()` - Switch to Position-based overload from Phase 2
- [ ] `PositionAvoidingSpecialElementBoundary()` - Already returns Position, reduce internal VP
- [ ] `BreakOutOfEmptyListItem()` / `BreakOutOfEmptyMailBlockquotedParagraph()` - Moderate VP usage

### Task 4.2: MoveParagraph Family [Person B - Weeks 6-7]

The `MoveParagraph`/`MoveParagraphs`/`MoveParagraphWithClones` methods are the heaviest VP users and take VisiblePosition parameters directly. This is the hardest migration:

- [ ] Design Position-based signatures for `MoveParagraph`, `MoveParagraphs`, `MoveParagraphWithClones`
- [ ] Migrate internal logic to use Position (paragraph range computation, destination handling)
- [ ] Update all callers (in `delete_selection_command.cc`, `indent_outdent_command.cc`, etc.)
- [ ] Extensive testing - these are critical for cut/paste, indent/outdent, list operations

CL sequencing within Task 4.2: one CL per method in this order:
1. `MoveParagraph` + `MoveParagraphs` together (shared body)
2. `MoveParagraphWithClones`
3. `CleanupAfterDeletion`
4. `ReplaceCollapsibleWhitespaceWithNonBreakingSpaceIfNeeded`

### Task 4.3: Heavy Composite Commands [Both - Weeks 7-8]

| Command | VP Refs | Estimated Effort |
|---------|---------|-----------------|
| `insert_paragraph_separator_command.cc` | 19 | M |
| `apply_block_element_command.cc` | 37 | L |
| `indent_outdent_command.cc` | 61 | L |
| `delete_selection_command.cc` | 64 | XL |
| `insert_list_command.cc` | 91 | XL |
| `replace_selection_command.cc` | 98 | XL |

- [ ] `insert_paragraph_separator_command.cc` [Person A]
- [ ] `apply_block_element_command.cc` [Person B]
- [ ] `indent_outdent_command.cc` [Person A] - Heavy user of `StartOfParagraph`/`EndOfParagraph`
- [ ] `delete_selection_command.cc` [Person B] - 64 refs, critical for delete/backspace behavior
- [ ] `insert_list_command.cc` [Person A] - 91 refs, heaviest single command
- [ ] `replace_selection_command.cc` [Person B] - 98 refs, critical for paste

**Note:** The XL commands (`delete_selection_command`, `insert_list_command`, `replace_selection_command`) may not fully complete within 8 weeks. The goal is to make significant progress and establish the pattern so remaining work is mechanical.

**Deliverables:**
- ~15-20 CLs migrating composite commands
- MoveParagraph family fully migrated
- At least 3 of the 6 heavy commands substantially cleaned up

---

## Timeline Summary

| Week | Person A | Person B |
|------|----------|----------|
| **W1** | Task 1.1: VP method inventory | Task 1.2: Classify all command VP usage |
| **W2** | Task 1.4: Bug triage | Task 1.3: Design doc (both) |
| **W3** | Task 2.1: visible_units Position overloads | Task 2.2: editing_commands_utilities overloads |
| **W4** | Task 2.1: (continued) | Task 2.3: CompositeEditCommand helper alternatives |
| **W5** | Task 3.1: Simple commands cleanup | Task 3.2: Medium commands cleanup |
| **W6** | Task 4.1: CompositeEditCommand private methods | Task 3.3: Shared utilities (both) + Task 4.2: MoveParagraph |
| **W7** | Task 4.1: (continued) + Task 4.3: indent_outdent, insert_list | Task 4.2: (continued) + Task 4.3: delete_selection, replace_selection |
| **W8** | Task 4.3: (continued) + buffer | Task 4.3: (continued) + buffer |

---

## Validation Strategy

### Per-CL Validation
- [ ] All existing `editing/` unit tests pass (`blink_unittests --gtest_filter="*Editing*"`)
- [ ] All Web Platform Tests for editing pass (`third_party/blink/web_tests/editing/`)
- [ ] No new layout test failures in `editing/` and `contenteditable/` suites
- [ ] Manual smoke test: basic typing, delete, paste, undo/redo, list operations, indent/outdent

### Phase Gate Validation
- [ ] Run full `blink_unittests` suite
- [ ] Run `content_browsing_tests` editing subset
- [ ] Performance check: no regression in editing benchmark (if one exists)
- [ ] Check `visible_position.h` include count decreasing over time

### Regression Monitoring
- [ ] Watch CQ (commit queue) for editing test failures in the weeks after landing
- [ ] Monitor crbug.com for new editing bugs that could be related

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| VP removal changes behavior subtly | High - editing correctness | Small CLs, extensive test coverage, each CL independently revertable |
| Some VP usage is actually necessary (e.g., for correct caret affinity at line wraps) | Medium - blocks cleanup | Classification in Phase 1 identifies these; leave legitimate uses, document why |
| visible_units functions deeply depend on VP internally | Medium - blocks overloads | May need to refactor visible_units.cc internals first |
| Large CLs in Phase 4 are hard to review | Medium - velocity | Break into smaller sub-CLs (e.g., one function per CL) |
| Chromium owners push back on approach | High - blocks all work | Design doc review in Week 2 catches this early |
| 8 weeks insufficient for XL commands | Medium - incomplete | XL commands are scoped as "significant progress" not "complete"; remaining work carries over |

---

## Success Criteria

### Minimum (8 weeks)
- Design doc approved by editing-dev owners
- All Position-based overloads landed (Phase 2)
- Simple and medium commands cleaned up (Phase 3)
- CompositeEditCommand private methods substantially cleaned up
- At least 3 heavy commands have significant VP reduction

### Stretch
- All 6 heavy composite commands fully cleaned up
- VP include removed from at least 5 command files
- `MoveParagraph` family fully migrated to Position-based API

### Measurable Metrics
- `VisiblePosition` references in `editing/commands/*.cc` reduced from ~391 to <100
- `CreateVisiblePosition` calls in commands reduced from ~126 to <30
- `.DeepEquivalent()` calls reduced from ~288 to <60
