# Audit Strategy: APPLICATION_LOGS and USAGE_LOGS

## Purpose

Logging in an agentic AI system serves two distinct purposes that require
two distinct log streams. Mixing them into a single stream reduces the
usefulness of both. This document explains what each stream captures, why
they are kept separate, and how that separation maps to established audit
principles in cloud security.

---

## The Two Log Streams

### APPLICATION_LOGS
Captures runtime events at the session and conversation level. What the
agent was asked to do, what it decided, what errors occurred, and how
sessions began and ended.

This stream answers operational questions: is the agent working correctly,
are there errors, what is the pattern of usage over time.

### USAGE_LOGS
Captures identity events at the tool call level. Which agent called which
tool, with what inputs, at what time, and what the result was.

This stream answers security questions: what actions were taken, by which
identity, against which resources, and when.

---

## Why Separation Matters

An agent that processes thousands of conversation turns per day generates
a high volume of APPLICATION_LOGS. Embedding tool call records into that
stream makes security-relevant events difficult to find and correlate.

Separating USAGE_LOGS gives the security team a dedicated stream they can
monitor, alert on, and route to a SIEM without processing the full volume
of runtime events. An anomalous tool call pattern is immediately visible
in the USAGE_LOGS without requiring the analyst to filter through session
data.

This is the same principle behind separating CloudTrail management events
from data events. Management events record control plane actions: who
created a role, who modified a policy. Data events record data plane
actions: who read an object, who wrote to a table. Security monitoring
for threat detection focuses on the data events. Compliance review focuses
on the management events. Mixing them degrades both use cases.

---

## Routing

Both log streams are routable independently to:

- CloudWatch Logs: for real-time monitoring and alerting
- S3: for long-term retention and compliance archiving
- Kinesis Data Firehose: for streaming to a SIEM or analytics platform

In a DoD environment, long-term retention of USAGE_LOGS in S3 satisfies
audit record retention requirements under AU-11 (Audit Record Retention).
Real-time routing to CloudWatch enables the continuous monitoring required
under AU-6 (Audit Review, Analysis, and Reporting).

---

## What to Monitor in USAGE_LOGS

Anomalous patterns that warrant investigation:

- An agent calling a tool it has never called before
- Tool call volume outside the normal range for that agent
- Failed authorization decisions from Verified Permissions
- Tool calls at times inconsistent with expected operation
- Calls to external tools from an agent not configured for external access

Each of these patterns is detectable in USAGE_LOGS without requiring access
to APPLICATION_LOGS. This is the value of the separated stream.

---

## RMF Control Mapping

- AU-2 (Event Logging): USAGE_LOGS provide the tool-level audit record
  required for non-human identities operating in the environment
- AU-6 (Audit Review): real-time CloudWatch routing enables continuous
  monitoring of agent behavior against defined baselines
- AU-9 (Protection of Audit Information): log streams must be protected
  from modification and deletion. S3 bucket policies and CloudTrail
  log file integrity validation apply to archived USAGE_LOGS
- AU-11 (Audit Record Retention): S3 routing with defined retention
  policies satisfies the retention requirement for audit records
- AU-12 (Audit Record Generation): AgentCore generates USAGE_LOGS
  automatically for every tool call. This is not optional and cannot
  be disabled by the agent itself.

---

## Connection to Existing Portfolio

The cloudtrail-threat-detection repository covers Scenario 02: CloudTrail
Tampering. In that scenario, an attacker with sufficient permissions
disables CloudTrail logging to create a blind spot in the audit record.

The same risk applies to USAGE_LOGS. If an agent's identity has permissions
to modify its own logging configuration, an attacker who compromises that
identity can disable the audit trail for that agent's actions.

Mitigation: the agent's IAM role must not include permissions to modify
logging configuration. Logging controls are managed by a separate security
identity, not by the agent being logged. This is the same principle as
restricting cloudtrail:StopLogging to a designated security account.
