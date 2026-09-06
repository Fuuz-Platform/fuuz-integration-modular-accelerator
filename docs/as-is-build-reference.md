# Integration Orchestrator — AS IS Build Reference

> Converted from the v1.1.0 Word document. Full data model and data flow reference.

### 1. Executive Summary

### 2. Package Overview

### 3. Architecture Overview

### 3.1 Layered Architecture

### 3.2 Data Flow Overview

### 3.3 Topics & Messaging

### 4. Core Components — Data Models

### 4.1 IntegrationStaging

### 4.2 StagingTableStatus

### 4.3 IntegrationType

### 4.4 IntegrationConfiguration

### 4.5 IntegrationMessage

### 4.6 IntegrationMessageType

### 4.7 StagingTableStatus Flag Matrix

### 4.8 IntegrationMessageType Matrix

### 5. Core Components — Data Flows

### 5.1 Integration: Main Handler

### 5.2 Integration: Main Flow Request Handler

### 5.3 Int_Outbound App Tenant Request Handler

### 5.4 Integration Orchestrator Dashboard

### 6. Core Components — Screens

### 7. Status & Message Lifecycle

### 7.1 Staging Record Status Lifecycle

### 7.2 Integration Message Types

### 8. How to Apply — Implementation Guide

### 8.1 Step 1: Define an IntegrationType

### 8.2 Step 2: Create an IntegrationConfiguration

### 8.3 Step 3: Build the Request Flow

### 8.4 Step 4: Build the Integration Data Flow

### 8.5 Step 5: Build the Response Handler Flow

### 8.6 Step 6: Outbound API Path (Optional)

### 8.7 Concurrency & Retry Configuration

### 9. Reference Data

### 9.1 Sequences

### 9.2 Integration Data Flow Context Contract

### 9.3 Response Handler Context Contract

### 9.4 Outbound API Contract

### 10. Package Dependencies & Prerequisites

### 11. Glossary

### 1. Executive Summary

This document is the AS IS build description of the Fuuz Integration Orchestrator package, version 0.0.9. It is the authoritative technical reference for integration architects, platform engineers, and developers who need to understand, install, configure, or extend the integration orchestration capabilities of the Fuuz Industrial Intelligence Platform.

The Integration Orchestrator is the packaged evolution of the earlier Integration Tenant Core Package (v0.0.1). It provides a formal typing system for integration endpoints (IntegrationType), per-endpoint configuration (IntegrationConfiguration), concurrency and retry controls, per-transaction message logging, file attachment support on staging records, and an outbound API handler flow that lets application tenants and external systems submit integration requests through a structured request interface. Integration Dashboard: a menu-accessible screen backed by a dedicated dashboard data flow that renders live KPI, topology, throughput, activity, connector, and error-queue views.

The core design principle is the Staging Table Pattern: every integration transaction is persisted as an IntegrationStaging record before processing. This decoupling provides reliability, retry-ability, observability, and isolation of failures without requiring changes to the core package.

### 2. Package Overview

| Attribute | Value |
|---|---|
| Package Name | Integration Orchestrator |
| Package ID | integrationOrchestrator |
| Package Version | 0.0.9 |
| Platform Version | 2026.6.1 |
| Spec Version | 2.0.0 |
| External Dependencies | None (dependencies: {} in manifest) |

The package ships the following artifacts:

| Artifact Type | Items |
|---|---|
| Data Models (6) | IntegrationConfiguration, IntegrationMessage, IntegrationMessageType, IntegrationStaging, IntegrationType, StagingTableStatus |
| Data Flows (4) | Integration: Main Handler; Integration: Main Flow Request Handler; Int_Outbound App Tenant Request Handler; Integration Orchestrator Dashboard |
| Screens (5) | Integration Setup; Integration Transactions; Manage Integration Configuration; Manage Integration Type; Integration Dashboard |
| Topics (4) | integration.main.handler; integration.main.request.handler; integration.outbound.record.processor; integration.outbound.request.handler |
| Sequences (2) | Integration Staging Number (prefix IS); Integration Message Number (prefix IM) |
| Reference Data | 10 StagingTableStatus records; 5 IntegrationMessageType records |
| Module Group / Module | System / Integration |

### 3. Architecture Overview

### 3.1 Layered Architecture

The orchestration framework is structured across three layers with a clear separation of responsibility:

- Layer 1 — Request Layer: application-specific or external-system request flows create records in IntegrationStaging, placing integration tasks in the queue with status New (or Hold for the outbound API path).

