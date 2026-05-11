# SIMS Start Processing — Logic Apps & API Integration Guide

**Document release:** aligned with descriptor **`VersionMajor`/`Minor`/`Build` = 1.0.0**, **`VersionRevision` = 6**. See [`CHANGELOG.md`](../CHANGELOG.md).

---

This document describes how **Microsoft Dynamics 365 Finance & Operations** (with the **SIMSStartProcessing** model) integrates with **Azure Logic Apps** and external callers. It is intended for integration architects and operators configuring **non-production** (e.g. sandbox / UAT) environments after a database refresh from production.

---

## 1. Integration overview

There are two complementary directions:


| Direction                      | Purpose                                                                                                                                        | Typical use                                                                                                                                           |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Outbound (F&O → Logic App)** | Finance calls an HTTPS endpoint when a startup action runs (batch at AOS start or manual run).                                                 | Notify an orchestration layer that F&O reached a known step; Logic App continues automation outside the database.                                     |
| **Inbound (Logic App → F&O)**  | Logic App calls a **JSON custom service** that executes **approved catalog SQL** or **controlled ad-hoc SQL** and returns a structured result. | Continue masking / cleanup / enable users via SQL **without** direct SQL access from Logic Apps; all execution goes through F&O policies and logging. |


**Environment safety:** Startup actions and the inbound SQL service both rely on **environment URL patterns** (wildcards on the host/FQDN). Configure patterns so that **only** your sandbox/UAT host can run the logic—never point Logic Apps or patterns at production endpoints for these flows unless that is an explicit, separate decision.

---

## 2. Outbound: F&O calls Logic App (REST startup action)

### 2.1 Concept

A row in **AOS start processing** (`SIMSStartTable`) can use task type **REST API call**. When the action runs, F&O sends an HTTP request to the URL you configure (typically the **HTTP trigger URL** of a Logic App).

### 2.2 Configuration in F&O

In **AOS start processing** (menu path depends on your menu extension; often under system administration):

1. Create or edit a line with **Task type** = **REST API call**.
2. Set **Env URL template** so it matches **only** the target environment’s FQDN (same idea as SQL/class steps)—e.g. `*sandbox`* or `*uat.operations.dynamics.com*`.
3. Approve the line per your process (**Approved by**).
4. On the **REST request** tab:
  - **HTTP method** — `GET`, `POST`, `PUT`, `PATCH`, or `DELETE` (Logic App HTTP triggers usually accept **POST**).
  - **Request URL** — full HTTPS URL of the Logic App HTTP trigger (including `sig=` or API key query parameters if you use them).
  - **Timeout (seconds)** — default `100` if left `0` or empty.
  - **Headers** (optional) — one header per line:  
  `Header-Name: value`  
  Example:  
  `Content-Type: application/json`
  - **Request body** (optional) — for `POST`/`PUT`/`PATCH`, often JSON (UTF-8, `application/json` when content-type is set).

### 2.3 What Logic App receives

The Logic App **HTTP Request** trigger receives the verb, headers, and body you configured. You can parse JSON in a **Parse JSON** action and branch on fields (e.g. correlation id, step name).

### 2.4 Example: minimal Logic App (HTTP trigger)

1. Create a **Consumption** or **Standard** Logic App.
2. Add trigger **When a HTTP request is received**.
  - Method: match what F&O sends (often **POST**).
  - Optionally define a JSON schema for the body F&O posts.
3. Copy the **HTTP POST URL** into F&O **Request URL** for the REST startup line.

**Sample body F&O might send** (you define this in **Request body** on the startup line):

```json
{
  "source": "SIMSStartProcessing",
  "event": "aos-start-step",
  "environmentHint": "sandbox-restore",
  "timestampUtc": "2026-05-05T12:00:00Z"
}
```

Adjust to whatever your orchestration needs.

---

## 3. Inbound: Logic App calls F&O — SQL execution JSON service

### 3.1 Endpoint shape

