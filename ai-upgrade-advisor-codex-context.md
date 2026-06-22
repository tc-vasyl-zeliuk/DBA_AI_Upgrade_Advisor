# Context for Codex: AI Upgrade Advisor for Managed Data Platforms

## Purpose of this file
This file captures the current project context for the **AI Upgrade Advisor for Managed Data Platforms** based on the conversation so far. It is intended to be pasted into or referenced by Codex in VS Code so implementation work stays aligned with the product concept, scope, terminology, and constraints already discussed.

---

## 1. Working product definition
Build an **AI-powered upgrade governance and decision-support platform** for AWS-managed data services.

The product should continuously assess managed data platforms for:
- engine/version support lifecycle risk
- upcoming end-of-standard-support deadlines
- Extended Support exposure and avoidable cost
- upgrade readiness and likely blockers
- business prioritization and execution guidance

This is **not** just a version dashboard. It should function as a **governance and upgrade decisioning layer**.

### Working name
**AI Upgrade Advisor for Managed Data Platforms**

### One-line pitch
An AI-powered platform that detects support risk across AWS data services, quantifies business and cost impact, and produces prioritized, evidence-backed upgrade plans with optional controlled automation.

---

## 2. Problem statement
The target organization runs multiple AWS-managed data services and needs a better way to stay within vendor-supported versions, avoid surprise upgrade projects, and reduce avoidable support costs.

Current pain points likely include:
- fragmented visibility of engine versions across services/accounts/environments
- support deadlines discovered too late
- reactive upgrade planning
- unclear prioritization across many assets
- weak linkage between technical upgrade risk and business impact
- inconsistent upgrade readiness checks
- risk of incurring paid Extended Support unnecessarily

The solution should convert raw service/version state into **actionable upgrade guidance**, not just alerts.

---

## 3. Target environment and technology scope
### Primary AWS-managed engines/services in scope
- **Amazon RDS for MariaDB**
- **Amazon RDS for PostgreSQL**
- **Amazon ElastiCache for Redis OSS / Valkey transition context**
- **Amazon OpenSearch Service**

### Initial bias
AWS-focused first. Avoid broad multi-cloud scope in MVP.

### Environment assumptions
The user operates in a database architecture / platform engineering context and already works with:
- database code deployments
- database modeling
- DBA routines in AWS
- Terraform / IaC automation
- centralized user management for some services

---

## 4. Product goals
### Primary goals
1. Keep fleets within supported engine/version windows.
2. Avoid or reduce Extended Support charges.
3. Detect and prioritize upgrade work earlier.
4. Produce actionable, evidence-backed upgrade preparation guidance.
5. Optionally automate safe, tightly controlled parts of the workflow.

### Success outcomes
- fewer assets in or near unsupported state
- lower avoidable support spend
- fewer emergency upgrade projects
- faster planning and approval cycles
- better leadership visibility into platform upgrade risk

---

## 5. Non-goals for MVP
Do **not** make the MVP any of the following:
- a full autonomous production upgrade system
- a general observability platform
- a generic LLM chatbot over AWS inventory
- a cross-cloud governance platform from day one
- a deep schema/data-model intelligence product (that is the separate “AI-assisted Data Management Control Plane” concept)

Keep the MVP focused on **version lifecycle governance + upgrade decision support**.

---

## 6. Core capabilities
### 6.1 Lifecycle monitoring
The system should:
- discover engine/version inventory
- identify service type, major/minor version, region, environment, ownership metadata where available
- map versions to support lifecycle state
- warn before standard support ends
- identify assets already in Extended Support or near end-of-life

### 6.2 Upgrade readiness assessment
The system should assess likely upgrade complexity and blockers, such as:
- major vs minor upgrade path
- allowed upgrade targets
- version path constraints
- parameter/configuration drift
- incompatible or risky extensions/plugins
- replica and topology considerations
- maintenance window considerations
- application/client dependency concerns
- rollback readiness
- known operational blockers for in-place upgrades

### 6.3 Prioritization
The system should rank upgrade work using factors like:
- support urgency
- production criticality
- business criticality
- cost exposure
- blast radius
- complexity/risk
- readiness/confidence

