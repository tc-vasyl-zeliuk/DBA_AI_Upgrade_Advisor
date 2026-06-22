# Implementation Decisions

## Current state

### Fact
- No implementation stack has been selected in the provided information.

### Inference
- The next material decision is not folder naming. It is choosing the first executable slice.

## Viable first slices

### Option 1. Report-first prototype
Pros:
- fastest way to validate policy logic and output usefulness
- low infrastructure overhead
- fits file-based prototype inputs

Cons:
- limited integration realism
- UI and workflow questions remain deferred

### Option 2. API-first service
Pros:
- clean contract boundary
- easier path to later UI and integrations

Cons:
- more up-front stack and deployment decisions
- higher setup cost before useful outputs appear

### Option 3. Dashboard-first application
Pros:
- strong leadership demo value
- makes prioritization visible quickly

Cons:
- highest risk of polishing presentation before policy logic is credible
- can hide weak underlying data and scoring contracts

## Recommendation
Start report-first unless there is a hard requirement for interactive workflow in the first release.
