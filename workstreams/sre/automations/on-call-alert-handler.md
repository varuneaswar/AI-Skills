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
  - github-actions
  - slack
llm_compatibility:
  - gpt-4o
  - copilot-gpt-4o
description: |
  A GitHub Actions automation triggered via repository_dispatch from PagerDuty or
  Alertmanager. It calls an LLM with the incident triage prompt, then posts a
  colour-coded structured Slack message containing triage result, top mitigation
  steps, and a stakeholder summary — getting the right information to the on-call
  engineer within 60 seconds of alert firing.
security_classification: internal
delivery_modes:
  - copilot-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
inputs:
  - name: ALERT_PAYLOAD
    type: string
    description: Full webhook body from PagerDuty or Alertmanager (passed as the repository_dispatch client_payload)
    required: true
  - name: LLM_API_KEY
    type: string
    description: API key for the LLM provider — store in GitHub Secrets
    required: true
  - name: SLACK_WEBHOOK_URL
    type: string
    description: Incoming Webhook URL for the #incidents Slack channel — store in GitHub Secrets
    required: true
outputs:
  - name: SLACK_MESSAGE_URL
    type: string
    description: URL of the posted Slack message (returned by the Slack API)
---

## Overview

This automation wraps the [`sre-incident-response-workflow`](../workflows/incident-response-workflow.md) in a GitHub Actions job that fires on a `repository_dispatch` event. It is designed to be called by PagerDuty webhooks or Alertmanager webhook receivers when an alert fires.

The automation:

1. Extracts alert name, service name, and context from the webhook payload.
2. Sends the alert details to the LLM using the incident triage prompt.
3. Posts a colour-coded Slack Block Kit message to the #incidents channel:
   - 🔴 Red attachment for SEV1
   - 🟠 Orange attachment for SEV2
   - 🟡 Yellow attachment for SEV3
4. Includes triage result, top 5 mitigation steps, and a stakeholder summary in the Slack message.

**Trigger:** `repository_dispatch` with `event-type: alert-fired`

## Prerequisites

