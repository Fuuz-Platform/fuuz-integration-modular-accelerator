# Integration Orchestrator — Technical Overview

> Converted from the technical overview deck shipped with the accelerator package.

## FUUZ INTEGRATION ORCHESTRATOR

- Technical Overview — Architecture, Data Model & Orchestration Mechanics
- FUUZ

## What It Is — and What It Is Not

- Clear boundaries: the orchestrator owns how integrations run, never what the data means.
- WHAT IT IS
- An orchestration backbone — a reusable engine that dispatches, executes, retries, and tracks integration transactions.
- A staging-table framework — every transaction persisted as an IntegrationStaging record with full request/response history.
- Configuration-driven — new endpoints onboarded via IntegrationType + IntegrationConfiguration records — no core changes.
- A reliability layer — mutex locking, concurrency caps, auto-retry rules, deduplication, and attempt limits built in.
- An observability layer — per-record message audit trail, transaction screens, and a live dashboard.
- Installed once per environment — shared by every integration in the tenant.
- WHAT IT IS NOT
- Not a pre-built connector library — the external-system calls (integration data flows) are built per integration.
- Not a mapping/transformation tool — payload mapping lives in your request, integration, and response flows.
- Not an ESB or message broker — it orchestrates Fuuz-centric integrations, not enterprise-wide messaging.
- Not a real-time streaming pipeline — transactional, record-based processing — event-triggered plus scheduled.
- Not self-contained business logic — it deliberately owns how integrations run, never what the data means.
- Not monitoring for the target system — it tracks the exchange, not what the external system does afterwards.

## Core Design: The Staging Table Pattern

- Every transaction is persisted as an IntegrationStaging record before processing — queue entry, audit log, and file container in one.
- Enqueue, never execute inline
- Request flows only create a staging record (statusId = New). The orchestrator owns everything after that.
- The record carries the transaction
- request / response / metadata JSON payloads, attemptCount, internal + external record references, an indexed dedup key (outboundRecordReference), and an optional fileId for EDI/CSV payloads.
- Full per-record message log
- Every transition writes an IntegrationMessage child (Failure / Warning / Informational / Success / Detail) — a timestamped audit trail per transaction.
- IntegrationStaging
- number IS000000000042
- statusId New → Inprocess → …
- request { JSON payload }
- response { external reply }
- metadata { caller context }
- attemptCount 2
- outboundRef tenant_config_rec
- fileId optional File FK
- messages[] IntegrationMessage

## Data Model: Six Reference Models

- Configuration is split from execution: types define connectivity, configurations define endpoints, staging records carry transactions.
- IntegrationType
- Connection + capacity profile: connectionId, concurrencyLimit, maxRecordsLimit, requestInterval, app scope (MES/WMS/CMMS/QMS), tenantId.
- IntegrationConfiguration
- One per endpoint: direction, integrationDataFlowId, responseDataFlowId, autoRetryResponses, maxRetryAttemps, parameters JSON. Name becomes the record ID.
- IntegrationStaging
- The transaction record: payloads, status, attempt count, references, optional file. Auto-numbered (IS prefix).
- StagingTableStatus
- 10 seeded lifecycle states. Flags (new / inProcess / processed / success / failure / autoRetry) drive orchestrator queries.
- IntegrationMessage
- Per-record audit events, auto-numbered (IM prefix), classified by type, with structured JSON detail.
- IntegrationMessageType
- 5 seeded severities: Failure, Warning, Informational, Success, Detail — with UI colors and usage flags.

## Orchestration Engine: The Packaged Flows

- Event-driven coordination over four topics — schedule, data-change events, and topic messages all converge on the same pipeline.
- Integration: Main Handler 15 nodes
Dispatcher. Triggers: Schedule, IntegrationStaging create event, topic integration.main.handler, direct request. Queries new/autoRetry statuses, groups by IntegrationType, pre-filters against concurrencyLimit, bulk-publishes each record to integration.main.request.handler. Serialized by a global mutex (TTL 30 s).
- Integration: Main Flow Request Handler 38 nodes
Per-record processor. Per-record mutex → guard branches (missing/invalid config, max retries, concurrency, status guard) → sets Inprocess + increments attemptCount → $executeFlow(integration flow) in try/catch → evaluates response → final status + messages → $executeFlow(response handler).
- Int_Outbound App Tenant Request Handler 15 nodes
API entry point. Validates active config, dedup-checks outboundRecordReference (tenantId_configId_recordId), creates staging record in Hold, publishes to integration.outbound.record.processor, returns { recordId }.
- Flow to receive the outbound request from the app tenant

## Transaction Lifecycle & Status Machine

- Status flags — not hardcoded lists — drive eligibility: the dispatcher picks up any status with new=true or autoRetry=true.
- New
- Inprocess
- Success
- Processed
- Queued
- autoRetry = true — re-dispatched next cycle; retried until maxRetryAttemps
- Integration Error
- external system returned an error
- Processing Error
- unhandled exception (try/catch)
- Cannot Process
- invalid config or max retries reached
- Hold
- parked — invisible to the dispatcher
- Response evaluation: RESPONSE.success → Success · match in autoRetryResponses (case-insensitive, error code or message) → Queued · success=false → Integration Error · anything else → Cannot Process

