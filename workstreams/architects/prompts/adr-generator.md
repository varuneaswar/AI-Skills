---
id: arch-adr-generator
title: Architecture Decision Record (ADR) Generator
type: prompt
version: "1.0.0"
status: active
workstream:
  - architects
  - system-designers
author: ai-skills-maintainer
created: 2024-01-01
updated: 2024-01-01
tags:
  - adr
  - architecture
  - decision-record
  - documentation
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - gemini-1.5-pro
description: |
  Generates a well-structured Architecture Decision Record (ADR) in the MADR format from
  a brief description of the decision context, options considered, and the chosen option.
security_classification: internal
inputs:
  - name: DECISION_CONTEXT
    type: string
    description: The problem or context that required a decision
    required: true
  - name: OPTIONS_CONSIDERED
    type: string
    description: Comma-separated list of options that were evaluated
    required: true
  - name: CHOSEN_OPTION
    type: string
    description: The option that was selected
    required: true
  - name: RATIONALE
    type: string
    description: Brief explanation of why this option was chosen over the others
    required: true
---

## Overview

Use this prompt to quickly generate a formal Architecture Decision Record (ADR) following the [MADR (Markdown Architectural Decision Records)](https://adr.github.io/madr/) format. The generated ADR should be reviewed and refined by the decision stakeholders before committing.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{DECISION_CONTEXT}}` | string | Yes | The problem or context requiring a decision |
| `{{OPTIONS_CONSIDERED}}` | string | Yes | Options evaluated (comma-separated) |
| `{{CHOSEN_OPTION}}` | string | Yes | The selected option |
| `{{RATIONALE}}` | string | Yes | Why this option was chosen |

## Prompt

```
You are a principal architect helping a team document architecture decisions.

Using the MADR (Markdown Architectural Decision Records) format, generate a complete ADR
from the following information:

Decision Context: {{DECISION_CONTEXT}}
Options Considered: {{OPTIONS_CONSIDERED}}
Chosen Option: {{CHOSEN_OPTION}}
Rationale: {{RATIONALE}}

Output a complete ADR in Markdown using this exact structure:

# ADR-NNNN: [Short title derived from the context]

## Status
Proposed

## Context and Problem Statement
[Expand the decision context into 2-3 sentences explaining the situation, constraints,
and why a decision was needed.]

## Decision Drivers
[Bulleted list of the key quality attributes or constraints that shaped the decision
(e.g., performance, cost, team expertise, time-to-market).]

## Considered Options
[For each option, provide: option name, a one-sentence description, key pros, and key cons.]

## Decision Outcome
**Chosen option:** [chosen option name]

[One paragraph explaining why this option was selected over the others, referencing
the decision drivers.]

### Positive Consequences
[Bulleted list of expected positive outcomes.]

### Negative Consequences / Trade-offs
[Bulleted list of known trade-offs or risks introduced by this decision.]

## Links
- [Add relevant links, RFC numbers, or related ADR IDs — leave blank if none]

Do not include placeholder text like "[fill in later]". Generate realistic content
based on the inputs provided.
```

## Usage

1. Copy the prompt above.
2. Replace the four placeholder values.
3. Paste into your LLM interface.
4. Review the generated ADR with your team and refine as needed.
5. Save the ADR to your repository as `docs/adr/ADR-NNNN-short-title.md`.

## Examples

### Example 1

**Input:**
- `DECISION_CONTEXT`: `We need to choose a message broker for our new event-driven microservices platform`
- `OPTIONS_CONSIDERED`: `Apache Kafka, RabbitMQ, AWS SQS/SNS`
- `CHOSEN_OPTION`: `Apache Kafka`
- `RATIONALE`: `High throughput requirements, need for log compaction, team already has Kafka expertise`

**Output (abbreviated):**
```markdown
# ADR-0001: Use Apache Kafka as the Message Broker for Event-Driven Microservices

## Status
Proposed

## Context and Problem Statement
Our new microservices platform requires a reliable, high-throughput message broker to
enable asynchronous communication between services. The broker must support event
replay, handle peak loads of 50k messages/second, and integrate with our existing
Kubernetes-based infrastructure.

## Decision Drivers
- High throughput: must sustain 50k events/second at peak
- Event replay capability for audit and recovery scenarios
- Team expertise to reduce onboarding time
- Long-term operational cost

## Considered Options
**Apache Kafka** — Distributed log-based message broker.
  Pros: Very high throughput, log compaction, strong ecosystem.
  Cons: Higher operational complexity than managed alternatives.

**RabbitMQ** — Traditional message queue with AMQP support.
  Pros: Simple setup, flexible routing.
  Cons: Lower throughput ceiling, no native log compaction.

**AWS SQS/SNS** — Fully managed AWS messaging service.
  Pros: Zero ops overhead, pay-per-use.
  Cons: Vendor lock-in, limited replay capabilities without additional setup.
...
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | Tested 2024-01-01. Produces complete, realistic ADRs. |
| claude-3-5-sonnet | ✅ | Tested 2024-01-01. Excellent prose quality. |

## Changelog

### 1.0.0 — 2024-01-01
- Initial version.