After deployment, Finance exposes a **JSON custom service** under the **service group** `SIMSStartProcessingServices` and service `SIMSStartSqlExecutionService`.

**Operation:** `executeSql`  
**HTTP method:** `POST` (standard for JSON custom services)

**Base URL pattern:**

```http
https://{your-environment-host}/api/services/SIMSStartProcessingServices/SIMSStartSqlExecutionService/executeSql
```

Replace `{your-environment-host}` with your environment base URL (same host you use for OData, without `/data/`).

**Discovery (optional):** You can inspect metadata with **GET** (browser or HTTP GET with auth):

- List service groups:  
`https://{host}/api/services`
- List services in group:  
`https://{host}/api/services/SIMSStartProcessingServices`
- List operations:  
`https://{host}/api/services/SIMSStartProcessingServices/SIMSStartSqlExecutionService`

Use these URLs only with appropriate authentication; they are useful to confirm deployment.

### 3.2 Authentication

Callers must authenticate like any **Finance OData / service** integration:

1. **Microsoft Entra ID (Azure AD)** application registration (client credentials or delegated flow, depending on policy).
2. Grant the app access to **Dynamics 365 Finance / ERP** APIs as per Microsoft documentation for your tenant.
3. In Finance, map an **application user** (or appropriate user) to the **security roles** that include the SIMS Start duties/privileges (your partner configures **SIMSStartProcessingUser** or equivalent).

Logic Apps typically use:

- **HTTP** action + **OAuth 2.0** (Azure AD) **managed identity** or **app registration** (client id/secret or certificate), **audience** / **resource** set to your Finance application ID URI or `https://{hostname}` as required by your tenant setup.

Exact OAuth settings vary by tenant; align with your existing **Logic App → D365 FO** pattern if you already integrate.

### 3.3 Service parameters (mandatory for inbound SQL)

In F&O, open **Service parameters** (menu item bound to **SIMSStartParameters**):


| Field                               | Meaning                                                                                                                                                                                   |
| ----------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Allowed environment URL pattern** | Wildcard pattern matched against the environment **FQDN** (same safety concept as startup lines). Must be set before inbound calls succeed. Example: `*sandbox.operations.dynamics.com`*. |
| **Allow ad hoc SQL from service**   | If **No**, only **catalog** execution by `LineRecId` is allowed. If **Yes`, callers may send raw` SqlScript`when`LineRecId` is not used—use only in tightly controlled scenarios.         |


### 3.4 Request contract (`SIMSStartSqlExecuteRequestContract`)

JSON property names (case-sensitive in many stacks):


| Property        | Type             | Description                                                                                                                                            |
| --------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `LineRecId`     | Integer (64-bit) | RecId of a **SIMSStartTable** row with **Task type** = **SQL script**, approved, and matching env template. Use `0` if not using catalog mode.         |
| `SqlScript`     | String           | Raw SQL for **ad hoc** mode. Ignored if catalog mode is used. Requires **Allow ad hoc SQL from service** = **Yes** and environment pattern validation. |
| `CorrelationId` | String           | Optional trace id echoed back in the response and written to **SIMSStartLog**.                                                                         |


**Modes:**

1. **Catalog SQL** — set `LineRecId` to an approved startup line that stores the script in **SQL**. Do **not** rely on ad hoc SQL for routine restore tasks (prefer catalog lines).
2. **Ad hoc SQL** — `LineRecId` = `0` (or omit / null per serializer), fill `SqlScript`, ensure ad hoc is enabled in parameters.

### 3.5 Response contract (`SIMSStartSqlExecuteResponseContract`)

The service returns a **list** with one element (Finance JSON custom service convention). Each element:


| Property        | Type    | Description                                                      |
| --------------- | ------- | ---------------------------------------------------------------- |
| `Success`       | Boolean | `true` if SQL completed without error.                           |
| `Message`       | String  | Human-readable message (rows affected, or error label/text).     |
| `RowsAffected`  | Integer | Rows affected for successful DML/DDL reporting where applicable. |
| `CorrelationId` | String  | Echo of request correlation id.                                  |


### 3.6 Example: execute catalog line by RecId

**Request — POST** to:

`https://{host}/api/services/SIMSStartProcessingServices/SIMSStartSqlExecutionService/executeSql`