## Concurrency, Retry & Safety Controls

- Two enforcement points: the dispatcher pre-filters batches; the request handler guards each record.
- Distributed mutexes
- Global dispatcher lock ("mainHandlerInprocess") plus a per-record lock keyed on the staging ID — both TTL 30 s. No double-dispatch, no double-processing.
- concurrencyLimit
- Per IntegrationType. Dispatcher sends only (limit − inProcess) records per cycle; the handler re-checks live counts before executing. 0 = unlimited.
- autoRetryResponses + maxRetryAttemps
- Configured per endpoint as JSON. Matching responses re-queue automatically; the attempt cap converts repeat failures to Cannot Process with a logged reason.
- Deduplication
- Indexed outboundRecordReference (tenantId_configId_recordId) rejects duplicate API submissions before a record is ever created.
- Batch & rate limits
- maxRecordsLimit (type) and maxRecords (endpoint) cap batch size; requestInterval paces calls to rate-limited APIs in milliseconds.
- Status guard
- Only records in New, Queued, or a failed state are processed — anything already in flight exits immediately.

## Building an Integration: Flows & Contracts

- Your logic plugs in via $executeFlow() — the orchestrator handles dispatch, locking, retries, status, and logging.
- 1
- Request Flow
- Your trigger (data change, schedule, screen action). Prepares the payload and creates the staging record with statusId New. Never calls the external system itself.
- 2
- Integration Data Flow
- Performs the external call (e.g. $integrate()) and writes the reply to the staging record’s response field.
- 3
- Response Handler Flow
- Invoked after the final status is set — success and failure. Applies the result to application data and logs messages.
- Context contracts
- // integration data flow receives
- { id, request,
- connection, // Connection name
- parameters } // config JSON
- // response handler receives
- { recordId, timeStamp }
- // outbound API
- POST { tenantId, configId,
- recordId, ... }
- → { recordId }

## Observability: Screens & Dashboard

- Integration Dashboard — six live sections: KPIs, topology, throughput, activity, connectors, error queue.
- Integration Setup — type & endpoint configuration.
- Integration Transactions — drill into payloads & message log.
- Every status transition is also written to IntegrationMessage — observability is in the data, not just the UI.

## Complete Visibility

- Operations teams see every transaction — live dashboard for the big picture, drill-down for the detail.
- KPI overview
- Daily volumes, success rates, and failures at a glance.
- Integration topology
- Hub-and-spoke map of every connected system.
- Throughput trends
- Message flow over time to spot bottlenecks early.
- Live activity
- Transactions streaming through, as they happen.
- Error queue
- Failed records front and center — nothing hides.
- Full audit trail
- Per-transaction message log: every step, timestamped.

## Why This Architecture Holds Up

- Durable by default
- Transactions survive restarts, outages, and slow endpoints — state lives in the database, not in memory.
- Near-real-time + scheduled
- Data-change events dispatch immediately; the schedule sweeps anything missed. Both paths converge on one pipeline.
- Configuration over code
- New endpoint = 1 IntegrationType + 1 IntegrationConfiguration + your three flows. Zero changes to the core package.
- Safe under load
- Mutexes, concurrency caps, batch limits, and pacing protect both Fuuz and rate-limited external APIs.
- Forensics built in
- Request, response, attempts, and a typed message log per record — root-cause analysis without log-diving.
- No package dependencies
- Ships self-contained; relies only on core platform capabilities (GraphQL, flows, topics, sequences, mutexes).

## Why Teams Choose the Orchestrator

- Nothing gets lost
- Every transaction is persisted, tracked, and recoverable — reliability by architecture, not by luck.
- Faster onboarding
- A new integration is configuration, not a project: one type, one configuration, and your business logic.
- Build once, reuse everywhere
- The orchestration engine is shared. Retry, concurrency, and audit come free with every new endpoint.
- Failures stay contained
- One failing endpoint never takes down the rest. Errors are isolated, flagged, and recoverable.
- Answers in minutes
- Complete request/response history ends the finger-pointing between system owners.
- Grows with you
- Inbound, outbound, files, APIs, Fuuz apps, and external systems — one consistent pattern for all of it.

## From First Endpoint to Full Landscape

- The orchestration engine is installed once per environment. Every integration after that is configuration plus three flows.
- 1
- Connection + IntegrationType
- Register the endpoint, set concurrency, batch, and pacing limits.
- 2
- Integration Configuration
- Wire direction, flows, retry rules (autoRetryResponses, maxRetryAttemps), and parameters.
- 3
- Request / Integration / Response flows
- Your business logic, plugged in via $executeFlow() contracts.
- 4
- Operate
- Monitor on the dashboard; tune limits without touching code.
- Reliable integration as a platform capability — measured, governed, and auditable.
