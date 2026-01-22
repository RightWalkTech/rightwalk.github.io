# Chatur AI - System Architecture

**System Architecture** for the WhatsApp Conversational Access Layer for NAPS (National Apprenticeship Promotion Scheme)

> This document describes the **NAPS-specific implementation of Citizen AI**, currently referred to as **Chatur AI** (working name), and should not be read as the full Citizen AI vision.

**Status:** Live technical architecture (near-term, reality-first)  
**Shelf life:** Evolving document – updated as system changes  
**Related vision document:** *RightWalk’s Citizen AI* (long-horizon, product & policy narrative)

**Positioning & Relationship to Citizen AI**

This document intentionally differs in tone and scope from *Citizen AI*:

- **Citizen AI** is a *vision-first*, long-shelf-life narrative covering past, present, and long-term futures.
- **Chatur AI** is a *reality-first*, operational snapshot of what is deployed today and what is planned in the near term.

Both documents are correct for their own domain, and are complementary:
- Citizen AI explains *why* and *where* the system is going.
- Chatur AI (this docuemnt) explains *what is running* and *how it is engineered* right now or for near-term

---

## 1. Objectives


A WhatsApp-first conversational system that enables **opportunity discovery, assisted registration, and apprenticeship application** on NAPS, using an AI-driven conversational layer with deterministic tool execution, human handoff, strong validation, and a secure, auditable AWS deployment.

> Opportunity discovery is the primary value surface; workflows are executed only when intent is clear.

### 1.1 What exists today
- Exploration-first UX is live: users can discover opportunities without prior login.
- Registration and application are triggered *only* when a user chooses to act.
- AI acts as a guided facilitator, not just a form-filling bot.

### 1.2 Design principles
- **Correctness over autonomy**
- **Auditability over speed**
- **Human override at every stage**
- **Field-realistic failure handling**

---

## 2. System Overview

### 2.1 User & HITL Layer
- **WhatsApp Cloud API** → sole user interface.
- **Chatwoot (self-hosted)**:
  - Conversation ownership (AI ↔ human).
  - Metadata, inbox routing, escalation.
  - Acts as the *primary conversational state holder*.

### 2.2 Backend (*chatwhat*, golang, private EC2)
  - Receives Chatwoot webhooks.
  - Decides whether AI or human owns the turn.
  - Performs context assembly.
  - Mediates *all* side effects.

### 2.3 AI Decision Engine (Agentor)
  - Currently single-agent, single-model.
  - self developed tool written in golang for stability, performance, and concurrency  
  - Responsible for intent understanding, planning, and response generation.
  - No direct database or external system access.

  **Agent Design & Evolution**

    * Current
      - Single agent.
      - Single model.
      - End-to-end reasoning loop.
  
    * Planned (if needed)
      - Hierarchical or classifier-first agents.
      - Validators for sensitive actions.
      - Cost-optimized executor models.

  This evolution is **optional**, not assumed.

### 2.4. Tool Control Plane (FastMCP, MCP server)

  The **MCP server** is the system’s safety and determinism boundary.

  Responsibilities:
  - Strongly typed, versioned tools.
  - Validation and schema enforcement.
  - Connectivity to NAPS APIs and internal datasets.
  - Full audit logging.

  LLMs decide *what* to do; MCP defines *how* it happens.

### 2.5. Opportunity Intelligence Plane (Part of MCP server, with PGvector)

  While NAPS remains the system of record, **Opportunity Intelligence** is a first-class internal plane:

  Components:
  - Normalized Opportunity Database (Postgres).
  - Semantic indexing for filters like `Sector/Industry`, `Course Name`, `Location` etc. (pgvector).
  - Deterministic ranking and filtering logic.
  - MCP tools:
    - `search_opportunities`
    - (planned) `recommend_opportunities`

  This plane:
  - Eliminates user-facing NAPS latency.
  - Enables exploration-first UX.
  - Provides a foundation for future recommendation and guidance systems.

