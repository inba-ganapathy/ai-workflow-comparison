# Battle Plan — W-21632772

## Bug Summary
The UEM (User Experience Message) structured output currently requires the LLM to repeat identical component blocks for every item in a list (e.g., 5 product cards = 5 separate component definitions). This is wasteful in output tokens and doesn't scale for complex input data. The work item asks for "for:each" loop parameterization in the LLM output template, allowing a single block definition to be annotated with iteration data, and then expanded into the full tree before returning to the client.

## Root Cause
**File**: `genui/processing/uem.py`
**Lines**: The `post_process_composable_ui` function (line 129) and `build_uem_hierarchy` (line 316) — these functions receive a flat `components` list and build a hierarchical UEM tree. They currently have no concept of a loop/iteration directive. There is also no expansion logic anywhere in the pipeline.

**Issue**: The `_get_composable_ui_output_schema()` in `genui/tools/core_tools.py` (line 823) defines the JSON schema sent to the LLM for composable UI generation. It does not include any `for_each` or `items` fields that would let the LLM output a compact looped structure. Similarly, `post_process_composable_ui` in `genui/processing/uem.py` does not expand any such directive.

**Confidence**: HIGH

## Affected Files

### Source Files
| File | Type | Role in Fix |
|------|------|-------------|
| `genui/processing/uem.py` | Backend | Add `expand_for_each_components()` function + call it in `post_process_composable_ui` |
| `genui/tools/core_tools.py` | Backend | Update `_get_composable_ui_output_schema()` to add `for_each`/`items` fields to component schema |

### Test Files
| File | Type | Status |
|------|------|--------|
| `tests/unit/test_genui/processing/test_uem.py` | Backend Test | Exists — needs new test class for for-each expansion |

## Fix Strategy

### AC1: Structured output response is parameterized
The LLM output schema for composable UI must support a `for_each` field on each component. When present, it signals that this component acts as a template to be replicated once per item in the `for_each.items` list.

Schema change in `_get_composable_ui_output_schema()` (core_tools.py):
- Add `for_each` optional object to each component's schema, with sub-fields:
  - `items`: array of objects (each item is a dict of parameter key/value pairs)
  - `index_attr`: optional string (which attribute name should receive the loop index)

### AC2: Parameterized response is correctly expanded into the full tree
In `post_process_composable_ui` (uem.py), before building the UEM hierarchy, iterate over `components` and expand any component with `for_each` into N copies — one per item. Each copy:
- Gets a unique `component_id` (e.g., `{original_id}_{i}`)
- Gets the `parent_id` and `region` from the template component
- Has its attributes merged/overridden with the item's parameter values (parameter values in attributes use `{{param_key}}` substitution)
- Gets an `index` = i for ordering

This expansion happens in a new `expand_for_each_components()` function called at the start of `post_process_composable_ui`, before the existing hierarchy-building logic. The expanded flat list is then processed exactly as before.

## Implementation Details

### Data contract (what the LLM emits)

```json
{
  "components": [
    {
      "component_id": "card_template",
      "definition": "lightning/card",
      "attributes": [{"key": "title", "value": "{{name}}"}, {"key": "subtitle", "value": "{{role}}"}],
      "parent_id": null,
      "region": "default",
      "for_each": {
        "items": [
          {"name": "Alice", "role": "Engineer"},
          {"name": "Bob", "role": "Manager"}
        ]
      }
    }
  ]
}
```

### After expansion (what `expand_for_each_components` produces)

```json
[
  {"component_id": "card_template_0", "definition": "lightning/card",
   "attributes": [{"key": "title", "value": "Alice"}, {"key": "subtitle", "value": "Engineer"}],
   "parent_id": null, "region": "default", "index": 0},
  {"component_id": "card_template_1", "definition": "lightning/card",
   "attributes": [{"key": "title", "value": "Bob"}, {"key": "subtitle", "value": "Manager"}],
   "parent_id": null, "region": "default", "index": 1}
]
```

### Template substitution rules
- Attribute values in `for_each` components that match `{{key}}` are replaced by the corresponding item value.
- Attribute values that don't contain `{{...}}` are kept as-is (shared across all expanded copies).
- Components without `for_each` are passed through unchanged.
- `for_each` with an empty `items` list results in 0 components (component is dropped).

## Engineers Needed
- [x] Backend Engineer (Python)

## Setup Requirements
No special environment setup — unit tests are fully in-process with no LLM or service calls.

## Quality Gates Discovered
| Gate | Command | Applies |
|------|---------|---------|
| Lint | `ruff check genui/ tests/` | Yes |
| Types | `mypy genui/ --strict` | Yes |
| Tests | `/Users/inbasagar.ganapathy/eng/beethoven/.venv/bin/python -m pytest tests/unit/ --tb=short -q` | Yes |
| Format | `black genui/ tests/ --line-length 100` | Yes |

## Risk Assessment
- **Schema extension is additive** — `for_each` is an optional field on each component, existing LLM responses without it work unchanged.
- **Only uem.py and core_tools.py change** — no other code paths are affected.
- **Child components referencing a `for_each` parent** need their `parent_id` rewritten to the expanded copies (or simply dropped if the template has children). This edge case should be documented but is out of scope for the core feature (the LLM typically doesn't add children to a looped template).
- **Nested for-each** (a for-each inside another for-each) is not required and not supported in this implementation.
