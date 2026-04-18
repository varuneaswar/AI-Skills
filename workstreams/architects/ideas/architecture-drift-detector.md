---
id: arch-drift-detector
title: Architecture Drift Detector
type: idea
version: "1.0.0"
status: draft
workstream:
  - architects
author: ai-skills-maintainer
created: 2024-01-01
updated: 2024-01-01
tags:
  - architecture
  - governance
  - drift
  - compliance
description: |
  An agent that compares the actual deployed system topology (inferred from infrastructure
  code, service meshes, or dependency graphs) against the documented architecture and flags
  deviations — keeping architecture documentation honest and triggering ADR reviews.
security_classification: internal
---

## Overview

Architecture documents describe intent; running systems reflect reality. Over time these diverge. This idea proposes an agent that continuously compares the two and flags drift for architect review.

## Proposed Approach

**Asset type:** `agent`  
**Potential delivery mode:** `api-endpoint` (scheduled weekly or triggered on IaC changes)

The agent would:
1. Parse infrastructure-as-code (Terraform, Helm charts, Kubernetes manifests) to extract the current topology.
2. Compare against the documented architecture (system design docs, ADRs).
3. Use `arch-trade-off-analysis` for each detected deviation to assess impact.
4. Raise a GitHub Issue with a summary of drifts, grouped by severity.
5. Suggest which ADRs need to be updated or superseded.

## Expected Benefits

- Architecture documentation stays aligned with reality.
- New drifts are caught early, before they become entrenched.
- Provides evidence for architecture review boards.

## Acceptance Criteria

- [ ] Agent detects when a new service is added that is not in the architecture diagram.
- [ ] Agent detects deprecated components still running in production.
- [ ] Agent groups findings by: new-addition, missing-component, changed-dependency.
- [ ] Agent output links to the relevant ADR(s).

## Open Questions

- How to represent the "expected" architecture in a machine-readable format?
- Should the agent integrate with draw.io, Structurizr, or C4 tooling?

## References

- Existing skill: `arch-trade-off-analysis` — for assessing impact of deviations
- Existing prompt: `arch-adr-generator` — for proposing new ADRs to document accepted drift
