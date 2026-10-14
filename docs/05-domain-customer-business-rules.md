# Customer Management Domain — Business Rules

Domain: **Customer Management**
Extracted from source code analysis of `base/src`. Formatted for publication to Confluence or equivalent wiki.

> **Scope**: Business validations, constraints, calculations, entity relationships, and business outcomes implemented in or inferred from the Customer Management domain programs. Generic error handling and routing logic are excluded unless they represent a business constraint.

---

## Customer Identity and Registration

| ID | Rule | Conditions | Outcome | Implemented in | Data involved | Explicit/Inferred | Verification example |
|---|---|---|---|---|---|---|---|
| BR-C-01 | A customer must be assigned a unique customer number at creation time | Add Customer operation is invoked | A unique integer customer number is generated and returned to the caller in `CA-CUSTOMER-NUM` | [`LGACDB01`](../base/src/LGACDB01.cbl) `Obtain-CUSTOMER-Number` + `INSERT-CUSTOMER` | `CA-CUSTOMER-NUM`, `DB2-CUSTOMERNUM-INT`, DB2 `CUSTOMER.customerNumber` | **Explicit** | Add customer → response must contain a non-zero `CA-CUSTOMER-NUM`; a second add must return a different number |
| BR-C-02 | Customer numbers are generated from a CICS Named Counter Service (NCS) when available; if NCS is unavailable, DB2's identity column generates the number automatically | `LGAC-NCS = 'ON'` → use NCS counter value; `LGAC-NCS = 'NO'` → use `INSERT … VALUES (DEFAULT, …)` and retrieve assigned number via `IDENTITY_VAL_LOCAL()` | Customer is registered with a system-assigned number in either path | [`LGACDB01`](../base/src/LGACDB01.cbl) `Obtain-CUSTOMER-Number`, `INSERT-CUSTOMER` | `GENACUSTNUM` NCS counter (pool `GENA`), `DB2-CUSTOMERNUM-INT`, `IDENTITY_VAL_LOCAL()` | **Explicit** | Disable NCS counter → customer add must still succeed; DB2 identity must assign a number starting from 1,000,001 |
| BR-C-03 | Customer numbers in the DB2 identity sequence start at 1,000,001 and increment by 1 | DB2 identity column definition in DDL | First system-created customer number ≥ 1,000,001 | [`db2cre.jcl`](../base/cntl/db2cre.jcl:111) `CUSTOMER` DDL | `CUSTOMER.customerNumber` identity `(START WITH 1000001, INCREMENT BY 1)` | **Explicit** | Fresh database: first inserted customer has `customerNumber = 1000001` |
| BR-C-04 | Every new customer automatically receives a security record with a default password | Add Customer always calls LGACDB02 after the CUSTOMER insert succeeds | A row is inserted into `CUSTOMER_SECURE` with a hardcoded default password hash, state `N` (new), and pass_changes `0` | [`LGACDB01`](../base/src/LGACDB01.cbl) `MAINLINE` (lines 179–188), [`LGACDB02`](../base/src/LGACDB02.cbl) `INSERT-CUSTOMER-PASSWORD` | `CUSTOMER_SECURE.customerPass = '5732fec825535eeafb8fac50fee3a8aa'`, `state_indicator = 'N'`, `pass_changes = 0` | **Explicit** | Add customer → `CUSTOMER_SECURE` row exists for the new customer number with state `N` and the known default hash |
| BR-C-05 | A customer record must be mirrored in the VSAM KSDS file at the time of creation | Add Customer always writes to KSDSCUST immediately after the DB2 insert | The VSAM file contains a 225-byte customer record keyed on `CA-Customer-Num` | [`LGACDB01`](../base/src/LGACDB01.cbl) `MAINLINE` (lines 174–177), [`LGACVS01`](../base/src/LGACVS01.cbl) `MAINLINE` | `KSDSCUST`, `CA-Customer-Num` (10-byte key), record size 225 bytes | **Explicit** | After add customer: both `CUSTOMER` DB2 row and `KSDSCUST` VSAM record must exist with matching customer number |

---

## Customer Data Fields and Constraints

