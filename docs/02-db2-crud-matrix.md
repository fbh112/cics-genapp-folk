# GenApp — DB2 CRUD Matrix

Generated from local metadata database (local scan of `base/src`).

Legend: **C** = INSERT · **R** = SELECT/FETCH (including cursor) · **U** = UPDATE · **D** = DELETE · **—** = not accessed · 📡 = cursor (OPEN/FETCH/CLOSE)

---

## 1. CRUD Matrix by Program

| Program | CUSTOMER | CUSTOMER_SECURE | POLICY | MOTOR | HOUSE | ENDOWMENT | COMMERCIAL |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| [`LGACDB01`](../base/src/LGACDB01.cbl) | **C** | — | — | — | — | — | — |
| [`LGACDB02`](../base/src/LGACDB02.cbl) | — | **C** | — | — | — | — | — |
| [`LGICDB01`](../base/src/LGICDB01.cbl) | **R** | — | — | — | — | — | — |
| [`LGUCDB01`](../base/src/LGUCDB01.cbl) | **U** | — | — | — | — | — | — |
| [`LGAPDB01`](../base/src/LGAPDB01.cbl) | — | — | **C R** | **C** | **C** | **C** | **C** |
| [`LGIPDB01`](../base/src/LGIPDB01.cbl) | — | — | **R** 📡 | **R** | **R** | **R** | **R** 📡 |
| [`LGUPDB01`](../base/src/LGUPDB01.cbl) | — | — | **R U** 📡 | **U** | **U** | **U** | — |
| [`LGDPDB01`](../base/src/LGDPDB01.cbl) | — | — | **D** | — | — | — | — |

> **Note**: No program issues a DELETE against CUSTOMER, MOTOR, HOUSE, ENDOWMENT, or COMMERCIAL directly. POLICY DELETE is done by [`LGDPDB01`](../base/src/LGDPDB01.cbl) only.
> **CUSTOMER_SECURE** is written only by the secure-add path ([`LGACDB02`](../base/src/LGACDB02.cbl)) and never read or updated by any program in this codebase.

---

## 2. CRUD Matrix by CICS Transaction (following call chain)

Each transaction entry-point calls down through orchestrator → DB2 backend programs. The matrix shows net access.

| Transaction | Entry Program | CUSTOMER | CUSTOMER_SECURE | POLICY | MOTOR | HOUSE | ENDOWMENT | COMMERCIAL |
|---|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| `SSC1` | [`LGTESTC1`](../base/src/LGTESTC1.cbl) | **C R U** | **C** | — | — | — | — | — |
| `SSP1` | [`LGTESTP1`](../base/src/LGTESTP1.cbl) | — | — | **C R U D** 📡 | **C R U** | **C R U** | **C R U** | **C R** 📡 |
| `SSP2` | [`LGTESTP2`](../base/src/LGTESTP2.cbl) | — | — | **C R U D** 📡 | **C R U** | **C R U** | **C R U** | **C R** 📡 |
| `SSP3` | [`LGTESTP3`](../base/src/LGTESTP3.cbl) | — | — | **C R U D** 📡 | **C R U** | **C R U** | **C R U** | **C R** 📡 |
| `SSP4` | [`LGTESTP4`](../base/src/LGTESTP4.cbl) | — | — | **C R U D** 📡 | **C R U** | **C R U** | **C R U** | **C R** 📡 |
| `SSST` | [`LGWEBST5`](../base/src/LGWEBST5.cbl) | — | — | — | — | — | — | — |

> `SSP1–SSP4` are four test harness transactions, all driving the same full set of policy operations (Add/Inquire/Update/Delete). They differ only in test data parameters.
> `SSST` (web service) does not issue SQL directly in the scanned codebase.

---

## 3. Table Access Count (programs and domains)

| Table | Programs accessing | Transactions accessing | Notes |
|---|:---:|:---:|---|
| `POLICY` | 4 | 4 | Most-accessed table; hub of all policy operations |
| `CUSTOMER` | 3 | 1 | Customer domain only |
| `MOTOR` | 3 | 4 | Policy sub-type |
| `HOUSE` | 3 | 4 | Policy sub-type |
| `ENDOWMENT` | 3 | 4 | Policy sub-type |
| `COMMERCIAL` | 2 | 4 | Policy sub-type; inquired via cursor |
| `CUSTOMER_SECURE` | 1 | 1 | Orphaned write — never read by any program |

---

## 4. Cursor Usage

| Program | Table | Cursor operation |
|---|---|---|
| [`LGIPDB01`](../base/src/LGIPDB01.cbl) | `POLICY` | DECLARE cursor → OPEN → FETCH → CLOSE |
| [`LGIPDB01`](../base/src/LGIPDB01.cbl) | `COMMERCIAL` | DECLARE cursor → OPEN → FETCH → CLOSE |
| [`LGUPDB01`](../base/src/LGUPDB01.cbl) | `POLICY` | OPEN → FETCH → CLOSE (positioned update) |

---

## 5. Tables Defined but Never Read

| Table | Accessed by | Observation |
|---|---|---|
| `CUSTOMER_SECURE` | [`LGACDB02`](../base/src/LGACDB02.cbl) (INSERT only) | Written on add-customer secure path; no SELECT or UPDATE in any scanned program. If the DB2 DDL defines it, it is a **write-only table** — data written but never retrieved through COBOL. |

---

## 6. Observations and Flags

| # | Observation | Risk |
|---|---|---|
| 1 | **`POLICY` is the most shared table** — 4 programs perform C/R/U/D. Any schema change to POLICY impacts all four programs. | HIGH |
| 2 | **`CUSTOMER_SECURE` is never read** — inserted by `LGACDB02` but no program queries it. Possible dead data path or read is handled outside this COBOL codebase (e.g., a separate report job). | MEDIUM |
| 3 | **No cascading deletes or triggers detected** in the COBOL source. `LGDPDB01` deletes only from `POLICY`; sub-type tables (MOTOR, HOUSE, etc.) are not deleted on policy delete — possible data orphaning. | MEDIUM |
| 4 | **`LGSTSQ`** is called by every DB and VSAM program. It is a universal cross-cutting dependency; any change to it touches all 20+ programs. | HIGH |
| 5 | **`SSP1–SSP4` are identical** in terms of DB2 access — all four test transactions exercise the same full CRUD path. They are likely load/variant test drivers, not functionally distinct operations. | LOW |
