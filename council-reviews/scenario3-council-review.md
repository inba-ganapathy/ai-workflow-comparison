# SFDC Council Review — Scenario 3: W-21632772
# "Parameterize output for loops in structured data"
# Reviewer: innerloop:innerloop-code-reviewer subagent
# Date: 2026-05-17

## Comparison vs Human Reference

Human reference (genui/processing/data_binding.py, 436 lines + 564-line tests):
- Template syntax: $data.field (dollar-sign paths), array indexing $data[0].field
- Loop syntax: $forEach on the component itself
- Tree model: Flat component list with parent_id; loop expansion replicates subtrees with ID suffixing
- Separate module: data_binding.py

## Innerloop (19/25)
- AC Coverage: 4/5 — Both ACs addressed; no test proving LLM schema includes forEach directive
- Code Quality: 4/5 — Clean, None sentinel issue (_resolve_path returns None for both missing key and key=None)
- Architecture: 4/5 — Good separate module, but hierarchical tree model diverges from codebase's flat-list+parent_id pattern
- Test Depth: 4/5 — 9 tests; missing: no _resolve_path isolation test, no deeply nested forEach test
- Completeness: 3/5 — No detect_structured_data(), no general expression engine, no array indexing

Key finding: _resolve_path at line 215 returns None for both missing key AND key with None value — indistinguishable.

## Direct (22/25)
- AC Coverage: 5/5 — Explicit test_ac1_structured_output_is_parameterized and test_ac2_parameterized_response tests
- Code Quality: 5/5 — Type preservation, configurable variable name, robust validation, deepcopy safety
- Architecture: 4/5 — Added to uem.py (natural home); could argue for separate module
- Test Depth: 5/5 — 29 tests across 3 classes, every helper tested in isolation
- Completeness: 3/5 — Narrower than human reference; no detect_structured_data(), no array indexing

Key finding: _substitute_value returns "{{missing}}" (sub-path without variable prefix) — slightly inconsistent debugging experience.

## ShipIt (17/25)
- AC Coverage: 3/5 — Both ACs functionally met but no explicit AC-named tests; no schema test
- Code Quality: 3/5 — No dot-path resolution ({{name}} works, {{item.name}} does NOT); no type preservation (always returns string via re.sub); shallow dict copy (aliasing bug risk)
- Architecture: 4/5 — Same integration point as Direct; clean minimal structure
- Test Depth: 4/5 — 24 tests but no helper unit tests; only integration coverage
- Completeness: 3/5 — No dot-path, no type preservation, no configurable variable name

Critical bugs:
1. _substitute_template_value always returns str(replacement) — numeric 95 becomes "95"
2. No dot-path: {{address.city}} looks up literal key "address.city", not traverses dict
3. Shallow copy: attributes dict shared between expanded components

## Rankings
1. Direct — 22/25 — Best AC coverage, type safety, test depth. Configurable variable name.
2. Innerloop — 19/25 — Clean architecture, good RAG integration. None sentinel issue.  
3. ShipIt — 17/25 — Two critical bugs (no type preservation, no dot-path). Simplest implementation.

## Feature Comparison Table
| Dimension | Human Ref | Innerloop | Direct | ShipIt |
|-----------|-----------|-----------|--------|--------|
| Template syntax | $data.field | {{item.field}} | {{variable.field}} | {{key}} |
| Dot-path access | Yes+array | Yes | Yes | No |
| Variable name | Fixed | Fixed | Configurable | N/A |
| Type preservation | Yes | Yes | Yes | No |
| Tree model | Flat+parent_id | Hierarchical | Flat+parent_id | Flat+parent_id |
| Deep copy safety | Yes | Yes | Yes | No |
| AC-named tests | No | No | Yes | No |
