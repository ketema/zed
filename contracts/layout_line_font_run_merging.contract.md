# Contract: layout_line() Font Run Merging

**Module**: `crates/gpui/src/text_system.rs`
**Method**: `WindowTextSystem::layout_line()`
**Issue**: #48670 — Highlighting `/` causes text shift

## Context

`layout_line()` converts `TextRun`s (which combine font + decoration info) into
`FontRun`s (which contain only `{len, font_id}`). FontRuns are passed to the
platform text shaper which determines glyph positions and ligatures.

Decorations (color, underline, strikethrough) are handled separately by
`DecorationRun`s in `shape_line()`, which are applied AFTER shaping.

## Contracts

### PRE-LLM-01: Valid Input Runs
TextRuns passed to `layout_line()` MUST have `len > 0` and valid font specifications.
The sum of all `run.len` values MUST equal the byte length of `text`.

### POST-LLM-01: Font-Only Merging
Adjacent TextRuns that resolve to the same `font_id` MUST be merged into a single
FontRun, regardless of any decoration differences (color, underline, strikethrough,
background_color).

**Rationale**: FontRun boundaries determine where the platform text shaper breaks
text for shaping. Breaking at decoration boundaries causes ligature breaks and
glyph repositioning — the root cause of #48670.

### POST-LLM-02: FontRun Boundaries At Font Changes Only
FontRun boundaries MUST occur if and only if adjacent TextRuns resolve to different
`font_id` values. No other property of TextRun (color, underline, strikethrough,
background_color) SHALL influence FontRun boundaries.

### POST-LLM-03: Merged FontRun Length Correctness
When adjacent TextRuns are merged into a single FontRun, the resulting FontRun's
`len` MUST equal the sum of the merged TextRuns' `len` values.

### POST-LLM-04: Total Length Preservation
The sum of all FontRun `len` values MUST equal the sum of all input TextRun `len`
values (i.e., the byte length of the input text).

### INV-LLM-01: Decoration Independence
The set of FontRuns produced by `layout_line()` MUST be identical for any two
invocations with the same `text`, `font_size`, and TextRuns that resolve to the
same sequence of `font_id` values — regardless of decoration differences in those
TextRuns.

**Rationale**: This is the core invariant that prevents text shifting. Selecting
text changes decorations (color via `ensure_minimum_contrast`) but must not change
FontRun boundaries, ensuring identical glyph positions.

### INV-LLM-02: Layout Stability Under Selection
For any given text with a fixed set of fonts, the `LineLayout` (glyph positions)
MUST be identical whether or not any portion of the text is selected/highlighted.
Selection only changes decoration (color), which MUST NOT affect shaping.

### ERRORS-LLM-01: Empty Runs
If `runs` is empty, `layout_line()` produces zero FontRuns. The resulting
LineLayout represents empty text. No panic.

## Test Traceability

Tests MUST cite clause IDs:
- `POST-LLM-01`: Test that same-font runs with different colors merge
- `POST-LLM-02`: Test that different-font runs do NOT merge
- `POST-LLM-03`: Test merged length equals sum of parts
- `POST-LLM-04`: Test total length preservation
- `INV-LLM-01`: Test decoration independence (same fonts, different colors → same FontRuns)
