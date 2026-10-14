# GenApp Application Inventory

Generated from local metadata database (local scan of `base/src`).

---

## 1. Program Catalogue

### Naming Convention

GenApp program names follow a consistent pattern:

| Segment | Position | Meaning |
|---|---|---|
| `LG` | 1–2 | Application prefix (Liberty GenApp) |
| `A/I/U/D` | 3 | Operation: **A**dd, **I**nquire, **U**pdate, **D**elete |
| `C/P` | 4 | Entity: **C**ustomer, **P**olicy |
| `OL/DB/VS/US/ST` | 5–6 | Layer: **OL**=orchestrator, **DB**=DB2 backend, **VS**=VSAM backend, **US**=UI/presentation, **ST**=statistics |
| `01` | 7–8 | Instance number |

---

### 1.1 Online CICS Programs (31 total)

#### Customer Operations

| Program | Paragraphs | Role | Layer |
|---|---|---|---|
| [`LGACUS01`](../base/src/LGACUS01.cbl) | 4 | Add Customer — UI/presenter | Presentation |
| [`LGACDB01`](../base/src/LGACDB01.cbl) | 5 | Add Customer — DB2 backend | DB2 |
| [`LGACDB02`](../base/src/LGACDB02.cbl) | 4 | Add Customer (secure path) — DB2 backend | DB2 |
| [`LGACVS01`](../base/src/LGACVS01.cbl) | 3 | Add Customer — VSAM backend | VSAM |
| [`LGICUS01`](../base/src/LGICUS01.cbl) | 5 | Inquire Customer — UI/presenter | Presentation |
| [`LGICDB01`](../base/src/LGICDB01.cbl) | 5 | Inquire Customer — DB2 backend | DB2 |
| [`LGICVS01`](../base/src/LGICVS01.cbl) | 2 | Inquire Customer — VSAM backend | VSAM |
| [`LGUCUS01`](../base/src/LGUCUS01.cbl) | 5 | Update Customer — UI/presenter | Presentation |
| [`LGUCDB01`](../base/src/LGUCDB01.cbl) | 5 | Update Customer — DB2 backend | DB2 |
| [`LGUCVS01`](../base/src/LGUCVS01.cbl) | 3 | Update Customer — VSAM backend | VSAM |

#### Policy Operations

| Program | Paragraphs | Role | Layer |
|---|---|---|---|
| [`LGAPOL01`](../base/src/LGAPOL01.cbl) | 3 | Add Policy — orchestrator | Orchestrator |
| [`LGAPDB01`](../base/src/LGAPDB01.cbl) | 8 | Add Policy — DB2 backend | DB2 |
| [`LGAPVS01`](../base/src/LGAPVS01.cbl) | 3 | Add Policy — VSAM backend | VSAM |
| [`LGIPOL01`](../base/src/LGIPOL01.cbl) | 3 | Inquire Policy — orchestrator | Orchestrator |
| [`LGIPDB01`](../base/src/LGIPDB01.cbl) | 13 | Inquire Policy — DB2 backend | DB2 |
| [`LGIPVS01`](../base/src/LGIPVS01.cbl) | 2 | Inquire Policy — VSAM backend | VSAM |
| [`LGUPOL01`](../base/src/LGUPOL01.cbl) | 5 | Update Policy — orchestrator | Orchestrator |
| [`LGUPDB01`](../base/src/LGUPDB01.cbl) | 10 | Update Policy — DB2 backend | DB2 |
| [`LGUPVS01`](../base/src/LGUPVS01.cbl) | 3 | Update Policy — VSAM backend | VSAM |
| [`LGDPOL01`](../base/src/LGDPOL01.cbl) | 4 | Delete Policy — orchestrator | Orchestrator |
| [`LGDPDB01`](../base/src/LGDPDB01.cbl) | 4 | Delete Policy — DB2 backend | DB2 |
| [`LGDPVS01`](../base/src/LGDPVS01.cbl) | 3 | Delete Policy — VSAM backend | VSAM |

#### Test Harness / Workload Programs

