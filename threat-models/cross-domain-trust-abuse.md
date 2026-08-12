# Threat Model: Cross-Domain Trust Abuse

## Overview

This threat model covers scenarios where the trust relationship between the
internal AWS trust domain and the external partner trust domain is exploited.
The boundary between these two domains is enforced through JWT Two-Legged
OAuth. Any weakness in how that boundary is established or verified creates
an attack surface.

---

## The Trust Boundary

The internal domain trusts IAM. The external domain trusts its own identity
provider. Neither trusts the other's credential system directly. The only
bridge between them is a JWT issued by the external identity provider and
presented by the internal agent when calling external tools.

For this bridge to be secure, three conditions must hold:

1. The external identity provider only issues tokens to authenticated clients
2. The internal agent stores its client credentials securely
3. The external MCP server verifies the JWT signature before acting on requests

If any of these conditions fails, the trust boundary is compromised.

---

## Threat Scenarios

### Scenario 1: Client Credential Theft

The internal agent authenticates to the external identity provider using
client credentials. If those credentials are stolen, an attacker can
obtain valid JWTs and call the external MCP server as if they were the
legitimate agent.

Attack path: attacker obtains the agent's client credentials through code
exposure, environment variable leakage, or secrets manager misconfiguration.
Attacker authenticates to the external identity provider, receives a valid
JWT, and calls the external MCP server with full permissions of the
legitimate agent.

Blast radius: limited to what the external MCP server exposes and what the
compromised agent's client credentials are authorized to access. The attacker
cannot use stolen client credentials to access internal AWS resources because
those credentials are only valid with the external identity provider.

Mitigating controls:
- Client credentials must be stored in a secrets manager, never in code or
  environment variables
- Client credentials should be rotated on a defined schedule
- The external identity provider should enforce short JWT lifetimes to
  limit the window of exposure from a compromised token
- USAGE_LOGS capture all external tool calls. Anomalous call volumes or
  patterns from the agent's identity are detectable.

### Scenario 2: JWT Forgery Attempt

An attacker attempts to forge a JWT to call the external MCP server without
obtaining legitimate client credentials.

Result: denied. JWTs are signed by the external identity provider using a
private key. The external MCP server verifies the signature using the
corresponding public key. A forged JWT that was not signed by the legitimate
identity provider will fail signature verification and be rejected.

This is why JWT signature verification is mandatory. A server that accepts
JWTs without verifying signatures provides no authentication guarantee.

### Scenario 3: Token Replay

An attacker intercepts a valid JWT in transit and attempts to replay it
to call the external MCP server.

Blast radius: limited to the permissions encoded in the intercepted token
and valid only until the token expires.

Mitigating controls:
- Short token lifetimes limit the replay window
- Transport encryption via TLS prevents interception in transit
- The external MCP server should validate the token expiration claim before
  processing any request

### Scenario 4: Confused Deputy via MCP Gateway

A malicious external MCP server tricks the AgentCore Gateway into making
calls on the attacker's behalf, using the agent's internal IAM credentials
to access internal AWS resources.

Result: mitigated by trust domain separation. The AgentCore Gateway uses
different authentication mechanisms for internal and external tools. Internal
tool calls use IAM. External tool calls use JWT. The gateway does not use
IAM credentials when calling external tools, so a malicious external server
cannot leverage the agent's internal IAM role.

This is the same confused deputy problem addressed by the ExternalId
condition in the CrossAccountReadOnly role in the iam-least-privilege
repository, applied to the AI agent context.

---

## Detection

- APPLICATION_LOGS record all agent sessions and the external tools called
  during each session. Unexpected external tool calls or call volumes outside
  normal patterns indicate possible abuse.
- The external identity provider should maintain its own access logs. Any
  authentication from an unexpected source IP or at an unexpected time
  warrants investigation.
- Failed JWT verification events at the external MCP server indicate either
  a forgery attempt or a misconfiguration.

---

## RMF Relevance

Cross-domain trust abuse affecting a DoD system under RMF requires:

- AC-17 (Remote Access): cross-domain tool access is a form of remote
  access and must be authorized, monitored, and logged
- SC-8 (Transmission Confidentiality): all cross-domain communication must
  be encrypted in transit
- IR-6 (Incident Reporting): confirmed cross-domain trust abuse must be
  reported to the Authorizing Official
- A POA&M entry must be opened covering both the incident and any
  architectural remediation required to prevent recurrence