**Headers:**

```http
Content-Type: application/json
Authorization: Bearer {access_token}
```

**Body:**

```json
{
  "LineRecId": 5637144576,
  "SqlScript": "",
  "CorrelationId": "logicapp-run-20260505-001"
}
```

Use the real **RecId** from **SIMSStartTable** for your SQL line.

**Sample response (shape):**

```json
[
  {
    "$id": "1",
    "Success": true,
    "Message": "SQL completed. Rows affected: 42.",
    "RowsAffected": 42,
    "CorrelationId": "logicapp-run-20260505-001"
  }
]
```

Exact serialization may include `$id` and metadata depending on platform version; parse the first array element for business fields.

### 3.7 Example: ad hoc SQL (only if enabled)

**Body:**

```json
{
  "LineRecId": 0,
  "SqlScript": "UPDATE dbo.SOMETABLE SET FIELD = 0 WHERE TEMPORARY = 1",
  "CorrelationId": "adhoc-001"
}
```

**Warning:** Ad hoc SQL is powerful. Keep **Allow ad hoc SQL from service** off in production-like tiers; restrict Entra app and network access.

---

## 4. Logic App — call Finance HTTP action (conceptual)

The following is a **pattern**, not a substitute for your tenant’s OAuth settings.

1. Add action **HTTP**.
2. **Method:** `POST`.
3. **URI:** full `executeSql` URL (section 3.1).
4. **Authentication:** pick **Managed identity** or **Active Directory OAuth** as approved by your platform team.
5. **Body:** JSON from sections 3.6–3.7.
6. **Parse JSON** on the response array; take `[0].Success` and `[0].Message` for branching.

If you chain **HTTP Request** (from F&O outbound) → **HTTP** (to F&O inbound), pass `CorrelationId` through variables so end-to-end traces align with **SIMSStartLog**.

---

## 5. Logging and troubleshooting

- **SIMSStartLog** records inbound service calls (correlation id, success flag, and status/details text).
- For **ad hoc SQL** (`SqlScript` path), the log line appends **`AdHocSqlPrefix (first 200 chars)`**: the beginning of the script with newlines collapsed to spaces (truncated after 200 characters). Catalog executions (`LineRecId`) do not append this preview.
- Startup REST actions log via the same logging pipeline as other startup types.
- If `Success` is `false`, read `Message` first; then check **Allowed environment URL pattern**, line approval, task type, and role assignments.

To change the preview length from **200**, adjust `#define.SqlAdhocPreviewLen` in class `SIMSStartSqlExecutionService` and update the label text if you reference the number there.

---

## 6. Client checklist (sandbox / UAT restore)

- Configure **SIMSStartParameters** with a **narrow** allowed FQDN pattern for this tier only.  
- Maintain **SIMSStartTable** lines for SQL steps (masking, delete staging data, enable users, etc.) with correct **Env URL template** and approvals.  
- Configure **REST** lines pointing at the **sandbox-only** Logic App HTTP trigger URL.  
- Register Entra app + Finance user roles for **executeSql** callers.  
- Keep **ad hoc SQL** disabled unless there is a documented exception.  
- Validate **GET** discovery URLs once after release to confirm the service group is deployed.

---

## 7. Support / assumptions

- Endpoint paths follow Microsoft’s **JSON custom services** convention for Finance:  
`/api/services/{ServiceGroup}/{Service}/{Operation}`  
- Behavior (exact JSON casing, array wrapping) follows your Finance platform version; validate with a test call in **Postman** or Logic Apps **Run trigger** before cutover.

For model-specific behavior changes, refer to the **SIMSStartProcessing** model version deployed in your build.