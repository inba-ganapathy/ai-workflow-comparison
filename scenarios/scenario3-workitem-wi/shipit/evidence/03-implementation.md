# Backend Fix — W-21632772

## Summary
Added `for_each` loop expansion to the UEM post-processing pipeline, allowing the LLM to emit a single parameterized template component instead of duplicating it for every data item.

## Changes
| File | Lines Changed | What | Why |
|------|--------------|------|-----|
| `genui/processing/uem.py` | +104 / -0 | Added `_substitute_template_value`, `_apply_substitutions`, `expand_for_each_components`; integrated call into `post_process_composable_ui` | Core expansion logic + integration point for AC2 |
| `genui/tools/core_tools.py` | +28 / -0 | Added `for_each` optional field to `_get_composable_ui_output_schema()` component schema | Teaches the LLM it can emit for_each (AC1) |
| `tests/unit/test_genui/processing/test_uem.py` | +209 / -0 | Added `TestExpandForEachComponents` (10 tests) and `TestPostProcessComposableUIWithForEach` (3 tests) | TDD coverage for both ACs |

## Test Results
- All 13 new tests: PASS
- Full suite: 944 passed, 5 skipped (baseline was 931 + 5)

## Diff Summary

### `genui/processing/uem.py`
**New functions added (before `post_process_block_instance`):**

1. `_substitute_template_value(value, item)` — regex-based `{{key}}` substitution in a single string value
2. `_apply_substitutions(attributes, item)` — applies substitution to both list-of-dicts and plain dict attribute formats
3. `expand_for_each_components(components)` — public function that:
   - passes non-`for_each` components through unchanged
   - drops `for_each` components with empty `items`
   - expands `for_each` components into N copies with unique IDs, sequential index, substituted attributes

**Integration point in `post_process_composable_ui`:**
After extracting the flat `components` list and before alias substitution, call `expand_for_each_components(components)`. The expanded list then flows through the rest of the hierarchy-building pipeline unchanged.

### `genui/tools/core_tools.py`
**`_get_composable_ui_output_schema()` updated:**
Added an optional `for_each` object property to each component item's schema. It contains:
- `items`: array of parameter objects (the loop data)
The description explicitly instructs the LLM on when and how to use it.

## Safety Analysis
- **Thread safety**: All new functions are pure / stateless — no shared mutable state.
- **API compatibility**: `expand_for_each_components` is a new export. `post_process_composable_ui` signature is unchanged. Schema change is additive (new optional field).
- **Performance impact**: O(N×M) where N=components and M=items per template. Both N and M are bounded by LLM output size limits. No database or I/O involved. Net effect: reduces LLM output tokens for looped data (the whole point of the feature).
- **Multi-tenant safety**: N/A — pure in-memory transformation.

## Edge Cases Handled
- `for_each` with empty `items` → component is dropped, log warning
- No `component_id` on template → fallback ID `for_each_{idx}` used as base
- Attribute values with no `{{...}}` placeholders → passed through unchanged
- Dict-format attributes (not just array format) → `_apply_substitutions` handles both
- `{{key}}` with missing key in item dict → placeholder left intact (safe degradation)
- Components without `for_each` coexist with `for_each` components in the same list