### 2.6. State & Workflow Management

  State is distributed across:
  - **Chatwoot conversations** – conversational and ownership state.
  - **NAPS portal** – authoritative workflow state.
  - **Postgres DB** – durable user mappings, audit data, opportunity data.
  - **Chatwhat** – Go backend that integrates Chatwoot + LLM + MCP + Postgres

---

## 3. Evaluation, Observability & Safety

Evaluation is treated as **both observability and infrastructure**.

### 3.1 Observability
- **Infra & app metrics:** Logs and Metrices in SigNoz.
- **LLM telemetry:** Traces in Signoz.

### 3.2 Evaluation as infrastructure
- Self-developed evaluation system (Assertyr) for agent correctness, safety, and compliance.
- Replay of real conversations.
- Tool correctness checks.
- Consent and safety assertions.
- Regression detection.

Evaluation outputs influence:
- Prompt changes.
- Model selection.
- Tool schema evolution.
- Release gating.

### 3.3 Assertyr (written in golang) – Evaluation & Safety Infrastructure

  **Assertyr** is RightWalk’s **agent evaluation and safety infrastructure**, designed specifically for non-deterministic, agentic conversational systems operating in public-sector workflows.

  Unlike traditional observability, Assertyr focuses on **system correctness, policy compliance, and regression immunity** across real conversations.

  #### Role in the architecture

  Assertyr operates **out of band** from the live request path, consuming artifacts produced by:

  * Chatwoot conversations,
  * Backend action logs,
  * MCP tool invocation records,
  * Signoz LLM traces.

  It does **not** participate in runtime decision-making, but instead governs **what is allowed to ship and scale**.

  #### Core capabilities

  * **Conversation replay**

    * Replays real Chatwoot conversations deterministically.
    * Reconstructs agent context, tool calls, and outcomes.

  * **Assertion-based evaluation**

    * Language consistency (multilingual correctness).
    * Tool correctness and schema adherence.
    * Consent and policy compliance.
    * Safety boundary violations.
    * High-risk action gating (e.g., irreversible submissions, planned).

  * **Regression detection**

    * Detects behavioral drift across:

      * prompt changes,
      * model swaps,
      * MCP schema evolution,
      * backend logic changes.

  * **Improvement signals**

    * Produces structured failure reports.
    * Feeds actionable insights into:

      * prompt iteration,
      * tool redesign,
      * escalation rules,
      * evaluation coverage expansion.

  #### Why Assertyr is infrastructure (not just observability)

  * Traditional metrics answer *“what happened?”*
  * Assertyr answers *“is the system still safe, correct, and policy-aligned?”*

  In practice, Assertyr acts as:

  * a **quality gate** before scale-up,
  * a **safety net** against silent regressions,
  * and a **compliance enabler** for public deployment.

  #### Positioning

  Assertyr is treated as a **first-class platform component**, alongside MCP and the backend, ensuring that agent behavior remains:

  * auditable,
  * reproducible,
  * and governable over time.

### 3.4. Infrastructure (AWS)

- Single VPC, private-by-default.
- Public ingress via Nginx / ALB.
- Private subnets for Chatwoot, Backend, MCP.
- NAT-based egress.
- Single-AZ today; Multi-AZ/region and ASG planned.

Stateless services enable horizontal scaling.

![AWS Architectural diagram for RightWalk NAPS Whatsapp BOT](./architectural_diagram.jpg)


---

## 4. Way forward and decisions

### 4.1 Employer Bot

The **Employer WhatsApp Bot** will be:
- A separate WhatsApp number.
- A separate backend.
- A separate MCP server.

While patterns and stack will be similar, it is **not** a shared runtime with the citizen system.

This document intentionally excludes employer architecture. A similar approach can be taken for creating a WA conversation layer for other portals like RTE.

### 4.2. Non-goals (Current)
- No multi-region deployment.
- No shared employer-citizen backend.
- No assumption that long-horizon features are imminent.

### 4.3. Open Decisions
- Cache reintroduction strategy.
- Recommendation engine boundaries.
- Token budget enforcement.

---

## Closing Note

This architecture encodes *what is real today*, while remaining compatible with the longer-term vision articulated in *Citizen AI*. It is intentionally conservative, auditable, and field-aligned — designed to evolve without rewriting core assumptions.
