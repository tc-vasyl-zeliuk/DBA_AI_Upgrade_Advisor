# SOP Automation Pattern

## Decision
Use a hybrid control pattern for RDS blue/green operational procedures:
- Terraform defines the steady-state infrastructure intent.
- GitHub Actions orchestrates the workflow.
- AWS CLI or SDK handles blue/green create, monitor, and switchover actions.
- Post-switchover automation reconciles Terraform state and operational integrations.

## Why this is the right pattern

### Facts
- Terraform AWS Provider supports RDS blue/green updates for `aws_db_instance`.
- That support works when the resource is a single self-contained DB instance.
- Terraform cannot treat a source instance and its replicas as one unit, so it cannot use a blue/green deployment to update an instance and its replicas together.
- AWS blue/green deployments rename DB instance identifiers during switchover and preserve resource IDs per physical resource, which means the new production resources do not have the old production resource IDs.

### Inference
- For a MariaDB primary plus replica topology, pure Terraform is the wrong operational abstraction.
- A hybrid workflow is not a workaround. It is the correct control plane split.

## Recommended ownership split

### Terraform should own
- DB instances in steady state
- subnet groups
- parameter groups
- option groups where applicable
- security groups
- alarms
- tags
- IAM policy scaffolding
- GitHub Actions role assumptions and runner-side configuration

### Workflow automation should own
- prechecks
- `describe-db-engine-versions` target validation
- `create-blue-green-deployment`
- deployment status polling
- gated switchover approval
- `switchover-blue-green-deployment`
- post-cutover reconciliation
- Jira and Confluence updates

## Recommended workflow shape

### 1. Precheck stage
- confirm source version, target version, Region, and instance topology
- confirm target version appears in `ValidUpgradeTarget`
- confirm custom parameter group strategy for the target major version
- confirm no unsupported blue/green feature is present:
  - cross-Region replicas
  - cascading replicas
  - RDS Proxy
- confirm rollback checkpoint:
  - snapshot baseline
  - ticket and change record created

### 2. Create green stage
- use AWS CLI to create the blue/green deployment with:
  - target engine version
  - target parameter group
  - optional storage or instance adjustments
- record the blue/green deployment identifier as workflow state
- publish deployment metadata to Jira and Confluence

### 3. Validate green stage
- wait for green environment readiness
- run smoke tests against green endpoints
- verify replica topology exists in green
- verify engine version and parameter group assignment
- verify application connection behavior

### 4. Approval stage
- require manual approval in GitHub Actions before production switchover
- attach validation evidence and rollback notes

### 5. Switchover stage
- use AWS CLI `switchover-blue-green-deployment`
- set an explicit switchover timeout
- monitor completion and capture renamed resources

### 6. Reconciliation stage
- discover new production DB instance identifiers and `DbiResourceId` values
- update integrations that key on resource IDs:
  - Performance Insights consumers
  - CloudTrail filters
  - AWS Backup selections
  - IAM database authentication policies
- reassociate attached IAM roles if needed
- reconcile Terraform state:
  - refresh configuration to the new production objects
  - import moved resources when required
  - remove or intentionally retain old blue resources under explicit control

### 7. Closeout stage
- update Jira status and change record outcome
- publish Confluence execution summary
- schedule deletion of the old blue environment only after confirmation and hold-period expiry

## What not to do
- Do not model a primary plus replica blue/green cutover as a single pure Terraform apply and call that IaC.
- Do not assume DB instance identifiers are the stable identity after switchover.
- Do not allow ad hoc green-side mutations that are not captured by workflow state.
- Do not destroy the old blue environment in the same step as the cutover.

## Minimal CLI surface for the workflow
```powershell
aws rds describe-db-engine-versions --engine mariadb --engine-version 10.6.25 --query "DBEngineVersions[*].ValidUpgradeTarget[*].{EngineVersion:EngineVersion,IsMajorVersionUpgrade:IsMajorVersionUpgrade}" --output table

aws rds create-blue-green-deployment --blue-green-deployment-name <name> --source <db-arn> --target-engine-version 10.11.16 --target-db-parameter-group-name <target-parameter-group>

aws rds describe-blue-green-deployments --blue-green-deployment-identifier <bgd-id>

aws rds switchover-blue-green-deployment --blue-green-deployment-identifier <bgd-id> --switchover-timeout 300
```

## Current recommendation
For the MariaDB 10.6.25 to 10.11.16 SOP, standardize on:
- deterministic prechecks
- AWS blue/green as the cutover mechanism
- GitHub Actions as the orchestrator
- Terraform as the steady-state definition
- CLI plus post-cutover state reconciliation as first-class workflow steps