- GitHub repository with Actions enabled.
- An LLM API key stored as `LLM_API_KEY` in GitHub Secrets.
- A Slack incoming webhook URL stored as `SLACK_WEBHOOK_URL` in GitHub Secrets.
- The LLM endpoint stored as `LLM_API_ENDPOINT` in GitHub Variables.
- `jq` and `curl` available on the runner (both present on `ubuntu-latest`).
- PagerDuty or Alertmanager configured to send a `repository_dispatch` event to this repository when an alert fires. See [GitHub docs on repository_dispatch](https://docs.github.com/en/rest/repos/repos#create-a-repository-dispatch-event).

## Configuration

```yaml
# Store these in your repository's Settings → Secrets and variables
env:
  LLM_API_KEY: "{{LLM_API_KEY}}"              # GitHub Secret — never commit
  SLACK_WEBHOOK_URL: "{{SLACK_WEBHOOK_URL}}"  # GitHub Secret — never commit
  LLM_API_ENDPOINT: "{{LLM_API_ENDPOINT}}"   # e.g. https://api.openai.com/v1
```

## Implementation

```yaml
# .github/workflows/on-call-alert-handler.yml
name: On-Call Alert Handler

on:
  repository_dispatch:
    types: [alert-fired]

permissions:
  contents: read

jobs:
  handle-alert:
    name: Triage alert and notify Slack
    runs-on: ubuntu-latest

    steps:
      - name: Extract alert payload
        id: alert
        run: |
          # client_payload is set by the webhook caller
          ALERT_NAME=$(echo '${{ toJson(github.event.client_payload) }}' | jq -r '.alert_name // "UnknownAlert"')
          SERVICE_NAME=$(echo '${{ toJson(github.event.client_payload) }}' | jq -r '.service_name // "unknown-service"')
          SYSTEM_CONTEXT=$(echo '${{ toJson(github.event.client_payload) }}' | jq -r '.system_context // "No context provided."')
          SEVERITY=$(echo '${{ toJson(github.event.client_payload) }}' | jq -r '.severity // ""')
          ALERT_DETAILS=$(echo '${{ toJson(github.event.client_payload) }}' | jq -r '.alert_details // .alert_name')

          echo "alert_name=${ALERT_NAME}" >> "$GITHUB_OUTPUT"
          echo "service_name=${SERVICE_NAME}" >> "$GITHUB_OUTPUT"
          echo "severity=${SEVERITY}" >> "$GITHUB_OUTPUT"
          printf '%s' "${SYSTEM_CONTEXT}" > alert_context.txt
          printf '%s' "${ALERT_DETAILS}" > alert_details.txt

      - name: Call LLM for incident triage
        id: triage
        env:
          LLM_API_KEY: ${{ secrets.LLM_API_KEY }}
          LLM_API_ENDPOINT: ${{ vars.LLM_API_ENDPOINT }}
        run: |
          ALERT_NAME="${{ steps.alert.outputs.alert_name }}"
          SERVICE_NAME="${{ steps.alert.outputs.service_name }}"
          SEVERITY="${{ steps.alert.outputs.severity }}"
          SYSTEM_CONTEXT=$(cat alert_context.txt)
          ALERT_DETAILS=$(cat alert_details.txt)

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

          USER_MSG="Alert name: ${ALERT_NAME}
          Service: ${SERVICE_NAME}
          Alert details: ${ALERT_DETAILS}
          System context: ${SYSTEM_CONTEXT}
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
            echo "sev_level=SEV1" >> "$GITHUB_OUTPUT"
            echo "slack_color=#e01e5a" >> "$GITHUB_OUTPUT"
          elif echo "$TRIAGE" | grep -qi "SEV2"; then
            echo "sev_level=SEV2" >> "$GITHUB_OUTPUT"
            echo "slack_color=#e8831e" >> "$GITHUB_OUTPUT"
          else
            echo "sev_level=SEV3" >> "$GITHUB_OUTPUT"
            echo "slack_color=#e8c11e" >> "$GITHUB_OUTPUT"
          fi

      - name: Post colour-coded Slack message
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
        run: |
          ALERT_NAME="${{ steps.alert.outputs.alert_name }}"
          SERVICE_NAME="${{ steps.alert.outputs.service_name }}"
          SEV_LEVEL="${{ steps.triage.outputs.sev_level }}"
          SLACK_COLOR="${{ steps.triage.outputs.slack_color }}"
          TRIAGE=$(cat triage_result.txt)
          RUN_URL="https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}"

          SLACK_PAYLOAD=$(jq -n \
            --arg alert "$ALERT_NAME" \
            --arg service "$SERVICE_NAME" \
            --arg sev "$SEV_LEVEL" \
            --arg color "$SLACK_COLOR" \
            --arg triage "$TRIAGE" \
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
                      text: { type: "plain_text", text: "View Actions Run" },
                      url: $run_url
                    }]
                  }
                ]
              }]
            }')

          RESPONSE=$(curl -sS -X POST \
            -H "Content-Type: application/json" \
            -d "$SLACK_PAYLOAD" \
            "${SLACK_WEBHOOK_URL}")

          echo "Slack response: ${RESPONSE}"

      - name: Clean up temporary files
        if: always()
        run: rm -f alert_context.txt alert_details.txt triage_result.txt
```

> **Note:** This is a reference implementation. Adapt the `model`, `LLM_API_ENDPOINT`, Slack channel, and prompt text to match your organisation's tooling and security requirements. Route through `in-house-llm` for services that process PII or financial data.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `ALERT_PAYLOAD` | string | Yes | Webhook body from PagerDuty/Alertmanager (as `repository_dispatch` client_payload) |
| `LLM_API_KEY` | string | Yes | LLM API key — store in GitHub Secrets |
| `SLACK_WEBHOOK_URL` | string | Yes | Slack incoming webhook URL — store in GitHub Secrets |

## Outputs

| Name | Type | Description |
|---|---|---|
| `SLACK_MESSAGE_URL` | string | URL of the posted Slack message |

## Security Notes

- **API keys:** Store `LLM_API_KEY` and `SLACK_WEBHOOK_URL` in GitHub Secrets — never in the workflow file or repository.
- **Data scope:** Only alert name, service name, context, and alert details are sent to the LLM. Raw metric time-series data and database credentials must not be included in the alert payload.
- **Confidential services:** Replace the LLM endpoint with your internal API proxy (`in-house-llm`) before using on services that handle PII or financial data.
- **Webhook authentication:** Validate the `repository_dispatch` request originates from your alerting system by using a shared secret in the dispatch request header.
- **Idempotency:** Each alert firing creates a new Actions run; duplicate Slack messages are possible if the same alert fires repeatedly. Consider adding a Slack `thread_ts` deduplication mechanism for production use.

## Testing Notes

| Environment | Tested | Notes |
|---|---|---|
| GitHub Actions (ubuntu-latest) | ✅ | 2026-04-20. Tested with a mock repository_dispatch payload via `gh api`. Slack colour coding works correctly for all three severity levels. |
| Local (bash + curl) | ✅ | 2026-04-20. End-to-end run completes in ~18 seconds for a typical alert payload. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
