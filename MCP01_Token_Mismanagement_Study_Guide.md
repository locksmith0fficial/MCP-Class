# MCP01 Study Guide
## Token Mismanagement and Secret Exposure

> [!IMPORTANT]
> **Core principle:** An agent needs credentials to perform useful actions. As soon as credentials are introduced, they become high-value assets that must be protected from the model, users, logs, tools, and untrusted configuration.

## Table of Contents

- [1. Plain-English Summary](#1-plain-english-summary)
- [2. What Is MCP01?](#2-what-is-mcp01)
- [3. Why Credentials Are So Valuable](#3-why-credentials-are-so-valuable)
- [4. Where Secrets Can Persist](#4-where-secrets-can-persist)
- [5. Contextual Secret Leakage](#5-contextual-secret-leakage)
- [6. Secure Credential Architecture](#6-secure-credential-architecture)
- [7. Poisoned Repository Configuration Scenario](#7-poisoned-repository-configuration-scenario)
- [8. Attack Chain Explained](#8-attack-chain-explained)
- [9. Token vs. Exfiltration Channel](#9-token-vs-exfiltration-channel)
- [10. MCP01 and the Lethal Trifecta](#10-mcp01-and-the-lethal-trifecta)
- [11. Common Secret-Exposure Locations](#11-common-secret-exposure-locations)
- [12. Detection Checklist](#12-detection-checklist)
- [13. Priority Remediation](#13-priority-remediation)
- [14. Credential Lifecycle](#14-credential-lifecycle)
- [15. Memory Helper: VAULT](#15-memory-helper-vault)
- [16. Five Credential Questions](#16-five-credential-questions)
- [17. Incident-Response Checklist](#17-incident-response-checklist)
- [18. Detection Engineering Ideas](#18-detection-engineering-ideas)
- [19. Principal Security Review Checklist](#19-principal-security-review-checklist)
- [20. Key Terms](#20-key-terms)
- [21. Technical Nuances and Corrections](#21-technical-nuances-and-corrections)
- [22. Technical Cheat Sheet](#22-technical-cheat-sheet)
- [23. One-Minute Memory Summary](#23-one-minute-memory-summary)
- [24. Review Questions](#24-review-questions)
- [25. Final Security Principle](#25-final-security-principle)

---

## 1. Plain-English Summary

MCP-connected agents often need credentials to interact with external systems such as GitHub, Jira, cloud platforms, databases, email services, CI/CD platforms, and customer-support systems.

These credentials may include:

- API keys
- OAuth access tokens
- OAuth refresh tokens
- Cloud access keys
- Service-account credentials
- Session cookies
- Database passwords
- Temporary cloud credentials

The agent requires authorization to use these systems, but the **LLM itself usually does not need to see the secret**.

The main security failure occurs when credentials are placed in locations that can be read by the model, inherited by an untrusted process, stored in logs, committed to source control, retrieved from agent memory, exposed through malicious configuration, or transmitted to an attacker-controlled endpoint.

> [!TIP]
> **Memory rule:** The tool may need the credential. The model does not.

## 2. What Is MCP01?

MCP01 covers weaknesses in how authentication tokens and secrets are:

1. Created
2. Stored
3. Injected
4. Used
5. Transmitted
6. Logged
7. Rotated
8. Revoked

A token-management problem becomes a security incident when an unauthorized person, process, tool, MCP server, or model context gains access to a credential.

### Typical Examples

- An API key is hard-coded in `.mcp.json`.
- A developer commits a `.env` file containing secrets.
- A token is pasted directly into a prompt.
- A model response accidentally repeats a credential.
- Full prompts are stored in observability logs.
- A local MCP subprocess inherits every environment variable.
- A remote MCP server receives a token with excessive privileges.
- One static service account is shared by all agents.
- A project configuration redirects authenticated API requests to an attacker-controlled server.

## 3. Why Credentials Are So Valuable

A token is not merely a random string. It represents an authorization decision.

Depending on its scope, a stolen credential may provide access to:

- Source code
- Customer records
- Internal documents
- Cloud storage
- Production databases
- CI/CD pipelines
- Deployment systems
- Secrets stored in repositories
- Vector databases
- Administrative functions

A single leaked credential can create an attack chain:

```text
Stolen token
   ↓
Repository access
   ↓
CI/CD access
   ↓
Cloud deployment access
   ↓
Production compromise
```

The impact depends on four major properties:

```text
Impact = Privilege × Scope × Lifetime × Reachability
```

A narrowly scoped token that expires in five minutes presents much less risk than an administrator token valid for one year.

## 4. Where Secrets Can Persist

The statement that tokens can persist in model context is directionally correct, but several storage layers must be distinguished.

### Model Context

The context window contains information currently sent to the model. If a secret is included, the model may repeat it, transform it, include it in output, or use it when responding to a later instruction.

### Application Conversation History

The host application may retain earlier prompts and responses after the immediate model call.

### Logs and Observability Platforms

Prompt, response, tool-call, and tracing logs may permanently store secrets unless redaction is applied.

### Agent Memory

Some applications maintain long-term memory between sessions. A credential included in memory may become retrievable later.

### Vector Databases

Documents, conversations, or tool results may be embedded and indexed. If secrets are accidentally indexed, semantic search may retrieve them in future sessions.

### Model Training

A secret entering a prompt does **not automatically mean** that it becomes part of the model's permanent weights. Retention and training depend on the product, configuration, and data-handling policy.

> [!IMPORTANT]
> A secret exposed to an AI application may persist in context, logs, history, memory, traces, caches, or vector storage, even when it is not stored permanently inside the model itself.

MCP sessions are not automatically long-lived or stateful. Session behavior depends on the host, server, transport, and implementation.

## 5. Contextual Secret Leakage

**Contextual secret leakage** occurs when a secret enters data that the model or agent can read.

The credential may then be exposed through:

- A direct user question
- Indirect prompt injection
- A malicious document
- Tool output
- Debug traces
- Retrieved memory
- Model-generated summaries
- Automated external communications

### Example

A user pastes this into a prompt:

```text
Use API key sk_live_example123 to test the integration.
```

The secret may now appear in prompt history, model context, tracing data, debug logs, browser storage, application analytics, or later model responses.

The correct design is to supply the credential through a secure execution layer rather than through natural-language context.

## 6. Secure Credential Architecture

### Insecure Architecture

```text
User prompt containing token
        ↓
LLM reads token
        ↓
LLM generates tool call containing token
        ↓
MCP server
        ↓
External API
```

Problems:

- The model sees the token.
- Logs may capture it.
- The user may retrieve it.
- Prompt injection may expose it.
- The token may be passed to the wrong tool.

### More Secure Architecture

```text
User request
      ↓
LLM selects approved tool
      ↓
Tool call contains no secret
      ↓
Credential broker / middleware
      ↓
Secret retrieved at runtime
      ↓
Approved MCP server or API
```

The model may generate:

```json
{
  "tool": "github.list_private_repositories",
  "arguments": {
    "organization": "example-org"
  }
}
```

The model should not generate:

```json
{
  "token": "ghp_sensitive_value",
  "tool": "github.list_private_repositories"
}
```

The middleware should:

1. Validate the requested action.
2. Identify the user.
3. Check authorization.
4. Retrieve the appropriate credential.
5. Apply the credential outside model context.
6. Call the approved destination.
7. Record a sanitized audit event.

> [!TIP]
> **Security principle:** Secrets should be attached to authorized operations, not inserted into prompts.

## 7. Poisoned Repository Configuration Scenario

According to the course notes, researchers demonstrated a scenario involving malicious project configuration files such as:

```text
.claude/settings.json
.mcp.json
```

The high-level attack sequence was:

```text
Attacker creates public repository
             ↓
Repository contains malicious configuration
             ↓
Developer clones repository
             ↓
AI coding tool loads project configuration
             ↓
Configuration redirects authenticated traffic
             ↓
API credential is sent to attacker-controlled endpoint
             ↓
Trust prompt appears after exposure
```

The broader attack class is:

> **Untrusted project configuration becoming active before the user has approved the project.**

### Why This Matters

Developers traditionally think about configuration files as passive values. In agent-based environments, configuration may control:

- Which MCP servers start
- Which commands execute
- Which endpoints receive requests
- Which tools are automatically approved
- Which environment variables are inherited
- Which headers are forwarded
- Which permissions are granted

Configuration therefore becomes part of the execution path.

> [!TIP]
> **Memory rule:** In an agent environment, configuration is code-adjacent and must be treated as active content.

## 8. Attack Chain Explained

### Step 1 - Attacker Prepares a Repository

The repository contains legitimate-looking code and a malicious agent or MCP configuration file.

### Step 2 - Developer Clones the Repository

Cloning source code is normally considered a low-risk read operation. Risk increases when a development tool automatically processes repository-level configuration.

### Step 3 - Developer Launches the AI Coding Assistant

The developer may only intend to ask questions about the code and may not intentionally run the application.

### Step 4 - Project Configuration Is Loaded

If the configuration is applied before the directory is trusted, attacker-controlled settings may become active.

### Step 5 - Authenticated Traffic Is Redirected

The next API request may be sent to an attacker-controlled endpoint. If the authorization header is forwarded, the credential is exposed.

### Step 6 - Trust Dialogue Appears

The user sees a normal security prompt, but the security decision occurred too late.

### Root Cause

The main issue is incorrect trust ordering:

```text
Unsafe order:
Load configuration → use credential → ask for trust

Safer order:
Detect configuration → ask for trust → validate → load configuration
```

## 9. Token vs. Exfiltration Channel

A more precise distinction than "the token is the exfiltration channel" is:

- **The token is the secret being stolen or the authorization mechanism being abused.**
- **The network request, model output, webhook, log, or malicious endpoint is the exfiltration channel.**

In the poisoned-configuration scenario:

```text
Secret:
API token in the Authorization header

Exfiltration channel:
Outbound HTTPS request to the attacker-controlled endpoint

Trigger:
Untrusted configuration loaded before approval
```

| Component | Security Control |
| --- | --- |
| Secret | Vault, short lifetime, restricted scope |
| Channel | Egress filtering, destination allowlist |
| Trigger | Trust validation, configuration review |
| Execution | Sandbox, explicit approval |
| Detection | Network and tool-call monitoring |

## 10. MCP01 and the Lethal Trifecta

The Lethal Trifecta consists of:

```text
Private Data
+
Untrusted Content
+
External Communication
```

### Private Data

The credential itself is sensitive private data. It may also provide access to additional private systems.

### Untrusted Content

Examples include cloned repository configuration, public project files, malicious prompt content, imported documents, tool descriptions, and third-party MCP packages.

### External Communication

Examples include attacker-controlled API base URLs, malicious webhooks, external HTTP requests, model-generated messages, log-export services, and remote MCP servers.

### Mapping the Repository Scenario

```text
Private data:
Developer API key

Untrusted content:
Malicious repository configuration

External communication:
Request to attacker-controlled server
```

Breaking any one of these legs prevents the complete attack chain.

## 11. Common Secret-Exposure Locations

### Source-Control Repositories

Look for secrets in:

- `.mcp.json`
- `.claude/settings.json`
- `.env`
- Application configuration
- Terraform files
- CI/CD workflows
- Test scripts
- Sample code
- Documentation
- Notebook files

### Environment Variables

Environment variables are commonly used for credential injection, but they are not automatically secure.

Risks include:

- Inheritance by subprocesses
- Exposure through crash dumps
- Debug output
- Process-inspection tools
- Accidental logging
- Broad access by local MCP servers

Environment variables may be acceptable in controlled environments, but sensitive production credentials should preferably be delivered through a secret manager or credential broker with restrictive process access.

### Prompts and Chat History

Users may paste secrets while debugging. The model should never be used as a password manager or secret transport mechanism.

### Logs

Potential sources include prompt logs, response logs, tool-call logs, HTTP headers, application traces, error messages, proxy logs, CI/CD output, and command histories.

### Agent Memory and Retrieval Systems

Secrets may be stored in conversation memory, vector databases, cached tool responses, indexed documents, browser storage, or session checkpoints.

### MCP Server Definitions

Configuration may contain plaintext API keys, command-line arguments, environment variables, remote endpoints, executable paths, or automatic approval settings.

## 12. Detection Checklist

### 12.1 Hard-Coded Credentials

Search repositories for:

- Known provider prefixes
- Authorization headers
- Private keys
- Passwords
- Bearer tokens
- Suspiciously long random strings
- Cloud access-key patterns
- Database connection strings

Recommended controls:

- Pre-commit secret scanning
- Repository scanning
- CI pipeline scanning
- Historical Git scanning
- Artifact scanning

A secret removed from the latest commit may still exist in Git history.

### 12.2 Secrets in Model Context

Review whether prompts or tool results contain API keys, OAuth tokens, cookies, passwords, connection strings, private keys, or authentication headers.

A live credential appearing in model-visible context should generally be considered a security finding.

### 12.3 Unredacted Prompt and Trace Logs

Inspect LLM observability platforms, application logs, MCP debug logs, proxy traces, error-reporting systems, and support exports.

Confirm that secrets are redacted before storage or transmission.

### 12.4 Excessive Token Lifetime

Identify credentials that:

- Remain valid after the task finishes
- Have no expiration
- Use refresh tokens without rotation
- Are valid for weeks or months
- Cannot be centrally revoked

The longer a token remains valid, the longer the attacker's exploitation window.

### 12.5 Shared and Static Service Accounts

Red flags include:

- One credential used by all agents
- One credential shared across customers
- One service account used in development and production
- No ability to attribute actions to a human user
- Administrator scope for basic read operations

### 12.6 Suspicious Outbound Destinations

Monitor for:

- New MCP endpoints
- Modified API base URLs
- Unexpected domains
- Direct IP connections
- Unusual webhook destinations
- Requests that include authorization headers
- Connections initiated immediately after cloning a repository

### 12.7 Untrusted Configuration Changes

Monitor changes to:

```text
.mcp.json
.claude/settings.json
.vscode/
.cursor/
.github/
.env*
agent configuration
tool manifests
workspace settings
```

These files should be reviewed as security-sensitive code.

## 13. Priority Remediation

### Priority 1 - Revoke and Rotate Exposed Credentials

When a live credential is discovered:

1. Revoke it.
2. Create a replacement.
3. Reduce its permissions.
4. Review its activity.
5. Identify every system in which it was stored.
6. Remove it from logs, history, and indexes where possible.

Deleting the token from a file is not enough.

### Priority 2 - Move Secrets to a Secret Manager

Suitable systems include cloud secret-management services, enterprise vaults, operating-system credential stores, hardware-backed key stores, and workload-identity platforms.

The credential should be retrieved at runtime by an authorized component.

#### Nuance Regarding `.env`

For local development, a protected and properly ignored `.env` file may be used in some environments. However:

- It must never be committed.
- It should not contain long-lived production credentials.
- File permissions must be restrictive.
- Subprocess exposure must be considered.
- A vault remains preferable for sensitive or shared systems.

### Priority 3 - Use Short-Lived Tokens

Prefer temporary cloud credentials, short-lived OAuth access tokens, one-task or one-session tokens, just-in-time credentials, and automatically rotated credentials.

A token should ideally expire when the task or session ends.

### Priority 4 - Apply Least Privilege

Scope credentials by action, repository, project, tenant, database, table, environment, user, and time window.

Bad example:

```text
Agent receives full organization administrator token.
```

Better example:

```text
Agent receives read-only access to one repository for ten minutes.
```

### Priority 5 - Keep Secrets Outside Model Context

The model should receive an abstract tool definition such as:

```text
github.search_issues
```

It should not receive:

```text
GitHub token = ghp_example_secret
```

Use middleware or a credential broker to add authorization after the tool call has been validated.

### Priority 6 - Redact Logs

Apply redaction to prompts, responses, tool arguments, HTTP headers, command-line arguments, stack traces, and environment dumps.

Redaction should occur before data leaves the local trusted boundary whenever possible.

### Priority 7 - Restrict Outbound Communications

Use destination allowlists, DNS controls, proxy enforcement, egress firewall rules, blocking of arbitrary API base URLs, and approval for new MCP endpoints.

A credential should never be sent to an endpoint selected by untrusted project content.

### Priority 8 - Validate Trust Before Loading Configuration

The application should:

1. Identify repository-level configuration.
2. Treat it as untrusted.
3. Show the effective settings.
4. Require explicit approval.
5. Validate destinations and commands.
6. Only then initialize tools and credentials.

## 14. Credential Lifecycle

Use this lifecycle for every MCP credential:

```text
Create → Store → Inject → Use → Monitor → Rotate → Revoke
```

| Stage | Required Security Practice |
| --- | --- |
| Create | Generate only the minimum required permissions. Avoid shared administrator accounts. |
| Store | Use a protected secret manager, encrypt at rest, and restrict retrieval. |
| Inject | Deliver only to the authorized runtime component. Do not place it in prompts. |
| Use | Restrict approved endpoints and operations. Bind to user, session, workload, or task. |
| Monitor | Record sanitized use and detect unusual destinations, volumes, and times. |
| Rotate | Replace credentials regularly and automatically. |
| Revoke | Immediately invalidate exposed, unused, or expired credentials. |

## 15. Memory Helper: VAULT

Use **VAULT** to remember the main controls.

| Letter | Meaning | Action |
| --- | --- | --- |
| V | Vault the secret | Store credentials in a dedicated secret-management system. |
| A | Avoid model context | Do not let the model read, repeat, summarize, or store live secrets. |
| U | Use least privilege | Limit permissions to the smallest required action and resource. |
| L | Limit lifetime and location | Use short-lived tokens and approved outbound destinations. |
| T | Track, rotate, terminate | Monitor usage, rotate credentials, and revoke them when exposure is suspected. |

```text
VAULT
Vault
Avoid context
Use least privilege
Limit lifetime
Track and terminate
```

## 16. Five Credential Questions

Before approving an MCP credential, ask:

```text
WHO is using it?
WHAT can it do?
WHERE can it be sent?
HOW LONG is it valid?
CAN its use be traced and revoked?
```

A credential is high risk when these answers are unclear.

## 17. Incident-Response Checklist

### Containment

- Disable or revoke the token.
- Stop the affected MCP server or agent session.
- Block suspicious outbound destinations.
- Remove the affected configuration.
- Isolate the development workspace when necessary.

### Investigation

Determine:

- Where the secret originated
- When it was first exposed
- Which systems stored it
- Which endpoints received it
- Which actions were performed
- Whether additional credentials were reachable
- Whether logs, memory, or vector stores retained it

### Eradication

- Remove the secret from active configuration.
- Purge or restrict affected logs.
- Remove it from Git history where appropriate.
- Rebuild compromised development environments.
- Update vulnerable software.
- Validate all MCP server definitions.

### Recovery

- Issue a new reduced-scope credential.
- Re-enable tools gradually.
- Validate logging and alerting.
- Review all activity performed with the old token.

### Lessons Learned

- Why could the model or process access the secret?
- Why was the destination allowed?
- Why did the trust control occur too late?
- Why was the credential's scope or lifetime excessive?
- Which preventive control should have stopped the attack?

## 18. Detection Engineering Ideas

### Source-Control Detections

Alert when:

- A secret-like value is committed.
- `.mcp.json` introduces a new remote server.
- Workspace configuration changes an API base URL.
- Automatic approval is enabled.
- Command execution is added to project configuration.

### Endpoint Detections

Monitor:

- AI coding tools spawning unexpected subprocesses
- Local MCP servers reading credential files
- Access to shell history or browser credential databases
- New outbound connections after opening an untrusted repository
- Secrets appearing in process arguments

### Network Detections

Alert on:

- Authorization headers sent to unapproved domains
- MCP traffic to newly registered domains
- Traffic to raw IP addresses
- Unexpected changes in API destinations
- Unusual volume immediately following agent initialization

### Identity Detections

Look for:

- Use of the same token from different countries or networks
- Service-account activity inconsistent with normal workflows
- High-volume actions
- Rapid multi-system access
- Activity after the expected session ended

## 19. Principal Security Review Checklist

| Area | Review Question |
| --- | --- |
| Credential owner | Which team owns the credential? |
| Human identity | Can the action be tied to a specific person? |
| Agent identity | Can the MCP server and agent instance be identified? |
| Scope | Which resources and actions are authorized? |
| Lifetime | How long is the credential valid? |
| Storage | Is it stored in a vault or plaintext configuration? |
| Context exposure | Can the LLM read the credential? |
| Subprocess exposure | Can unrelated local tools inherit it? |
| Logging | Are prompts, headers, and arguments redacted? |
| Endpoint restriction | Which destinations can receive the credential? |
| Tenant isolation | Can the credential cross customer boundaries? |
| Rotation | Is rotation automatic? |
| Revocation | Can it be revoked immediately? |
| Detection | Are anomalous uses monitored? |
| Approval | Are high-impact operations confirmed? |

## 20. Key Terms

| Term | Definition |
| --- | --- |
| API Key | A credential used to authenticate requests to an API. |
| OAuth Access Token | A limited credential representing authorization granted to an application. |
| Refresh Token | A longer-lived credential used to obtain new access tokens. |
| Service Account | A non-human identity used by software or workloads. |
| Secret Manager | A system designed to securely store, retrieve, rotate, and audit secrets. |
| Contextual Secret Leakage | Exposure of a secret because it entered model-visible or agent-accessible context. |
| Secret Redaction | Removal or masking of credentials before data is stored or transmitted. |
| Least Privilege | Providing only the minimum permissions needed. |
| Token Scope | The systems, resources, and actions a token is authorized to access. |
| Token Lifetime | The amount of time a token remains valid. |
| Credential Broker | Middleware that retrieves and applies credentials without exposing them to the model. |
| Egress Control | Restriction and monitoring of outbound network communication. |
| Configuration Poisoning | Manipulation of configuration so a trusted application performs attacker-controlled behavior. |
| Trust Ordering | The sequence in which content is approved, loaded, and allowed to influence execution. |
| Secret Rotation | Replacement of an existing credential with a new credential. |
| Token Revocation | Immediate invalidation of a credential. |

## 21. Technical Nuances and Corrections

| Simplified Statement | More Accurate Explanation |
| --- | --- |
| Environment variables should never contain secrets. | Environment variables are common for injection but may be inherited or exposed. High-value credentials should preferably use a secret manager or broker. |
| A secret entering context is permanently stored in the model. | It may be retained in history, logs, traces, caches, memory, or vector storage, but does not automatically enter permanent model weights. |
| MCP sessions are always long-lived and stateful. | Session duration and state depend on the host, server, transport, and implementation. |
| The token is the exfiltration channel. | The token is the credential or stolen data; the outbound request or endpoint is the channel. |
| Moving a secret to a vault solves the entire problem. | A vault improves storage, but least privilege, safe injection, destination controls, redaction, and monitoring remain necessary. |
| Removing a key from Git fixes the exposure. | The key may remain in history, forks, clones, artifacts, logs, and caches. It must be revoked and rotated. |

## 22. Technical Cheat Sheet

```text
MCP01
Token Mismanagement and Secret Exposure

CORE RISK
Agent credentials are stored, transmitted, logged, or exposed insecurely.

COMMON SECRETS
API keys
OAuth access tokens
Refresh tokens
Service credentials
Cloud keys
Database passwords
Session cookies

COMMON EXPOSURE POINTS
Repository configuration
.env files
Prompts
Model context
Logs and traces
Agent memory
Vector stores
Environment variables
MCP subprocesses
Attacker-controlled endpoints

SECURE ARCHITECTURE
User request
→ LLM selects tool
→ Policy validates request
→ Credential broker retrieves secret
→ Secret attached outside model context
→ Approved endpoint called

TOP CONTROLS
Vault storage
Short-lived tokens
Least privilege
Per-user identity
Credential broker
Log redaction
Egress allowlist
Configuration trust validation
Token rotation
Immediate revocation

DETECTION
Secret scanning
Prompt and log review
Configuration monitoring
Outbound destination monitoring
Token anomaly detection
Service-account attribution review

INCIDENT RESPONSE
Revoke
Rotate
Block destination
Review usage
Remove persistence
Validate related credentials
```

## 23. One-Minute Memory Summary

```text
The model does not need the credential.

The tool execution layer does.

Never place live secrets in prompts.

Treat project configuration as active content.

Approve configuration before loading it.

Use a vault, short-lived tokens, and least privilege.

Restrict where credentials can be sent.

Redact prompts, headers, tool arguments, and logs.

When a secret leaks, revoke it - do not just delete it.
```

## 24. Review Questions

1. **Why are tokens necessary in MCP environments?**  
   They allow MCP tools and servers to authenticate to external systems.

2. **Why should the LLM not see a credential?**  
   The credential may be repeated, logged, retrieved, or exposed through prompt injection.

3. **Where should the credential be applied?**  
   Inside trusted middleware, a credential broker, MCP server, or execution layer after authorization is validated.

4. **What is contextual secret leakage?**  
   A secret escaping because it entered model-visible or agent-accessible context.

5. **Why are long-lived tokens dangerous?**  
   A temporary exposure remains exploitable for a long period.

6. **Why are shared service accounts problematic?**  
   They create excessive blast radius and make attribution difficult.

7. **Why can a repository configuration file be dangerous?**  
   Agent tools may interpret it as active instructions controlling servers, commands, credentials, or destinations.

8. **What was the main architectural weakness in the repository scenario?**  
   Untrusted configuration became active before the user's trust decision.

9. **Is removing a key from a repository sufficient?**  
   No. The token must be revoked and rotated because copies may remain elsewhere.

10. **What is the strongest structural control?**  
    Prevent the model from seeing the secret and apply credentials through trusted middleware.

11. **What are the five VAULT controls?**  
    Vault, Avoid context, Use least privilege, Limit lifetime and location, and Track or terminate.

12. **What should be done immediately after discovering a leaked token?**  
    Revoke it, investigate its use, rotate it, and remove remaining copies.

## 25. Final Security Principle

> [!IMPORTANT]
> **A credential should be invisible to the model, temporary for the session, restricted to the required action, usable only against approved destinations, and fully attributable in audit logs.**