| Program | Paragraphs | Transaction | Role |
|---|---|---|---|
| [`LGTESTC1`](../base/src/LGTESTC1.cbl) | 10 | `SSC1` | Test driver — Customer operations (Add/Inquire/Update) |
| [`LGTESTP1`](../base/src/LGTESTP1.cbl) | 10 | `SSP1` | Test driver — Policy operations (Add/Inquire/Update/Delete) |
| [`LGTESTP2`](../base/src/LGTESTP2.cbl) | 10 | `SSP2` | Test driver — Policy operations (variant 2) |
| [`LGTESTP3`](../base/src/LGTESTP3.cbl) | 10 | `SSP3` | Test driver — Policy operations (variant 3) |
| [`LGTESTP4`](../base/src/LGTESTP4.cbl) | 10 | `SSP4` | Test driver — Policy operations (variant 4) |

#### Infrastructure / Utility Programs

| Program | Paragraphs | Transaction | Role |
|---|---|---|---|
| [`LGSETUP`](../base/src/LGSETUP.cbl) | 2 | — | VSAM/Db2 setup utility |
| [`LGSTSQ`](../base/src/LGSTSQ.cbl) | 2 | — | Shared TSQ (Temporary Storage Queue) logging utility — called by virtually every program |
| [`LGASTAT1`](../base/src/LGASTAT1.cbl) | 2 | — | Statistics accumulator (batch-like) |
| [`LGWEBST5`](../base/src/LGWEBST5.cbl) | 4 | `SSST` | Web service entry-point (CICS web services) |

---

## 2. Copybooks

| Copybook | Used by | Purpose |
|---|---|---|
| [`LGCMAREA`](../base/src/LGCMAREA.cpy) | All programs except `LGWEBST5` | COMMAREA layout — the universal inter-program communication area |
| [`LGPOLICY`](../base/src/LGPOLICY.cpy) | LGACDB01, LGACDB02, LGAPDB01, LGICDB01, LGICUS01, LGIPDB01, LGIPOL01, LGUCDB01, LGUPDB01 | Policy data structures |
| [`SQLCA`](../base/src/SQLCA.cpy) | LGACDB01, LGACDB02, LGAPDB01, LGICDB01, LGIPDB01, LGUCDB01, LGUPDB01, LGDPDB01 | SQL Communication Area — embedded SQL error handling |
| [`SSMAP`](../base/src/ssmap.bms) | LGTESTC1, LGTESTP1, LGTESTP2, LGTESTP3, LGTESTP4 | BMS screen map — test harness display |

---

## 3. BMS Screen

| Map | Used by | Purpose |
|---|---|---|
| `SSMAP` (in [`ssmap.bms`](../base/src/ssmap.bms)) | All `LGTEST*` programs | Interactive test screen for driving customer and policy operations |

---

## 4. DB2 Tables

| Table | Purpose |
|---|---|
| `CUSTOMER` | Core customer records |
| `CUSTOMER_SECURE` | Sensitive/secure customer data (separate table) |
| `POLICY` | Policy header records |
| `MOTOR` | Motor insurance policy details |
| `HOUSE` | House/property insurance policy details |
| `ENDOWMENT` | Endowment insurance policy details |
| `COMMERCIAL` | Commercial insurance policy details |

---

## 5. VSAM Files

| Logical Name | Physical Name | Purpose |
|---|---|---|
| `KSDSCUST` | `KSDSCUST` | KSDS VSAM file — Customer records (alternate data store) |
| `KSDSPOLY` | `KSDSPOLY` | KSDS VSAM file — Policy records (alternate data store) |

---

## 6. Transactions (CICS)

| Transaction ID | Entry-point Program | Purpose |
|---|---|---|
| `SSC1` | `LGTESTC1` | Test harness — Customer CRUD |
| `SSP1` | `LGTESTP1` | Test harness — Policy CRUD (variant 1) |
| `SSP2` | `LGTESTP2` | Test harness — Policy CRUD (variant 2) |
| `SSP3` | `LGTESTP3` | Test harness — Policy CRUD (variant 3) |
| `SSP4` | `LGTESTP4` | Test harness — Policy CRUD (variant 4) |
| `SSST` | `LGWEBST5` | Web service entry-point |

