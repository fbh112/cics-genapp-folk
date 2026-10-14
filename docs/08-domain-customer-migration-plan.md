# Customer Domain — Migration Effort Plan

Domain: **Customer Management**
Based on all prior analysis: inventory ([`01`](01-inventory.md)), CRUD matrix ([`02`](02-db2-crud-matrix.md)), domain map ([`03`](03-functional-domains.md)), boundary ([`04`](04-domain-customer-boundary.md)), business rules ([`05`](05-domain-customer-business-rules.md)), hotspots ([`06`](06-domain-customer-migration-hotspots.md), [`07`](07-application-wide-hotspots.md)).

> **Goal**: Migrate the Customer Management domain off the IBM Z mainframe to a target platform (cloud or distributed), while keeping the remaining GenApp domains (Policy, Web Services, Test Harness) operational on mainframe.

---

## 1. Migration Scope

### Programs to migrate (10)

| Program | Lines | Paragraphs | Cyclomatic | SQL operations | VSAM operations |
|---|---|---|---|---|---|
| [`LGACUS01`](../base/src/LGACUS01.cbl) | 90 | 4 | 6 | None | None |
| [`LGACDB01`](../base/src/LGACDB01.cbl) | 187 | 5 | 12 | 2× INSERT, 1× SET ASSIGN | None (calls LGACVS01) |
| [`LGACDB02`](../base/src/LGACDB02.cbl) | 101 | 4 | 7 | 1× INSERT | None |
| [`LGACVS01`](../base/src/LGACVS01.cbl) | 55 | 3 | 5 | None | WRITE |
| [`LGICUS01`](../base/src/LGICUS01.cbl) | 78 | 5 | 6 | None | None |
| [`LGICDB01`](../base/src/LGICDB01.cbl) | 131 | 5 | 9 | 1× SELECT | None |
| [`LGICVS01`](../base/src/LGICVS01.cbl) | 136 | 2 | 13 | None | READ GTEQ |
| [`LGUCUS01`](../base/src/LGUCUS01.cbl) | 79 | 5 | 6 | None | None |
| [`LGUCDB01`](../base/src/LGUCDB01.cbl) | 107 | 5 | 8 | 1× UPDATE | None (calls LGUCVS01) |
| [`LGUCVS01`](../base/src/LGUCVS01.cbl) | 69 | 3 | 6 | None | READ for UPDATE + REWRITE |
| **Total** | **1,033 lines** | **41 paragraphs** | avg 7.8 | 5 SQL stmts across 3 programs | 4 VSAM ops across 3 programs |

### Data stores to migrate or retire

| Store | Type | Decision |
|---|---|---|
| `CUSTOMER` DB2 table | DB2 | **Migrate** — primary customer data store |
| `CUSTOMER_SECURE` DB2 table | DB2 | **Decision required** — write-only in current code; carry forward as stub or retire |
| `KSDSCUST` VSAM KSDS | VSAM | **Retire** — replace with DB query in `LGICVS01` equivalent |

### Components to keep on mainframe (not migrated)

| Component | Reason |
|---|---|
| `LGSTSQ` | Called by Policy domain — must remain; Customer programs must replace their calls |
| `LGCMAREA` copybook | Stays as shared contract; Customer API must be derived from it |
| `LGPOLICY` copybook | Stays for Policy domain; Customer can inline the length constant |
| `POLICY` FK to `CUSTOMER` | The DB2 FK constraint links Policy to Customer — must be managed across the split |
|   |   |

---

## 2. Migration Effort Items — Ranked by Risk

### ME-01 — Define the Customer API Contract 🔴 CRITICAL

**Effort**: Medium | **Risk**: Critical — must be done first

The current interface is a 32,500-byte flat binary COMMAREA (`LGCMAREA`) passed over CICS LINK. Before any code is written, this must be translated into a formal, versioned API contract for the target platform.

**Fields to map** (from `CA-CUSTOMER-REQUEST` in [`LGCMAREA`](../base/src/LGCMAREA.cpy)):

