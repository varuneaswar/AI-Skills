---
id: test-intelligent-suite-maintenance
title: Intelligent Test Suite Maintenance Agent
type: idea
version: "1.0.0"
status: draft
workstream:
  - testers
author: ai-skills-maintainer
created: 2024-01-01
updated: 2024-01-01
tags:
  - test-maintenance
  - automation
  - agent
  - flaky-tests
description: |
  An agent that monitors the test suite for flaky tests, outdated test data, and redundant
  coverage, and proposes targeted fixes — reducing the time teams spend maintaining tests
  rather than writing new ones.
security_classification: internal
---

## Overview

Test suites become maintenance burdens: flaky tests erode confidence, outdated tests generate false failures, and redundant tests slow the pipeline. This agent addresses all three automatically.

## Proposed Approach

**Asset type:** `agent`  
**Potential delivery mode:** `api-endpoint` (triggered after each CI run)

The agent would:
1. Ingest CI run history to identify tests that fail intermittently (flaky tests).
2. Use `test-coverage-gap-analysis` to find redundant tests covering the same scenario.
3. Cross-reference tests against feature flags to identify tests for removed features.
4. Raise PRs with targeted fixes: stabilise flaky tests, remove dead tests, update stale fixtures.
5. Never delete tests — always moves them to a `deprecated/` folder in the PR for human review.

## Expected Benefits

- CI pipeline is faster and more reliable.
- Engineers trust test results more when flaky tests are fixed.
- QA capacity is freed from reactive maintenance to proactive coverage expansion.

## Acceptance Criteria

- [ ] Agent correctly identifies flaky tests with >3 intermittent failures in 30 days.
- [ ] Agent PRs include the failure history as evidence for each flagged test.
- [ ] Agent never modifies production source code — only test files.
- [ ] Redundancy detection has <5% false positive rate.

## Open Questions

- How should the agent handle flakiness caused by environmental issues vs. test code?
- Which CI systems to integrate with first? (GitHub Actions, Jenkins, GitLab CI)

## References

- Existing skill: `test-coverage-gap-analysis` — for redundancy detection
- Existing prompt: `test-generate-test-cases-from-user-story` — for generating replacement tests