- Layer 2 — Orchestration Layer: the core package flows (Main Handler and Main Flow Request Handler) poll for eligible records, enforce concurrency and retry limits, execute the integration, and manage the record lifecycle.

- Layer 3 — Response Layer: per-integration response handler flows are called after execution completes. They transform the external system response and write results back to application data models.

A fourth entry point, the Int_Outbound App Tenant Request Handler, exposes a structured request API through which application tenants submit outbound integration requests without direct access to the orchestration tenant GraphQL API.

### 3.2 Data Flow Overview

| Data Flow | Role |
|---|---|
| Integration: Main Handler | Orchestration entry point. Triggered by schedule, IntegrationStaging create events, topic message, or direct request. Queries eligible (new / auto-retry) staging records, groups them by IntegrationType, pre-filters against the concurrency limit, and dispatches each record to the request handler topic. |
| Integration: Main Flow Request Handler | Per-record processor. Validates configuration, acquires a per-record mutex, enforces concurrencyLimit and maxRetryAttemps, executes the integration data flow, evaluates the response, updates staging status, and calls the response handler. |
| Int_Outbound App Tenant Request Handler | API entry point for application tenants. Accepts a structured request with tenantId, configId, and recordId, validates the IntegrationConfiguration, deduplicates, creates the staging record in Hold status, and publishes to integration.outbound.record.processor. Returns the new staging record ID to the caller. |
| Integration Orchestrator Dashboard | Read-only presentation flow that feeds the Integration Dashboard screen. Called on demand via $executeFlow() with a section parameter; each section queries staging/message data and renders an HTML view returned to the screen. |

### 3.3 Topics & Messaging

Four topics coordinate event-driven messaging between flows. All four ship in the package as reference data:

| Topic Name | Usage |
|---|---|
| integration.main.handler | Subscribed to by the Main Handler. Other flows can kick off the orchestration cycle by publishing to this topic. |
| integration.main.request.handler | Subscribed to by the Main Flow Request Handler. The Main Handler bulk-publishes each eligible staging record to this topic. |
| integration.outbound.record.processor | Published to by the Int_Outbound App Tenant Request Handler after creating a staging record. No flow inside this package subscribes to it — a tenant-built outbound record processor flow must subscribe, prepare the request payload, and move the record from Hold to New. |
| integration.outbound.request.handler | Reserved for triggering the outbound request handler path from application tenants / external systems. |

IMPORTANT  The outbound API path is intentionally open-ended: the package creates the staging record and raises the integration.outbound.record.processor event, but the record processor flow that formats the payload and releases the record (Hold to New) is implemented per tenant and is not part of this package.

### 4. Core Components — Data Models

All six models are of kind Reference and belong to module Integration (module group System). In v0.0.5 every model carries embedded field descriptions; the tables below reproduce the field definitions as shipped.

### 4.1 IntegrationStaging

The central staging table. Every integration transaction — whether initiated by an application flow, a schedule, or the outbound API — is represented as a record here. The staging record serves simultaneously as the integration queue entry, the execution audit log, and the file attachment container.

Model Version: 8  |  Kind: Reference  |  Model ID: integrationStaging

| Field | Type | Description |
|---|---|---|
| id | ID! | The primary unique ID of the record. Every model should have an id field. |
| attemptCount | Int | Number of processing attempts made for this staging record, used for retry logic. |
| appPostError | Boolean | Whether an error occurred while posting this record to the target applications. |
| number | String! | Auto-sequenced staging number using sequence integrationStagingNumber (format IS000000000001). |
| internalRecordReference | String | Reference to the internal (Fuuz) record that produced this staging entry, such as a record number. |
| externalRecordReference | String | Reference to the corresponding record in the external system. |
| outboundRecordReference | String | Reference identifier of the outbound transaction used to verify uniqueness and avoid duplicate submissions. Built by the outbound API handler as tenantId_configId_recordId. |
| appPostedAt | DateTime | Timestamp when the record was successfully posted to the target application. |
| processedAt | DateTime | Timestamp when the integration with the external system completed. |
| metadata | JSON | JSON metadata describing the staging record (context captured during processing). |
| request | JSON | JSON request payload sent to / received for processing. |
| response | JSON | JSON response payload returned from processing. |
| integrationConfigId | ID! | FK to the IntegrationConfiguration that governs this record. |
| integrationConfig | IntegrationConfiguration | Resolved parent IntegrationConfiguration. |
| statusId | ID | FK to StagingTableStatus. Tracks the current lifecycle position of the record. |
| status | StagingTableStatus | Resolved StagingTableStatus. |
| integrationMessages | [IntegrationMessage!]! | Child IntegrationMessage records forming the per-record execution log. |
| fileId | ID | Optional FK to a Fuuz File record, enabling file attachments (e.g. EDI files, CSVs, PDFs) to be carried with the staging record. |
| file | File | Resolved File object linked via fileId. |