| COMMAREA field | Type | Target API field |
|---|---|---|
| `CA-REQUEST-ID` (6) | Operation code | HTTP method + path or message type |
| `CA-RETURN-CODE` (2) | Result code | HTTP status code or response field |
| `CA-CUSTOMER-NUM` (10) | Customer identifier | `customerId` (integer) |
| `CA-FIRST-NAME` (10) | String | `firstName` |
| `CA-LAST-NAME` (20) | String | `lastName` |
| `CA-DOB` (10) | Date `YYYY-MM-DD` | `dateOfBirth` |
| `CA-HOUSE-NAME` (20) | String | `houseName` |
| `CA-HOUSE-NUM` (4) | String | `houseNumber` |
| `CA-POSTCODE` (8) | String | `postcode` |
| `CA-PHONE-MOBILE` (20) | String | `phoneMobile` |
| `CA-PHONE-HOME` (20) | String | `phoneHome` |
| `CA-EMAIL-ADDRESS` (100) | String | `emailAddress` |
| `CA-NUM-POLICIES` (3) | Counter | `numberOfPolicies` |

**Return codes to preserve** (business rules BR-C-12 to BR-C-16):

| Current RC | Meaning | Target equivalent |
|---|---|---|
| `00` | Success | HTTP 200 |
| `01` | Not found / deadlock | HTTP 404 |
| `98` | COMMAREA too small | HTTP 400 Bad Request |
| `99` | Invalid request ID | HTTP 400 Bad Request |
| `90` | DB2 error | HTTP 500 Internal Server Error |

---

### ME-02 — Re-route Inbound CICS LINKs (2 cut points) 🔴 CRITICAL

**Effort**: Low | **Risk**: Critical — these are the only two integration seams

Exactly **two call sites** on the mainframe must be redirected from CICS LINK to the new API:

| # | Caller program | Domain | Current mechanism | Calls | Migration action |
|---|---|---|---|---|---|
| 1 | [`LGTESTC1`](../base/src/LGTESTC1.cbl) | Test Harness | `EXEC CICS LINK PROGRAM('LGACUS01/LGICUS01/LGUCUS01')` | Add, Inquire, Update Customer | Replace CICS LINKs with calls to new Customer API (via CICS Web Services, CTG, or a COBOL stub that issues HTTP) |
| 2 | [`LGWEBST5`](../base/src/LGWEBST5.cbl) | Web Services | `EXEC CICS LINK` to Customer presentation programs | All three operations | Replace LINKs with forwarding calls to new Customer API; or retire LGWEBST5 for Customer operations and expose the new service's endpoint directly |

**Data passed at cut points**: The COMMAREA (`LGCMAREA` structure) at each CICS LINK — this becomes the request/response payload of the new API call.

---

### ME-03 — Replace LGSTSQ Calls in 10 Programs 🔴 CRITICAL

**Effort**: Low per program, Medium total | **Risk**: Critical — every Customer DB/VSAM program calls it

All 10 Customer programs call [`LGSTSQ`](../base/src/LGSTSQ.cbl) via `EXEC CICS LINK` for error/diagnostic logging. This is a CICS-specific call that cannot run on the target platform.

**Programs affected**: LGACUS01, LGACDB01, LGACDB02, LGACVS01, LGICUS01, LGICDB01, LGUCUS01, LGUCDB01, LGUCVS01 (+ LGICVS01 via shared error path).

**Migration options**:

| Option | Description | Tradeoff |
|---|---|---|
| A | Replace with target platform logger (e.g., `console.log`, cloud logging SDK, structured log write) | Clean; requires editing all 10 programs |
| B | Keep LGSTSQ on mainframe; expose it as a REST endpoint; call from target | Preserves existing log format; adds network hop on every error |
| C | Suppress logging in migrated Customer service; use platform-native observability | Simplest; loses existing log message format |

**Recommended**: Option A — replace inline. The `WRITE-ERROR-MESSAGE` paragraph is structurally identical in all programs; it can be replaced with a single shared logging function/method in the target language.

