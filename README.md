# AI Developer Workflow Comparison Study

**SVP Executive Briefing — AI-Assisted Software Engineering Workflows**

🌐 **Live Report:** [https://inba-ganapathy.github.io/ai-workflow-comparison](https://inba-ganapathy.github.io/ai-workflow-comparison)

---

## Overview

A rigorous, data-driven comparison of three AI developer workflows evaluated against four real engineering scenarios on production codebases (claude-talk and beethoven/GenUI Services).

| Workflow | Description |
|----------|-------------|
| 🔬 **Innerloop** | Structured 3-phase pipeline: Investigate → Execute → Review. Specialist subagents, TDD, plan audit, independent code review. |
| ⚡ **Direct Implementation** | No structured workflow. Read → identify → fix → verify. Single agent, no phases, no review. |
| 🚀 **ShipIt** | Salesforce `automation-platform-skill-poc`. Full multi-agent pipeline: Analyst → QA → Engineer → [Reviewer ∥ Validator] → Ship. Designed for GUS W-numbers with Manager approval gates. |

---

## Scenarios

| # | Scenario | Task Type | Codebase | Status |
|---|----------|-----------|----------|--------|
| 1 | Open Task | Cold-start bug discovery from raw prompt | claude-talk | 🔄 Re-running with real ShipIt |
| 2 | Pre-identified Bug Fix | 10 known bugs, exact file:line locations given | claude-talk | 📋 Pending |
| 3 | Work Item W-21632772 | GUS User Story — "Parameterize output for loops in structured data" | beethoven (GenUI Services) | 📋 Pending |
| 4 | Second Brain Feature | Feature build from structured product prompt | claude-talk | 📋 Pending |

---

## Repository Structure

```
scenarios/
  scenario1-open-task/{innerloop,direct,shipit}/
    plan.md            ← Generated plan / battle plan
    metrics.json       ← Timing, tokens, tool calls, bugs found
    diff.patch         ← Git diff of changes made
    council-review.md  ← SFDC Council findings
  scenario2-bugfix/...
  scenario3-workitem-wi/...   ← W-21632772
  scenario4-secondbrain/...
council-reviews/       ← Full SFDC Council (5-model) review reports
docs/
  index.html           ← SVP analysis website (GitHub Pages)
```

---

## Data Integrity

Every data point is verifiable with a cited source:
- GUS W-21632772 queried live from `gus.lightning.force.com` on 2026-05-17
- beethoven commits cited by SHA: `46062b3`, `9a3951f` (baseline: `372d164`)
- Token counts from actual agent `usage` fields
- Zero assumptions — all gaps are explicitly labeled in the report

---

*Generated: May 17, 2026 | Repositories: claude-talk + beethoven | Workflows: Innerloop, Direct, ShipIt (automation-platform-skill-poc)*