---

## 7. JCL Jobs (base/cntl)

| JCL Member | Purpose |
|---|---|
| `cobol.jcl` | Compile and link-edit all COBOL programs |
| `asmmap.jcl` | Assemble BMS map |
| `db2cre.jcl` | Create Db2 tables (DDL) |
| `db2del.jcl` | Drop Db2 tables |
| `db2bind.jcl` | Bind Db2 DBRM packages |
| `adef121–adef125.jcl` | CICS resource definitions (programs, transactions, files) |
| `cdef121–cdef125.jcl` | CPSM/CICSPlex resource definitions |
| `cpsmde2.jcl` | CICSPlex dynamic routing definitions |
| `defdrep.jcl` | Define CICS event bindings (Data Repository) |
| `defwrep.jcl` | Define CICS event bindings (Write Repository) |
| `itpentr.jcl` / `itpll.jcl` / `itpstl.jcl` | ITP (instrumentation) setup |
| `sampcma.jcl` / `sampncs.jcl` / `samptsq.jcl` / `sampwui.jcl` | Sample execution scripts |
| `wsaac01–wsavp01.jcl` | Web Service Assistant generation jobs (one per operation) |

---

## 8. Program Dependencies (Call Graph)

### Calls from CICS LINK / DYNAMIC_LINK

```
LGTESTC1 (SSC1)
  ├── LGACUS01  ──> LGACDB01 ──> LGACDB02 ──> LGSTSQ
  │                          └──> LGACVS01 ──> LGSTSQ
  │                          └──> LGSTSQ
  ├── LGICUS01  ──> LGICDB01 ──> LGSTSQ
  │                          └──> (LGICVS01 — VSAM read, no sub-call)
  └── LGUCUS01  ──> LGUCDB01 ──> LGUCVS01 ──> LGSTSQ
                             └──> LGSTSQ

LGTESTP1/2/3/4 (SSP1–SSP4)
  ├── LGAPOL01  ──> LGAPDB01 ──> LGAPVS01 ──> LGSTSQ
  │                          └──> LGSTSQ
  ├── LGIPOL01  ──> LGIPDB01 ──> LGSTSQ
  ├── LGUPOL01  ──> LGUPDB01 ──> LGUPVS01 ──> LGSTSQ
  │                          └──> LGSTSQ
  └── LGDPOL01  ──> LGDPDB01 ──> LGDPVS01 ──> LGSTSQ
                             └──> LGSTSQ

LGWEBST5 (SSST)
  └── (standalone — web service entry, no LINK calls in DB)
```

### LGSTSQ — Universal Shared Utility

[`LGSTSQ`](../base/src/LGSTSQ.cbl) is called (via CICS LINK) by **every** DB and VSAM tier program.
It writes diagnostic/audit messages to a CICS Temporary Storage Queue (TSQ).

---

## 9. Dependency Summary per Program

