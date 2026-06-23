# Reference Upgrade Scenarios

Purpose:
- Capture concrete scenarios that should shape prototype scoring, assessment output, and workflow design.
- Keep scenario facts, inferences, and unresolved validation points explicit.

## Scenario 1: RDS for PostgreSQL 17.8 to 18.3

### Facts
- The prototype instance identifier provided by the user is `test-ops-windmill-us-east-2-postgres-01`.
- The user identified this instance as belonging to the `test` tier.
- RDS for PostgreSQL 17 standard support ends on 2030-02-28 and Extended Support starts on 2030-03-01.
- RDS for PostgreSQL 18 standard support ends on 2031-02-28.
- RDS for PostgreSQL 17.8 was released by Amazon RDS on 2026-02-12.
- RDS for PostgreSQL 18.3 was released by Amazon RDS on 2026-02-27.
- For blue/green major version upgrades, RDS for PostgreSQL uses logical replication when the source version is 16.1 and higher.
- The validation command run by the user on 2026-04-27 returned `17.9`, `18.2`, and `18.3` as valid upgrade targets for source version `17.8`.
- In that validation output, `18.2` and `18.3` were marked as `IsMajorVersionUpgrade=True` and `AutoUpgrade=False`.

### Inferences
- Upgrading from 17.8 to 18.3 is not a support-deadline remediation case. It is a proactive currency and feature-adoption case.
- The business case for this upgrade should emphasize runway, platform standardization, and earlier certification against PostgreSQL 18, not Extended Support avoidance.
- If blue/green is used for this upgrade, logical replication limitations become part of the risk model and the SOP.

### Validated conclusion
- Direct major upgrade from `17.8` to `18.3` is supported in the validated execution context.
- `18.2` is also a direct major upgrade target, but `18.3` is the better default target because it is the newer supported minor.

### Remaining boundary
- This cannot be determined from the information provided alone: whether the actual AWS Region should be recorded authoritatively as `us-east-2`, or whether that is only embedded in the instance naming convention.
- This matters for auditability, but it does not change the prototype decision that `17.8 -> 18.3` is a directly supported path in the validated environment.

### Fastest validation
```powershell
aws rds describe-db-engine-versions `
  --engine postgres `
  --engine-version 17.8 `
  --query "DBEngineVersions[*].ValidUpgradeTarget[*].{EngineVersion:EngineVersion,IsMajorVersionUpgrade:IsMajorVersionUpgrade,AutoUpgrade:AutoUpgrade}" `
  --output table
```

### Prototype assessment stance
- urgency: low from a lifecycle perspective
- urgency: medium if PostgreSQL 18 alignment is a platform standardization objective
- recommended action: treat 18.3 as a confirmed direct target in the validated environment
- key checks:
  - custom parameter group compatibility
  - instance class support for PostgreSQL 18
  - extension compatibility and required `ALTER EXTENSION ... UPDATE`
  - logical replication slot review
  - application driver and ORM certification
  - blue/green logical replication constraints if used

### Blue/green-specific risk points for this scenario
- DDL changes on blue are not replicated to green during logical replication.
- DCL changes on blue are not replicated to green.
- Each database requires a logical replication slot, which can create lag pressure on under-sized instances.
- Sequence synchronization can lengthen switchover time in sequence-heavy databases.

### Validation evidence
```text
Source version: 17.8
Direct targets returned: 17.9, 18.2, 18.3
Direct major upgrade targets: 18.2, 18.3
Preferred target: 18.3
AutoUpgrade: False for all returned targets
Validation date: 2026-04-27
Validation context: AWS_PROFILE=test
Prototype instance: test-ops-windmill-us-east-2-postgres-01
Tier: test
```

## Scenario 2: RDS for MariaDB 10.6.25 to 10.11.16 with primary plus one read replica

### Facts
- MariaDB 10.6.25 is a supported RDS for MariaDB minor version, with RDS end of standard support listed as July 2026.
- MariaDB 10.11.16 is a supported RDS for MariaDB minor version, with RDS end of standard support listed as February 2027.
- Amazon RDS supports in-place major upgrades from any MariaDB version to MariaDB 10.11.
- Blue/green deployments are supported for RDS for MariaDB 10.2 and higher 10 versions.
- When a blue environment has read replicas, RDS copies them as replicas of the green primary.

### Inferences
- This is a credible prototype case for deadline-driven remediation because 10.6.25 has materially shorter support runway than 10.11.16.
- Blue/green should be the default preferred path for this topology because AWS explicitly positions it as the best downtime-reduction option for MariaDB upgrades.
- The presence of a replica does not block blue/green in AWS, but it does weaken the case for pure Terraform orchestration.

