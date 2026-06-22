# Data Model

## Core entities

### Asset
Represents a managed data resource under assessment.

Candidate fields:
- asset_id
- provider
- service_type
- engine
- engine_version
- engine_major
- engine_minor
- resource_id
- account_id
- region
- environment
- owner_team
- business_service
- criticality
- topology_summary
- tags
- discovered_at
- updated_at

### LifecyclePolicy
Represents support facts and recommended targets for a version family.

Candidate fields:
- policy_id
- service_type
- engine
- version_family
- major_version
- standard_support_end
- extended_support_start
- extended_support_end
- recommended_target_version
- source_url
- source_refreshed_at

### UpgradeAssessment
Represents point-in-time evaluation of an asset against policy.

Candidate fields:
- assessment_id
- asset_id
- lifecycle_policy_id
- support_phase
- days_to_standard_support_end
- days_to_extended_support_end
- extended_support_cost_risk
- upgrade_complexity_score
- business_priority_score
- overall_priority_score
- confidence_score
- blockers
- recommendations
- assessed_at

### Finding
Represents a discrete issue, recommendation, or blocker.

Candidate fields:
- finding_id
- asset_id
- assessment_id
- category
- severity
- title
- detail
- evidence
- status
- created_at

### GeneratedBrief
Represents AI-generated human-readable output.

Candidate fields:
- brief_id
- assessment_id
- audience_type
- markdown_body
- model_name
- generated_at
