# Changelog

All notable changes to **SIMS Start Processing** are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

Version numbers follow the **Dynamics model descriptor** in [`Descriptor/SIMSStartProcessing.xml`](Descriptor/SIMSStartProcessing.xml) (`VersionMajor.Minor.Build` + `VersionRevision`).

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