---

### ME-04 — Replace CICS Named Counter Service (GENACUSTNUM) 🟠 MEDIUM-HIGH

**Effort**: Low | **Risk**: Medium-High

[`LGACDB01`](../base/src/LGACDB01.cbl) uses `EXEC CICS GET COUNTER(GENACUSTNUM) POOL(GENA)` to generate customer numbers. This is a CICS platform service unavailable outside CICS.

**Good news**: The fallback path already exists in the code — when NCS fails (`LGAC-NCS = 'NO'`), it uses `INSERT INTO CUSTOMER VALUES (DEFAULT, …)` and retrieves the DB2 identity-assigned number via `IDENTITY_VAL_LOCAL()`. The DB2 identity column starts at 1,000,001 and increments by 1.

**Migration action**: Remove the NCS path entirely. Use `INSERT … DEFAULT` + `IDENTITY_VAL_LOCAL()` (or equivalent sequence mechanism in the target DB). The existing DB2 DDL already supports this — no schema change required.

---

### ME-05 — Handle the DB2 ↔ VSAM Dual-Write (Non-Atomic) 🟠 MEDIUM-HIGH

**Effort**: Medium | **Risk**: Medium-High

Every Customer write operation (Add, Update) currently writes to both DB2 (`CUSTOMER`) and VSAM (`KSDSCUST`). The two writes are not in the same unit of work.

**Migration action**: **Retire KSDSCUST**. In the target platform, DB2 (or replacement relational store) is the single system of record. The VSAM mirror becomes unnecessary.

**Impact on remaining mainframe programs**:
- `LGICVS01` uses KSDSCUST for random customer lookup. If KSDSCUST is retired, this program must be re-implemented to query the target Customer API for a random customer number (e.g., `SELECT customerNumber FROM CUSTOMER ORDER BY RAND() FETCH FIRST 1 ROW ONLY`), or retired if it only serves test workload purposes.

**Transaction atomicity**: The current non-atomic pattern (BR-C-18 — DB2 updated, VSAM not, if VSAM fails) disappears when the dual-write is retired. The target service need only guarantee atomicity of a single DB write.

---

### ME-06 — Resolve CUSTOMER_SECURE Write-Only Table 🟠 MEDIUM

**Effort**: Low (decision) | **Risk**: Medium

[`LGACDB02`](../base/src/LGACDB02.cbl) inserts a row into `CUSTOMER_SECURE` for every new customer with:
- A hardcoded default password hash (`5732fec825535eeafb8fac50fee3a8aa` — an MD5 hash of the string `NewPass`)
- `state_indicator = 'N'` (new, password not yet changed)
- `pass_changes = 0`

No program ever reads this data.

**Options**:

| Option | Description | Recommendation |
|---|---|---|
| A | **Carry forward**: Include `CUSTOMER_SECURE` in target schema; continue writing on Add | Preserves intended behaviour; keeps open for future auth implementation |
| B | **Retire**: Drop `CUSTOMER_SECURE` from target; remove LGACDB02 equivalent | Simplest migration; loses the security stub |
| C | **Defer**: Keep `CUSTOMER_SECURE` on mainframe only; migrated service does not write it | Splits the table — DB2 FK from `CUSTOMER` remains on mainframe but has no data |

**Recommended**: Option A — carry forward. The table schema and hardcoded default are minimal effort and the `state_indicator = 'N'` flag suggests a future password-change flow was planned. Retiring it may eliminate a security feature that a downstream system already depends on silently.

**FK cascade risk**: `CUSTOMER_SECURE` has `ON DELETE CASCADE` from `CUSTOMER`. If the target platform manages `CUSTOMER` and issues deletes (which the current code never does), the cascade must be replicated or handled explicitly.

---

### ME-07 — Re-implement LGICVS01 Random Customer Lookup 🟠 MEDIUM

**Effort**: Medium | **Risk**: Medium