### 4.2 StagingTableStatus

Reference model defining all lifecycle states of a staging record. The status flags drive the Main Handler query logic — only statuses with new=true or autoRetry=true are picked up for processing.

Model Version: 5  |  Kind: Reference  |  Model ID: stagingTableStatus

| Field | Type | Description |
|---|---|---|
| id | ID! | The primary unique ID of the record. Every model should have an id field. |
| name | String | Display name of the status, used as the record's label. |
| new | Boolean | Flag marking this status as the initial 'new / not yet processed' state. |
| inProcess | Boolean | Flag marking records currently being processed. |
| processed | Boolean | Flag marking records that have finished processing, regardless of outcome. |
| success | Boolean | Flag marking successful integration completion. |
| failure | Boolean | Flag marking failed processing. |
| autoRetry | Boolean | Flag indicating records in this status should be automatically retried. |
| integrationStagings | [IntegrationStaging!]! | Back-relation to staging records currently in this status. |
| color | Color | Display color used to visually represent the status in the UI. |

### 4.3 IntegrationType

Defines the connectivity and capacity profile for a class of integration endpoints: the external system connection, the applicable Fuuz application modules, ERP references, and capacity limits shared by all IntegrationConfiguration records of this type.

Model Version: 6  |  Kind: Reference  |  Model ID: integrationType

| Field | Type | Description |
|---|---|---|
| id | ID! | The primary unique ID of the record. Every model should have an id field. |
| integrationName | String! | Descriptive name for this integration type (e.g. ERP Outbound Orders). |
| concurrencyLimit | Int | Maximum number of simultaneous requests allowed (records In Process at the same time). 0 = unlimited. |
| maxRecordsLimit | Int | Maximum number of records processed in one request/orchestration cycle. |
| requestInterval | Int | Time interval between requests in milliseconds, where applicable (rate-limited external APIs). |
| fuuzApps | Boolean! | Flag to differentiate if the integration is to a FUUZ App |
| connectionId | ID! | Connection used to reach the application tenants. |
| connection | Connection | Resolved Connection object. |
| appMes | Boolean | Whether the MES application is enabled for this tenant configuration. |
| appWms | Boolean | Whether the WMS application is enabled for this tenant configuration. |
| appCmms | Boolean | Whether the CMMS application is enabled for this tenant configuration. |
| appQms | Boolean | Whether the QMS application is enabled for this tenant configuration. |
| siteId | String | External ERP facility/location/plant/site identifier mapped to this tenant configuration. |
| enterpriseId | String | External ERP subsidiary/entity/company identifier mapped to this tenant configuration. |
| additionalDefinitions | JSON | Additional definitions that can be used as defaults in integration if required |
| tenantId | String | Identifier of the tenant this configuration belongs to. This should be exact tenantId of the tenant |
| integrationConfigurations | [IntegrationConfiguration!]! | Back-relation to all IntegrationConfiguration records of this type. |

### 4.4 IntegrationConfiguration

Defines a single inbound or outbound integration endpoint, including the data flows used to process the payload and the response, and incremental-extraction state. Each integration requires exactly one IntegrationConfiguration record; its name becomes the record ID via the create trigger.

Model Version: 9  |  Kind: Reference  |  Model ID: integrationConfiguration