### 6.4 Actionable guidance
The system should generate:
- per-asset upgrade brief
- recommended target version
- explanation of why action is needed
- blocker summary
- preparation checklist
- test and validation suggestions
- rollback considerations
- owner-facing summary
- leadership-facing status summary

### 6.5 Optional automation
Automation is possible later, but only under strict controls.
Candidate automations:
- open tickets
- generate change records / change request drafts
- schedule non-prod rehearsals
- generate checklists/runbooks
- validate parameter groups against a target version
- trigger approved low-risk prechecks

Production upgrade execution should require explicit approval and should not be default MVP behavior.

---

## 7. Product principles
1. **Rules and authoritative vendor data are source of truth.**
2. **AI explains, summarizes, drafts, and recommends.**
3. **Severity and policy state should come from deterministic logic, not free-form LLM judgment.**
4. **Production changes require human approval.**
5. **Every recommendation should have evidence and confidence metadata.**
6. **The system should be useful even with incomplete metadata.**
7. **Decision support first, automation second.**

---

## 8. Why this idea is strong
Compared with a broader governance idea, this concept is stronger because it maps directly to:
- operational risk
- security/support hygiene
- platform cost control
- measurable outcomes

It is easier to justify because it can tie directly to:
- support deadlines
- billable Extended Support exposure
- upgrade backlog reduction
- business service risk

---

## 9. Current factual anchors to preserve
These are included to keep the implementation concept aligned with current AWS lifecycle realities. They should still be treated as vendor-data-backed facts that the product refreshes from authoritative sources rather than hard-coding permanently.

- **RDS for PostgreSQL 13** reached end of standard support on **2026-02-28**, and Extended Support starts **2026-03-01**. **PostgreSQL 14** standard support ends **2027-02-28**. Source: AWS RDS PostgreSQL release calendar.  
  Reference: https://docs.aws.amazon.com/AmazonRDS/latest/PostgreSQLReleaseNotes/postgresql-release-calendar.html

- **ElastiCache Redis OSS 4 and 5** entered Extended Support starting **2026-02-01** after standard support ended on **2026-01-31**. AWS states these versions are eligible for Extended Support up to **2029-01-31**, after which remaining caches are automatically upgraded to the latest Valkey.  
  References:  
  - https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/extended-support-versions.html  
  - https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/extended-support.html

- **Amazon OpenSearch Service** has published standard and extended support timelines for older Elasticsearch and OpenSearch versions, and supports in-place upgrades across documented version paths.  
  References:  
  - https://aws.amazon.com/blogs/big-data/amazon-opensearch-service-announces-standard-and-extended-support-dates-for-elasticsearch-and-opensearch-versions/  
  - https://docs.aws.amazon.com/opensearch-service/latest/developerguide/version-migration.html

- **RDS Extended Support** is relevant especially for **RDS for PostgreSQL** and should be explicitly modeled as a cost/risk dimension.  
  References:  
  - https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/extended-support.html  
  - https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/extended-support-versions.html

These references should be used by the eventual product as seed sources or test cases, not necessarily as permanent static inputs.

---

## 10. Expected users
### Primary users
- database architects
- platform engineers
- DBAs
- cloud infrastructure engineers
- service owners

### Secondary users
- engineering managers
- CTO / head of platform / leadership stakeholders
- governance / risk / security stakeholders

### User needs by role
#### Engineers / DBAs
- tell me what is urgent
- tell me what blocks the upgrade
- tell me the likely target version and prep steps
- give me an evidence-backed checklist

#### Managers / leadership
- show portfolio risk
- show what is already incurring cost
- show what must be done in the next 30/60/90/180 days
- show ownership and backlog

---

## 11. Suggested MVP
The MVP should **not** start with automation.

### Recommended MVP scope
1. fleet inventory collection
2. support lifecycle mapping
3. risk and priority scoring
4. per-asset generated upgrade brief
5. portfolio dashboard / report view

