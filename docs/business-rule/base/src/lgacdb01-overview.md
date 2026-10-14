# Business rules: Overview

**Program:** LGACDB01

## Table of Contents

- [1. Purpose](#1-purpose)
- [2. Function: Customer Record Creation](#2-function-customer-record-creation)
  - [2.1 SubFunction: Request Validation](#21-subfunction-request-validation)
  - [2.2 SubFunction: Customer Number Assignment](#22-subfunction-customer-number-assignment)
  - [2.3 SubFunction: Customer Database Record Insertion](#23-subfunction-customer-database-record-insertion)
  - [2.4 SubFunction: Customer Record Propagation to Secondary Systems](#24-subfunction-customer-record-propagation-to-secondary-systems)
  - [2.5 SubFunction: Customer Security Record Initialisation](#25-subfunction-customer-security-record-initialisation)

## 1. Purpose
This document presents a program-level functional view of the business logic in LGACDB01. Business rules are organised into a Function → SubFunction → Business Rule hierarchy representing the business functions performed in the program. Semantically related rules are consolidated into named SubFunctions that group related business capabilities, and purely technical or setup logic is excluded. Each rule documents its purpose, ordered execution steps with conditions, actions, and outcomes, references to the contributing source paragraphs, and the business significance of each rule.

## 2. Function: Customer Record Creation

### 2.1 SubFunction: Request Validation

#### 2.1.1 Business Rule: Communication Area Length Validation

##### 2.1.1.1 Purpose
Before any customer creation processing begins, the incoming communication area must be verified to contain sufficient data. The minimum acceptable length is calculated as the sum of the fixed header length and the required customer data length. Requests that fall below this threshold are immediately rejected without any database interaction.

##### 2.1.1.2 Rule Steps
1. **Calculate the minimum required communication area length**
   - Conditions: Always executed at transaction entry.
   - Actions: Sum `WS-CA-HEADER-LEN` (fixed header length) and the customer data length into `WS-REQUIRED-CA-LEN`.
   - Outcomes: A baseline threshold is established against which the actual incoming payload is measured.
   - Relevant Paragraphs: `MAINLINE_FIRST_SENTENCES`

2. **Reject the request if the incoming payload is below the minimum length**
   - Conditions: Actual communication area length < `WS-REQUIRED-CA-LEN`.
   - Actions: Set return code `98` in the communication area and return control immediately to the caller; no database operations are initiated.
   - Outcomes: The transaction is aborted early, protecting downstream processing from malformed or truncated input.
   - Relevant Paragraphs: `MAINLINE_FIRST_SENTENCES`

##### 2.1.1.3 Business Significance
Failing to enforce this guard allows malformed or incomplete customer requests to reach the database layer, risking partial inserts, data corruption, or unhandled SQL exceptions. Return code `98` provides the caller with an unambiguous rejection signal for retry or error-escalation handling.

---

### 2.2 SubFunction: Customer Number Assignment

#### 2.2.1 Business Rule: Unique Customer Number Generation with Fallback Strategy

##### 2.2.1.1 Purpose
Every new customer must receive a system-unique primary key before a database record is created. The primary mechanism is the CICS Named Counter Service (pool `GENA`, counter `GENACUSTNUM`). If that service is unavailable, the system falls back to DB2's built-in IDENTITY column auto-assignment, ensuring that customer number generation is never a single point of failure.

##### 2.2.1.2 Rule Steps
1. **Request the next sequential customer number from the Named Counter Service**
   - Conditions: Executed unconditionally as the first step of customer number acquisition.
   - Actions: Issue a CICS Named Counter GET against pool `GENApool` / counter `GENAcount`; store the returned value in `LastCustNum`; capture the service response in `WS-RESP`.
   - Outcomes: A candidate customer number is obtained or a failure response is recorded.
   - Relevant Paragraphs: `MAINLINE_OBTAIN-CUSTOMER-NUMBER`

2. **Evaluate the Named Counter Service response and select the assignment strategy**
   - Conditions: `WS-RESP` is compared to `DFHRESP(NORMAL)`.
   - Actions:
     - **Success path**: Move `LastCustNum` into `DB2-CUSTOMERNUM-INT`; leave `LGAC-NCS` set to its default `ON` state, signalling that an explicit customer number will be used for the INSERT.
     - **Failure path**: Set `LGAC-NCS` to `NO` and initialise `DB2-CUSTOMERNUM-INT` to zero, signalling that DB2 IDENTITY DEFAULT will govern number assignment.
   - Outcomes: The `LGAC-NCS` flag unambiguously controls which INSERT path is executed in the subsequent database step.
   - Relevant Paragraphs: `MAINLINE_OBTAIN-CUSTOMER-NUMBER`

##### 2.2.1.3 Business Significance
An incorrect or non-unique customer number assignment would cause primary key violations, duplicate customer records, or lost customer data. The dual-path strategy ensures business continuity when the Named Counter Service is degraded, while still producing a traceable, unique customer identifier through either mechanism.

---

### 2.3 SubFunction: Customer Database Record Insertion

#### 2.3.1 Business Rule: Customer Record Insert with Dual-Path Number Handling

##### 2.3.1.1 Purpose
A new customer record is written to the `CUSTOMER` table using one of two INSERT strategies determined by the outcome of the Named Counter Service call. Regardless of strategy, the transaction must either complete a successful INSERT and recover the assigned customer number, or abort with a clearly communicated error code. The assigned customer number is always returned to the caller.

##### 2.3.1.2 Rule Steps
1. **Insert with explicit customer number (Named Counter path)**
   - Conditions: `LGAC-NCS` = `ON` (Named Counter Service succeeded).
   - Actions: Execute a DB2 INSERT into the `CUSTOMER` table using `DB2-CUSTOMERNUM-INT` as the explicit primary key, together with all personal details from the communication area.
   - Outcomes: A customer record is created with a pre-determined, traceable primary key.
   - Relevant Paragraphs: `MAINLINE_INSERT-CUSTOMER`

2. **Insert with DB2 IDENTITY DEFAULT (fallback path)**
   - Conditions: `LGAC-NCS` ≠ `ON` (Named Counter Service was unavailable).
   - Actions: Execute a DB2 INSERT into the `CUSTOMER` table using `DEFAULT` for the customer number column, allowing DB2 to auto-generate the key; then immediately issue `IDENTITY_VAL_LOCAL()` to retrieve the generated value into `DB2-CUSTOMERNUM-INT`.
   - Outcomes: A customer record is created with a DB2-generated primary key that is captured for downstream use.
   - Relevant Paragraphs: `MAINLINE_INSERT-CUSTOMER`

3. **Handle INSERT failure on either path**
   - Conditions: SQLCODE ≠ 0 after either INSERT attempt.
   - Actions: Set return code `90` in the communication area; invoke `MAINLINE_WRITE-ERROR-MESSAGE` to log the failure; terminate the transaction immediately without proceeding to security record creation.
   - Outcomes: The transaction is rolled back / abandoned; the caller receives return code `90` as an unambiguous database error signal.
   - Relevant Paragraphs: `MAINLINE_INSERT-CUSTOMER`

4. **Propagate the assigned customer number to the communication area**
   - Conditions: INSERT succeeded on either path and `DB2-CUSTOMERNUM-INT` holds the authoritative customer number.
   - Actions: Move `DB2-CUSTOMERNUM-INT` into the corresponding field in `DFHCOMMAREA`.
   - Outcomes: The caller receives the newly assigned customer number, enabling downstream reference and confirmation.
   - Relevant Paragraphs: `MAINLINE_INSERT-CUSTOMER`

##### 2.3.1.3 Business Significance
If the assigned customer number is not correctly persisted or returned, all subsequent policy, security, and VSAM records created in the same transaction would reference an incorrect or unknown identifier, leading to orphaned records and broken customer relationships across storage systems.

---

### 2.4 SubFunction: Customer Record Propagation to Secondary Systems

#### 2.4.1 Business Rule: VSAM Customer Record Synchronisation via LGACVS01

##### 2.4.1.1 Purpose
After the DB2 customer INSERT succeeds, the transaction must also create a corresponding customer record in the VSAM-based storage layer. This is achieved by linking to program `LGACVS01` and passing the full communication area (`DFHCOMMAREA`), ensuring that both DB2 and VSAM storage systems hold consistent customer data.

##### 2.4.1.2 Rule Steps
1. **Transfer control to LGACVS01 for VSAM record creation**
   - Conditions: DB2 INSERT completed successfully.
   - Actions: Issue a CICS LINK to `LGACVS01` passing `DFHCOMMAREA` (containing all customer input data including the newly assigned customer number).
   - Outcomes: A VSAM customer record is created, synchronising the customer's existence across both storage systems.
   - Relevant Paragraphs: `MAINLINE_FIRST_SENTENCES`

##### 2.4.1.3 Business Significance
Omitting or failing the VSAM synchronisation step would leave the customer present in DB2 but absent from VSAM, causing read inconsistencies for any business process that queries the VSAM store — including policy administration and customer lookup functions.

---

### 2.5 SubFunction: Customer Security Record Initialisation

#### 2.5.1 Business Rule: Mandatory Security Record Creation for New Customers

##### 2.5.1.1 Purpose
Every newly created customer must have an associated security record provisioned in the same transaction. The security record is initialised with a default password, a zero security count, and an inactive state (`N`). This is accomplished by populating the `CDB2AREA` communication structure and linking to program `LGACDB02` with request type `02ACUS`.

##### 2.5.1.2 Rule Steps
1. **Populate the security record communication structure**
   - Conditions: DB2 and VSAM customer inserts have completed successfully; `DB2-CUSTOMERNUM-INT` holds the authoritative new customer number.
   - Actions: Move `DB2-CUSTOMERNUM-INT` into `D2-CUSTOMER-NUM`; set `D2-REQUEST-ID` to `02ACUS`; populate `CDB2AREA` with the default initial password, a security count of `0000`, and a security state of `N`.
   - Outcomes: `CDB2AREA` is fully prepared to instruct `LGACDB02` to create a new customer security record.
   - Relevant Paragraphs: `MAINLINE_FIRST_SENTENCES`

2. **Link to LGACDB02 to create the security record**
   - Conditions: `CDB2AREA` is populated with all required security initialisation values.
   - Actions: Issue a CICS LINK to `LGACDB02` passing `CDB2AREA`.
   - Outcomes: A customer security record is created in the security database, associated with the newly assigned customer number.
   - Relevant Paragraphs: `MAINLINE_FIRST_SENTENCES`

##### 2.5.1.3 Business Significance
A customer record without a corresponding security record leaves the account in an unprotectable state — authentication, access control, and audit trail functions would all fail or be bypassed for that customer. The default inactive state (`N`) and zero count ensure the account requires explicit activation before it can be used, enforcing a controlled onboarding policy.

---

Generated by IBM Bob Premium Package for Z
