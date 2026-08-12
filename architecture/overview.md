# Architecture Overview

## Purpose

This document describes the full component architecture of the SEC307 agentic
AI system and the security rationale behind each design decision. The
architecture was demonstrated at the AWS Summit SEC307 builders workshop:
"Design Authentication, Authorization, and Logging Logic in Agentic AI Apps."

---

## Component Breakdown

### Strands Agent SDK
The agent framework. Strands defines how the agent reasons, selects tools,
and handles responses. It is the application layer: the developer writes
agent behavior using Strands, and that behavior executes on AgentCore Runtime.

Security relevance: the agent's tool access is defined at this layer. What
tools the agent can call, what inputs it passes, and what it does with
responses are all governed by the Strands configuration. Least privilege
starts here.

---

### AgentCore Runtime
The AWS execution environment for Strands agents. AgentCore Runtime is to
a Strands agent what EC2 is to application code: it provides the compute
environment, scaling, and connectivity to other AWS services.

Security relevance: the runtime is where the IAM role from AgentCore Identity
is attached. Just as an EC2 instance receives its role at launch, an agent
receives its identity at runtime. The runtime enforces that each agent
operates under its own dedicated identity.

---

### AgentCore Identity
The per-agent identity service. AgentCore Identity assigns a unique IAM role
to each agent instance. AWS issues temporary credentials automatically through
the same mechanism used by EC2 instance roles: no stored credentials, no
access keys in code or environment variables, automatic rotation.

Security relevance: this is the core non-human identity control in the
architecture. Shared identities across agents are a least-privilege failure.
If one agent is compromised and all agents share an identity, the blast radius
is the full scope of that shared identity. Per-agent identity limits the blast
radius to a single agent's permissions.

The EC2 connection: EC2 instance roles assign an IAM role to a server. AWS
makes temporary credentials available through the instance metadata service
at 169.254.169.254. AgentCore Identity applies the identical principle to
AI agents. The mechanism is the same. The compute resource is different.

---

### Amazon Verified Permissions
The runtime authorization service. Verified Permissions evaluates whether a
specific agent is allowed to perform a specific action in a specific context
at the moment the action is requested.

Security relevance: authentication confirms who the agent is. Authorization
confirms what the agent is allowed to do. These are separate problems and
require separate controls. Verified Permissions enforces authorization at
runtime rather than relying solely on IAM policies set at deploy time.
This allows authorization decisions to incorporate context that was not
known when the agent was deployed.

---

### AgentCore Gateway
The tool exposure layer. AgentCore Gateway exposes tools to agents as MCP
endpoints. MCP (Model Context Protocol) is a standard interface: any
MCP-compliant tool can be called by any MCP-compliant agent without
custom integration work for each tool-agent pair.

Security relevance: the gateway is the enforcement point for tool access.
Rather than agents calling tools directly, all tool access is routed through
the gateway. This creates a single control point where authentication,
authorization, and logging can be applied consistently regardless of what
tool is being called.

---

## Data Flow

1. A request reaches the Strands agent running on AgentCore Runtime.
2. The agent determines it needs to call a tool.
3. AgentCore Identity provides the agent credentials for the call.
4. Verified Permissions is queried: is this agent allowed to call this tool?
5. If authorized, the agent calls the tool through AgentCore Gateway.
6. The gateway routes the call to the appropriate tool using the correct
   authentication mechanism for that tool type.
7. The response returns through the gateway to the agent.
8. All steps are logged to APPLICATION_LOGS and USAGE_LOGS independently.

---

## Why Each Component Is Separate

Each component solves one problem. Identity is separate from authorization.
Authorization is separate from tool access. Tool access is separate from
logging. This separation mirrors defense-in-depth: no single component
failure compromises the entire security posture. A misconfigured tool policy
does not bypass identity controls. A compromised agent identity does not
automatically grant access to tools the agent was not authorized to call.