### Required control points
- create a target parameter group for MariaDB 10.11 if custom parameters are in use
- validate application compatibility and SQL mode differences across major versions
- validate replica health and replication lag before cutover
- validate monitoring, backups, tagging, IAM roles, and alert wiring after switchover

### Working product implication
The prototype should model this scenario as both:
- a version policy finding
- a workflow pattern where deterministic assessment leads to hybrid execution steps

## Scenario 3: RDS for MariaDB 10.11.16 to 10.11.18 in-place minor update (primary plus one read replica)

### Facts
- Governing Jira: CLOPS-4564 ([TEST] Minor upgrade of MariaDB RDS from 10.11.16 to 10.11.18).
- Environment: TEST tier, region `us-east-2`.
- Instances in scope: `test-gateway` (primary, `db.t4g.micro`) and `test-gateway-replica` (read replica, `db.t4g.micro`).
- Latest supported MariaDB 10.11 minor is `10.11.18`, as reported by the external `AWS_Maintenance_Automation/rds/get-latest-rds-version.sh` helper (lives in the separate `AWS_Maintenance_Automation` repo and wraps `aws rds describe-db-engine-versions`, which is the authoritative source).
- `engine_version` is not pinned in Terraform for these MariaDB instances (per `terraform-module-db-access-patterns` notes); no IaC PR is required for the change itself.
- RDS for MariaDB 10.11 standard support ends 2027-02-28 (see `prototype_input/lifecycle/lifecycle_policies.example.yaml#rds-mariadb-10-11`).
- For an in-place minor upgrade, the SOP requires applying the change to read replicas before the primary and explicitly verifying each replica's engine version after the upgrade.
- Confluence SOP: `https://tradecentric.atlassian.net/wiki/spaces/OP/pages/3457777667` (AWS RDS Minor Update, v1.3).

### Inferences
- This is a routine currency / patch case, not a deadline-driven remediation. Urgency from a lifecycle standpoint is low; urgency from a vulnerability / patch hygiene standpoint is medium.
- The right execution pattern is `in_place_minor_update` (see `plans/sop_in_place_minor_update_pattern.md`), not `blue_green_hybrid_iac`.
- Replica-first ordering applies. TEST is the leading tier in the TEST -> QUALITY -> PROD promotion chain modeled via Jira `Blocks` links.

### Required control points
- Confirm `AWS_PROFILE=test` and account `226344076232` before any `--apply-immediately` action.
- Confirm target version `10.11.18` is in the `ValidUpgradeTarget` list for current source `10.11.16`.
- Apply the upgrade to `test-gateway-replica` first, wait for `available`, then apply to `test-gateway`.
- Validation: `SELECT VERSION();` returns `10.11.18` on the primary; both instances report `DBInstanceStatus=available`; replica lag returns to zero; application smoke test against the TEST gateway endpoint succeeds.
- Snapshot before the change is optional for TEST (automated backups present); it becomes mandatory for the PROD counterpart.

### Working product implication
The prototype should model this scenario as:
- a routine-patch finding (not deadline-driven)
- an `in_place_minor_update` execution pattern with `replica_first_ordering=true`
- a Jira `[System] Change` per tier, linked by `Blocks` for tier promotion

## Scenario 4: RDS for PostgreSQL 17.9 to 18.4 in-place major upgrade via Terraform (single-instance, TEST)

### Facts
- Governing Jira: CLOPS-4563 ([TEST] Major upgrade of PostgreSQL RDS from 17.9 to 18.4).
- Environment: TEST tier, account `226344076232`, region `us-east-2`.
- Instance in scope: `test-ops-windmill-us-east-2-postgres-01` (`db.t3.micro`, single-AZ, no replicas, `BackupRetentionPeriod=7`, `DeletionProtection=true`).
- Current engine version: `17.9`. Current parameter group: `test-ops-windmill-us-east-2-postgres-01-postgres17`, `ParameterApplyStatus=in-sync`.
- Target engine version `18.4` is a valid `IsMajorVersionUpgrade=true` target from `17.9` per `aws rds describe-db-engine-versions`.
- `engine_version` is pinned in Terraform at `Terraform_Windmill/environments/test/main.tf`, calling `module "database"` with `source = "github.com/PunchOut2Go/Terraform_Modules.git//databases/postgres?ref=v1.3.8"`.
- The shared `databases/postgres` module derives `local.major_version = split(".", var.engine_version)[0]` and creates `aws_db_parameter_group.postgres_pg` with `family = "postgres${local.major_version}"`, using `create_before_destroy = true`.
- The module exposes `allow_major_version_upgrade` (default `false`); the current TEST main.tf does not set it.
- No Confluence SOP currently exists for PostgreSQL major upgrades; the governing pattern document is `plans/sop_in_place_major_upgrade_via_terraform_pattern.md`.

