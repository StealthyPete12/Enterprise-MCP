<div align="center">

# Gatehouse

**A secure Model Context Protocol (MCP) server in C# that lets AI assistants work inside a company's systems — authenticated, least-privilege and fully audited.**

![.NET 10 LTS](https://img.shields.io/badge/.NET-10%20LTS-512BD4)
![MCP spec 2026-07-28](https://img.shields.io/badge/MCP%20spec-2026--07--28-0D9488)
![MCP C# SDK v2](https://img.shields.io/badge/MCP%20C%23%20SDK-v2-0F766E)
[![Roadmap: Phase 0](https://img.shields.io/badge/roadmap-Phase%200-F59E0B)](docs/roadmap.md)

</div>

## What it is

Gatehouse sits between AI assistants — Claude Code, VS Code or any other MCP client — and the internal systems of Contoso, a fictional company: a SQL Server–backed **IT service desk** and an internal **Directory API** of employees and their devices. It exposes them as a small set of task-shaped MCP tools, so an assistant can search and triage tickets, look up who raised them and what hardware they use, and — when the signed-in person is allowed to — update them.

The tools are the easy part. The point of the project is the gate in front of them:

- every call is **authenticated** with OAuth 2.1, as the person using the assistant;
- every call is **authorized three ways** — what the user granted the AI client, what their role allows, and which records they may touch;
- every write is **confirmed by a human**, who sees the exact change first;
- every call, including the denied ones, lands in an **append-only, tamper-evident audit trail**.

> Contoso and both of its systems are simulated in this repository, but Gatehouse treats them as systems it doesn't own: it can't change their schemas or their APIs. That is the realistic constraint of connecting AI to someone else's stack.

## What the project focuses on

| Focus | What that means here |
|---|---|
| **Connecting AI to systems I don't own** | Database-first EF Core over a legacy schema, an anti-corruption layer, and a second connector over REST — added without touching the core. |
| **Identity, done the MCP way** | Gatehouse is an OAuth 2.1 resource server: protected-resource metadata, audience-bound tokens and step-up consent, with Keycloak as the identity provider. |
| **Read vs write, with least privilege** | Read-only by default. A write needs an explicit grant, the right role, rights to that specific record and a human's confirmation. |
| **Accountability** | Fail-closed auditing of every tool call into a SQL Server ledger table, linked to OpenTelemetry traces. |

## Why I'm building it

Connecting AI to a client's existing systems is the core of forward-deployed engineering. MCP has become the standard way to do it, yet few C# developers have shipped an MCP server — and fewer still one that would survive a security review. Gatehouse is my attempt at the production-shaped version: the authentication, permissions and audit trail a real client asks about first, built on the official C# SDK, ASP.NET Core and EF Core.

## Architecture

<p align="center">
  <img src="docs/images/architecture-overview.svg" width="100%" alt="Gatehouse architecture. An AI assistant on the user's machine signs in with Keycloak using OAuth 2.1 and PKCE, then calls Gatehouse over HTTPS with a bearer token. Inside Gatehouse, requests pass transport and authentication, guardrail filters, connector tools, application services and data access. Gatehouse reaches Contoso's Helpdesk database and Directory API with its own least-privilege credentials, writes to an append-only audit database and sends telemetry to the Aspire dashboard.">
</p>

1. **Sign in.** The assistant signs the user in with Keycloak (OAuth 2.1 + PKCE) and receives a token bound to Gatehouse, carrying read scopes only.
2. **Call.** Every MCP request carries that token. Gatehouse checks the origin, the rate limit and the token itself, and challenges the client to step up when a write needs more than read access.
3. **Guard.** Before any tool runs, an audit record is opened; then the caller's role, granted scopes, the read-only kill switch and write quotas are checked.
4. **Act.** The connector applies data rules, asks the human to confirm any write, and reaches Contoso's systems with Gatehouse's own least-privilege credentials. The user's token never leaves Gatehouse.

The full design — trust boundaries, data model, threat model and decision records — is in **[docs/architecture.md](docs/architecture.md)**.

## Anatomy of a tool call

<p align="center">
  <img src="docs/images/tool-call-pipeline.svg" width="100%" alt="The ten ordered steps of a tool call: edge checks, authentication and a scope check in ASP.NET Core; opening the audit record, role authorization and guardrails in the MCP SDK's filters; validation with data scoping, human confirmation for writes and execution in the connector; then closing the audit record and responding. Each step shows the denial it can produce, and which denials are audited.">
</p>

The order is deliberate. The audit record is opened *before* authorization, so denied calls are recorded too — and it is opened fail-closed: if the record can't be written, nothing runs.

## Security model at a glance

| Layer | Question it answers | How |
|---|---|---|
| Authentication | Who is calling, and through which client? | A Keycloak-issued JWT; signature, issuer, expiry and **audience = Gatehouse** checked on every request |
| Scope | What has the user let this AI client do? | `helpdesk:read` and `directory:read` at sign-in; `helpdesk:write` only through a step-up `403 insufficient_scope` |
| Role | What is this person allowed to do? | `requester`, `agent` and `auditor` roles decide which tools are visible and callable |
| Data | Which records? | Requesters see only their own tickets; agents change only tickets in their queues; internal notes are hidden from requesters |
| Confirmation | Does the human agree with *this* change? | An MCP elicitation showing the exact change, backed by a sealed, user-bound, single-use, two-minute confirmation |
| Accountability | What happened, and who did it? | An append-only ledger table recording every call — denials included — with field-level changes |
| Blast radius | What if the model is manipulated? | No bulk or delete tools, write quotas, a read-only kill switch, and user-written text clearly labelled as untrusted |

## Tool catalogue

| Tool | Access | Who | What it does |
|---|---|---|---|
| `helpdesk_search_tickets` | read | requester, agent | Find tickets by text, status, priority, queue, assignee or date |
| `helpdesk_get_ticket` | read | requester, agent | A ticket with its comments — internal notes for agents only |
| `helpdesk_queue_summary` | read | agent | Queue health: open counts, SLA breaches, oldest unassigned |
| `helpdesk_create_ticket` | **write** | requester, agent | Raise a ticket — for yourself, or for an employee if you're an agent |
| `helpdesk_add_comment` | **write** | requester, agent | A public reply, or an internal note for agents |
| `helpdesk_assign_ticket` | **write** | agent | Assign to an agent who works the ticket's queue |
| `helpdesk_set_status` | **write** | agent | Move through allowed status transitions; resolving needs a note |
| `helpdesk_update_triage` | **write** | agent | Change priority, category or queue |
| `directory_find_employee` | read | agent | Department, location and manager for an employee |
| `directory_get_assets` | read | agent | An employee's devices, serial numbers and warranty status |

Plus two prompts, `triage_ticket` and `queue_briefing`. No tool deletes anything, and there is deliberately no "run SQL" tool.

## Tech stack

| Area | Choice |
|---|---|
| Runtime | .NET 10 (LTS), C# |
| MCP | The official C# SDK — `ModelContextProtocol.AspNetCore` v2.x — targeting MCP spec `2026-07-28`, with `2025-11-25` clients still supported |
| Web | ASP.NET Core — JWT bearer authentication, rate limiting, Data Protection |
| Data | EF Core 10 on SQL Server — database-first for the legacy helpdesk, code-first for the audit store (ledger tables) |
| Identity | Keycloak — OAuth 2.1 and OpenID Connect, Client ID Metadata Documents and Dynamic Client Registration |
| Local orchestration | Aspire — SQL Server, Keycloak and every project started together |
| Observability | OpenTelemetry — traces, metrics and logs in the Aspire dashboard |
| Testing | xUnit, Testcontainers, the SDK's own `McpClient`, MCP Inspector |

## Key design decisions

- **Remote over Streamable HTTP, never stdio** — per-user OAuth only exists on HTTP.
- **A resource server only** — Keycloak issues tokens; Gatehouse validates them and never becomes an auth server.
- **Read-only by default** — write access is a step-up the user consents to, and every write is confirmed.
- **Curated tools, not raw SQL** — better model accuracy, a smaller attack surface and meaningful audit records.
- **No token passthrough** — downstream systems see Gatehouse's own identity, never the user's token.
- **Audit wraps authorization** — a denied call is still a recorded call.

All thirteen decisions and their trade-offs are in [architecture §14](docs/architecture.md#14-architecture-decisions).

---

<sub>A personal learning and portfolio project. Contoso is fictional and all data is generated.</sub>
