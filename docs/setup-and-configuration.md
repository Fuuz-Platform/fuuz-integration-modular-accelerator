# Integration Orchestrator — Setup & Configuration Guide

> Converted from the v1.0.0 Word document shipped with the accelerator package.

### 1. Purpose & Scope

### 2. Prerequisites

### 3. Configuration Sequence

### 4. Step 1 — Configure the Integration Type

### 5. Step 2 — Configure the Integration Configuration

### 6. Step 3 — Verify the Configuration

### 7. Monitoring — Integration Transactions Screen

### 7.1 Status Reference

### 7.2 Common Failure Messages

### 8. Tuning Reference

### 1. Purpose & Scope

This guide provides step-by-step instructions for configuring the Fuuz Integration Orchestrator (package version 0.0.9) in a tenant where the package is already installed. It covers the creation and maintenance of IntegrationType and IntegrationConfiguration records through the packaged screens, verification of a working configuration, and day-to-day monitoring through the Integration Transactions screen.

Building the per-integration data flows (request flow, integration data flow, response handler flow, outbound record processor flow) is outside the scope of this guide. Refer to section 8 of the AS IS Build Document (v1.0.0) for the flow-building reference.

### 2. Prerequisites

Confirm the following before starting configuration:

| Prerequisite | Detail |
|---|---|
| Package installed | Integration Orchestrator 0.0.9 imported into the tenant (module Integration under module group System is visible in the menu). |
| Connection record | A Fuuz Connection holding the credentials and endpoint of each external system or application tenant you will integrate with. IntegrationType cannot be saved without one. |
| Data flows deployed | The integration data flow and response handler data flow referenced by each IntegrationConfiguration must already exist in the tenant. |
| Main Handler schedule | The Schedule trigger on the "Integration: Main Handler" flow must be configured (via the flow Schedule editor) for polling-based pickup. Record-create events and topic triggers work independently of the schedule. |
| Tenant ID | The exact tenantId of each application tenant, required on IntegrationType records. |
| Screen access | User permissions for the routes /system/integration/integrationSetup and /system/integration/integrationTransactions. |

### 3. Configuration Sequence

- Create (or verify) the Connection for the external system or application tenant.

- Create the IntegrationType — the connection and capacity profile (section 4).

- Create one IntegrationConfiguration per integration endpoint (section 5).

- Activate the configuration and verify with a test transaction (section 6).

- Monitor ongoing transactions in Integration Transactions (section 7).

### 4. Step 1 — Configure the Integration Type

Navigate to Integration Setup (menu: System > Integration, route /system/integration/integrationSetup) and open the Integration Type Setup tab. Use Create to open the Manage Integration Type form (Update edits the selected row; the Integration Name is locked in edit mode).

| Field | Required | How to Configure |
|---|---|---|
| Integration Name | Yes | Descriptive name for this class of integration (e.g. "ERP NetSuite", "MES App Tenant"). Locked after creation. |
| Fuuz Apps (switch) | Yes | Switch between External System (off) and Fuuz Apps (on). Controls which of the fields below are shown. |
| Concurrency Limit | No | External System types only. Maximum number of records In Process simultaneously across all configurations of this type. 0 or empty = unlimited. Enforced by both the Main Handler pre-filter and the Request Handler. |
| Max Records Limit | No | External System types only. Maximum number of records processed in one request/cycle. |
| Request Interval | No | External System types only. Interval between requests in milliseconds, for rate-limited external APIs. |
| MES / WMS / CMMS / QMS App | Yes (Fuuz Apps mode) | Select the target Fuuz application. Selecting MES App automatically clears the WMS/CMMS/QMS checkboxes. |
| Site ID | No | External ERP facility / plant / site identifier mapped to this tenant configuration. |
| Enterprise ID | No | External ERP subsidiary / entity / company identifier. |
| Tenant ID | Yes | The exact tenantId of the tenant this type belongs to. |
| Connection | Yes | The Fuuz Connection used to reach the external system or application tenant. Its name is passed to the integration data flow at runtime. |
| Additional Definitions | No | JSON container for defaults that integrations of this type can read at runtime. |

TIP  Create one IntegrationType per external system or application tenant, and reuse it across all integration configurations that share the same connection and capacity limits (e.g. PO Outbound and Invoice Outbound sharing one ERP type).

### 5. Step 2 — Configure the Integration Configuration

In Integration Setup, open the Integration Configuration tab and use Create to open the Manage Integration Configuration form. Create exactly one record per distinct integration endpoint.