| Field | Type | Description |
|---|---|---|
| id | ID! | The primary unique ID of the record. Every model should have an id field. |
| name | String! | Name of the integration configuration. Becomes the record ID via the create trigger. |
| active | Boolean! | Whether this configuration is currently enabled and eligible to run. |
| isInbound | Boolean! | Direction flag: true when data flows inbound (into Fuuz), false when outbound (Fuuz to external system). |
| integrationTypeId | ID! | FK to IntegrationType. Supplies the connection, capacity limits, and app-module scope. |
| requestHandlerDataFlowId | ID | Configure which flow main handler has to use for processing integration data flow and response data flow |
| integrationDataFlowId | ID! | The DataFlow that processes the integration payload. |
| responseDataFlowId | ID! | The DataFlow that processes the integration response after execution. |
| autoRetryResponses | JSON | JSON configuration describing which response conditions should trigger an automatic retry. |
| integrationType | IntegrationType! | Resolved IntegrationType. The orchestrator reads concurrencyLimit and connection from this relation. |
| requestHandlerDataFlow | DataFlow | Resolved request handler DataFlow. |
| integrationDataFlow | DataFlow! | Resolved integration DataFlow. |
| responseDataFlow | DataFlow! | Resolved response handler DataFlow. |
| lastTransactionAt | DateTime | Timestamp of the most recent transaction processed by this configuration. |
| latestRecordId | String | External reference/cursor of the most recently processed record, used to drive incremental (delta) extraction. |
| savedScriptId | String | Identifier of a saved script associated with this configuration. |
| maxRetryAttemps | Int | Maximum number of times a failed record is automatically retried before it is permanently failed (Cannot Process). Note: field id is spelled maxRetryAttemps in the model. |
| maxRecords | Int | Configure the maximum number of records extracted in one request |
| parameters | JSON! | JSON parameters/settings that drive this configuration's behavior. |
| scopedDataModelId | ID | The DataModel scoped to the FUUZ Apps integration. |
| scopedDataModel | DataModel | Resolved scoped DataModel object. |
| integrationStagings | [IntegrationStaging!]! | Back-relation to all staging records for this configuration. |

### 4.5 IntegrationMessage

Captures individual log/event messages for each staging record, providing a full per-transaction audit trail. Every status transition, error, warning, or informational event is written here.

Model Version: 9  |  Kind: Reference  |  Model ID: integrationMessage

| Field | Type | Description |
|---|---|---|
| id | ID! | The primary unique ID of the record. Every model should have an id field. |
| number | String! | Auto-sequenced message number using sequence integrationMessageNumber (format IM000000000001). |
| global | Boolean! | If true, the message is a global platform-level event rather than scoped to a staging record. |
| message | String | Human-readable message text (error description, status note, processing detail). |
| data | JSON | Structured JSON payload with additional data relevant to this message (e.g. response body, error details). |
| typeId | ID! | FK to IntegrationMessageType (Failure, Warning, Informational, Success, Detail). |
| integrationStagingId | ID | FK to the parent IntegrationStaging record. Null for global messages. |
| integrationStaging | IntegrationStaging | Resolved parent staging record. |
| type | IntegrationMessageType | Resolved message type object, including color and behavior flags. |

### 4.6 IntegrationMessageType

Reference model classifying the nature and severity of each IntegrationMessage. Five standard types are seeded by the package.

Model Version: 2  |  Kind: Reference  |  Model ID: integrationMessageType

| Field | Type | Description |
|---|---|---|
| id | ID! | The primary unique ID of the record. Every model should have an id field. |
| name | String! | Display name of the message type. |
| order | Int! | Sort order used when displaying message types. |
| fail | Boolean! | Marks the type as a failure classification. |
| informational | Boolean! | Marks the type as informational. |
| success | Boolean! | Marks the type as a success classification. |
| system | Boolean! | Marks the type as platform-reserved. |
| usable | Boolean! | Controls whether custom flows may write messages of this type. |
| color | String! | Display color used to render the message type in the UI. |
| customData | JSON | Optional JSON container for additional type attributes. |
| integrationMessages | [IntegrationMessage!]! | Back-relation to messages of this type. |

### 4.7 StagingTableStatus Flag Matrix

| Status | New | In Process | Processed | Success | Failure | Auto Retry | Color |
|---|---|---|---|---|---|---|---|
| Cannot Process | N | N | N | N | Y | N | #ff5656 |
| Error | N | N | Y | N | Y | N | #ff5656 |
| Hold | N | N | N | N | N | N | #ffff00 |
| Inprocess | N | Y | N | N | N | N | #fff700 |
| Integration Error | N | N | Y | N | Y | N | #ff5656 |
| New | Y | N | N | N | N | N | #808080 |
| Processed | N | N | N | Y | N | N | #2be84f |
| Processing Error | N | N | Y | N | Y | N | #ff5656 |
| Queued | N | N | N | N | Y | Y | #fff700 |
| Success | N | N | Y | Y | N | N | #ceff00 |

Only Queued has autoRetry=true. All failure statuses (Cannot Process, Error, Integration Error, Processing Error) have autoRetry=false — they require manual intervention. Hold has no flags set, so held records are invisible to the Main Handler until they are released.

### 4.8 IntegrationMessageType Matrix

