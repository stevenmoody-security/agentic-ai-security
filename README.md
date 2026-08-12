# SEC307: Agentic AI Security Architecture

Security analysis of the architecture demonstrated in the AWS Summit SEC307
builders workshop: "Design Authentication, Authorization, and Logging Logic
in Agentic AI Apps." This repository documents the security design decisions
behind an AWS-native agentic AI system, with focus on identity, authentication,
authorization, and audit logging.

This is not a deployment guide. It is a security engineering analysis of why
the architecture is designed the way it is, what threats each design decision
mitigates, and how the patterns connect to established cloud security principles.

---

## Why This Architecture Matters

AI agents introduce a category of identity that most security teams are not
yet equipped to reason about: non-human, autonomous, and capable of taking
actions across multiple systems without human involvement in each step.

The security challenges are not new. Least privilege, credential management,
audit logging, and trust boundaries are foundational. What is new is applying
those principles to identities that are not people, do not log in, and may
be operating at a speed and scale that makes human-in-the-loop review
impractical for every action.

This architecture is AWS's answer to that problem. Every design decision
maps back to a security principle that applies equally to human identities,
EC2 instances, and AI agents.

---

## Architecture Components

### Strands Agent SDK
The framework used to define agent behavior, tool use, and decision logic.
Strands provides the structure for how an agent operates. The developer
defines what the agent does. Strands defines how it runs.

### AgentCore Runtime
The AWS execution environment for Strands agents. Provides compute,
scaling, and connectivity to AWS services. The relationship between
Strands and AgentCore Runtime mirrors the relationship between application
code and EC2: the code needs infrastructure to run on.

### AgentCore Identity
Assigns a unique IAM role to each agent. No stored credentials anywhere.
AWS issues temporary credentials automatically, the same mechanism used
by EC2 instance roles. This is the direct extension of the EC2 instance
role pattern to AI agent workloads.

### Amazon Verified Permissions
Makes runtime authorization decisions. Before an agent takes an action,
Verified Permissions is queried: is this agent allowed to perform this
action in this context? Authorization is enforced at runtime, not just
at deploy time.

### AgentCore Gateway
Exposes tools to the agent as MCP endpoints. MCP (Model Context Protocol)
is a standard interface that allows any compliant tool to be called by
any compliant agent without custom integration work for each pair.

---

## Contents

- **architecture/overview.md** — full component breakdown and data flow
- **architecture/trust-domains.md** — internal and external trust domain design
- **authentication/decision-matrix.md** — authentication mechanism selection by tool type
- **threat-models/agent-identity-compromise.md** — agent credential theft and blast radius
- **threat-models/cross-domain-trust-abuse.md** — cross-trust-domain attack scenarios
- **logging/audit-strategy.md** — APPLICATION_LOGS vs USAGE_LOGS separation and routing

---

## Connection to Established Patterns

The security principles in this architecture are not new. They are extensions
of patterns already proven in traditional cloud environments:

- AgentCore Identity extends the EC2 instance role pattern to AI agents
- Cross-domain JWT authentication extends the CrossAccountReadOnly pattern
  to external trust domains that do not share AWS IAM
- Separate identity logging extends the principle of separating CloudTrail
  management events from data events

Understanding this architecture requires no AI-specific security knowledge.
It requires understanding identity, least privilege, and audit logging well
enough to recognize where the same principles apply in a new context.

---

Active TS/SCI | AWS SAA | SSCP | Relocating to Raleigh, NC February 2027
