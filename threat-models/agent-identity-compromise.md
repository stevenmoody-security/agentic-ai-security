# Threat Model: Agent Identity Compromise

## Overview

This threat model covers scenarios where an AI agent's identity credentials
are stolen or abused. AgentCore Identity issues temporary IAM credentials
to each agent. If those credentials are obtained by an attacker, the attacker
can impersonate the agent for the duration of the credential lifetime.

---

## What the Agent Identity Can Do

Determined by the IAM role attached through AgentCore Identity. In a
least-privilege design, the role is scoped to the minimum set of tools
the agent needs to complete its function. An agent that reads from DynamoDB
and calls one Lambda function has a role that permits exactly those two
actions and nothing else.

## What the Agent Identity Cannot Do

- Access tools or resources not explicitly permitted in the IAM role
- Assume other roles unless explicitly permitted
- Access the external trust domain using IAM credentials
- Persist beyond the credential expiration window

---

## Threat Scenarios

### Scenario 1: Credential Exfiltration via Prompt Injection

An attacker crafts malicious input that causes the agent to execute
unintended actions, including exfiltrating its own credentials from the
runtime environment.

Attack path: malicious prompt reaches the agent, agent is manipulated into
calling a tool that sends credential data to an attacker-controlled endpoint,
attacker uses the temporary credentials from outside the environment.

Blast radius: limited to the permissions of the compromised agent's IAM role.
If the role is scoped to read access on two DynamoDB tables, the attacker
can read those two tables for the duration of the credential lifetime.
No other resources are reachable regardless of what the attacker attempts.

Mitigating controls:
- Per-agent identity through AgentCore Identity limits blast radius to one
  agent's permissions. Shared identities would expand the blast radius to
  every agent sharing that identity.
- Temporary credentials expire automatically. The attack window is bounded.
- USAGE_LOGS capture every tool call the agent makes, including any anomalous
  calls that preceded the exfiltration. The exfiltration attempt itself is
  logged even if the credential use after exfiltration is not.
- Verified Permissions enforces authorization at runtime. Even with valid
  credentials, the agent cannot call tools it was not authorized to call.

### Scenario 2: Lateral Movement Attempt

An attacker with stolen agent credentials attempts to access other agents,
other AWS resources, or the external trust domain.

Result: denied at multiple layers.

Other AWS resources: the IAM role is scoped to specific resources. Any
attempt to access resources outside the role's permissions is denied by
IAM with an explicit AccessDenied response.

Other agents: each agent has its own role. Credentials from one agent do
not grant access to another agent's identity or another agent's permitted
resources.

External trust domain: IAM credentials are not valid in the external trust
domain. The external partner MCP server requires a JWT issued by its own
identity provider. Stolen IAM credentials cannot be used to obtain that JWT.

### Scenario 3: Credential Replay After Expiration

An attacker obtains temporary credentials but does not use them immediately.
By the time they attempt to use them, the credentials have expired.

Result: denied. Temporary credentials issued by AgentCore Identity have a
defined lifetime. Expired credentials are rejected by AWS regardless of
whether they were legitimately issued.

---

## Detection

USAGE_LOGS record every tool call made by the agent including the tool name,
inputs, and timestamp. Anomalous patterns to monitor:

- Calls to tools the agent has never called before
- High volume of calls in a short window inconsistent with normal operation
- Calls originating from an unexpected source IP after credential export
- Failed authorization decisions from Verified Permissions indicating the
  agent attempted to access something outside its permitted scope

---

## RMF Relevance

In a DoD environment operating under RMF, a confirmed agent identity
compromise constitutes a security incident requiring:

- AC-2 (Account Management): compromised non-human identity must be
  disabled immediately and a new identity provisioned
- AU-9 (Protection of Audit Information): USAGE_LOGS must be preserved
  for forensic review
- IR-6 (Incident Reporting): the Authorizing Official must be notified
  if the compromise affected a system operating under an ATO
- A POA&M entry must be opened to document the incident, track remediation,
  and prevent recurrence