| ID | Rule | Conditions | Outcome | Implemented in | Data involved | Explicit/Inferred | Verification example |     |
|---|---|---|---|---|---|---|---| --- |
| BR-C-06 | A customer record contains: customer number, first name (10), last name (20), date of birth, house name (20), house number (4), postcode (8), mobile phone (20), home phone (20), email address (100) | Customer add or update operation | All nine non-key fields are stored as provided; no server-side format validation beyond field length | [`LGACDB01`](../base/src/LGACDB01.cbl) `INSERT-CUSTOMER`, [`LGUCDB01`](../base/src/LGUCDB01.cbl) `UPDATE-CUSTOMER-INFO`, [`db2cre.jcl`](../base/cntl/db2cre.jcl:110) | `CA-FIRST-NAME`, `CA-LAST-NAME`, `CA-DOB`, `CA-HOUSE-NAME`, `CA-HOUSE-NUM`, `CA-POSTCODE`, `CA-PHONE-MOBILE`, `CA-PHONE-HOME`, `CA-EMAIL-ADDRESS` | **Explicit** | Add customer with all fields populated → select from DB2 → all nine fields match the input values |     |
| BR-C-07 | All nine mutable customer fields are replaced in their entirety on every update; there is no partial-field update | Update Customer is called with `CA-REQUEST-ID = '01UCUS'` | DB2 `UPDATE CUSTOMER SET col1=:v1, col2=:v2, … WHERE CUSTOMERNUMBER = :n` replaces all fields | [`LGUCDB01`](../base/src/LGUCDB01.cbl) `UPDATE-CUSTOMER-INFO` (lines 155–169) | `CA-FIRST-NAME` … `CA-EMAIL-ADDRESS`, `CUSTOMER.customerNumber` | **Explicit** | Update customer changing only first name → all other fields must retain their pre-update values (caller must supply all fields) |     |
| BR-C-08 | The VSAM record for a customer is also replaced in its entirety on every update | Update Customer always calls LGUCVS01 after the DB2 update | KSDSCUST record is READ for UPDATE then REWRITTEN with the full 225-byte COMMAREA slice | [`LGUCDB01`](../base/src/LGUCDB01.cbl) `MAINLINE` (lines 136–138), [`LGUCVS01`](../base/src/LGUCVS01.cbl) `MAINLINE` | `KSDSCUST`, `CA-Customer-Num` | **Explicit** | Update customer → KSDSCUST record reflects the new values |     |

---

## Customer Lookup

| ID | Rule | Conditions | Outcome | Implemented in | Data involved | Explicit/Inferred | Verification example |
|---|---|---|---|---|---|---|---|
| BR-C-09 | Customer inquiry is by customer number; a non-existent customer number returns return code `01` (not found) | Inquire Customer with a customer number not in `CUSTOMER` | `SQLCODE = 100` → `CA-RETURN-CODE = '01'` | [`LGICDB01`](../base/src/LGICDB01.cbl) `GET-CUSTOMER-INFO` (lines 192–203) | `CA-CUSTOMER-NUM`, `CA-RETURN-CODE`, `CUSTOMER.customerNumber` | **Explicit** | Inquire with customer number 9999999 (does not exist) → `CA-RETURN-CODE = '01'` |
| BR-C-10 | A deadlock or timeout on customer inquiry returns return code `01` (treated equivalently to not found) | `SQLCODE = -913` during SELECT | `CA-RETURN-CODE = '01'` | [`LGICDB01`](../base/src/LGICDB01.cbl) `GET-CUSTOMER-INFO` (line 198) | `CA-RETURN-CODE`, SQLCODE | **Explicit** (code defensive) | Simulate lock contention → caller receives `CA-RETURN-CODE = '01'`, not a system abend |
| BR-C-11 | The VSAM inquire path (`LGICVS01`) returns a **random existing customer number** — it does not look up a specific customer | LGICVS01 is invoked (standalone, not part of the standard Add/Inquire/Update DB2 path) | Reads a random KSDSCUST record using a random key generated from `EIBTASKN` as seed, bounded by low/high customer number range from `GENACNTL` TSQ | [`LGICVS01`](../base/src/LGICVS01.cbl) `MAINLINE` | `KSDSCUST`, `GENACNTL` TSQ, `EIBTASKN` (random seed) | **Explicit** | Call LGICVS01 multiple times → different customer numbers returned each time; all returned numbers must exist in KSDSCUST |

---

## COMMAREA Contract