### Inferences
- This is a routine currency upgrade, not a deadline-driven remediation; 17.9 is not at end of support at the time of the change.
- The right execution pattern is `in_place_major_upgrade_via_terraform` (see `plans/sop_in_place_major_upgrade_via_terraform_pattern.md`), not `blue_green_hybrid_iac` (no replica / Multi-AZ topology to preserve) and not `in_place_minor_update` (engine version is IaC-pinned and the major bump requires the `allow_major_version_upgrade` toggle).
- The parameter group family flip (`postgres17` to `postgres18`) is the module's responsibility within the same apply; no separate parameter group resource needs to be added to the environment main.tf.
- The same PR that changes `engine_version` to `18.4` must also set `allow_major_version_upgrade = true`, otherwise the AWS apply will fail. Resetting it back to `false` post-upgrade is a hardening choice, not a correctness requirement.

### Required control points
- Confirm `AWS_PROFILE=test` and account `226344076232` before any state-changing action.
- Confirm `18.4` is in the `ValidUpgradeTarget` list for current source `17.9` with `IsMajorVersionUpgrade=true`.
- Take a manual pre-upgrade snapshot named `clops-4563-pre-upgrade-<UTC-timestamp>` and wait for `available` before opening the apply window.
- The Terraform PR must include exactly two functional edits in `environments/test/main.tf`: `engine_version = "18.4"` and `allow_major_version_upgrade = true`. No parameter group changes.
- The `terraform plan` must show: new parameter group create, instance update (engine_version, parameter group attachment, `allow_major_version_upgrade`), old parameter group destroy. Anything additional must be reconciled before apply.
- Validation: `DBInstanceStatus=available`, `EngineVersion=18.4`, parameter group `...-postgres18` attached with `ParameterApplyStatus=in-sync`, `SELECT version();` reports PostgreSQL 18.4, post-apply `terraform plan` is a no-op.

### Working product implication
The prototype should model this scenario as:
- a routine-currency finding (not deadline-driven)
- an `in_place_major_upgrade_via_terraform` execution pattern with `iac_pinned_engine_version=true`, `allow_major_version_upgrade_toggle_required=true`, `parameter_group_family_derived_by_module=true`, `replica_first_ordering=false` (no replicas)
- a Jira `[System] Change` per tier, linked by `Blocks` for tier promotion (TEST -> QUALITY -> PROD)

### Validation evidence
```text
Source version: 17.9
Target version: 18.4 (valid IsMajorVersionUpgrade=true target confirmed via describe-db-engine-versions)
Source instance class: db.t3.micro
Target instance class: db.t4g.micro (x86 to Graviton; AWS-transparent for stock RDS extensions)
Source parameter group: test-ops-windmill-us-east-2-postgres-01-postgres17
Target parameter group: test-ops-windmill-us-east-2-postgres-01-postgres18 (in-sync, attached)
Allow major version upgrade: false -> true
Manual snapshot: skipped (TEST tier; AWS RDS auto-snapshot before major upgrade is the safety net)
Validation date: 2026-06-23
Validation context: AWS_PROFILE=test, account 226344076232, region us-east-2
Governing PR: PunchOut2Go/Terraform_Windmill#59 (merged), deploy tag test-v1.0.5
Deferred-modification race observed: yes (first apply failed at old parameter group destroy step because apply_immediately=false caused the AWS-side attachment swap to lag the Terraform-side instance update; recovered on retry after AWS completed the in-place modification)
Post-upgrade state: DBInstanceStatus=available, EngineVersion=18.4, DBInstanceClass=db.t4g.micro, PendingModifiedValues={}, old parameter group destroyed
```

### Lessons learned (folded into SOP)
- The `databases/postgres` module does not currently set `apply_immediately = true` on `aws_db_instance`. Combined with `create_before_destroy = true` on `aws_db_parameter_group`, this creates a deterministic race between Terraform-perceived instance update completion and AWS-side parameter group attachment swap. The first `terraform apply` of a major upgrade should be expected to fail at the old parameter group destroy step until either AWS finishes the deferred modification or the modification is flushed via `modify-db-instance --apply-immediately`.
- The race is now documented in `plans/sop_in_place_major_upgrade_via_terraform_pattern.md` under `Known race conditions`, and Step 4 (Apply stage) cross-references it. A future module enhancement to expose `apply_immediately` as a variable would let the upgrade PR pre-empt the race without an out-of-band CLI step.
