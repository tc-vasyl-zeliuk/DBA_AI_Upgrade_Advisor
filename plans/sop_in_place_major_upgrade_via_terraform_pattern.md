# SOP In-Place Major Upgrade via Terraform Pattern

## Decision
Use a Terraform-driven, in-place major version upgrade pattern for RDS instances whose `engine_version` is pinned in IaC and whose topology does not warrant blue/green:
- The reviewed Terraform pull request is the trigger and the source of truth for the change.
- AWS RDS performs the major-version upgrade as a side effect of `terraform apply` once `engine_version` and `allow_major_version_upgrade` are changed in the same plan.
- The shared internal database module derives the parameter group family from the new major version, so the parameter group flip is part of the same apply, not a separate operator step.
- No CLI `modify-db-instance` invocation is performed by hand for the engine bump itself. Manual CLI is reserved for the pre-upgrade snapshot and read-only validation.
- Source of truth: this document; Confluence has no current page for major upgrades, so until one is added the governing Jira `[System] Change` ticket is the operational record.

## Why this is the right pattern

### Facts
- AWS RDS supports in-place major upgrades when the proposed target appears in `aws rds describe-db-engine-versions --engine <engine> --engine-version <current> --query "DBEngineVersions[*].ValidUpgradeTarget[?IsMajorVersionUpgrade==\`true\`]"`.
- AWS RDS rejects a major upgrade unless `AllowMajorVersionUpgrade=true` is set on the modification request.
- The internal `databases/postgres` module (and its peers) derives `local.major_version` from `var.engine_version` and constructs the parameter group as `family = "postgres${local.major_version}"`, named `"<prefix>-postgres-<id>-postgres<major>"`. The parameter group resource uses `create_before_destroy = true`, so the new family parameter group is created before the old one is detached.
- Within the same apply, RDS attaches the new parameter group and changes the engine version atomically from the operator point of view. AWS schedules the version change immediately when `apply_immediately = true` or via the next maintenance window otherwise.
- A major upgrade may invalidate query plans, extensions, and stored procedures. AWS does not roll back automatically on validation failure.
- Major upgrades disable read replicas during the operation. Replicas must be re-created from the upgraded primary after cutover when present.
- Single-instance assets (no replica, no Multi-AZ standby) are the safest first targets for this pattern.

### Inference
- Because `engine_version` is pinned in Terraform, any out-of-band CLI engine change would create immediate drift. The PR is therefore not optional documentation; it is the change.
- Because the parameter group family is derived inside the module, no separate parameter group bump step is required and no manual parameter group needs to be passed to the module.
- Because `allow_major_version_upgrade` defaults to `false` in the module, the same PR that changes `engine_version` must also set this variable to `true` for the apply to succeed. Once the upgrade is complete, the variable may be left at `true` for the next major or reset to `false` as a hardening step.
- Because RDS does not auto-rollback a major upgrade, the manual pre-upgrade snapshot is the only reliable backout. It must be taken before `terraform apply`, not after the PR is merged but before it is applied.
- Because blue/green is not used here, validation must verify the engine version, the parameter group attachment status, and a minimal application smoke test in the same window.

## Recommended ownership split

### Terraform should own
- `engine_version` value
- `allow_major_version_upgrade` value (set to `true` for the apply that performs the upgrade)
- parameter group family (derived inside the module from `engine_version`)
- DB instance steady state (instance class, storage, networking, option groups, alarms, tags)
- subnet groups, security groups
- IAM scaffolding

### CLI / operator workflow should own
- pre-upgrade manual snapshot creation (`aws rds create-db-snapshot`)
- precheck of valid upgrade targets (`aws rds describe-db-engine-versions`)
- precheck of current instance state (`aws rds describe-db-instances`)
- status polling until `available` after the apply
- post-apply `terraform plan` no-op confirmation
- Jira and Confluence updates and validation evidence capture

### Explicitly out of scope for this pattern
- blue/green deployment creation and switchover (covered by `sop_automation_pattern.md`)
- pure minor / OS patching on instances whose Terraform does not pin `engine_version` (covered by `sop_in_place_minor_update_pattern.md`)
- logical-replication-based zero-downtime upgrades

## Recommended workflow shape

### 1. Precheck stage
- confirm AWS CLI v2 is installed on the operator machine and the correct `AWS_PROFILE` is exported for the target tier
- confirm the target engine version is a valid major upgrade target from the current version using `describe-db-engine-versions` (see CLI surface below)
- describe the current instance: engine version, parameter group name and `ParameterApplyStatus`, deletion protection, backup retention, Multi-AZ, replica list
- confirm there are no replicas (single-instance pattern); if replicas exist, escalate to the blue/green pattern or to a replica re-creation plan
- locate the Terraform module call: confirm the module derives parameter group family from `engine_version` and exposes `allow_major_version_upgrade`
- confirm the current Terraform main does not set `allow_major_version_upgrade` (or sets it to `false`) so the PR is the explicit toggle

### 2. Snapshot stage
- always create a manual, named snapshot of the primary, regardless of tier and regardless of automated backup retention
- snapshot identifier convention: `<jira-ticket-lower>-pre-upgrade-<UTC-timestamp>` (for example, `clops-4563-pre-upgrade-20260623t125500z`)
- wait for snapshot status `available` before opening the apply window
- record the snapshot identifier as backout evidence in the Jira ticket

