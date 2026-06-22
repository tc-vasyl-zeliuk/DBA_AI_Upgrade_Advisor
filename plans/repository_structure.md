# Repository Structure

Decision:
- Keep the repository technology-agnostic until the first implementation mode is chosen explicitly.

Created now:
```text
DBA_AI_Upgrade_Advisor/
  context/
    ai/
    domain/
    product/
  plans/
  prototype_input/
    assessments/
    inventory/
    lifecycle/
  artifacts/
    assessments/
    briefs/
    portfolio/
  ai-upgrade-advisor-codex-context.md
  README.md
  .gitignore
```

Why this shape:
- `context/` holds durable product and domain truth.
- `plans/` isolates changeable execution decisions.
- `prototype_input/` gives the future pipeline a file-based prototyping boundary.
- `artifacts/` defines where generated outputs will land.

Intentionally deferred:
- `src/`
- `apps/`
- framework-specific API folders
- deployment and infrastructure directories
- test harness layout

Why deferred:
- Fact: the product scope is defined.
- Fact: the implementation stack is not defined.
- Inference: choosing code and deployment folders now would be guesswork.
