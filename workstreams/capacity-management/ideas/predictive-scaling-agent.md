---
id: cap-predictive-scaling-agent
title: Predictive Auto-Scaling Recommendation Agent
type: idea
version: "1.0.0"
status: draft
workstream:
  - capacity-management
author: ai-skills-maintainer
created: 2024-01-01
updated: 2024-01-01
tags:
  - auto-scaling
  - forecasting
  - agent
  - cost-optimisation
description: |
  An agent that analyses historical traffic patterns and business event calendars to
  pre-emptively recommend or apply scaling actions — preventing capacity-driven incidents
  before reactive auto-scaling can respond.
security_classification: internal
---

## Overview

Reactive auto-scaling responds after load increases; this agent acts before it arrives. By correlating historical usage patterns with upcoming business events (product launches, marketing campaigns, end-of-month processing), it generates scaling recommendations in advance.

## Proposed Approach

**Asset type:** `agent`  
**Potential delivery mode:** `api-endpoint` (scheduled daily + event-triggered)

The agent would:
1. Ingest historical utilization data and use `cap-utilization-analysis` to identify seasonal patterns.
2. Query a business events calendar for upcoming high-load events.
3. Generate a scaling plan with time-boxed recommendations (e.g., "scale out by 3 nodes from Monday 08:00 to 18:00").
4. Optionally integrate with cloud provider APIs to apply the plan automatically (with human approval gate).
5. Report post-event on whether the plan was sufficient.

## Expected Benefits

- Prevents capacity-driven SEV1/SEV2 incidents during predictable peak events.
- Reduces over-provisioning during off-peak periods.
- Provides evidence for capacity planning budgets.

## Acceptance Criteria

- [ ] Agent correctly predicts peak windows with >85% accuracy on historical data.
- [ ] Recommendations include confidence intervals.
- [ ] Agent integrates with at least one cloud provider API (AWS, Azure, or GCP) for plan execution.
- [ ] Human approval gate is mandatory before any scaling action is executed.

## Open Questions

- How to model irregular events (e.g., viral marketing campaigns with short notice)?
- Should cost impact be included in the recommendation?

## References

- Existing skill: `cap-utilization-analysis` — core analysis capability
