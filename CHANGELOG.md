# Changelog

All notable changes to **SIMS Start Processing** are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

Version numbers follow the **Dynamics model descriptor** in [`Descriptor/SIMSStartProcessing.xml`](Descriptor/SIMSStartProcessing.xml) (`VersionMajor.Minor.Build` + `VersionRevision`).

---

## [1.0.7] – 2026-05-11 (descriptor `VersionRevision` **7**)

### Overview

Production-ready release with **outbound REST webhooks** for AOS startup orchestration and **inbound JSON SQL service** for sandbox automation (e.g., Azure Logic Apps). Complete documentation, labels, and security controls.

### Scenarios & Use cases

| Use case | How it works |
|----------|------|
| **Sandbox after DB refresh** | REST webhook notifies Logic App → executes approved SQL lines (masking, enable users, cleanup) via inbound `executeSql` service. Environment-guarded to prevent accidental production runs. |
| **UAT automation** | Configure startup SQL lines for environment setup, add REST line to notify orchestration system. Sub-admins approve, batch runs at AOS start. |
| **Minimal external scripts** | Run only essential SQL/scripts via startup processor with audit logging. Ad-hoc SQL disabled by default (requires explicit opt-in). |

### Added

- **Outbound REST webhook** from startup processing: **REST API call** task type sends HTTPS requests to external services (e.g., Logic App HTTP trigger) with configurable method, URL, headers, body, and timeout.
- **Inbound JSON SQL service** (`SIMSStartSqlExecutionService.executeSql`): **Logic App** or any OAuth client can run **approved catalog SQL** or **optional ad-hoc SQL** (must enable via **Service parameters**).
- **Service parameters form** (`SIMSStartParameters`): **Allowed environment URL pattern** (environment guard for inbound calls), **Allow ad-hoc SQL** toggle.
- **Full documentation**: [`docs/SIMSStartProcessing-LogicApps-and-API.md`](docs/SIMSStartProcessing-LogicApps-and-API.md) with OAuth notes, Logic App patterns, request/response contracts, examples.
- **Security privilege** (`SIMSStartInboundSqlPrivilege`): role-based access to service parameters.
- **Audit logging** for inbound calls: **SIMSStartLog** records correlation ID, success flag, SQL preview (first 200 chars for ad-hoc mode).

### How it works

1. **POST to Logic App at AOS startup** — configure REST line with Logic App HTTP trigger URL; batch sends notification when F&O starts.
2. **Logic App → F&O SQL execution** — call `executeSql` service with catalog line RecId to run pre-approved SQL scripts (masking, user setup, cleanup).
3. **Ad-hoc SQL from Logic Apps** — enable **Allow ad-hoc SQL from service**, send `SqlScript` in request (admin-controlled feature for controlled scenarios only).

### Metadata

- Descriptor **`VersionRevision`**: **7**.
- Model version: **1.0.0** (build **0**), release date **2026-05-11**.

---

## [1.0.6] – 2026-05-05 (descriptor `VersionRevision` **6**)

### Documentation

- Rewrote [`README.md`](README.md) for a **public repository**: capabilities summary, REST + inbound API overview, configuration tables, links to deeper docs.
- Added this **`CHANGELOG.md`**.
- Updated [`docs/SIMSStartProcessing-LogicApps-and-API.md`](docs/SIMSStartProcessing-LogicApps-and-API.md) with release/version notice and aligned logging notes.

### Metadata

- Descriptor **`VersionRevision`**: **6**.

---

## [1.0.5] – 2026-05-05

### Added

- **Ad hoc SQL audit snippet** in **`SIMSStartLog`**: when inbound execution uses **`SqlScript`** (ad hoc path), the log appends **`AdHocSqlPrefix (first 200 chars)`** after collapsing whitespace (constant `#SqlAdhocPreviewLen` in `SIMSStartSqlExecutionService`).

### Metadata

- Descriptor **`VersionRevision`**: **5**.

---

## [1.0.4] – 2026-05-05

### Added

- **`SIMSStartParameters`** table and **Service parameters** menu: **Allowed environment URL pattern**, **Allow ad hoc SQL from service**.
- **`SIMSStartSqlExecutionService`** JSON custom service + **`SIMSStartProcessingServices`** service group: operation **`executeSql`** (catalog `LineRecId` and optional ad hoc `SqlScript`).
- **`SIMSStartSqlRunner`** shared SQL execution; **`SIMSStartSqlExecutionValidator`** environment / catalog checks.
- **`SIMSStartInboundSqlPrivilege`** and duty update for parameter maintenance.
- English integration guide: [`docs/SIMSStartProcessing-LogicApps-and-API.md`](docs/SIMSStartProcessing-LogicApps-and-API.md).

### Changed

- **`SIMSStartRunActionSQL`** delegates execution to **`SIMSStartSqlRunner`**.

### Metadata

- Descriptor **`VersionRevision`**: **4**.

---

## [1.0.3] – 2026-05-05

### Added

- **`SIMSStartTaskType::RestApi`** and **`SIMSStartRunActionRest`**: outbound HTTP calls from startup processing (method, URL, headers, body, timeout).
- **`SIMSStartHttpRequestMethod`** enum.
- **`SIMSStartTable`** REST-related fields; **SIMSStartTable** form tab **REST request**.
- Validation: REST tasks require **Request URL**; startup **`validate`** extended.

### Metadata

- Descriptor **`VersionRevision`**: **3**.

---

## Earlier revisions

Revisions **1–2** and the original **1.0.x** baseline introduced the core **AOS startup batch**, **`SIMSStartTable`** processing types (**SQL**, **Class**, **RunBase**), approval/sub-admin flows, and logging (**`SIMSStartLog`**, **`SIMSStartTrans`**). See Git history for file-level detail.