| ID | Rule | Conditions | Outcome | Implemented in | Data involved | Explicit/Inferred | Verification example |
|---|---|---|---|---|---|---|---|
| BR-C-12 | A COMMAREA must be present on every invocation; absence abends the task with code `LGCA` | `EIBCALEN = 0` on entry to any Customer presentation program | `EXEC CICS ABEND ABCODE('LGCA') NODUMP` | [`LGACUS01`](../base/src/LGACUS01.cbl) line 95–98, [`LGICUS01`](../base/src/LGICUS01.cbl) line 87–90, [`LGUCUS01`](../base/src/LGUCUS01.cbl) line 99–103 | `EIBCALEN`, `CA-RETURN-CODE` | **Explicit** | Call any Customer program with no COMMAREA → CICS task abends with code `LGCA` |
| BR-C-13 | The COMMAREA must be at least `WS-CA-HEADER-LEN (18) + WS-CUSTOMER-LEN (72) = 90 bytes` for Add and Inquire operations; if undersized, return code `98` is returned without processing | `EIBCALEN < 90` | `CA-RETURN-CODE = '98'`, `EXEC CICS RETURN` | [`LGACUS01`](../base/src/LGACUS01.cbl) lines 108–115, [`LGICUS01`](../base/src/LGICUS01.cbl) lines 102–107 | `EIBCALEN`, `WS-REQUIRED-CA-LEN`, `CA-RETURN-CODE` | **Explicit** | Pass 89-byte COMMAREA to LGACUS01 → `CA-RETURN-CODE = '98'` returned immediately |
| BR-C-14 | Update Customer requires `CA-REQUEST-ID = '01UCUS'`; any other request ID returns `CA-RETURN-CODE = '99'` without processing | `CA-REQUEST-ID ≠ '01UCUS'` | `CA-RETURN-CODE = '99'` | [`LGUCUS01`](../base/src/LGUCUS01.cbl) lines 110–113 | `CA-REQUEST-ID`, `CA-RETURN-CODE` | **Explicit** | Call LGUCUS01 with `CA-REQUEST-ID = 'XXXXX'` → `CA-RETURN-CODE = '99'` |
| BR-C-15 | The Add Customer security sub-call uses a separate `CDB2AREA` structure (not `LGCMAREA`), with request ID `02ACUS`; any other request ID to LGACDB02 returns return code `99` | `D2-REQUEST-ID ≠ '02ACUS'` in LGACDB02 | `D2-RETURN-CODE = '99'` | [`LGACDB02`](../base/src/LGACDB02.cbl) lines 143–152 | `D2-REQUEST-ID`, `D2-RETURN-CODE` | **Explicit** | Call LGACDB02 directly with a different request ID → `D2-RETURN-CODE = '99'` |

---

## Error Outcomes

| ID | Rule | Conditions | Outcome | Implemented in | Data involved | Explicit/Inferred | Verification example |
|---|---|---|---|---|---|---|---|
| BR-C-16 | Any DB2 SQL error (non-zero SQLCODE, not SQLCODE 100) during customer operations returns return code `90` and logs an error message | `SQLCODE ≠ 0` and `SQLCODE ≠ 100` | `CA-RETURN-CODE = '90'`, error message written to TSQ via LGSTSQ | [`LGACDB01`](../base/src/LGACDB01.cbl) lines 245–248, [`LGICDB01`](../base/src/LGICDB01.cbl) lines 200–202, [`LGUCDB01`](../base/src/LGUCDB01.cbl) lines 172–179 | `CA-RETURN-CODE`, SQLCODE, `LGSTSQ` TSQ | **Explicit** | Simulate DB2 unavailability → `CA-RETURN-CODE = '90'` |
| BR-C-17 | A VSAM write failure on customer add abends the task with code `LGV0`; a VSAM read failure on update abends with `LGV1`; a VSAM rewrite failure on update abends with `LGV2` | CICS RESP ≠ NORMAL on VSAM operations | CICS task abend (`LGV0`, `LGV1`, or `LGV2`), error logged to LGSTSQ | [`LGACVS01`](../base/src/LGACVS01.cbl) line 79, [`LGUCVS01`](../base/src/LGUCVS01.cbl) lines 81, 94 | `WS-RESP`, `CA-RETURN-CODE (80/81/82)`, `KSDSCUST` | **Explicit** | Simulate VSAM file unavailable → appropriate abend code issued |
| BR-C-18 | Update Customer does not roll back the DB2 update if the subsequent VSAM rewrite fails — the DB2 and VSAM updates are not atomic | LGUCVS01 abends after LGUCDB01's DB2 UPDATE completes successfully | DB2 `CUSTOMER` table updated; KSDSCUST not updated → data inconsistency between DB2 and VSAM | [`LGUCDB01`](../base/src/LGUCDB01.cbl) lines 134–143, [`LGUCVS01`](../base/src/LGUCVS01.cbl) | `CUSTOMER`, `KSDSCUST` | **Inferred** (no compensating transaction) | Force LGUCVS01 VSAM failure mid-update → DB2 row updated, KSDSCUST record not; state divergence exists |

---

## Customer–Policy Relationship

| ID | Rule | Conditions | Outcome | Implemented in | Data involved | Explicit/Inferred | Verification example |
|---|---|---|---|---|---|---|---|
| BR-C-19 | A customer can have zero or more policies; a policy cannot exist without a customer | DB2 FK `POLICY.customerNumber REFERENCES CUSTOMER(customerNumber)` | Inserting a policy for a non-existent customer violates the referential integrity constraint | [`db2cre.jcl`](../base/cntl/db2cre.jcl:174) POLICY DDL FK | `POLICY.customerNumber`, `CUSTOMER.customerNumber` | **Explicit** | Attempt to add a policy with a non-existent customer number → DB2 SQL error, policy not created |
| BR-C-20 | No customer delete operation exists; customers cannot be removed from the system through any implemented operation | No `DELETE FROM CUSTOMER` SQL in any program | Customer records are permanent (append + update only) in this codebase | All Customer programs | `CUSTOMER` table | **Explicit** (by absence) | Confirm: no delete customer API exists; attempting to send a delete request is not supported |