| Program | Copybooks | BMS | DB2 Tables (mode) | VSAM Files (mode) | Calls | Called by |
|---|---|---|---|---|---|---|
| `LGACUS01` | LGCMAREA, LGPOLICY | — | — | — | LGACDB01, LGSTSQ | LGTESTC1 |
| `LGACDB01` | LGCMAREA, LGPOLICY, SQLCA | — | CUSTOMER (W) | — | LGACDB02, LGACVS01, LGSTSQ | LGACUS01 |
| `LGACDB02` | LGPOLICY, SQLCA | — | CUSTOMER_SECURE (W) | — | LGSTSQ | LGACDB01 |
| `LGACVS01` | LGCMAREA | — | — | KSDSCUST (W) | LGSTSQ | LGACDB01 |
| `LGICUS01` | LGCMAREA, LGPOLICY | — | — | — | LGICDB01, LGSTSQ | LGTESTC1 |
| `LGICDB01` | LGCMAREA, LGPOLICY, SQLCA | — | CUSTOMER (R) | — | LGSTSQ | LGICUS01 |
| `LGICVS01` | — | — | — | KSDSCUST (R) | — | — |
| `LGUCUS01` | LGCMAREA | — | — | — | LGUCDB01, LGSTSQ | LGTESTC1 |
| `LGUCDB01` | LGCMAREA, LGPOLICY, SQLCA | — | CUSTOMER (W/UPDATE) | — | LGUCVS01, LGSTSQ | LGUCUS01 |
| `LGUCVS01` | LGCMAREA | — | — | KSDSCUST (R+W) | LGSTSQ | LGUCDB01 |
| `LGAPOL01` | LGCMAREA | — | — | — | LGAPDB01, LGSTSQ | LGTESTP1–4 |
| `LGAPDB01` | LGCMAREA, LGPOLICY, SQLCA | — | POLICY (R+W), MOTOR (W), HOUSE (W), ENDOWMENT (W), COMMERCIAL (W) | — | LGAPVS01, LGSTSQ | LGAPOL01 |
| `LGAPVS01` | LGCMAREA | — | — | KSDSPOLY (W) | LGSTSQ | LGAPDB01 |
| `LGIPOL01` | LGCMAREA, LGPOLICY | — | — | — | LGIPDB01, LGSTSQ | LGTESTP1–4 |
| `LGIPDB01` | LGCMAREA, LGPOLICY, SQLCA | — | POLICY (R×cursor), MOTOR (R), HOUSE (R), ENDOWMENT (R), COMMERCIAL (R×cursor) | — | LGSTSQ | LGIPOL01 |
| `LGIPVS01` | — | — | — | KSDSPOLY (R) | — | — |
| `LGUPOL01` | LGCMAREA | — | — | — | LGUPDB01, LGSTSQ | LGTESTP1–4 |
| `LGUPDB01` | LGCMAREA, LGPOLICY, SQLCA | — | POLICY (R+W), MOTOR (W), HOUSE (W), ENDOWMENT (W) | — | LGUPVS01, LGSTSQ | LGUPOL01 |
| `LGUPVS01` | LGCMAREA | — | — | KSDSPOLY (R+W) | LGSTSQ | LGUPDB01 |
| `LGDPOL01` | LGCMAREA | — | — | — | LGDPDB01, LGSTSQ | LGTESTP1–4 |
| `LGDPDB01` | LGCMAREA, SQLCA | — | POLICY (W/DELETE) | — | LGDPVS01, LGSTSQ | LGDPOL01 |
| `LGDPVS01` | LGCMAREA | — | — | KSDSPOLY (W/DELETE) | LGSTSQ | LGDPDB01 |
| `LGTESTC1` | LGCMAREA, SSMAP | SSMAP | — | — | LGACUS01, LGICUS01, LGUCUS01 | — (entry: SSC1) |
| `LGTESTP1` | LGCMAREA, SSMAP | SSMAP | — | — | LGAPOL01, LGIPOL01, LGUPOL01, LGDPOL01 | — (entry: SSP1) |
| `LGTESTP2` | LGCMAREA, SSMAP | SSMAP | — | — | LGAPOL01, LGIPOL01, LGUPOL01, LGDPOL01 | — (entry: SSP2) |
| `LGTESTP3` | LGCMAREA, SSMAP | SSMAP | — | — | LGAPOL01, LGIPOL01, LGUPOL01, LGDPOL01 | — (entry: SSP3) |
| `LGTESTP4` | LGCMAREA, SSMAP | SSMAP | — | — | LGAPOL01, LGIPOL01, LGUPOL01, LGDPOL01 | — (entry: SSP4) |
| `LGSTSQ` | — | — | — | — | — | All DB + VSAM programs |
| `LGSETUP` | — | — | — | — | — | — (utility) |
| `LGASTAT1` | LGCMAREA | — | — | — | — | — (statistics) |
| `LGWEBST5` | — | — | — | — | — | — (entry: SSST) |