[`LGICVS01`](../base/src/LGICVS01.cbl) is the most structurally unusual Customer program (cyclomatic 13). It does not perform a standard customer lookup. Instead it:
1. Uses CICS ENQ/DEQ to lock a control TSQ (`GENACNTL`)
2. Reads the TSQ to find the current low/high customer number range
3. Generates a random number using `EIBTASKN` as a seed
4. Reads KSDSCUST with `GTEQ` semantics to find the nearest existing customer

This program exists for **workload simulation** (the test harness uses it to get valid customer numbers to drive transactions). It has no DB2 equivalent and no standard business inquiry path.

**Migration action**: Re-implement as a simple DB2 range query in the target service:
```sql
SELECT customerNumber FROM CUSTOMER
WHERE customerNumber BETWEEN :lowRange AND :highRange
ORDER BY RAND()
FETCH FIRST 1 ROW ONLY
```
The ENQ/DEQ and TSQ range management (`GENACNTL`) is no longer needed — the range is available directly from the DB.

---

### ME-08 — Manage the POLICY FK Dependency Across the Split 🟠 MEDIUM

**Effort**: Medium | **Risk**: Medium

The DB2 `POLICY` table (owned by the Policy domain, staying on mainframe) holds a FK:
```sql
FOREIGN KEY (customerNumber) REFERENCES customer (customerNumber)
```

After migration, `CUSTOMER` moves to the target platform's data store. This **breaks the DB2 referential integrity constraint** — the FK target table no longer exists in the mainframe DB2.

**Options**:

| Option | Description |
|---|---|
| A | **Drop the FK constraint** from the POLICY table; enforce referential integrity at the application level (Policy programs validate customer existence via the Customer API before inserting a policy) |
| B | **Keep a shadow `CUSTOMER` table on mainframe** containing only `customerNumber` (PK); sync it from the migrated Customer service via events or batch feed; POLICY FK references the shadow table |
| C | **Dual-write `CUSTOMER`**: migrated service writes to both target DB and mainframe DB2 `CUSTOMER` table; FK constraint preserved |

**Recommended**: Option A for a demo migration — drop the FK, enforce at application layer. Option B is more robust for production.

---

### ME-09 — Replace CICS GENACNTL TSQ Range Control 🟡 LOW

**Effort**: Low | **Risk**: Low

`LGICVS01` reads and writes a CICS Temporary Storage Queue named `GENACNTL` to store the low/high customer number range used for random lookups. This is a CICS-specific construct.

**Migration action**: Replace with a DB2 range query (see ME-07). The TSQ and its ENQ/DEQ locking become unnecessary.

---

### ME-10 — Handle CICS Abend Codes (LGCA, LGV0, LGV1, LGV2) 🟡 LOW

**Effort**: Low | **Risk**: Low

Customer programs issue CICS abends for critical failures:
- `LGCA` — no COMMAREA received (all presentation programs)
- `LGV0` — VSAM write failure on add ([`LGACVS01`](../base/src/LGACVS01.cbl))
- `LGV1` — VSAM read failure on update ([`LGUCVS01`](../base/src/LGUCVS01.cbl))
- `LGV2` — VSAM rewrite failure on update ([`LGUCVS01`](../base/src/LGUCVS01.cbl))

**Migration action**: Replace with structured exceptions or error responses appropriate to the target platform. `LGV0/1/2` abends disappear naturally when VSAM is retired (ME-05). `LGCA` (no COMMAREA) becomes a standard HTTP 400 Bad Request.

---

## 3. Migration Execution Sequence

The items above are best tackled in the following order to minimise risk:

