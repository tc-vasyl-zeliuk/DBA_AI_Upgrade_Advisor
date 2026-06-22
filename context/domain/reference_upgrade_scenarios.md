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
