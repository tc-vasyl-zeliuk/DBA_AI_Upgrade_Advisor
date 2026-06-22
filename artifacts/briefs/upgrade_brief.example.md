# Upgrade Brief: rds-prod-orders-pg13

## Summary
This production RDS PostgreSQL 13 asset is in Extended Support as of 2026-03-01. It should be prioritized for upgrade planning because the current state likely creates avoidable support cost and leaves a customer-facing service on a retired major version.

## Recommended direction
- Target a supported PostgreSQL major version.
- Start with PostgreSQL 14 unless application certification requires a different approved target.

## Required checks
- extension compatibility
- parameter group differences
- replica topology impact
- application driver compatibility
- rollback and rehearsal readiness

## Evidence
- AWS RDS PostgreSQL release calendar: standard support for PostgreSQL 13 ended on 2026-02-28, and Extended Support started on 2026-03-01.
