---
id: sysdesign-design-pattern-recommender
title: Design Pattern Recommender
type: idea
version: "1.0.0"
status: draft
workstream:
  - system-designers
  - architects
author: ai-skills-maintainer
created: 2024-01-01
updated: 2024-01-01
tags:
  - design-patterns
  - system-design
  - recommendations
  - agent
description: |
  An agent that analyses a system design description or problem statement and recommends
  the most appropriate design patterns (GoF, enterprise integration, cloud patterns) with
  pros, cons, and implementation guidance tailored to the team's technology stack.
security_classification: internal
---

## Overview

System designers often apply patterns they know rather than evaluating the best fit for the problem. This agent systematically evaluates applicability and provides tailored recommendations.

## Proposed Approach

**Asset type:** `agent`  
**Potential delivery mode:** `api-endpoint` (portal chat or IDE integration)

The agent would:
1. Parse the system design description to identify the core problem type (e.g., event sourcing need, cross-service data consistency, cache invalidation).
2. Match the problem type against a curated pattern library.
3. Rank patterns by fit score based on the stated technology stack and constraints.
4. Output a recommendation report with rationale and a Mermaid diagram for each recommended pattern.
5. Use `sysdesign-sequence-diagram-generator` to visualise the recommended pattern.

## Expected Benefits

- Reduces reliance on individual designers' pattern knowledge.
- Surfaces less-commonly-used but well-suited patterns.
- Provides a starting point for documentation and ADRs.

## Acceptance Criteria

- [ ] Agent considers at least 50 patterns from GoF, EIP, and cloud pattern catalogues.
- [ ] Recommendations include a fit score and explicit reasoning.
- [ ] Each recommendation includes a Mermaid diagram.
- [ ] Agent filters patterns that conflict with stated constraints (e.g., "no external messaging broker").

## Open Questions

- How to keep the pattern library current as new cloud-native patterns emerge?
- Should the agent integrate with existing design tools (draw.io, Lucidchart)?

## References

- Existing skill: `sysdesign-sequence-diagram-generator` — for visualising recommended patterns
- Existing skill: `arch-trade-off-analysis` — for comparing pattern options