| Type | Order | Fail | Informational | Success | System | Color |
|---|---|---|---|---|---|---|
| Failure | 0 | Y | N | N | Y | #ff5656 |
| Warning | 1 | N | Y | N | Y | #ffee00 |
| Success | 3 | N | N | Y | Y | #2be84f |
| Informational | 4 | N | Y | N | Y | #43c5ff |
| Detail | 5 | N | Y | N | Y | #cccccc |

### 5. Core Components — Data Flows

### 5.1 Integration: Main Handler

Flow Version: 0.0.5 |  Nodes: 15

The orchestration entry point. It identifies eligible staging records and dispatches them for processing. Four trigger mechanisms operate in parallel:

- Schedule — configurable polling via the flow Schedule node (the schedule itself is configured in the environment after import).

- Data Changes — fires on IntegrationStaging Create events, enabling near-real-time processing without waiting for the next schedule tick.

- Topic — subscribes to integration.main.handler for programmatic triggering.

- Request — direct invocation for manual runs.

Processing sequence:

- Mutex Lock (resource "mainHandlerInprocess", TTL 30 s, 1 retry) — prevents concurrent Main Handler executions; does not throw when the lock is not acquired.

- Query: Staging Status — retrieves all StagingTableStatus IDs where new=true or autoRetry=true.

- Query: Integration Staging — fetches all staging records in those statuses, ordered by createdAt ascending, including each record’s integrationConfig, its integrationType.concurrencyLimit, and the current In Process totals across all configurations of that type.

- Prepare Data (transform) — excludes records whose integrationConfig is missing, groups the remainder by integrationTypeId, and pre-filters each group against the concurrency limit: if concurrencyLimit is set and non-zero, only (limit − currently In Process) records are dispatched for that type; a limit of 0 means unlimited.

- Accept if record exists (ifElse) — exits cleanly (unlocking the mutex) when no eligible records remain.

- Publish (bulk) — publishes each eligible record to integration.main.request.handler.

- Mutex Unlock and execution logs complete the run.

### 5.2 Integration: Main Flow Request Handler

Flow Version: 0.0.6  |  Nodes: 38

Processes a single staging record end-to-end. Triggered by the integration.main.request.handler topic (published by the Main Handler) or by direct request. The record ID arrives in the request context.

Validation and guard branches (Route), evaluated in order:

- Incorrect Integration Config — integrationConfig missing: record set to Cannot Process with Failure message "Invalid or inactive integration configuration in FUUZ".

- Invalid Config — integrationDataFlowId or responseDataFlowId empty: Cannot Process with Failure message "Integration Configuration Missing Integration/Response dataflow in FUUZ".

- Reached Max Retry Attempts — attemptCount equals IntegrationConfiguration.maxRetryAttemps: Cannot Process with Failure message "Stopped retrying as the record has reached maximum retry attempts".

- Exceeds Concurrency — when concurrencyLimit > 0 and the count of In Process records across the type’s configurations is at or above the limit: the record receives an Informational message "Added to the Queue, maximum concurrency reached" and is left for a later cycle.

- Record Inprocess guard — only records whose status is one of New, Queued, Cannot Process, Integration Error, or Processing Error are processed; anything else exits without action (prevents double-processing).

Happy-path sequence:

- Mutex Lock on the staging record ID (TTL 30 s, 1 retry) — per-record lock prevents duplicate processing.

- Query: In Process / If Else — when a concurrency limit is configured, counts In Process records across all configurations of the integration type before proceeding.

- Mutate: Inprocess — sets statusId=Inprocess, increments attemptCount, writes Informational message "Integration Started".

- Execute Flow: Integration — $executeFlow(integrationDataFlowId, payload) inside a Try/Catch. The payload contains id, request, connection (the IntegrationType connection name), and parameters.

- Query: Int Stagings RES — re-fetches the staging record to read the response written by the integration flow.

Response evaluation (Route):

- Success — RESPONSE.success is true: statusId=Success with a Success message.

- AutoRetryError — the response error code (RESPONSE.error.error.code) or message (RESPONSE.data.message) matches, case-insensitively, an entry in IntegrationConfiguration.autoRetryResponses: statusId=Queued with Informational message "Added to the Queue to reprocess".

- Error — RESPONSE.success is false: statusId=Integration Error with a Failure message.

- Unhandled response — any other outcome: statusId=Cannot Process with a Failure message. Unexpected exceptions caught by the Try/Catch produce Cannot Process with Failure message "Integration failed with unhandled error check log".

- Execute Flow: Response Handler — $executeFlow(responseDataFlowId, { recordId, timeStamp }) is invoked after the final status is set, followed by mutex unlock and the final processed log.

