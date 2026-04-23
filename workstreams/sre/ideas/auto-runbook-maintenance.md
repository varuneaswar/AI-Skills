---
id: sre-auto-runbook-maintenance
title: Automated Runbook Currency Agent
type: idea
version: "1.0.0"
status: draft
workstream:
  - sre
author: ai-skills-maintainer
created: 2024-01-01
updated: 2024-01-01
tags:
  - runbook
  - automation
  - agent
  - operations
description: |
  An agent that monitors service changes (deployments, infrastructure changes, alert
  threshold updates) and automatically proposes runbook updates to keep operational
  documentation current — eliminating stale runbooks as a source of incident delay.
security_classification: internal
---

## Overview

Runbooks go stale shortly after they are written. Alert thresholds change, commands change, services are renamed. This idea proposes an agent that listens to change events and proactively raises PRs to update affected runbooks.

## Proposed Approach

**Asset type:** `agent`  
**Potential delivery mode:** `api-endpoint` (event-triggered via webhook)

The agent would:
1. Subscribe to deployment pipeline events and infrastructure change notifications.
2. Identify which runbooks reference the changed service or alert.
3. Use the `sre-runbook-generator` skill to propose updated sections.
4. Raise a PR with a diff showing the proposed changes and the triggering event.
5. Tag the runbook owner for review.

## Expected Benefits

- Runbooks stay accurate with minimal manual effort.
- On-call engineers can trust runbook content during high-pressure incidents.
- Reduces post-incident findings of "runbook was outdated".

## Acceptance Criteria

- [ ] Agent detects service rename and updates all runbook references automatically.
- [ ] Agent proposes updated CLI commands when service interface changes.
- [ ] All proposals are PRs — agent never merges without human approval.
- [ ] Agent sends a Slack notification to the runbook owner when a PR is raised.

## Open Questions

- Which change event sources should be integrated first? (Bitbucket Pipelines, PagerDuty, Terraform)
- How to handle runbooks that contain confidential system details?

## References

- Existing skill: `sre-runbook-generator` — core generation capability
- Existing prompt: `sre-incident-triage-prompt` — for contextual alert descriptions
