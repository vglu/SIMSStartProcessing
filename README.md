# SIMS Start Processing

**Dynamics 365 Finance & Operations** extension model that runs **approved, ordered steps when AOS starts** (batch-based startup processor). Typical uses: **sandbox / UAT** refresh workflows—mask data, flip integration endpoints, enable users, trigger external orchestration—**without running the same automation on production by mistake**.

| | |
|---|---|
| **Model version** | **1.0.0** (build **0**), **`VersionRevision` 6** — [`Descriptor/SIMSStartProcessing.xml`](Descriptor/SIMSStartProcessing.xml) |
| **Changelog** | [`CHANGELOG.md`](CHANGELOG.md) |
| **Docs index** | [`docs/README.md`](docs/README.md) |
| **Logic Apps & APIs** | [`docs/SIMSStartProcessing-LogicApps-and-API.md`](docs/SIMSStartProcessing-LogicApps-and-API.md) |

---

## ⚠️ Warning & disclaimer

This model can execute **SQL**, **custom X++**, and **outbound HTTP** in your environment. Misconfiguration can harm data or availability. Use **only** in environments where you accept that risk, with **narrow URL patterns**, **approval workflow**, and **least-privilege** security roles.

**Disclaimer:** Authors and contributors are not liable for damage or loss from use of this software. A short overview video (legacy UI): [YouTube](https://youtu.be/L4anM9fJYiU).

---

## What it does

1. On **system startup**, a subscription ensures a batch job (**AOS Start task processor**) exists.
2. That job walks **`SIMSStartTable`** rows (sorted) where **Run on start** is set.
3. Each row runs only if the **current environment FQDN** matches **Env URL template** *and* the line is **approved** (four-eyes style).
4. **Task type** decides how the step runs: **SQL script**, **Class** (`SIMSStartActionRunClassInterface`), **RunBase** (`SysRunnable`), or **REST API** (outbound `HttpClient` call—for example an **Azure Logic App HTTP trigger**).
5. Execution history goes to **`SIMSStartTrans`** / **`SIMSStartLog`**.

---

## Key capabilities (release 1.0.x)

| Capability | Description |
|------------|-------------|
| **Environment guard** | Wildcard **Env URL template** per line so the same model deployed everywhere does not fire sensitive steps on the wrong host. |
| **Approval** | Lines require approval; admin/sub-admin rules apply (see forms below). |
| **Startup REST call** | Task type **REST API call**: configurable method, URL, headers, body, timeout—use for **F&O → Logic App** notifications. |
| **Inbound SQL JSON service** | Custom service **`SIMSStartSqlExecutionService.executeSql`**: **Logic App → F&O** (or any OAuth client) can run **catalog SQL** (by `SIMSStartTable` RecId) or **ad hoc SQL** when explicitly enabled. See [integration doc](docs/SIMSStartProcessing-LogicApps-and-API.md). |
| **Service parameters** | Form **Service parameters** (`SIMSStartParameters`): **Allowed environment URL pattern** for inbound calls; **Allow ad hoc SQL from service** toggle. |
| **Audit logging** | Inbound calls logged to **`SIMSStartLog`**; ad hoc executions append a **200-character prefix** of the script for traceability. |

---

## Requirements

- Microsoft Dynamics 365 **Finance** / **Supply Chain Management** (F&O) development/deploy target aligned with module references in the descriptor (e.g. **ApplicationFoundation**, **ApplicationPlatform**, **ApplicationSuite**, **Directory**, **ContactPerson**).

---

## Where to click in F&O

- **Menu:** **System administration** → reference menu **SIMS AOT** → items include **AOS run process**, **Service parameters**, etc. (exact translations depend on language.)
- **Direct menu item examples** (adjust host & company):

  - Startup configuration: `...?cmp=usmf&mi=SIMSStartTable`
  - Sub admins: `...?cmp=usmf&mi=SIMSStartSubAdmins`

Screenshots in `./img/` may show an older UI; behavior matches current labels.

---

## Documentation in this repo

| Document | Purpose |
|----------|---------|
| [`CHANGELOG.md`](CHANGELOG.md) | Version history and notable changes. |
| [`docs/SIMSStartProcessing-LogicApps-and-API.md`](docs/SIMSStartProcessing-LogicApps-and-API.md) | Logic Apps integration: outbound REST from F&O, inbound `executeSql`, JSON contracts, OAuth pointers, examples. |

---

## Configuration cheat sheet

### Startup lines (`SIMSStartTable` – “AOS start functionality”)

| Field | Role |
|-------|------|
| **Sort order** | Execution order. |
| **Env URL template** | Must match current environment host (wildcards allowed). |
| **Task type** | SQL / Class / RunBase / **REST API call**. |
| **Run on start** | Include in AOS startup batch. |
| **Start each time** vs **Executed on** | Run once vs every startup. |
| **Continue on error** | Stop the chain or continue. |
| **Approval** | Required before run (except governed admin rules). |

REST-specific fields appear on the **REST request** tab when task type is REST.

### Inbound SQL service (`SIMSStartParameters` – “Service parameters”)

| Field | Role |
|-------|------|
| **Allowed environment URL pattern** | Inbound service rejects calls unless the environment FQDN matches (additional safety vs prod). |
| **Allow ad hoc SQL from service** | When **No**, only **catalog** mode (`LineRecId`) works; when **Yes**, callers may send raw `SqlScript` (still subject to URL pattern). |

Security roles: see duty **SIMSStartProcessingDuty** / role **SIMSStartProcessingUser** and privilege **SIMSStartInboundSqlPrivilege** for maintenance access.

---

## JSON service endpoint (reference)

After deployment, discovery follows Microsoft’s pattern:

`POST https://{environment-host}/api/services/SIMSStartProcessingServices/SIMSStartSqlExecutionService/executeSql`

Full contract, authentication notes, and Logic App samples are in the [integration guide](docs/SIMSStartProcessing-LogicApps-and-API.md).

---

## Building & deploying

1. Use **Visual Studio** with **Finance and Operations** tools; add this repo as a **metadata project** / package matching your branch’s binary references.
2. Build the **SIMSStartProcessing** model; deploy via your pipeline (**RCS / LCS**, **build VM**, or classic **deployable package** flow).

---

## Contributing

Issues and PRs welcome for **documentation clarity**, **safe defaults**, and **tests/linter alignment** with Microsoft guidelines. Breaking or risky behavior changes should be discussed with maintainers.

---

## Buttons & flows (legacy section)

### ▶️ Run!

Runs the **selected** line immediately (same guards as batch: URL + approval).

### 📜 History of transactions / 📘 Common log

Per-line and global logs for approvals, runs, and errors.

### ✅ Approve

Governed approval (admin / sub-admin). See Sub admin form: `mi=SIMSStartSubAdmins` (optional `&user=Admin` for elevated maintenance per your org policy).

---

## Field reference table (startup processing)

| Name | Example | Description |
|------|---------|-------------|
| Description | Mask & enable users | Human-readable step name |
| Sort order | 10 | Order of execution |
| Env URL template | `*sandbox.operations.dynamics.com*` | Template matched against environment FQDN |
| Approved by | Admin | Approver user id |
| Has error | No | Last run failed |
| Task type | SQL script / Class / RunBase / **REST API call** | How the step executes |
| Run on start | Yes | Participate in startup batch |
| Continue on error | Yes | Whether to run following steps if this fails |
| Start each time | No | Run only once vs every startup |
| Executed on | (date) | Last execution date |
| Delete line | Yes | Remove line after successful run |
| SQL script to run | *(SQL)* | For **SQL script** task type |
| Class names | *(class name)* | For **Class** / **RunBase** task types |

REST-specific: **HTTP method**, **Request URL**, **Headers**, **Body**, **Timeout** on the REST tab.
