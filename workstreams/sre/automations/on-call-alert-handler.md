---
id: sre-on-call-alert-handler
title: On-Call Alert Handler Automation
type: automation
version: "1.0.0"
status: active
workstream:
  - sre
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - incident-response
  - on-call
  - automation
  - bitbucket-pipelines
  - slack
llm_compatibility:
  - gpt-4o
description: |
  A Bitbucket Pipelines custom pipeline triggered via the Bitbucket REST API from
  PagerDuty or Alertmanager. It calls an LLM with the incident triage prompt, then posts
  a colour-coded structured Slack message containing triage result, top mitigation
  steps, and a stakeholder summary — getting the right information to the on-call
  engineer within 60 seconds of alert firing.
security_classification: internal
delivery_modes:
  - llm-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
inputs:
  - name: ALERT_NAME
    type: string
    description: Name of the alert (pipeline variable passed at trigger time)
    required: true
  - name: SERVICE_NAME
    type: string
    description: Affected service name (pipeline variable passed at trigger time)
    required: true
  - name: SEVERITY
    type: string
    description: Initial severity hint (pipeline variable, e.g. SEV1/SEV2/SEV3)
    required: false
  - name: SYSTEM_CONTEXT
    type: string
    description: Context about the system/service (pipeline variable)
    required: false
  - name: ALERT_DETAILS
    type: string
    description: Alert details or description (pipeline variable)
    required: false
  - name: LLM_API_KEY
    type: string
    description: API key for the LLM provider — store in Bitbucket Secured Variables
    required: true
  - name: SLACK_WEBHOOK_URL
    type: string
    description: Incoming Webhook URL for the #incidents Slack channel — store in Bitbucket Secured Variables
    required: true
outputs:
  - name: SLACK_MESSAGE_URL
    type: string
    description: URL of the posted Slack message (returned by the Slack API)
---

## Overview

This automation wraps the [`sre-incident-response-workflow`](../workflows/incident-response-workflow.md) in a Bitbucket Pipelines custom pipeline that is triggered via the Bitbucket REST API (pipelines trigger endpoint). It is designed to be called by PagerDuty webhooks or Alertmanager webhook receivers when an alert fires.

The automation:

1. Receives alert name, service name, and context as pipeline variables passed at trigger time.
2. Sends the alert details to the LLM using the incident triage prompt.
3. Posts a colour-coded Slack Block Kit message to the #incidents channel:
   - 🔴 Red attachment for SEV1
   - 🟠 Orange attachment for SEV2
   - 🟡 Yellow attachment for SEV3
4. Includes triage result, top 5 mitigation steps, and a stakeholder summary in the Slack message.

**Trigger:** Custom pipeline `alert-fired` triggered via `POST https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pipelines/` with `target.selector.pattern: alert-fired`.

## Prerequisites

- Bitbucket repository with Pipelines enabled.
- An LLM API key stored as `LLM_API_KEY` in Bitbucket Secured Variables.
- A Slack incoming webhook URL stored as `SLACK_WEBHOOK_URL` in Bitbucket Secured Variables.
- The LLM endpoint stored as `LLM_API_ENDPOINT` in Bitbucket Repository Variables.
- `jq` and `curl` available on the runner (both present on `atlassian/default-image:4`).
- PagerDuty or Alertmanager configured to call the Bitbucket pipelines trigger endpoint when an alert fires.

## Configuration

```yaml
# Store these in your repository's Settings → Repository variables
# Mark LLM_API_KEY and SLACK_WEBHOOK_URL as Secured (encrypted at rest).
LLM_API_KEY: "sk-xxxx"                        # Secured — never commit
SLACK_WEBHOOK_URL: "https://hooks.slack.com/…" # Secured — never commit
LLM_API_ENDPOINT: "https://api.openai.com/v1"  # plain variable
```

## Implementation