### Minimum viable outputs
#### Per asset
- service type
- engine/version
- environment
- account/region
- owner (if known)
- support phase
- deadline(s)
- risk severity
- recommended target version
- likely blockers/checks
- suggested due date
- confidence/evidence

#### Per portfolio
- assets already in Extended Support
- assets entering risk in 30/60/90/180 days
- count by engine/version/support phase
- cost-at-risk by engine/team/environment
- prioritized remediation backlog

---

## 12. Example decision outputs the product should be able to generate
### Example: PostgreSQL 13 on RDS in production
- already in Extended Support as of 2026-03-01
- incurring avoidable cost
- recommend upgrade to a supported target, subject to app certification constraints
- validate extensions, parameter groups, replica topology, maintenance window, and application driver compatibility
- high priority if customer-facing or production-critical

### Example: ElastiCache Redis OSS 5
- already in Extended Support as of 2026-02-01
- target path should consider Redis OSS 6+/7+ or Valkey depending policy and compatibility
- check client compatibility, Lua/script behavior, ACL expectations, cluster mode, and failover rehearsal needs
- higher priority when node count / spend / production criticality are high

### Example: OpenSearch older version
- determine support status against AWS lifecycle timeline
- confirm supported in-place migration path
- surface integration compatibility checks (ingestion pipelines, clients, plugins)
- generate rehearsal and rollback guidance

---

## 13. Proposed logical architecture
This does **not** need to be final implementation architecture. It is the working conceptual architecture Codex should align to.

### 13.1 Inventory and normalization layer
Collect from AWS APIs and metadata sources:
- RDS instances / clusters
- ElastiCache clusters / replication groups
- OpenSearch domains
- tags and ownership metadata
- environment/account/region data
- optional CMDB / service catalog data

Normalize assets into a common model, e.g.:
- asset_id
- service_family
- engine
- engine_version
- major_version
- minor_version
- env
- account_id
- region
- owner_team
- business_service
- criticality
- topology_summary
- last_seen_at

### 13.2 Lifecycle knowledge base
Maintain support facts such as:
- standard support start/end
- Extended Support start/end
- supported upgrade targets
- forced transition dates if applicable
- notes relevant to compatibility or policy

This knowledge should be refreshable from authoritative vendor data.

### 13.3 Policy and scoring engine
Deterministic rules should classify:
- support state
- urgency window
- policy compliance
- cost exposure state
- severity / priority score

A simple scoring model may include:
- support_risk
- cost_risk
- criticality
- complexity_modifier
- readiness_score
- confidence_score

Example conceptual formula:
`priority = support_risk + cost_risk + criticality + complexity_modifier - readiness_score`

### 13.4 AI reasoning layer
Use AI to:
- generate concise asset summaries
- explain severity and rationale
- draft upgrade plans/checklists
- summarize known blockers and next steps
- create leadership narratives

Do **not** use AI as the source of truth for lifecycle dates or severity assignment.

### 13.5 Workflow / integration layer
Potential outputs:
- Slack / Teams alerts
- Jira tickets
- change record drafts
- email digest / summary report
- dashboard cards
- non-prod rehearsal job creation

---

## 14. Data model suggestions for implementation
Codex can use this as a starting point for initial domain modeling.

### Core entities
#### Asset
Represents a managed data service instance/domain/cluster.

Possible fields:
- id
- provider
- service_type
- engine
- engine_version
- engine_major
- engine_minor
- arn_or_resource_id
- account_id
- region
- environment
- owner_team
- business_service
- criticality
- topology_json
- tags_json
- discovered_at
- updated_at

#### LifecyclePolicy
Represents support lifecycle facts for a product/version.

Possible fields:
- id
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

#### UpgradeAssessment
Represents a point-in-time evaluation of an asset.

Possible fields:
- id
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
- blockers_json
- recommendations_json
- assessed_at

#### Finding
Represents a specific issue or recommendation.

Possible fields:
- id
- asset_id
- assessment_id
- category
- severity
- title
- detail
- evidence_json
- status
- created_at

#### GeneratedBrief
Represents AI-generated human-readable output.