### 3. Pull request stage
- create a feature branch in the Terraform repository following the existing naming convention (for example, `CLOPS-XXXX_upgrade_<scope>` or `feat/clops-XXXX-...`)
- in the same diff, change `engine_version` to the new major and set `allow_major_version_upgrade = true`
- do not bump the parameter group manually; the module handles it
- run `terraform fmt` and `terraform validate` locally before pushing
- open the PR against `main`, link the Jira ticket, request review
- merge only after green plan and reviewer approval

### 4. Apply stage
- run `terraform init` and `terraform plan` from the environment directory; confirm the plan shows exactly the engine_version change, the `allow_major_version_upgrade` flip, the new parameter group create (`create_before_destroy`), the instance update with the new parameter group, and the old parameter group destroy
- run `terraform apply`; do not exit the terminal until the apply completes
- expected duration for single-instance major upgrade on small `db.t3` class instances is on the order of 10-20 minutes; larger classes scale accordingly

### 5. Validate stage
- poll instance status with `describe-db-instances` until `DBInstanceStatus = available` and `EngineVersion = <target>`
- confirm the new parameter group is attached and `ParameterApplyStatus = in-sync`
- run `SELECT version();` from the application or a controlled session to confirm the catalog-reported version
- run an application smoke test against the instance endpoint
- run `terraform plan` once more from the environment directory and confirm it is a no-op (no drift, no pending changes)

### 6. Closeout stage
- post the validation evidence as a single, structured "Implementation Highlights" comment on the governing Jira `[System] Change` ticket
- transition the Jira ticket to its resolved state with `resolution = Done`
- if a tier-promotion ticket chain exists, link the next tier ticket via `Blocks`

## Backout
- the only supported backout for a completed major upgrade is restore from the manual snapshot taken in stage 2 to a new instance, with a follow-up cutover plan
- in-place downgrade is not supported by RDS for major versions
- if `terraform apply` fails before AWS starts the engine modification, revert the PR and re-apply the prior state; no AWS-side action needed
- if `terraform apply` fails partway through, do not retry blindly; describe the instance and the parameter group state and decide explicitly between retrying the apply, restoring from snapshot, or escalating

## What not to do
- Do not run `modify-db-instance --engine-version` from the CLI for an instance whose `engine_version` is pinned in Terraform. It creates drift immediately and the next `terraform plan` will try to revert.
- Do not change `engine_version` in Terraform without also setting `allow_major_version_upgrade = true` in the same PR. The apply will fail.
- Do not create a new parameter group manually for the new family. The module owns it.
- Do not skip the manual snapshot, even in TEST, even when automated backups are enabled. Backout depends on it.
- Do not run the major upgrade against an instance with active replicas using this pattern. Either re-create replicas after, or use the blue/green pattern.
- Do not merge the PR and walk away. The apply must happen in the same window with active operator validation.
- Do not declare the change complete until `terraform plan` is a no-op.

## Minimal CLI surface for the workflow

```bash
# Confirm the target major is a valid upgrade target
aws rds describe-db-engine-versions \
  --engine <postgres|mysql|mariadb> \
  --engine-version <current-version> \
  --query "DBEngineVersions[*].ValidUpgradeTarget[?IsMajorVersionUpgrade==\`true\`].{EngineVersion:EngineVersion,AutoUpgrade:AutoUpgrade}" \
  --output table

# Describe current instance state
aws rds describe-db-instances \
  --db-instance-identifier <db-instance-id> \
  --query "DBInstances[].{Id:DBInstanceIdentifier,Engine:Engine,Version:EngineVersion,Class:DBInstanceClass,Status:DBInstanceStatus,MultiAZ:MultiAZ,PG:DBParameterGroups[0].DBParameterGroupName,PGStatus:DBParameterGroups[0].ParameterApplyStatus,DeletionProtection:DeletionProtection}" \
  --output table

# Pre-upgrade manual snapshot
aws rds create-db-snapshot \
  --db-instance-identifier <db-instance-id> \
  --db-snapshot-identifier <jira-ticket-lower>-pre-upgrade-<UTC-timestamp>

# Wait for snapshot available
aws rds wait db-snapshot-available \
  --db-snapshot-identifier <jira-ticket-lower>-pre-upgrade-<UTC-timestamp>

# Terraform-driven apply (run from the environment directory)
terraform init
terraform plan
terraform apply

# Post-apply status confirmation
aws rds describe-db-instances \
  --db-instance-identifier <db-instance-id> \
  --query "DBInstances[0].{Status:DBInstanceStatus,Version:EngineVersion,PG:DBParameterGroups[0].DBParameterGroupName,PGStatus:DBParameterGroups[0].ParameterApplyStatus,Pending:PendingModifiedValues}" \
  --output table

# Drift sanity
terraform plan
```

## Reference
- AWS docs: `https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_UpgradeDBInstance.PostgreSQL.html`
- AWS docs: `https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_UpgradeDBInstance.Upgrading.html`
- Companion pattern (minor and OS patching, no IaC pin): `plans/sop_in_place_minor_update_pattern.md`
- Companion pattern (blue/green for topology-sensitive major upgrades): `plans/sop_automation_pattern.md`

## Current recommendation
For single-instance RDS upgrades where `engine_version` is pinned in Terraform and the shared module derives the parameter group family from the major version, standardize on this Terraform-driven in-place major upgrade pattern. Use the blue/green pattern when replicas, Multi-AZ standbys, or near-zero downtime tolerance constraints apply. Use the minor in-place pattern when `engine_version` is not pinned in IaC.
