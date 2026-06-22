# Policy And Scoring

## Deterministic rule boundary
These states should come from rules, not from LLM inference:
- support phase
- urgency window
- policy compliance
- Extended Support exposure
- severity class
- priority score

## Example scoring dimensions
- support_risk
- cost_risk
- criticality
- complexity_modifier
- readiness_score
- confidence_score

## Conceptual priority formula
`priority = support_risk + cost_risk + criticality + complexity_modifier - readiness_score`

## Required evidence posture
- every lifecycle fact should point to a source
- every recommendation should carry confidence metadata
- target version recommendations should only be emitted when evidence exists

## Policy layering model
1. vendor lifecycle facts
2. internal organizational policy overlays
3. asset-specific overrides with audit trail