### 5.3 Int_Outbound App Tenant Request Handler

Flow Version: 0.0.5  |  Nodes: 15

A structured request entry point through which application tenants submit outbound integration requests to the orchestration tenant. The caller sends a payload whose DATA object carries tenantId, configId, and recordId (plus any additional data).

Processing sequence:

- Query: Integration Setup — fetches the IntegrationConfiguration by configId where active=true, and simultaneously checks IntegrationStaging for an existing record with outboundRecordReference equal to tenantId_configId_recordId (deduplication).

- Route — "Setup Missing": when no active configuration matches, logs "Setup not found error" and returns an error response. "Record Already Exists": when a staging record with the same outbound reference exists, logs and returns an error response.

- Mutate: Int Staging — creates the IntegrationStaging record with integrationConfigId, outboundRecordReference (tenantId_configId_recordId), attemptCount 0, metadata set to the full request context, and statusId=Hold.

- Publish (bulk) — publishes to integration.outbound.record.processor. The tenant-built record processor flow is expected to subscribe, build the request payload, and set the record to New so the Main Handler picks it up.

- Format Payload / Response — returns { recordId: <IntegrationStaging.id> } to the caller, which can be used to poll the staging record status.

KNOWN EXCEPTION  The "Record Already Exists" branch contains a hardcoded bypass: duplicate checking is skipped when the configuration ID is "NS Inv Adjust". This is a tenant-specific exception embedded in the packaged flow.

### 5.4 Integration Orchestrator Dashboard

Flow Version: 0.0.19  |  Nodes: 21  |

A read-only presentation flow that feeds the Integration Dashboard screen. It is invoked on demand from the screen via $executeFlow("integrationOrchestratorDashboard", { section, themeMode, ... }) and routed by the requested section. Each section branch follows the same three-node pattern: a GraphQL query over the orchestrator data, a JavaScript transform that renders the result as themed HTML, and a response node returning the markup to the calling screen element.

Sections served by the flow:

- header — KPI cards: per-status transaction counts and failure totals for the current day (statusId aggregates and appPostError/failure-status counts over processedAt).

- topology — hub-and-spoke view of the integration landscape (integration types and their configurations around the orchestrator).

- throughput — message throughput chart over a requested time range (e.g. range: "1h").

- activity — live activity feed of recent staging transactions and their statuses.

- connectors — connector/integration-type status summary.

- errorQueue — current error-queue listing of failed records requiring attention.

The flow performs no mutations: it does not alter staging records, statuses, or configuration, and can be called freely without affecting orchestration.

### 6. Core Components — Screens

| Screen | Version | Description & Purpose |
|---|---|---|
| Integration Setup | 0.0.10 | Configuration hub, routed at /system/integration/integrationSetup (menu-accessible). Two tabs: Integration Type Setup and Integration Configuration, each with a filter form, a data table, and Search / Create / Update / Delete actions. Create and Update open the corresponding manage screens. |
| Integration Transactions | 0.0.2 | Monitoring screen, routed at /system/integration/integrationTransactions (menu-accessible). Inbound and Outbound tabs list IntegrationStaging records with date-range and status filters; selecting a record exposes its request/response payloads and child IntegrationMessage tables. |
| Manage Integration Configuration | 0.0.12 | Standalone routed form (hidden from menu) at /system/integration/manageIntegrationConfiguration, opened in Add or Edit mode via URL parameter. Captures all IntegrationConfiguration fields; the Name field is locked in Edit mode because it is the record ID. |
| Manage Integration Type | 0.0.6 | Standalone routed form (hidden from menu) at /system/integration/manageIntegrationType, opened in Add or Edit mode. Captures IntegrationType fields; capacity fields (Concurrency Limit, Max Records Limit, Request Interval) are shown only when the type targets an external system (Fuuz Apps switch off). Integration Name is locked in Edit mode. |
| Integration Dashboard | 0.0.2 | Operational dashboard, routed at /system/integration/integrationDashboard (menu-accessible). Composed of section panels (KPI header, hub-and-spoke topology, message throughput, live activity, connectors, error queue), each backed by a form that calls the Integration Orchestrator Dashboard flow for its section and renders the returned HTML in an embedded webpage element. Panels follow the active theme via the themeMode parameter. |

DEPENDENCY NOTE  The Integration Transactions screen contains a table bound to the IntegrationAppTenantScope model, which is not shipped in this package. In environments without that model the affected table has no data source; all IntegrationStaging and IntegrationMessage tables are unaffected.