```
Phase 1 — Prepare (no code changes)
  ├── ME-01: Define Customer API contract from LGCMAREA
  ├── Agree on CUSTOMER_SECURE strategy (ME-06)
  └── Agree on POLICY FK strategy (ME-08)

Phase 2 — Build target Customer service
  ├── Implement Add Customer  (LGACDB01 logic, without NCS — ME-04)
  ├── Implement Inquire Customer  (LGICDB01 logic)
  ├── Implement Update Customer  (LGUCDB01 logic)
  ├── Implement security record write  (LGACDB02 logic — ME-06)
  ├── Replace LGSTSQ calls with target logging  (ME-03)
  ├── Remove KSDSCUST dual-write  (ME-05)
  └── Implement random customer lookup via DB query  (ME-07)

Phase 3 — Cut over
  ├── ME-02: Re-route LGTESTC1 CICS LINKs to new Customer API
  ├── ME-02: Re-route LGWEBST5 CICS LINKs to new Customer API
  └── ME-08: Apply POLICY FK strategy (drop constraint / shadow table / dual-write)

Phase 4 — Clean up
  ├── ME-09: Remove GENACNTL TSQ usage (done in Phase 2)
  ├── ME-10: Verify abend codes replaced with structured responses
  └── Decommission KSDSCUST VSAM file on mainframe
```

---

## 4. Migration Risk Register

| ID | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R-01 | `LGCMAREA` API contract misinterpretation — fields mapped incorrectly to target API | Medium | 🔴 High — all callers break | Validate against all 20 business rules in [`05-domain-customer-business-rules.md`](05-domain-customer-business-rules.md) before coding |
| R-02 | Non-atomic Add Customer (4 steps) leaves partial records on failure | Medium | 🟠 Medium — orphaned CUSTOMER row without CUSTOMER_SECURE | Wrap Add in atomic transaction on target; or implement compensating delete |
| R-03 | POLICY FK breaks after CUSTOMER moves to target DB | High | 🟠 Medium — policy inserts fail silently if customer validation not enforced | Implement application-level customer existence check in Policy domain (ME-08 Option A) |
| R-04 | DB2 ↔ VSAM data divergence on mainframe before cutover | Low | 🟡 Low — existing known risk (BR-C-18) | Reconcile DB2 and KSDSCUST before cutover; retire KSDSCUST cleanly |
| R-05 | `LGICVS01` random lookup semantic not preserved — test harness sends invalid customer numbers | Medium | 🟡 Low — test workload only | Re-implement random query in target API; validate test harness still functions |
| R-06 | Default password hash hardcoded in `LGACDB02` (`5732fec825535eeafb8fac50fee3a8aa`) treated as a real security credential | Low | 🟡 Low — only a stub | Document clearly that this is a placeholder; do not deploy as-is in production |
| R-07 | CICS NCS counter (`GENACUSTNUM`) and DB2 identity sequence produce overlapping numbers if both used during cutover | Low | 🟠 Medium — duplicate customer numbers | Freeze NCS counter; drain in-flight transactions before switching to DB2 identity exclusively |

---

## 5. Summary — Effort Estimate by Category

| Category | Items | Relative effort | Key risk |
|---|---|---|---|
| API contract definition | ME-01 | Medium | Correctness — all callers depend on it |
| Cut-point re-routing | ME-02 | Low | 2 programs, well-defined seam |
| LGSTSQ replacement | ME-03 | Low (per program) × 10 | Mechanical but repetitive across 10 programs |
| NCS → DB2 identity | ME-04 | Low | Fallback path already coded |
| VSAM retirement | ME-05, ME-09 | Low-Medium | LGICVS01 needs re-implementation |
| CUSTOMER_SECURE decision | ME-06 | Low | Decision cost > implementation cost |
| LGICVS01 re-implementation | ME-07 | Medium | Most complex Customer program (cyclomatic 13) |
| POLICY FK management | ME-08 | Medium | Cross-domain DB constraint |
| Abend code replacement | ME-10 | Low | Mechanical replacements |

**Overall assessment**: The Customer domain is a **well-bounded, manageable migration target**. The 10 programs total ~1,033 lines with straightforward SQL (5 statements across 3 programs, no cursors). The highest complexity program (`LGICVS01`, cyclomatic 13) provides workload simulation, not core business logic. The two inbound CICS LINK cut points are clearly defined. The biggest risks are API contract accuracy, POLICY FK management, and the non-atomic Add Customer sequence.
