# Logical Architecture

## Working architecture

### 1. Inventory and normalization layer
Inputs:
- AWS APIs
- tags and ownership metadata
- optional CMDB or service catalog inputs

Outputs:
- normalized asset records with stable identity and metadata needed for assessment

### 2. Lifecycle knowledge base
Inputs:
- authoritative vendor lifecycle sources
- internal policy overlays when approved

Outputs:
- version lifecycle records with source traceability and refresh timestamps

### 3. Policy and scoring engine
Responsibilities:
- determine support phase
- determine policy compliance
- compute urgency windows
- model cost exposure
- generate deterministic priority and severity

### 4. AI reasoning layer
Responsibilities:
- explain assessments
- draft briefs and checklists
- summarize blockers and next steps

Constraint:
- AI is not allowed to invent lifecycle dates or severity states.

### 5. Workflow layer
Outputs:
- briefs
- portfolio reports
- ticket drafts
- alert payloads
- review and approval artifacts
