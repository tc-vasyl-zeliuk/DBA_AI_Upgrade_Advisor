# Upgrade Brief: test-ops-windmill-us-east-2-postgres-01

## Summary
This test-tier RDS PostgreSQL instance is a proactive major upgrade candidate from 17.8 to 18.3. It is not driven by imminent support expiration. The value case is rehearsal of the major upgrade path, earlier PostgreSQL 18 compatibility validation, and platform standardization.

## Position
- Treat 18.3 as the preferred direct target in the validated environment.
- The user validation run on 2026-04-27 confirmed direct upgrade targets from 17.8 to 18.2 and 18.3.

## Why it matters
- PostgreSQL 17 standard support runs through 2030-02-28.
- PostgreSQL 18 standard support runs through 2031-02-28.
- The incremental runway is real, but the bigger benefit is platform standardization and earlier compatibility testing.

## Key checks
- create and validate a PostgreSQL 18 parameter group
- validate extension upgrade path
- validate driver and application behavior
- if blue/green is used, freeze DDL on blue during sync and account for logical replication limitations
- record the authoritative AWS Region explicitly in the final SOP inputs instead of inferring it only from the instance name
