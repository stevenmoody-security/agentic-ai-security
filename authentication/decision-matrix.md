# Authentication Decision Matrix

## Purpose

Authentication mechanism selection in an agentic AI architecture is not
arbitrary. Each mechanism is appropriate for a specific type of tool access.
Using the wrong mechanism creates either a security gap or an unnecessary
integration burden. This document defines which mechanism to use and why.

---

## The Three Mechanisms

### IAM Roles with SigV4 Signing
Use when: the tool or resource lives inside AWS.

How it works: the agent holds an IAM role issued by AgentCore Identity. When
calling an AWS-native service, the AWS SDK signs the request automatically
using SigV4, a signing protocol that uses the temporary credentials from the
IAM role. The receiving service verifies the signature against IAM. No token
is passed explicitly. No credential is stored anywhere.

Examples: calling a Lambda function, reading from a DynamoDB table, accessing
an S3 bucket.

Why this mechanism: AWS services are designed to verify IAM credentials
natively. Using IAM for AWS-native tool access requires no additional
infrastructure and provides the strongest available authentication guarantee
within the AWS trust domain.

---

### API Keys
Use when: accessing an API Gateway endpoint where the integration is simple
and the data is not sensitive.

How it works: a shared secret key is passed in the request header. The API
Gateway validates the key and allows or denies the request.

Examples: calling a public or semi-public API where OAuth overhead is not
justified and the exposed data does not warrant stronger authentication.

Why this mechanism: API keys are low friction. For non-sensitive integrations
where the cost of implementing OAuth exceeds the security benefit, API keys
are acceptable. They are not appropriate for sensitive data or privileged
actions because a leaked API key grants permanent access until manually
rotated.

Security note: API keys are static secrets. Treat them with the same
discipline as passwords. Rotate them regularly and store them in a secrets
manager, never in code or environment variables.

---

### JWT Two-Legged OAuth (2LO)
Use when: the tool lives outside your AWS account in a separate trust domain.

How it works: the agent authenticates to the external identity provider using
its client credentials. The identity provider issues a signed JSON Web Token.
The agent presents that JWT when calling the external MCP server. The server
verifies the JWT signature independently without calling back to AWS.
Two-Legged means machine-to-machine: no human user is involved at any step.

Examples: calling a partner API, accessing an external MCP server in a
different trust domain, any tool that lives outside AWS IAM's authority.

Why this mechanism: IAM credentials are not valid outside AWS. A standards-
based token format is required so the receiving system can verify the
credential without sharing an identity provider with the caller. JWT is
the standard. Two-Legged OAuth is the flow for automated, non-human access.

---

## Decision Logic

Ask one question: where does the tool live?

- Inside AWS: use IAM Role with SigV4
- API Gateway endpoint, low sensitivity: use API Key
- Outside AWS in a separate trust domain: use JWT Two-Legged OAuth

---

## Connection to the IAM Portfolio

The EC2S3ReadRole in the iam-least-privilege repository demonstrates IAM
Role authentication in practice. The EC2 instance holds a role, AWS handles
credential issuance through the instance metadata service, and S3 verifies
the request through IAM. No stored credentials anywhere.

AgentCore Identity applies the identical mechanism to AI agents. The agent
holds a role, AWS handles credential issuance through AgentCore Identity,
and AWS-native tools verify requests through IAM. The authentication pattern
is the same. The identity type is different.
