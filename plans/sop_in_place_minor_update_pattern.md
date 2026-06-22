# SOP In-Place Minor Update Pattern

## Decision
Use a CLI-driven, in-place update pattern for RDS minor version and operating system maintenance:
- AWS CLI (preferred) or AWS Console handles the action.
- Terraform is intentionally not in the critical path because affected instances do not pin `engine_version` in IaC for this class of change.
- Pending-maintenance updates and on-demand minor-version upgrades share the same procedural envelope (pre-check, apply, validate).
- Source of truth: Confluence page **AWS RDS Minor Update** (id `3457777667`).

## Why this is the right pattern

### Facts
- AWS RDS supports applying pending maintenance actions (`system-update`, `db-upgrade`) on a single DB instance through `aws rds apply-pending-maintenance-action`.
- AWS RDS supports on-demand minor engine version upgrades through `aws rds modify-db-instance --engine-version <target> --apply-immediately`.
- When a primary has read replicas, AWS guidance is to apply the maintenance action to the replica first, then the primary. Replica engine versions must be confirmed against the target after the upgrade, not assumed.
- Multi-AZ instances with cross-region replication may incur additional cost during a minor update; a reboot with forced failover is a documented mitigation when standby placement matters.
- PROD-tier instances require an explicit manual snapshot as a named restore point, independent of automated backup retention.
- The MariaDB instances currently in scope (for example `test-gateway`, `test-gateway-replica`) do not pin `engine_version` in Terraform.

### Inference
- For minor version and OS patching, blue/green is operational overkill. The blue/green pattern is reserved for major-version upgrades or topologies where downtime tolerance is near zero.
- Because `engine_version` is not pinned in Terraform for these assets, no Terraform drift remediation is required after the change. This keeps the change a pure operational event, not an IaC event.
- Replica-first ordering is a property of the change, not the orchestrator. Any automation onboarded later must preserve that ordering.

## Recommended ownership split

### Terraform should own
- DB instance steady state (instance class, storage, networking, parameter and option groups)
- subnet groups, security groups, alarms, tags
- IAM scaffolding

### CLI / operator workflow should own
- inventory of pending maintenance actions (`rds-pending-maintenance.sh`)
- engine version discovery (`get-latest-rds-version.sh`, `describe-db-instances`)
- snapshot creation for PROD when backup retention is zero
- `apply-pending-maintenance-action` invocation
- `modify-db-instance --engine-version <target> --apply-immediately` invocation
- status polling until `available`
- Jira and Confluence updates and validation evidence capture

### Explicitly out of scope for this pattern
- blue/green deployment creation, validation, switchover (covered by `sop_automation_pattern.md`)
- post-cutover Terraform state reconciliation
- replica re-creation as a normal step (it is only a backout action)

## Recommended workflow shape

### 1. Precheck stage
- confirm AWS CLI v2 is installed on the operator machine
- confirm the correct `AWS_PROFILE` for the target tier (`test`, `quality`, `prod`, `cicd`)
- list pending maintenance actions for the tier: `AWS_Maintenance_Automation/inventory/rds-pending-maintenance.sh <tier>` (script lives in the separate `AWS_Maintenance_Automation` repository)
- list current engine versions and latest supported minors: `AWS_Maintenance_Automation/inventory/rds-inventory.sh <tier>` and `AWS_Maintenance_Automation/rds/get-latest-rds-version.sh <tier>` (same external repo)
- confirm the topology (single primary, primary plus replicas, Multi-AZ, cross-region replication)
- decide whether the change is `system-update` (pending), `db-upgrade` (pending), or an on-demand minor upgrade

### 2. Snapshot stage
- skip if automated backups are enabled and the tier is non-prod
- in PROD: always create a manual, named snapshot of the primary
- record the snapshot identifier as backout evidence

### 3. Apply stage
- if replicas exist: apply the maintenance action to each replica first, wait for `available`, then apply to the primary
- for a pending maintenance action: `aws rds apply-pending-maintenance-action --resource-identifier <arn> --apply-action <system-update|db-upgrade> --opt-in-type immediate`
- for an on-demand minor upgrade: `aws rds modify-db-instance --db-instance-identifier <id> --engine-version <target-minor> --apply-immediately`

### 4. Validate stage
- poll instance status: `aws rds describe-db-instances --db-instance-identifier <id> --query "DBInstances[0].{Status:DBInstanceStatus,PendingModifiedValues:PendingModifiedValues}" --output table`
- confirm `DBInstanceStatus = available` on every affected instance
- confirm engine version matches the target on the primary (and replicas, for on-demand minor upgrades)
- run an application smoke test against the instance endpoint
- confirm replica lag returns to zero where applicable

### 5. Closeout stage
- post validation evidence as a comment on the governing Jira `[System] Change` ticket
- transition the Jira ticket to its resolved state
- update Confluence run log if one is maintained for the SOP

## Backout
- if validation fails after a minor upgrade, restore the primary from the snapshot taken in stage 2 to a new instance and cut over
- re-create the replica from the restored primary only if its state is unrecoverable
- pending maintenance actions cannot be rolled back in place; backout for those is the same snapshot-restore path

## What not to do
- Do not apply the change to the primary before the replicas.
- Do not skip the manual snapshot in PROD even when automated backups are enabled.
- Do not use blue/green for a pure minor or OS patch on instances whose Terraform does not pin `engine_version`.
- Do not introduce `engine_version` pinning in Terraform as a side effect of this change without an explicit IaC ticket.
- Do not run `--apply-immediately` without confirming the change is in an approved tier window.

## Minimal CLI surface for the workflow

The inventory helpers below live in the separate `AWS_Maintenance_Automation` repository, not in this repo. Paths are shown relative to that repo's root.

```bash
# Discover pending maintenance per tier
AWS_Maintenance_Automation/inventory/rds-pending-maintenance.sh <tier>

# Discover current and latest minor per engine major
AWS_Maintenance_Automation/inventory/rds-inventory.sh <tier>
AWS_Maintenance_Automation/rds/get-latest-rds-version.sh <tier>

# Apply a pending maintenance action immediately
aws rds apply-pending-maintenance-action \
  --resource-identifier <db-instance-arn> \
  --apply-action <system-update|db-upgrade> \
  --opt-in-type immediate

# On-demand minor version upgrade
aws rds modify-db-instance \
  --db-instance-identifier <db-instance-id> \
  --engine-version <target-minor-version> \
  --apply-immediately

# Status check
aws rds describe-db-instances \
  --db-instance-identifier <db-instance-id> \
  --query "DBInstances[0].{Status:DBInstanceStatus,PendingModifiedValues:PendingModifiedValues}" \
  --output table
```

## Reference
- Confluence: `https://tradecentric.atlassian.net/wiki/spaces/OP/pages/3457777667` (AWS RDS Minor Update, SOP v1.3)
- AWS docs: `https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_UpgradeDBInstance.Maintenance.html`
- Inventory helpers: `AWS_Maintenance_Automation/inventory/rds-inventory.sh`, `AWS_Maintenance_Automation/inventory/rds-pending-maintenance.sh`, `AWS_Maintenance_Automation/rds/get-latest-rds-version.sh`
- Companion pattern (major / topology-sensitive upgrades): `plans/sop_automation_pattern.md`

## Current recommendation
For minor version upgrades and pending system / db updates on RDS instances whose Terraform does not pin `engine_version`, standardize on this in-place CLI pattern. Tier promotion (TEST -> QUALITY -> PROD) is modeled as separate Jira `[System] Change` tickets linked by `Blocks`, one per `(tier, engine, version pair)`.
