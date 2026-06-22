# Context

Purpose:
- Hold reusable product, domain, and AI-grounding documents.
- Keep stable facts and working assumptions separate from execution plans.

Structure:
- `product/` defines scope, value, users, and outcomes.
- `domain/` defines capabilities, logical architecture, data model, and scoring logic.
- `ai/` defines how AI is allowed to participate in the product.

Constraint:
- The original seed document remains in the repository root for traceability.
