# AI Upgrade Advisor for Managed Data Platforms

This repository now treats `ai-upgrade-advisor-codex-context.md` as the seed document, not the ongoing source of truth.

The working set is split into:
- `context/` for durable product, domain, and AI-grounding context
- `plans/` for repository shape, delivery order, and unresolved decisions
- `prototype_input/` for file-based inventory and lifecycle policy inputs used in prototyping
- `artifacts/` for generated assessments, briefs, and portfolio outputs

Fact:
- The product scope is defined.
- The implementation stack is not defined.

Inference:
- The repository stays technology-agnostic for now.
- Creating `src/`, framework folders, or deployment scaffolding now would be premature.

Recommended read order:
1. `context/product/product_brief.md`
2. `context/product/value_proposition.md`
3. `context/product/scope_and_non_goals.md`
4. `context/product/user_and_decision_views.md`
5. `context/domain/capability_map.md`
6. `context/domain/reference_upgrade_scenarios.md`
7. `context/domain/logical_architecture.md`
8. `context/domain/data_model.md`
9. `context/domain/policy_and_scoring.md`
10. `context/domain/ai_reasoning_and_workflow.md`
11. `context/ai/assistant_operating_contract.md`
12. `plans/repository_structure.md`
13. `plans/delivery_sequence.md`
14. `plans/sop_automation_pattern.md`
15. `plans/sop_in_place_minor_update_pattern.md`
16. `plans/sop_in_place_major_upgrade_via_terraform_pattern.md`
17. `plans/open_questions.md`
