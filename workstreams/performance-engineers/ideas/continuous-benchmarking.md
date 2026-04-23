---
id: perf-continuous-benchmarking
title: Continuous Performance Benchmarking Agent
type: idea
version: "1.0.0"
status: draft
workstream:
  - performance-engineers
author: ai-skills-maintainer
created: 2024-01-01
updated: 2024-01-01
tags:
  - benchmarking
  - regression
  - ci-cd
  - agent
description: |
  An agent that runs lightweight performance benchmarks on every PR, compares results
  against the historical baseline, and blocks merge if a configurable regression threshold
  is exceeded — making performance a first-class CI/CD gate.
security_classification: internal
---

## Overview

Performance regressions are often introduced silently by individual PRs. This agent embeds performance testing into the CI pipeline so regressions are caught at the PR stage, not in production.

## Proposed Approach

**Asset type:** `agent`  
**Potential delivery mode:** `api-endpoint` (CI/CD webhook) + `bitbucket-pipelines`

The agent would:
1. Trigger a lightweight load test on a preview environment when a PR is opened.
2. Use `perf-bottleneck-analysis` to compare new results against the stored baseline.
3. Post a structured comment on the PR with a performance delta table.
4. Block merge if P99 latency increases by more than a configurable threshold (default: 15%).
5. Update the baseline after merge if the change is intentional and approved.

## Expected Benefits

- Performance regressions are caught in hours, not weeks.
- Engineers get immediate feedback on the performance impact of their changes.
- Reduces the volume of performance-related production incidents.

## Acceptance Criteria

- [ ] Agent posts a performance summary comment on every PR within 10 minutes.
- [ ] Merge block engages correctly when the regression threshold is exceeded.
- [ ] Baseline update mechanism requires explicit engineer confirmation.
- [ ] Agent integrates with k6 or Locust as the test runner.

## Open Questions

- How to handle PRs that intentionally change performance characteristics (e.g., trading latency for throughput)?
- How to provision and tear down preview environments cost-effectively?

## References

- Existing skill: `perf-bottleneck-analysis` — for interpreting results
