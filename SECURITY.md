# Security Policy

## Responsible AI Usage

All content in this repository must adhere to responsible AI principles. Contributors are responsible for ensuring that their prompts, skills, agents, workflows, and automations do not:

- Produce or amplify harmful, biased, or discriminatory content.
- Facilitate unauthorized access to systems, data, or services.
- Embed or transmit personally identifiable information (PII) or sensitive business data.
- Bypass or circumvent security controls.
- Violate applicable laws, regulations, or organizational policies.

---

## Data Classification

Every asset must declare a `security_classification` in its front matter:

| Classification | Description | Examples |
|---|---|---|
| `public` | No restrictions; safe to share externally. | Generic prompts, open-source tool integrations. |
| `internal` | For internal use only; do not share outside the organization. | Prompts referencing internal tooling names or workflows. |
| `confidential` | Restricted access; requires explicit approval to view. | Agents with production system access, sensitive automation scripts. |

**Default classification is `internal`** when not specified.

### Rules by Classification

- **Public** content may be referenced in external documentation or demos.
- **Internal** content must not be included in any external-facing portal without a separate review.
- **Confidential** content requires:
  - Approval from a designated security reviewer before merging.
  - Storage without any real credentials or tokens.
  - A documented data-handling justification in the asset's `security_notes` field.

---

## Secrets and Credentials

**Never commit secrets.** This includes:

- API keys, tokens, passwords, certificates.
- Connection strings or database URIs containing credentials.
- Internal IP addresses or hostnames of production systems.

Use placeholder variables instead:

```
{{API_KEY}}
{{DATABASE_URL}}
{{SERVICE_ENDPOINT}}
```

Document how these values should be supplied (environment variables, a secrets manager, etc.) in the asset's `inputs` or `configuration` section.

If a secret is accidentally committed, immediately:

1. Rotate the credential.
2. Open a Jira ticket with the `Security` label (or email the security team directly).
3. Follow the repository's incident-response runbook.

---

## LLM Safety Considerations

When designing prompts or agents:

- Include explicit instructions to avoid generating harmful content.
- Validate and sanitize any user-supplied input before passing it to an LLM.
- Do not pass raw PII to external LLM APIs without data-handling agreements in place.
- Prefer system-prompt constraints over relying on model defaults.
- Test for prompt injection vulnerabilities when agents accept untrusted input.

---

## Dependency and Supply-Chain Security

- Pin versions of any tools or libraries referenced in automations.
- Review third-party MCP servers or plugins before integrating them.
- Prefer well-known, actively maintained packages.
- Run dependency vulnerability scans before publishing an automation.

---

## Reporting a Vulnerability

If you discover a security vulnerability in this repository (e.g., a prompt injection risk, an automation that could leak data, or an accidentally committed secret):

1. **Do not open a public Jira ticket.**
2. Email the security team at: `security@example.com` *(replace with your organization's address)*
3. Include:
   - A description of the vulnerability.
   - The file path(s) affected.
   - Steps to reproduce.
   - Potential impact.
4. You will receive an acknowledgment within **2 business days** and a resolution timeline within **5 business days**.

---

## Supported Versions

Security fixes are applied to the `main` branch. Older release tags are not actively patched unless a critical vulnerability affects them.

---

## Compliance

Content in this repository should comply with:

- Your organization's AI Acceptable Use Policy.
- GDPR / applicable data-privacy regulations when prompts handle personal data.
- SOC 2 / ISO 27001 controls where relevant to automations that interact with production systems.
