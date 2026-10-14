# Business rules: Inventory

**Program:** LGACDB01

**Total rules extracted:** 9

## Table of Contents

- [1. Purpose](#1-purpose)
- [2. Rule Inventory](#2-rule-inventory)
  - [2.1 Paragraph: MAINLINE_FIRST_SENTENCES](#21-paragraph-mainline_first_sentences)
  - [2.2 Paragraph: MAINLINE_OBTAIN-CUSTOMER-NUMBER](#22-paragraph-mainline_obtain-customer-number)
  - [2.3 Paragraph: MAINLINE_INSERT-CUSTOMER](#23-paragraph-mainline_insert-customer)

## 1. Purpose

This document is an inventory of all business rules extracted from lgacdb01, organised at the Paragraph level. Each Paragraph contributes one or more named rule blocks, where each block captures a discrete business policy with its title, decision logic, conditions, actions, outcomes, and the exact source code that realises it. Key variables, when available, are annotated with their role (input or output) and their specific purpose within the rule. References to related Paragraphs called or invoked by each rule are also recorded.

## 2. Rule Inventory

### 2.1 Paragraph: MAINLINE_FIRST_SENTENCES

#### 2.1.1 Validate Communication Area Length Against Required Minimum

**Description:** Calculates the minimum required communication area length by summing the header length and customer data length, then rejects the request with return code '98' if the actual incoming communication area is shorter than the required minimum.

**Core decision logic:**
- If the incoming communication area length is less than the sum of the header length and customer data length, set return code '98' and immediately return to the caller without processing.

**Code:**
```cobol
      * check commarea length
           ADD WS-CA-HEADER-LEN TO WS-REQUIRED-CA-LEN
           ADD WS-CUSTOMER-LEN  TO WS-REQUIRED-CA-LEN

      * if less set error return code and return to caller
           IF EIBCALEN IS LESS THAN WS-REQUIRED-CA-LEN
             MOVE '98' TO CA-RETURN-CODE
             EXEC CICS RETURN END-EXEC
           END-IF
```

**Variables:**

| Variable name | Role | Purpose |
|---|---|---|
| `WS-CA-HEADER-LEN` | input | Provides the fixed byte length of the communication area header, added to the customer data length to compute the total minimum required length. |
| `WS-REQUIRED-CA-LEN` | output | Accumulates the minimum required communication area length used to validate the incoming data is sufficient for processing. |

#### 2.1.2 Invoke Customer Number Retrieval and Customer Record Insertion

**Description:** Orchestrates the core customer creation workflow by first obtaining a unique customer number and then inserting the customer record into the database.

**Core decision logic:**
- A unique customer number must be obtained before attempting to insert the customer record.
- Customer number retrieval and customer record insertion are mandatory sequential steps in the add-customer business process.

**Code:**
```cobol
      * Call routine to Insert row in Customer table                   *        
           PERFORM Obtain-CUSTOMER-Number.
           PERFORM INSERT-CUSTOMER.
```

**Paragraphs:**

* MAINLINE_OBTAIN-CUSTOMER-NUMBER
* MAINLINE_INSERT-CUSTOMER
#### 2.1.3 Link to Policy Administration Program for Customer Record Processing

**Description:** Transfers control to the LGACVS01 program passing the full communication area to perform additional customer record processing (e.g., VSAM-based policy/customer data handling) as part of the add-customer business transaction.

**Core decision logic:**
- After the DB2 customer insert, the add-customer transaction must also invoke LGACVS01 to complete the customer record lifecycle across storage systems.

**Code:**
```cobol
           EXEC CICS LINK Program(LGACVS01)
                Commarea(DFHCOMMAREA)
                LENGTH(225)
           END-EXEC.
```

**Variables:**

| Variable name | Role | Purpose |
|---|---|---|
| `DFHCOMMAREA` | input | Carries all customer input data to LGACVS01 for further customer record processing. |

#### 2.1.4 Prepare and Invoke Customer Security Record Creation

**Description:** Populates the CDB2AREA communication structure with the newly assigned customer number, request type '02ACUS', initial security password, security count, and security state, then links to LGACDB02 to create the customer security record as part of the add-customer business process.

**Core decision logic:**
- Every newly created customer must have an associated security record created in LGACDB02 immediately after the customer INSERT.
- The security record is initialised with a default password, a count of '0000', and a state of 'N' (inactive/new).
- The request identifier '02ACUS' designates this as an Add Customer security operation for LGACDB02.

**Code:**
```cobol
           MOVE DB2-CUSTOMERNUM-INT TO D2-CUSTOMER-NUM.
           Move '02ACUS'     To  D2-REQUEST-ID.
           move '5732fec825535eeafb8fac50fee3a8aa'
                             To  D2-CUSTSECR-PASS.
           Move '0000'       To  D2-CUSTSECR-COUNT.
           Move 'N'          To  D2-CUSTSECR-STATE.

           EXEC CICS LINK Program(LGACDB02)
                Commarea(CDB2AREA)
                LENGTH(32500)
           END-EXEC.
```

**Variables:**

| Variable name | Role | Purpose |
|---|---|---|
| `DB2-CUSTOMERNUM-INT` | input | Provides the newly assigned customer number to be conveyed to LGACDB02 for security record creation. |
| `D2-CUSTOMER-NUM` | output | Receives the newly assigned customer number within CDB2AREA to identify the customer for the security record operation. |
| `D2-REQUEST-ID` | output | Set to '02ACUS' to signal to LGACDB02 that this is an Add Customer security record request. |
| `CDB2AREA` | output | The communication structure populated with customer number, request type, password, count, and state before being passed to LGACDB02. |

### 2.2 Paragraph: MAINLINE_OBTAIN-CUSTOMER-NUMBER

#### 2.2.1 Retrieve Unique Customer Number from Named Counter Service

**Description:** Requests the next sequential customer number from the CICS Named Counter Service using the GENA pool and GENACUSTNUM counter, storing the retrieved value for use as the new customer's primary key.

**Core decision logic:**
- The primary mechanism for assigning a unique customer number is the CICS Named Counter Service using pool 'GENA' and counter 'GENACUSTNUM'.

**Code:**
```cobol
           Exec CICS Get Counter(GENAcount)
                         Pool(GENApool)
                         Value(LastCustNum)
                         Resp(WS-RESP)
           End-Exec.
```

**Variables:**

| Variable name | Role | Purpose |
|---|---|---|
| `GENAcount` | input | Identifies the specific named counter (GENACUSTNUM) to retrieve from the CICS Named Counter Service. |
| `GENApool` | input | Identifies the CICS Named Counter pool (GENA) from which the customer number counter is drawn. |
| `LastCustNum` | output | Receives the current counter value returned by the CICS Named Counter Service, representing the next available customer number. |
| `WS-RESP` | output | Captures the CICS response code to determine whether the counter retrieval succeeded or failed. |

#### 2.2.2 Determine Customer Number Assignment Strategy Based on Counter Response

**Description:** Evaluates whether the Named Counter Service call succeeded. On failure, sets the LGAC-NCS flag to 'NO' to trigger the DB2 IDENTITY DEFAULT insert path. On success, assigns the counter value as the explicit customer number for the DB2 INSERT.

**Core decision logic:**
- If the CICS Named Counter Service fails, the customer number assignment falls back to the DB2 IDENTITY DEFAULT mechanism (LGAC-NCS set to 'NO').
- If the CICS Named Counter Service succeeds, the counter value is used as the explicit customer number in the subsequent DB2 INSERT.

**Code:**
```cobol
           If WS-RESP Not = DFHRESP(NORMAL)
             MOVE 'NO' TO LGAC-NCS
             Initialize DB2-CUSTOMERNUM-INT
           ELSE
             Move LastCustNum  To DB2-CUSTOMERNUM-INT
           End-If.
```

**Variables:**

| Variable name | Role | Purpose |
|---|---|---|
| `WS-RESP` | input | Tested against DFHRESP(NORMAL) to decide which customer number assignment strategy to apply. |
| `LGAC-NCS` | output | Set to 'NO' when the Named Counter Service fails, switching the INSERT strategy to DB2 IDENTITY DEFAULT. |
| `LastCustNum` | input | Provides the counter value to be moved into DB2-CUSTOMERNUM-INT when the Named Counter Service call succeeds. |
| `DB2-CUSTOMERNUM-INT` | output | Receives either the Named Counter value or is initialised to zero depending on the counter service outcome, serving as the customer number for the DB2 INSERT. |

### 2.3 Paragraph: MAINLINE_INSERT-CUSTOMER

#### 2.3.1 Insert Customer Record Using Explicit Customer Number from Named Counter

**Description:** When the CICS Named Counter Service successfully provided a customer number, inserts a new customer record into the CUSTOMER table using that explicit number as the primary key along with all personal details from the communication area. If the INSERT fails, sets return code '90' and aborts processing.

**Core decision logic:**
- When LGAC-NCS is 'ON', the customer record is inserted with an explicitly assigned customer number derived from the Named Counter Service.
- A failed INSERT (non-zero SQLCODE) results in return code '90' and immediate termination of the transaction.

**Code:**
```cobol
           IF LGAC-NCS = 'ON'
             EXEC SQL
               INSERT INTO CUSTOMER
                         ( CUSTOMERNUMBER,
                           FIRSTNAME,
                           LASTNAME,
                           DATEOFBIRTH,
                           HOUSENAME,
                           HOUSENUMBER,
                           POSTCODE,
                           PHONEMOBILE,
                           PHONEHOME,
                           EMAILADDRESS )
                  VALUES ( :DB2-CUSTOMERNUM-INT,
                           :CA-FIRST-NAME,
                           :CA-LAST-NAME,
                           :CA-DOB,
                           :CA-HOUSE-NAME,
                           :CA-HOUSE-NUM,
                           :CA-POSTCODE,
                           :CA-PHONE-MOBILE,
                           :CA-PHONE-HOME,
                           :CA-EMAIL-ADDRESS )
             END-EXEC
             IF SQLCODE NOT EQUAL 0
               MOVE '90' TO CA-RETURN-CODE
               PERFORM WRITE-ERROR-MESSAGE
               EXEC CICS RETURN END-EXEC
             END-IF
```

**Variables:**

| Variable name | Role | Purpose |
|---|---|---|
| `LGAC-NCS` | input | Controls which INSERT path is executed — 'ON' triggers the explicit customer number INSERT path. |
| `DB2-CUSTOMERNUM-INT` | input | Provides the explicit customer number obtained from the Named Counter Service to be inserted as the CUSTOMER table primary key. |

**Paragraphs:**

* MAINLINE_WRITE-ERROR-MESSAGE
#### 2.3.2 Insert Customer Record Using DB2 Identity Default and Retrieve Assigned Number

**Description:** When the Named Counter Service was unavailable, inserts a new customer record using the DB2 IDENTITY DEFAULT to auto-assign the customer number, then retrieves the DB2-generated customer number via IDENTITY_VAL_LOCAL() for downstream use. A failed INSERT results in return code '90' and transaction termination.

**Core decision logic:**
- When LGAC-NCS is not 'ON' (Named Counter Service failed), the customer number is auto-assigned by the DB2 IDENTITY column rather than an explicit value.
- After a successful DEFAULT INSERT, the DB2-generated customer number is retrieved via IDENTITY_VAL_LOCAL() to maintain the customer number for subsequent processing.
- A failed INSERT (non-zero SQLCODE) results in return code '90' and immediate termination of the transaction.

**Code:**
```cobol
           ELSE
             EXEC SQL
               INSERT INTO CUSTOMER
                         ( CUSTOMERNUMBER,
                           FIRSTNAME,
                           LASTNAME,
                           DATEOFBIRTH,
                           HOUSENAME,
                           HOUSENUMBER,
                           POSTCODE,
                           PHONEMOBILE,
                           PHONEHOME,
                           EMAILADDRESS )
                  VALUES ( DEFAULT,
                           :CA-FIRST-NAME,
                           :CA-LAST-NAME,
                           :CA-DOB,
                           :CA-HOUSE-NAME,
                           :CA-HOUSE-NUM,
                           :CA-POSTCODE,
                           :CA-PHONE-MOBILE,
                           :CA-PHONE-HOME,
                           :CA-EMAIL-ADDRESS )
             END-EXEC
             IF SQLCODE NOT EQUAL 0
               MOVE '90' TO CA-RETURN-CODE
               PERFORM WRITE-ERROR-MESSAGE
               EXEC CICS RETURN END-EXEC
             END-IF
      *    get value of assigned customer number
               EXEC SQL
                 SET :DB2-CUSTOMERNUM-INT = IDENTITY_VAL_LOCAL()
               END-EXEC
           END-IF.
```

**Variables:**

| Variable name | Role | Purpose |
|---|---|---|
| `LGAC-NCS` | input | When not equal to 'ON', triggers the fallback DB2 IDENTITY DEFAULT INSERT path. |
| `DB2-CUSTOMERNUM-INT` | output | Receives the DB2-generated customer number from IDENTITY_VAL_LOCAL() after the DEFAULT INSERT, making it available for downstream processing. |

**Paragraphs:**

* MAINLINE_WRITE-ERROR-MESSAGE
#### 2.3.3 Propagate Assigned Customer Number to Communication Area

**Description:** Transfers the newly assigned customer number (whether from the Named Counter Service or DB2 IDENTITY) into the communication area so the caller receives the customer number that was created.

**Core decision logic:**
- The newly assigned customer number must be returned to the caller via the communication area regardless of which INSERT path was used.

**Code:**
```cobol
           MOVE DB2-CUSTOMERNUM-INT TO CA-CUSTOMER-NUM.
```

**Variables:**

| Variable name | Role | Purpose |
|---|---|---|
| `DB2-CUSTOMERNUM-INT` | input | Provides the newly assigned customer number to be propagated back to the caller through the communication area. |

---

Generated by IBM Bob Premium Package for Z