### 7. Status & Message Lifecycle

### 7.1 Staging Record Status Lifecycle

Happy path:

- Record created by a request flow with status New (or via the outbound API with status Hold, then released to New by the tenant record processor flow).

- Main Handler dispatches the record to the Request Handler (concurrency pre-filtered).

- Request Handler validates, locks, and sets status Inprocess (attemptCount incremented, "Integration Started" message).

- Integration data flow executes against the external system and writes the response to the staging record.

- Response evaluated as success — status Success; the response handler flow is then executed.

Retry path:

- Response matches autoRetryResponses — status Queued (autoRetry=true); the Main Handler re-dispatches it on a later cycle.

- When attemptCount reaches maxRetryAttemps, the record is permanently failed with status Cannot Process.

Concurrency path:

- Both the Main Handler (pre-filter) and the Request Handler (guard) enforce IntegrationType.concurrencyLimit. A record that exceeds the limit at the Request Handler receives the message "Added to the Queue, maximum concurrency reached" and is retried on a later cycle.

Error paths:

- Cannot Process — invalid or missing configuration, missing data flow references, max retries exceeded, or unhandled exception. Not auto-retried.

- Integration Error — the external system returned an error response. Not auto-retried.

- Processing Error / Error — general processing failures. Not auto-retried.

- Hold — set manually by an operator or by the outbound API on creation. Invisible to the Main Handler until released.

### 7.2 Integration Message Types

| Message Type | When Written |
|---|---|
| Failure | Error conditions: exception caught, integration call failed, configuration missing or invalid, max retries reached. |
| Warning | Non-fatal issues raised by custom flows. |
| Informational | Status transitions: integration started, queued to reprocess, concurrency queue events. |
| Success | Successful completion of the integration call. |
| Detail | Verbose diagnostic data: raw payloads, debug context. |

### 8. How to Apply — Implementation Guide

To build a new integration on top of the orchestrator you create: an IntegrationType record, an IntegrationConfiguration record, a Request Flow, an Integration Data Flow, a Response Handler Flow, and — for the outbound API path — an Outbound Record Processor Flow. The companion document "Integration Orchestrator — Configuration Guide" covers the screen-level configuration steps in detail.

### 8.1 Step 1: Define an IntegrationType

- Open Integration Setup, tab Integration Type Setup, and use Create.

- Provide the Integration Name, the Connection (credentials and endpoint of the external system), and the Tenant ID.

- For external-system types (Fuuz Apps switch off), set Concurrency Limit, Max Records Limit, and Request Interval as required by the target system.

- For Fuuz App types, select the target application (MES / WMS / CMMS / QMS) and set Site ID / Enterprise ID where applicable.

### 8.2 Step 2: Create an IntegrationConfiguration

- Open Integration Setup, tab Integration Configuration, and use Create. The Name becomes the record ID and cannot be changed later.

- Select the Integration Type, direction (Inbound / Outbound), the Integration Data Flow, and the Response Data Flow. Optionally set a custom Request Handler Data Flow.

- Configure Auto Retry Responses (JSON array of error codes/messages that should re-queue instead of fail), Max Retry Attemps, Max Records, and Parameters.

### 8.3 Step 3: Build the Request Flow

Triggered by application logic (e.g. a data change on a Work Order). It queries the source data, prepares the request payload, and creates the IntegrationStaging record with integrationConfigId, request, statusId=New, references, metadata, and optionally fileId. It must not execute the integration itself — the orchestrator handles everything from the staging record onward.

### 8.4 Step 4: Build the Integration Data Flow

Called by the Request Handler via $executeFlow() with { id, request, connection, parameters }. It calls the external system (e.g. via $integrate() or an Integration node), writes the response back to the staging record’s response field, and sets processedAt. The Request Handler re-reads the response afterwards to determine the final status.

### 8.5 Step 5: Build the Response Handler Flow

Called with { recordId, timeStamp } after the final status is set, regardless of outcome. It queries the staging record, applies business logic to the response, updates application records, and writes IntegrationMessage entries. It must handle both success and failure statuses.

### 8.6 Step 6: Outbound API Path (Optional)

For application tenants submitting requests through the orchestration tenant: the caller invokes the Int_Outbound App Tenant Request Handler with DATA { tenantId, configId, recordId, ... } and receives { recordId }. A tenant-built Outbound Record Processor flow must subscribe to integration.outbound.record.processor, prepare the request payload, and release the staging record from Hold to New.