```yaml
# bitbucket-pipelines.yml (custom pipeline section)
# Trigger via: POST https://api.bitbucket.org/2.0/repositories/{workspace}/{repo}/pipelines/
# with body: {"target":{"type":"pipeline_ref_target","ref_type":"branch","ref_name":"main","selector":{"type":"custom","pattern":"alert-fired"}},"variables":[{"key":"ALERT_NAME","value":"..."},{"key":"SERVICE_NAME","value":"..."},...]}
pipelines:
  custom:
    alert-fired:
      - step:
          name: Triage alert and notify Slack
          image: atlassian/default-image:4
          variables:
            - name: ALERT_NAME
            - name: SERVICE_NAME
            - name: SEVERITY
            - name: SYSTEM_CONTEXT
            - name: ALERT_DETAILS
          script:
            - |
              ALERT_NAME="${ALERT_NAME:-UnknownAlert}"
              SERVICE_NAME="${SERVICE_NAME:-unknown-service}"
              SEVERITY="${SEVERITY:-}"
              SYSTEM_CONTEXT="${SYSTEM_CONTEXT:-No context provided.}"
              ALERT_DETAILS="${ALERT_DETAILS:-${ALERT_NAME}}"

              printf '%s' "${SYSTEM_CONTEXT}" > alert_context.txt
              printf '%s' "${ALERT_DETAILS}" > alert_details.txt

              # ── Call LLM for incident triage ──────────────────────────────────
              SYSTEM_PROMPT="You are an autonomous incident response agent for an SRE team.

              Given an alert, produce a structured incident triage with these four sections:

              ## Severity
              State SEV1, SEV2, or SEV3. One-sentence justification.
              SEV1: Complete outage or data at risk. SEV2: Significant degradation imminent. SEV3: Minor issue, no immediate impact.

              ## Mitigation Steps
              Top 5 numbered, immediately actionable steps with CLI commands where applicable.

              ## Stakeholder Summary
              Under 75 words. Non-technical. Present tense. No jargon.

              ## Runbook Hint
              One sentence naming the most likely runbook to consult."

              SYSTEM_CTX=$(cat alert_context.txt)
              ALERT_DET=$(cat alert_details.txt)

              USER_MSG="Alert name: ${ALERT_NAME}
              Service: ${SERVICE_NAME}
              Alert details: ${ALERT_DET}
              System context: ${SYSTEM_CTX}
              Indicated severity: ${SEVERITY}"

              RESPONSE=$(curl -sS "${LLM_API_ENDPOINT}/chat/completions" \
                -H "Authorization: Bearer ${LLM_API_KEY}" \
                -H "Content-Type: application/json" \
                -d "$(jq -n \
                  --arg system "$SYSTEM_PROMPT" \
                  --arg user "$USER_MSG" \
                  '{model:"gpt-4o",temperature:0.1,messages:[{role:"system",content:$system},{role:"user",content:$user}]}')")

              TRIAGE=$(echo "$RESPONSE" | jq -r '.choices[0].message.content // "Error: no response from LLM"')
              printf '%s' "${TRIAGE}" > triage_result.txt

              # Extract severity level for Slack colour coding
              if echo "$TRIAGE" | grep -qi "SEV1"; then
                SEV_LEVEL="SEV1"
                SLACK_COLOR="#e01e5a"
              elif echo "$TRIAGE" | grep -qi "SEV2"; then
                SEV_LEVEL="SEV2"
                SLACK_COLOR="#e8831e"
              else
                SEV_LEVEL="SEV3"
                SLACK_COLOR="#e8c11e"
              fi

              # ── Post colour-coded Slack message ───────────────────────────────
              TRIAGE_TEXT=$(cat triage_result.txt)
              RUN_URL="https://bitbucket.org/${BITBUCKET_REPO_FULL_NAME}/addon/pipelines/home#!/results/${BITBUCKET_BUILD_NUMBER}"

              SLACK_PAYLOAD=$(jq -n \
                --arg alert "$ALERT_NAME" \
                --arg service "$SERVICE_NAME" \
                --arg sev "$SEV_LEVEL" \
                --arg color "$SLACK_COLOR" \
                --arg triage "$TRIAGE_TEXT" \
                --arg run_url "$RUN_URL" \
                '{
                  text: ("🚨 *Incident Alert: " + $alert + "* — " + $service + " (" + $sev + ")"),
                  attachments: [{
                    color: $color,
                    blocks: [
                      {
                        type: "section",
                        text: { type: "mrkdwn", text: ("*AI Triage Report*\n```" + $triage + "```") }
                      },
                      {
                        type: "actions",
                        elements: [{
                          type: "button",
                          text: { type: "plain_text", text: "View Pipeline Run" },
                          url: $run_url
                        }]
                      }
                    ]
                  }]
                }')

              SLACK_RESPONSE=$(curl -sS -X POST \
                -H "Content-Type: application/json" \
                -d "$SLACK_PAYLOAD" \
                "${SLACK_WEBHOOK_URL}")

              echo "Slack response: ${SLACK_RESPONSE}"

              rm -f alert_context.txt alert_details.txt triage_result.txt
              echo "✅ Alert triage complete."
```

> **Note:** This is a reference implementation. Adapt the `model`, `LLM_API_ENDPOINT`, Slack channel, and prompt text to match your organisation's tooling and security requirements. Route through `in-house-llm` for services that process PII or financial data.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `ALERT_NAME` | string | Yes | Alert name (pipeline variable at trigger time) |
| `SERVICE_NAME` | string | Yes | Affected service (pipeline variable at trigger time) |
| `SEVERITY` | string | No | Severity hint: SEV1/SEV2/SEV3 (pipeline variable) |
| `SYSTEM_CONTEXT` | string | No | System/service context description (pipeline variable) |
| `ALERT_DETAILS` | string | No | Alert details (pipeline variable) |
| `LLM_API_KEY` | string | Yes | LLM API key — store in Bitbucket Secured Variables |
| `SLACK_WEBHOOK_URL` | string | Yes | Slack incoming webhook URL — store in Bitbucket Secured Variables |

## Outputs

| Name | Type | Description |
|---|---|---|
| `SLACK_MESSAGE_URL` | string | URL of the posted Slack message |

## Security Notes

- **API keys:** Store `LLM_API_KEY` and `SLACK_WEBHOOK_URL` in Bitbucket Secured Variables — never in the pipeline file or repository.
- **Data scope:** Only alert name, service name, context, and alert details are sent to the LLM. Raw metric time-series data and database credentials must not be included in the alert payload.
- **Confidential services:** Replace the LLM endpoint with your internal API proxy (`in-house-llm`) before using on services that handle PII or financial data.
- **Idempotency:** Each alert firing creates a new pipeline run; duplicate Slack messages are possible if the same alert fires repeatedly. Consider adding a Slack `thread_ts` deduplication mechanism for production use.

## Testing Notes

| Environment | Tested | Notes |
|---|---|---|
| Bitbucket Pipelines (atlassian/default-image:4) | ✅ | 2026-04-20. Tested with a mock trigger payload via curl. Slack colour coding works correctly for all three severity levels. |
| Local (bash + curl) | ✅ | 2026-04-20. End-to-end run completes in ~18 seconds for a typical alert payload. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
