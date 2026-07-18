# Study Guide — MCP and AI Agent Security

> **Lessons 3 to 5: Architecture, Trust Flows, and the Lethal Trifecta**  
> Study summary, memory aid, and technical cheat sheet.

> [!IMPORTANT]
> **The model proposes. Policy authorizes. The backend verifies.**

## Table of Contents

- [1. Plain-English Summary](#1-plain-english-summary)
- [2. The Three Application Architecture Eras](#2-the-three-application-architecture-eras)
- [3. Anatomy of an MCP Architecture](#3-anatomy-of-an-mcp-architecture)
- [4. MCP Transports](#4-mcp-transports)
- [5. The MCP Request Journey](#5-the-mcp-request-journey)
- [6. Trust Boundaries](#6-trust-boundaries)
- [7. Identity Loss and the Confused Deputy Problem](#7-identity-loss-and-the-confused-deputy-problem)
- [8. The Agent as a New API Consumer](#8-the-agent-as-a-new-api-consumer)
- [9. The Lethal Trifecta](#9-the-lethal-trifecta)
- [10. Examples Presented in the Course](#10-examples-presented-in-the-course)
- [11. Why Prompts and Filters Are Not Enough](#11-why-prompts-and-filters-are-not-enough)
- [12. How to Break the Lethal Trifecta](#12-how-to-break-the-lethal-trifecta)
- [13. Priority Security Controls](#13-priority-security-controls)
- [14. Operational Scorecard](#14-operational-scorecard)
- [15. Technical Red Flags](#15-technical-red-flags)
- [16. Technical Nuances and Corrections](#16-technical-nuances-and-corrections)
- [17. Key Terms to Study](#17-key-terms-to-study)
- [18. Memory Aids](#18-memory-aids)
- [19. Technical Cheat Sheet](#19-technical-cheat-sheet)
- [20. Review Questions](#20-review-questions)

---

## 1. Plain-English Summary

Before MCP, an AI model could reason and generate text, but it could not easily act inside external systems. Every connection to Jira, GitHub, Gmail, a database, or Stripe had to be built separately.

> [!TIP]
> **Mental model: the LLM is the brain; MCP gives it hands and tools.**

MCP, or **Model Context Protocol**, standardizes how an AI application discovers and uses external capabilities. It does not download new knowledge into the model. Instead, it presents available tools, their descriptions, their parameters, and how to call them.

> [!TIP]
> **Remember: MCP gives the model capabilities, not necessarily judgment.**

## 2. The Three Application Architecture Eras

### 2.1 Traditional Application

```text
User → Application → Backend/API
```

The user clicks buttons, and the application translates those actions into structured API calls. Possible actions are constrained by the interface and by coded application logic.

### 2.2 Application with an AI Agent

```text
User → Application/AI Agent → Backend/API
```

The model interprets a natural-language request and produces a structured action. A new interpretive layer appears. It may misunderstand intent, select the wrong tool, hallucinate a parameter, or follow malicious instructions.

### 2.3 External Agent with MCP

```text
User → AI Host/LLM → MCP Client → MCP Server → Backend/API
```

The user can now ask a general-purpose agent to act inside an external system. The architecture now includes more components, owners, credentials, and trust boundaries.

## 3. Anatomy of an MCP Architecture

| Component | Role | Main Risk |
| --- | --- | --- |
| Host | Application used by the user; orchestrates the model and MCP connections. | Weak permission, context, or installed-server management. |
| LLM / Agent | Interprets requests, chooses tools, and builds arguments. | Probabilistic decisions, hallucination, prompt injection, unsafe tool chaining. |
| MCP Client | Manages the connection to one MCP server and transports calls. | Trusts the agent's decision; may have weak validation or authentication. |
| MCP Server | Exposes tools and translates MCP calls into real API operations. | Access to secrets, files, databases, and privileged functions. |
| Backend | Final system that reads or modifies data. | Accepts valid credentials without knowing the original identity or intent. |

> [!TIP]
> **The Host chooses, the Client transmits, the Server translates, and the Backend executes.**

### MCP Does Not Replace APIs

```text
Agent → MCP → Existing API → Backend
```

The real change is the addition of a new API consumer: the AI agent. The backend already existed, but its exposure and possible behaviors have changed.

## 4. MCP Transports

### 4.1 STDIO — Local Server

```text
Local Host <-> Local MCP Process
```

- Common in desktop applications and IDEs.
- The MCP server runs as a local subprocess.
- Risks include malicious packages, software supply-chain compromise, access to local files, environment variables, secrets, and local code execution.

### 4.2 Streamable HTTP — Remote Server

```text
Host → Network/Internet → Remote MCP Server
```

- Common for SaaS services and multi-tenant environments.
- Risks include missing authentication, cross-tenant data leaks, token theft, TLS misconfiguration, and excessive trust in the service provider.

> [!TIP]
> **Essential question: who controls the server, where does it run, and which credentials does it use?**

## 5. The MCP Request Journey

> [!TIP]
> **Ask → Decide → Request → Execute → Respond**

1. **Ask:** The user expresses an intent in natural language.
2. **Decide:** The model selects a tool and builds its parameters.
3. **Request:** The MCP client wraps the decision in a structured request, often using JSON-RPC.
4. **Execute:** The MCP server calls the target API or system.
5. **Respond:** The backend returns the result through the chain to the user.

### Simplified JSON-RPC Example

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "jira.search",
    "arguments": {
      "query": "status = New"
    }
  }
}
```

## 6. Trust Boundaries

```text
User → Agent → MCP Client → MCP Server → Backend
```

| Boundary | Trust Assumption | Possible Failure |
| --- | --- | --- |
| User → Agent | The request reflects the user's real intent. | Ambiguity, social engineering, malicious indirect content. |
| Agent → Client | The agent chose the correct tool and arguments. | Wrong tool, incorrect amount, hallucinated parameters. |
| Client → Server | The request is legitimate and authorized. | Missing authentication, replay, impersonation, weak validation. |
| Server → Backend | The technical credential represents a legitimate action. | The backend does not know the original user, intent, or business context. |

> [!TIP]
> **A valid technical identity does not prove valid human intent.**

## 7. Identity Loss and the Confused Deputy Problem

```text
User → Agent → MCP → Backend sees only "service-mcp"
```

In a weak architecture, the backend may no longer know which human user initiated the action. A privileged component can then be manipulated into using its authority for a less-privileged actor. This is the **confused deputy** problem.

### Recommended Controls

- Propagate delegated per-user identity whenever possible.
- Use short-lived tokens and minimal scopes.
- Have the backend revalidate authorization for every object and tenant.
- Require explicit confirmation for sensitive actions.
- Avoid "god mode" service accounts for workflows that process untrusted content.

## 8. The Agent as a New API Consumer

| Consumer | Typical Behavior | Risk Profile |
| --- | --- | --- |
| Human | Low volume, one click at a time, constrained by the interface. | Human error and interactive abuse. |
| Traditional Automation | High volume, coded logic, known endpoints. | Predictable; drift is usually measurable. |
| AI Agent | Dynamic decisions, tool chaining, natural-language interpretation. | Emergent behavior, prompt injection, loops, excessive privilege. |

> [!TIP]
> **An API is a tool. An agent decides how to use tools.**

A model-generated decision must never be the sole authorization mechanism for a sensitive action.

## 9. The Lethal Trifecta

| Capability | Definition | Examples |
| --- | --- | --- |
| Private Data | Access to sensitive information. | Email, private repositories, customer records, secrets, databases. |
| Untrusted Content | Reads content controlled by an external or untrusted source. | Email, support ticket, public issue, webpage, uploaded document. |
| External Communication | Can send or expose data outside the trusted environment. | Email, webhook, public pull request, customer reply, remote Markdown image. |

```text
Private Data + Untrusted Content + External Communication
= Structural Data Exfiltration Path
```

Each capability may be acceptable on its own. When the same agent or workflow has all three, untrusted content can influence the agent, access private data, and transmit it to an external destination.

> [!TIP]
> **The attacker needs all three legs; the defender only needs to remove one.**

## 10. Examples Presented in the Course

### Echo Leak

- **Private data:** Email inbox contents.
- **Untrusted content:** An attacker-controlled email.
- **External communication:** Loading a remote Markdown image or URL.

### Supabase MCP

- **Private data:** A database accessed using a highly privileged credential.
- **Untrusted content:** A support ticket submitted by a customer.
- **External communication:** A reply posted in the attacker-visible ticket.

### GitHub MCP

- **Private data:** Private repositories.
- **Untrusted content:** A public issue.
- **External communication:** A pull request or publicly visible output.

> [!TIP]
> **In these scenarios, the system did not necessarily fail. It used its tools inside a poorly separated trust context.**

## 11. Why Prompts and Filters Are Not Enough

A system prompt, classifier, or output filter can reduce risk, but it should not be the primary security boundary. The underlying problem often comes from the structure of access, permissions, and output channels.

> [!TIP]
> **You cannot solve an architectural security problem with prompts alone.**

- Separate trusted instructions from untrusted data.
- Do not give one workflow unnecessary access to all three legs of the trifecta.
- Enforce authorization through deterministic code and backend controls.
- Apply technical restrictions even when the model claims an action is safe.

## 12. How to Break the Lethal Trifecta

| Leg to Remove or Restrict | Possible Controls |
| --- | --- |
| Private Data | Least privilege, restricted views, row-level security, tenant isolation, secret masking, reduced-scope tokens. |
| Untrusted Content | Isolation, provenance tracking, structured data conversion, sandboxing, separate read-only agents, no privileged actions in the same workflow. |
| External Communication | Destination allowlists, block arbitrary URLs, DLP, human approval, separate write tools, disable remote resource loading. |

## 13. Priority Security Controls

### Least Privilege

Each tool should receive only the permissions it requires. A support agent should not have administrative access to the entire production database.

### Identity and Authorization

- Identify the human user, agent, MCP server, and credential involved.
- Authorize each action based on object, tenant, amount, and business context.
- Prefer delegated user tokens over shared service accounts.

### Human Approval

Show the exact action before confirmation, including the action type, target, amount, destination, initiating identity, and affected data.

### Behavioral Limits

- Rate limits, task budgets, maximum tool-call counts, and timeouts.
- Transaction-value limits, destination restrictions, and loop detection.
- Idempotency controls and tool-chain restrictions.

### Logging

```text
User → Session → Model → Tool → Arguments → MCP → Credential → Backend → Result → Destination
```

Logs should allow defenders to reconstruct the complete action chain without exposing secrets in clear text.

## 14. Operational Scorecard

| Tool | Private Data? | Untrusted Content? | External Output? | Credential | Approval? |
| --- | --- | --- | --- | --- | --- |
| `repos.read` | Yes | No | No | User OAuth | No |
| `issues.read` | No | Yes | No | User OAuth | No |
| `pull_request.create` | No | No | Yes | Organization OAuth | Yes |
| `database.query` | Yes | Possible | No | Service role | Yes |
| `ticket.reply` | No | Yes | Yes | Support account | Yes |

Do not analyze tools only in isolation. Multiple tools combined in the same task can complete all three legs of the trifecta.

### Review Questions for Each Tool

1. Who develops and maintains the server?
2. Does it use local STDIO or remote HTTP?
3. How are the client and user authenticated?
4. Is the human identity propagated to the backend?
5. Which tenants, objects, and data can the credential access?
6. Does the tool ingest external content?
7. Can it write or send data externally?
8. Which actions require approval?
9. What volume and rate limits exist?
10. Can the entire chain be reconstructed from logs?

## 15. Technical Red Flags

- Public MCP server with no authentication.
- MCP package installed from an unknown source.
- Shared administrator credential or service-role token.
- Simultaneous access to private data and untrusted content.
- Ability to execute arbitrary HTTP requests, SQL queries, or system commands.
- Agent can publish or send data without approval.
- Human identity is not propagated to the backend.
- No rate limits or multi-tenant isolation.
- Secrets are accessible through environment variables.
- Logs cannot identify the original user or final destination.

## 16. Technical Nuances and Corrections

| Simplified Statement | Important Nuance |
| --- | --- |
| "MCP downloads a skill." | It exposes tools; it does not necessarily modify the model's weights or knowledge. |
| "No API or documentation is needed." | The MCP server still integrates APIs and manages authentication, permissions, schemas, and maintenance. |
| "The MCP server is the only attack point." | The host, content, tool descriptions, client, transport, and backend are also part of the threat model. |
| "The backend simply follows the rules." | It must revalidate identity, authorization, tenant, target, and business context. |
| "AI is random." | Models may be nondeterministic, but they are not completely chaotic or random. |

## 17. Key Terms to Study

| Term | Short Definition |
| --- | --- |
| MCP | Standardized protocol that allows an AI application to use external capabilities. |
| Host | Application that hosts the AI experience and its MCP connections. |
| MCP Client | Component that manages one connection to an MCP server. |
| MCP Server | Component that exposes tools and calls target systems. |
| Tool | Structured function the model can request to execute. |
| JSON-RPC | Structured request-and-response format. |
| Indirect Prompt Injection | Malicious instruction embedded in content the agent reads. |
| Confused Deputy | A privileged component manipulated into acting for an unauthorized actor. |
| Identity Propagation | Preservation of the original user identity throughout the action chain. |
| Tool Chaining | Sequential use of multiple tools for one task. |
| Tenant Isolation | Separation of data and actions between customers or organizations. |
| Human-in-the-Loop | Human review or approval before a sensitive action. |

## 18. Memory Aids

> [!TIP]
> **Components: H → C → S → B — Host, Client, Server, Backend**

> [!TIP]
> **Flow: Ask → Decide → Request → Execute → Respond**

> [!TIP]
> **Trifecta: P + U + E — Private, Untrusted, External**

- MCP gives the model hands, not wisdom.
- A valid badge proves who entered, not why the action is correct.
- The model proposes; policy authorizes; the backend verifies.
- An API is a tool; an agent decides how to use tools.

## 19. Technical Cheat Sheet

```text
ARCHITECTURE
User → Host/LLM → MCP Client → MCP Server → Backend/API

FLOW
Ask → Decide → Request → Execute → Respond

TRANSPORTS
STDIO: local subprocess; supply-chain, secrets, filesystem, code execution
HTTP: remote server; authentication, TLS, tenant isolation, provider trust

LETHAL TRIFECTA
Private Data + Untrusted Content + External Communication

PRIORITY CONTROLS
Least privilege | Per-user identity | Narrow scopes | Tenant isolation
Destination allowlists | Human approval | Rate limits | Tool-call logging
Backend authorization | Separation of read/write capabilities
```

> [!TIP]
> **Final rule: never use the probabilistic judgment of an LLM as the only authorization boundary for a sensitive action.**

## 20. Review Questions

1. **Does MCP replace APIs?**  
   No. It generally relies on existing APIs.

2. **Which component chooses the tool?**  
   The model or agent logic inside the host.

3. **Which component actually calls Jira or Stripe?**  
   The MCP server.

4. **Why is a valid credential not enough?**  
   It does not prove the identity or intent of the original user.

5. **What are the three legs of the Lethal Trifecta?**  
   Private data, untrusted content, and external communication.

6. **Must one tool contain all three capabilities?**  
   No. Their combination across one workflow is enough.

7. **Why is a better prompt not sufficient?**  
   The risk also comes from permissions, architecture, and available output channels.

8. **Which defense provides the highest leverage?**  
   Remove or isolate at least one leg of the trifecta.

9. **Which rule must remain in the backend?**  
   Verify identity, authorization, and business context.

10. **Which principle summarizes agent security?**  
    The model proposes; the deterministic system decides and controls.

---

## Recommended Use

- Use the **operational scorecard** before adding a new MCP tool or server.
- Evaluate the **Lethal Trifecta** across the complete workflow, not only tool by tool.
- Keep sensitive authorization in deterministic code and backend controls.

## License and Attribution

This file is a study guide derived from course notes supplied by the user. Adapt the license section to the target repository and the original course's applicable terms.
