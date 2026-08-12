# Trust Domains

## What Is a Trust Domain

A trust domain is a boundary within which a single identity system governs
authentication. Inside one trust domain, identities are issued, verified,
and trusted by the same authority. Across trust domain boundaries, that
authority does not extend. A separate mechanism is required to establish
trust between domains.

In AWS, IAM is the identity authority within your account. IAM issues
credentials, verifies them, and enforces access decisions. IAM does not
extend to systems outside your AWS account. Any system in a different
AWS account or outside AWS entirely is in a different trust domain.

---

## The Two Trust Domains

### Internal Trust Domain
Contains the primary Strands agent running on AgentCore Runtime, the
AgentCore Identity service issuing per-agent IAM roles, Amazon Verified
Permissions enforcing runtime authorization, and AgentCore Gateway exposing
internal tools as MCP endpoints.

Within this domain, authentication uses IAM roles with SigV4 signing.
AWS handles credential issuance and verification automatically. No tokens
need to be passed explicitly because AWS services recognize and verify
IAM credentials natively.

Tools in this domain: Lambda functions, DynamoDB tables, S3 buckets, and
other AWS-native services. All are reachable through IAM.

### External Trust Domain
Contains a partner system with its own identity provider and its own MCP
server exposing tools that the primary agent needs to call.

IAM does not reach this domain. The partner system has no AWS account and
no concept of IAM roles. A standards-based authentication mechanism is
required: JWT Two-Legged OAuth. The primary agent authenticates to the
partner identity provider, receives a signed JSON Web Token, and presents
that token when calling the partner MCP server. The partner system verifies
the token independently without calling back to AWS.

---

## Why the Boundary Matters

The trust domain boundary is a security control, not just an architectural
concept. It forces an explicit authentication decision at every cross-domain
call. There is no implicit trust between domains.

If an agent in the internal domain is compromised, the attacker holds IAM
credentials scoped to that agent's role. Those credentials do not grant
access to the external trust domain. The external domain requires a valid
JWT issued by its own identity provider. The blast radius of an internal
agent compromise stops at the trust domain boundary.

The reverse is also true. A compromised JWT from the external domain does
not grant access to internal AWS resources. IAM and JWT are separate
credential systems. Possession of one does not imply possession of the other.

---

## Connection to Existing Portfolio Patterns

The CrossAccountReadOnly role in the iam-least-privilege repository
demonstrates the same concept at the AWS account level. One AWS account
delegates read-only access to another through a trust policy and an
ExternalId condition. The delegating account and the trusted account are
in different trust domains within AWS IAM.

The SEC307 external trust domain extends this concept beyond AWS entirely.
The external partner does not share AWS IAM at all. JWT Two-Legged OAuth
is the cross-domain trust mechanism when IAM is not an option.

Both patterns enforce the same principle: trust is explicit, bounded, and
requires a specific credential type that only the authorized party can present.