| Field | Required | How to Configure |
|---|---|---|
| Name | Yes | Human-readable integration name (e.g. "ERP-PurchaseOrder-Outbound"). Becomes the record ID via the create trigger and is locked after creation. |
| Integration Type | Yes | The IntegrationType created in Step 1. Supplies the connection, capacity limits, and app scope. |
| Scoped Data Model | No | Optional DataModel that scopes this integration for FUUZ Apps integrations. Can be used to identify the scope of the configuration for FUUZ Apps data ingestion. |
| Outbound / Inbound (switch) | Yes | Direction flag: Inbound = data flows into Fuuz; Outbound = Fuuz to external system. |
| Integration Data Flow | Yes | The data flow that performs the external system call. Invoked by the orchestrator via $executeFlow(). |
| Response Data Flow | Yes | The data flow that processes the result after the integration completes (called for success and failure). |
| Request Handler Data Flow | Conditional | Custom per-integration request handler. Leave as the packaged "Integration: Main Flow Request Handler" behavior unless a custom handler is required. |
| Auto Retry Responses | No | JSON array of error codes or message fragments that should re-queue the record (status Queued) instead of failing it. Matching is case-insensitive against the response error code (error.error.code) or message (data.message). Example: ["TIMEOUT", "RATE_LIMIT_EXCEEDED"]. |
| Saved Script ID | No | Identifier of a saved script for reusable logic within integration flows. |
| Max Retry Attemps | No | Number of attempts before the record is permanently failed as Cannot Process (e.g. 5). Empty = retry indefinitely. |
| Max Records | No | Maximum records extracted/processed in one request for this configuration. |
| Parameters | No | JSON settings available to the integration data flow at runtime (endpoint paths, field mappings, constants). |
| Active | Yes | Must be checked for the orchestrator to process records of this configuration. Uncheck to disable the integration without deleting it. |

IMPORTANT  The Name becomes the record ID and is also the configId that callers of the outbound API must send. Choose a stable, meaningful name before creating the record.

### 6. Step 3 — Verify the Configuration

- Confirm the configuration row appears in the Integration Configuration tab with Active checked.

- Create a test transaction: have the request flow (or a manual mutation) create an IntegrationStaging record with the new integrationConfigId, a valid request payload, and statusId "New".

- The Main Handler picks the record up via the IntegrationStaging create event or on the next schedule tick.

- Open Integration Transactions and locate the record (Inbound or Outbound tab depending on direction).

- Confirm the expected status progression: New, then Inprocess (message "Integration Started"), then Success — or an error status with a Failure message explaining the cause.

- Verify the response payload was written to the staging record and that the response handler updated the application data as designed.

OUTBOUND API PATH  Records submitted through the Int_Outbound App Tenant Request Handler are created in Hold status and published to the integration.outbound.record.processor topic. They will not be processed until a tenant-built record processor flow prepares the payload and sets the status to New. Verify that this flow is deployed before testing the outbound API path.

### 7. Monitoring — Integration Transactions Screen

Route: /system/integration/integrationTransactions (menu: System > Integration). The screen presents Inbound and Outbound tabs listing IntegrationStaging records, with filter forms (date range, status, configuration) and a Search action. Selecting a record exposes its request and response payloads and the child IntegrationMessage log.

### 7.1 Status Reference

| Status | Auto Retry | Meaning / Operator Action |
|---|---|---|
| New | No (initial) | Waiting for pickup by the Main Handler. No action needed. |
| Inprocess | No | Currently executing. If stuck, check the flow execution logs. |
| Queued | Yes | Will be retried automatically (auto-retry response matched, or concurrency limit reached). |
| Success | No | Integration completed successfully. |
| Processed | No | Post-success processing completed. |
| Integration Error | No | External system returned an error. Review the response payload and messages, correct the cause, and reprocess. |
| Processing Error | No | Unexpected exception during processing. Review flow logs. |
| Error | No | General error. Review messages. |
| Cannot Process | No | Configuration invalid/missing, required flows missing, or max retries reached. Fix configuration before reprocessing. |
| Hold | No | Parked: set manually or created by the outbound API awaiting the record processor. Not visible to the orchestrator until released. |

### 7.2 Common Failure Messages

| Message | Cause & Resolution |
|---|---|
| Invalid or inactive integration configuration in FUUZ | The staging record references a missing or inactive IntegrationConfiguration. Verify the configId and the Active flag. |
| Integration Configuration Missing Integration/Response dataflow in FUUZ | integrationDataFlowId or responseDataFlowId is empty. Edit the configuration and select both flows. |
| Added to the Queue, maximum concurrency reached | Informational: the concurrency limit on the IntegrationType was reached; the record is retried automatically. Raise the limit if throughput is too low. |
| Added to the Queue to reprocess | Informational: the response matched an autoRetryResponses entry; the record will be retried. |
| Stopped retrying as the record has reached maximum retry attempts | attemptCount reached Max Retry Attemps. Investigate the underlying failure; reset the record (e.g. status New, adjusted attempt count) to force reprocessing after the fix. |
| Integration failed with unhandled error check log | An unhandled exception occurred in the integration data flow. Check the data flow execution logs. |
| Setup not found error (outbound API) | The configId sent to the outbound API does not match an active IntegrationConfiguration. |
| Record already exists error (outbound API) | A staging record with the same tenantId_configId_recordId reference already exists. Duplicate submission was rejected. |

### 8. Tuning Reference

| Goal | Setting |
|---|---|
| Limit parallel calls to a rate-limited system | IntegrationType.Concurrency Limit (e.g. 5). Applies across all configurations of the type. |
| Cap batch size per cycle | IntegrationType.Max Records Limit, and/or IntegrationConfiguration.Max Records for a single endpoint. |
| Pace requests | IntegrationType.Request Interval (milliseconds). Need to handle this logic in the integration data flow. |
| Retry transient errors automatically | IntegrationConfiguration.Auto Retry Responses + Max Retry Attemps. |
| Pause one integration | Uncheck Active on its IntegrationConfiguration. |
| Park individual records | Set the staging record status to Hold; release by setting it back to New. |
