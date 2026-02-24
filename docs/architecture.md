# Pachara Engine Architecture

Pachara Engine is an open-source, AI-native, multi-tenant service management core designed for the next generation of global operations.

It is built around:

- Event-driven architecture
- AI as a first-class system actor
- Governance and auditability by default
- Multi-tenant logical isolation
- Multi-language support from day one
- API-first design

This document defines the foundational architecture.

---

# Core Principles

1. Event-First
   Every meaningful state change emits a domain event.

2. AI as a Consumer, Not the Owner
   AI proposes actions.
   The domain enforces policies.

3. Multi-Tenant by Design
   Tenant isolation is mandatory at every layer.

4. Multi-Language by Default
   Language is part of the domain, not a UI concern.

5. Governance by Default
   Every AI decision is traceable and reviewable.

6. API as a Contract
   OpenAPI defines system boundaries.

---

# High-Level Architecture

Clients (UI, CLI, integrations)
        |
        v
Tenant Resolution Layer
        |
        v
API Layer (FastAPI)
        |
        v
Domain Core
        |
        v
Domain Events (Outbox Pattern)
        |
        +--> AI Orchestrator
        |
        +--> Integrations / Webhooks
        |
        +--> Metrics / Observability

External Services:
- PostgreSQL (domain + audit + outbox)
- Redis (locks + async coordination)
- Vector Database (tenant-scoped memory)
- Worker Services (AI + event consumers)

---

# Multi-Tenant Architecture

Tenant isolation is logical (Phase 1).

## Tenant Context Resolution

Each request must resolve tenant context via:

- Subdomain (tenant.pachara.io)
- Header (X-Tenant-ID)
- Auth token claims

The resolved `tenant_id` is attached to the request lifecycle.

## Data Isolation

All tenant-owned entities include:

- tenant_id (UUID)

Applied to:

- Ticket
- Message
- Queue
- SLA Policy
- AI Decision Log
- Events
- Embeddings metadata
- Users (scoped to tenant unless global role)

All database queries must enforce tenant filtering.

No cross-tenant joins allowed.

## AI Isolation

Vector memory queries must always include:

tenant_id filter

Embeddings must never mix tenants.

---

# Multi-Language Architecture

Language is part of the domain model.

## Tenant Configuration

Tenant entity includes:

- default_language
- supported_languages (array)
- timezone
- locale

## Ticket Model

Ticket includes:

- language (detected or declared)
- original_content
- optional_translations (future phase)

## Language Handling Strategy

1. Detect language on inbound message.
2. Store original language content.
3. Process AI in native language.
4. Respond in the same language.
5. Only translate when required.

System messages use i18n keys.

AI prompts are language-aware.

---

# Domain Core

Source of truth for business rules.

## Entities (Phase 1)

Ticket
- id
- tenant_id
- type (incident)
- status
- priority
- category
- service
- queue_id
- assignee_id
- language
- timestamps

Message
- id
- tenant_id
- ticket_id
- channel
- author_type
- body
- language

Queue
- tenant_id
- name
- routing_rules

SLA Policy
- tenant_id
- priority mapping
- response target
- resolution target

User
- tenant_id
- role
- locale

---

# Domain Events

All state transitions emit events.

Examples:

- TicketCreated
- MessageReceived
- TicketUpdated
- TicketTriaged
- PriorityChanged
- TicketResolved
- SLAWarning
- TicketClusterCandidateDetected

Event structure:

- event_id
- tenant_id
- event_type
- occurred_at
- actor
- entity_ref
- schema_version
- payload

Outbox pattern ensures reliable publishing.

---

# AI Orchestrator

AI components subscribe to domain events.

They never directly mutate domain state without policy validation.

## AI Responsibilities (Phase 1)

- Classification (category, type)
- Priority suggestion
- Queue suggestion
- Field extraction
- Summarization
- Similarity search
- Duplicate suggestion

## AI Design Requirements

- Deterministic interface
- Pluggable model providers
- Language-aware prompt templates
- Tenant-aware context isolation
- Confidence scoring per field

---

# AI Governance and Audit

Every AI action produces an AI Decision Log entry.

## AI Decision Log Schema

- decision_id
- tenant_id
- event_id
- ticket_id
- decision_type
- model_provider
- model_name
- model_version
- prompt_hash
- input_snapshot (sanitized)
- output_snapshot
- confidence
- suggested_actions
- policy_gate_result
- human_review_status
- human_override_diff
- created_at

## Policy Gates

Examples:

- P1 priority requires confirmation
- Auto-routing allowed if confidence > threshold
- Auto-close not allowed (Phase 1)

All high-risk actions require human validation.

---

# Vector Memory Layer

Purpose:
Tenant-scoped semantic similarity.

Stored:

- ticket embeddings
- optional knowledge base embeddings

Operations:

- upsert(tenant_id, entity_id, embedding, metadata)
- query(tenant_id, embedding, filters)

No cross-tenant memory allowed.

---

# Integration Layer

Integrations operate via events.

Inbound:
- Email → MessageReceived
- Chat → MessageReceived
- API → TicketCreated

Outbound:
- Webhooks
- Notifications
- Analytics export

All integration flows preserve tenant context.

---

# Deployment Model

Initial deployment:

- API service
- Worker service
- PostgreSQL
- Redis
- Vector database

Docker-first.
Helm later.

SaaS-ready by architecture.

---

# Security and Privacy

- Strict tenant filtering
- Sanitized AI input snapshots
- Minimal PII in embeddings
- Secret management via environment variables
- Idempotent event consumers
- Rate limiting (future phase)

---

# Phasing

Phase 1:
AI-native Incident Engine
Multi-tenant + Multi-language core
Governed AI suggestions

Phase 2:
Clustering + Major Incident
Cross-ticket correlation

Phase 3:
Analyst Assistant (RAG + structured resolution)

Phase 4:
Plugin SDK + Marketplace-ready extensions

---

# Non-Goals (Early Phases)

- Full CMDB
- Full change enablement workflows
- Deep reporting suite
- Enterprise compliance modules

---

# Positioning

Pachara is not a ticketing tool.

It is a multi-tenant, globally aware, AI-native service operations engine built for modern infrastructure.