Possible fields:
- id
- assessment_id
- audience_type
- markdown_body
- model_name
- generated_at

---

## 15. UX expectations
### Engineer-facing view
Should answer:
- what is wrong
- why it matters
- how urgent it is
- what target version to consider
- what checks to run before scheduling change
- what evidence supports the recommendation

### Manager-facing view
Should answer:
- what is the overall portfolio risk
- which teams own the risky assets
- where costs are accumulating
- what must be completed this quarter

### Design bias
Favor clarity and prioritization over dense telemetry.

---

## 16. Safety, trust, and control requirements
The product should be conservative.

### Must-have controls
- human approval for production change execution
- evidence attached to recommendations
- confidence score on generated output
- deterministic policy evaluation before AI narrative generation
- auditability of assessments and generated advice

### Avoid
- autonomous upgrade actions by default
- unsupported claims about compatibility without evidence
- mixing policy truth with speculative model output

---

## 17. Integration ideas
Potential data/integration sources:
- AWS SDK / boto3
- AWS Config
- AWS Resource Groups Tagging API
- CloudWatch for maintenance or operational signals
- service catalogs / CMDB if available
- Jira / ServiceNow for work item creation
- Slack / Teams for delivery of alerts and summaries

Potential implementation modes:
- scheduled assessment job
- event-driven change detection where feasible
- API + UI + background worker architecture

---

## 18. Relationship to the separate concept
There are **two separate product ideas** discussed in the broader conversation:
1. **AI-assisted Data Management Control Plane**
2. **AI Upgrade Advisor for Managed Data Platforms**

This file is for **#2 only**.

Do not let implementation drift into a broad data catalog / schema governance product unless explicitly requested.

---

## 19. Open questions that remain unresolved
These should be preserved as open design questions rather than silently assumed.

1. What is the authoritative refresh mechanism for lifecycle timelines?
2. How exactly should cost exposure be estimated for each service type?
3. Where will ownership and business criticality metadata come from?
4. What evidence threshold is required before recommending a target version?
5. What automation level is acceptable in non-prod vs prod?
6. Should the first release be API-first, dashboard-first, or report-first?
7. How much custom policy overlay should be supported on top of AWS/vendor rules?

---

## 20. Recommended implementation sequence for Codex
A sensible build order:

### Phase 1
- define domain models
- implement AWS inventory collectors
- normalize asset records
- store version/lifecycle data

### Phase 2
- implement lifecycle rule evaluation
- compute support phase and urgency windows
- create risk scoring and prioritization

### Phase 3
- generate markdown upgrade briefs per asset
- generate portfolio summary outputs
- attach evidence/confidence

### Phase 4
- add workflow integrations (ticketing, notifications)
- add review/approval flow
- add optional rehearsal automation for non-prod

---

## 21. Guidance for Codex on decision boundaries
When implementing this concept, prefer:
- deterministic classification for support state
- explicit schemas and typed models
- audit-friendly storage of inputs and outputs
- pluggable vendor lifecycle sources
- explainable scoring
- modular engine-specific rule packs

Avoid:
- hard-coding too many AWS facts directly into business logic
- unstructured prompts as the main source of decisioning
- mixing data collection, rule evaluation, AI generation, and workflow execution into one monolith without boundaries

---

## 22. Suggested repository framing
Possible top-level structure:

```text
/src
  /collectors
    /aws
      rds.py
      elasticache.py
      opensearch.py
  /models
  /rules
    /lifecycle
    /scoring
    /engine_packs
  /services
    inventory_service.py
    assessment_service.py
    brief_generation_service.py
  /integrations
    jira.py
    slack.py
  /api
  /jobs
  /ui
/docs
  architecture.md
  scoring.md
  lifecycle-sources.md
/tests
```

This is illustrative, not prescriptive.

---

## 23. Final instruction to Codex
Treat this as a **focused AWS data-platform upgrade governance product**.

Optimize for:
- clarity of scope
- explicit domain modeling
- deterministic policy evaluation
- useful upgrade preparation outputs
- evidence-backed AI summaries

Do not expand scope into unrelated data governance or generic observability unless specifically asked.