### 8.7 Concurrency & Retry Configuration

| Control | Where Set |
|---|---|
| Main Handler global mutex | Fixed in flow: resource key "mainHandlerInprocess", TTL 30 s, 1 retry. |
| Per-record mutex | Fixed in flow: resource key = staging record ID, TTL 30 s, 1 retry. |
| Concurrency limit per type | IntegrationType.concurrencyLimit — enforced by the Main Handler pre-filter and the Request Handler guard. 0 = unlimited. |
| Max records per cycle | IntegrationType.maxRecordsLimit (type level) and IntegrationConfiguration.maxRecords (configuration level). |
| Request pacing | IntegrationType.requestInterval (milliseconds between requests, where applicable). |
| Auto-retry on response | IntegrationConfiguration.autoRetryResponses — entries matched case-insensitively against the response error code or message. |
| Max retry attempts | IntegrationConfiguration.maxRetryAttemps — attempts before permanent failure (Cannot Process). |

### 9. Reference Data

### 9.1 Sequences

| Sequence | Details |
|---|---|
| Integration Staging Number | ID: integrationStagingNumber | Prefix: IS | Example: IS000000000001 |
| Integration Message Number | ID: integrationMessageNumber | Prefix: IM | Example: IM000000000001 |

### 9.2 Integration Data Flow Context Contract

Payload passed by the Request Handler to the integration data flow via $executeFlow():

| Context Key | Value |
|---|---|
| id | The IntegrationStaging record ID |
| request | The request JSON payload from the staging record |
| connection | The name of the Connection configured on the IntegrationType |
| parameters | The parameters JSON from the IntegrationConfiguration |

### 9.3 Response Handler Context Contract

| Context Key | Value |
|---|---|
| recordId | The IntegrationStaging record ID |
| timeStamp | ISO timestamp of processing completion ($now()) |

### 9.4 Outbound API Contract

| Element | Value |
|---|---|
| Request | DATA: { tenantId, configId, recordId, ... } — used to resolve the configuration and build the dedup key tenantId_configId_recordId |
| Success response | { recordId: "" } |
| Error responses | Setup not found (no active configuration for configId); Record already exists (duplicate outboundRecordReference) |

### 10. Package Dependencies & Prerequisites

The package declares no external package dependencies. It relies on the following Fuuz platform capabilities: the Data Flow engine (schedule, query, mutate, transform, javascriptTransform, setContext, switch, ifElse, tryCatch, topic, publish, request, response, mutexLock, mutexUnlock, mergeContext, log, and dataChanges nodes), the GraphQL API, the $executeFlow() built-in, distributed mutex infrastructure, topic/publish messaging, the Sequence engine, the Connection model (referenced by IntegrationType), the File model (optional attachment on IntegrationStaging), the DataModel / DataFlow models (referenced by IntegrationConfiguration), and the screen webpage element (used by the Integration Dashboard to render flow-generated HTML).

Prerequisites for operating the package:

- Platform version 2026.6.1 or higher.

- At least one Connection record must exist before creating IntegrationType records.

- Integration data flows and response handler flows must be built and deployed before IntegrationConfiguration records reference them.

- The Main Handler Schedule must be configured in the target environment after import.

- For the outbound API path, a tenant-built record processor flow subscribing to integration.outbound.record.processor is required.

### 11. Glossary

| Term | Definition |
|---|---|
| Staging Table Pattern | Integration pattern where transaction records are persisted in an intermediate table before processing, enabling decoupling, retry, and full audit. |
| Mutex | Mutual exclusion lock preventing concurrent access to a shared resource; used globally by the Main Handler and per record by the Request Handler. |
| $executeFlow() | Fuuz built-in that synchronously invokes another data flow and returns its result. |
| autoRetry | StagingTableStatus flag marking records in that status as eligible for re-dispatch by the Main Handler. |
| IntegrationType | Connection and capacity profile shared by multiple IntegrationConfiguration records. |
| IntegrationConfiguration | Per-endpoint definition specifying which data flows to execute, retry rules, and parameters. |
| Request Flow | Custom flow built by the integrator that creates the staging record (the enqueue step). |
| Integration Data Flow | Custom flow performing the external system call, invoked by the Request Handler. |
| Response Handler Flow | Custom flow that processes the integration response and updates application records. |
| Outbound Record Processor Flow | Tenant-built flow subscribing to integration.outbound.record.processor; prepares the payload and releases Hold records to New. |
| TTL | Time To Live — the maximum duration a mutex lock is held (30 seconds in this package). |
