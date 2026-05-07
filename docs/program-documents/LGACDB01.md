# LGACDB01 - Add Customer to DB2 Database

## 1. Program Purpose

LGACDB01 is a CICS-DB2 data management program responsible for adding new customer records to the DB2 CUSTOMER table. The program serves as the database layer in a three-tier architecture, receiving customer information through a communication area (COMMAREA), obtaining a unique customer number from either a CICS named counter or DB2 identity column, inserting the customer record into DB2, and coordinating with VSAM file updates through linked programs. It implements comprehensive error handling with detailed logging to temporary storage queues and supports both counter-based and DB2-generated customer number assignment strategies.

## 2. Program Inputs

### 2.1. CICS Communication Area (DFHCOMMAREA)
- **Source**: Passed from calling program via CICS LINK
- **Structure**: Defined in copybook [`LGCMAREA`](base/src/lgcmarea.cpy)
- **Required Fields**:
  - `CA-FIRST-NAME`: Customer first name
  - `CA-LAST-NAME`: Customer last name
  - `CA-DOB`: Customer date of birth
  - `CA-HOUSE-NAME`: House name
  - `CA-HOUSE-NUM`: House number
  - `CA-POSTCODE`: Postal code
  - `CA-PHONE-MOBILE`: Mobile phone number
  - `CA-PHONE-HOME`: Home phone number
  - `CA-EMAIL-ADDRESS`: Email address
- **Validation**: Program validates COMMAREA length against required minimum (header + customer data length)

### 2.2. CICS Named Counter
- **Counter Name**: `GENACUSTNUM`
- **Pool Name**: `GENA`
- **Purpose**: Provides sequential customer numbers when Named Counter Service (NCS) is available
- **Fallback**: If counter unavailable, DB2 identity column generates the customer number

### 2.3. CICS Execution Interface Block (EIB)
- **EIBCALEN**: Length of COMMAREA (validates input)
- **EIBTRNID**: Transaction ID (logged for debugging)
- **EIBTRMID**: Terminal ID (logged for debugging)
- **EIBTASKN**: Task number (used as random seed and logged)

### 2.4. Copybooks
- **LGPOLICY**: Defines customer data structure lengths
- **LGCMAREA**: Defines communication area structure
- **SQLCA**: DB2 SQL Communication Area

## 3. Program Outputs

### 3.1. DB2 CUSTOMER Table Insert
- **Table**: CUSTOMER
- **Columns Inserted**:
  - `CUSTOMERNUMBER`: Generated customer number (from counter or DB2 identity)
  - `FIRSTNAME`: From `CA-FIRST-NAME`
  - `LASTNAME`: From `CA-LAST-NAME`
  - `DATEOFBIRTH`: From `CA-DOB`
  - `HOUSENAME`: From `CA-HOUSE-NAME`
  - `HOUSENUMBER`: From `CA-HOUSE-NUM`
  - `POSTCODE`: From `CA-POSTCODE`
  - `PHONEMOBILE`: From `CA-PHONE-MOBILE`
  - `PHONEHOME`: From `CA-PHONE-HOME`
  - `EMAILADDRESS`: From `CA-EMAIL-ADDRESS`

### 3.2. Updated COMMAREA (Return Values)
- **CA-RETURN-CODE**: Status code returned to caller
  - `00`: Successful completion
  - `90`: DB2 insert failure
  - `98`: Invalid COMMAREA length
- **CA-CUSTOMER-NUM**: Assigned customer number (populated on success)

### 3.3. Linked Program Calls
- **LGACVS01**: Called to add customer record to VSAM file
  - Receives same COMMAREA with populated customer number
- **LGACDB02**: Called to add customer security data
  - Receives `CDB2AREA` with customer number and security initialization data

### 3.4. Error Logging (via LGSTSQ)
- **ERROR-MSG Structure**: Contains formatted error information
  - Date and time of error
  - Program name (LGACDB01)
  - Customer number
  - SQL request type
  - SQLCODE
- **CA-ERROR-MSG**: Contains COMMAREA dump (first 90 bytes)
- **Destination**: Temporary storage queue via [`LGSTSQ`](base/src/lgstsq.cbl) program

## 4. Processing Logic

### 4.1. Initialization
1. Initialize working storage variables (`WS-HEADER`, `DB2-OUT-INTEGERS`)
2. Capture CICS environment information (transaction ID, terminal ID, task number)
3. Validate COMMAREA presence (ABEND with code 'LGCA' if missing)

### 4.2. COMMAREA Validation
1. Calculate required COMMAREA length (header + customer data)
2. Compare actual length (`EIBCALEN`) with required length
3. Return with code '98' if length insufficient

### 4.3. Customer Number Assignment
1. Attempt to obtain customer number from CICS named counter (`GENACUSTNUM` in pool `GENA`)
2. If counter available (`LGAC-NCS` = 'ON'):
   - Use counter value as customer number
   - Insert with explicit customer number
3. If counter unavailable (`LGAC-NCS` = 'NO'):
   - Insert with DEFAULT for customer number
   - Retrieve DB2-generated value using `IDENTITY_VAL_LOCAL()`

### 4.4. DB2 Insert Operation
1. Execute SQL INSERT into CUSTOMER table
2. Check SQLCODE for success
3. If failure:
   - Set return code to '90'
   - Log error message
   - Return to caller
4. If success:
   - Move customer number to COMMAREA
   - Continue processing

### 4.5. VSAM and Security Updates
1. Link to [`LGACVS01`](base/src/lgacvs01.cbl) to add customer to VSAM file
2. Prepare security data in `CDB2AREA`:
   - Request ID: '02ACUS'
   - Customer number: From DB2 insert
   - Initial password hash
   - Counter: '0000'
   - State: 'N' (new)
3. Link to [`LGACDB02`](base/src/lgacdb02.cbl) to add security record

### 4.6. Completion
1. Return to calling program with updated COMMAREA
2. COMMAREA contains customer number and return code

## 5. Program Paragraphs

### 5.1. MAINLINE
**Purpose**: Main program flow control

**Processing**:
- Performs initialization
- Validates COMMAREA
- Calls `Obtain-CUSTOMER-Number` paragraph
- Calls `INSERT-CUSTOMER` paragraph
- Links to VSAM update program
- Links to security update program
- Returns to caller

### 5.2. Obtain-CUSTOMER-Number
**Purpose**: Obtains next customer number from CICS named counter

**Processing**:
- Executes `CICS GET COUNTER` for `GENACUSTNUM`
- If successful: Stores counter value in `DB2-CUSTOMERNUM-INT`, sets `LGAC-NCS` to 'ON'
- If unsuccessful: Initializes `DB2-CUSTOMERNUM-INT` to zero, sets `LGAC-NCS` to 'NO'

**Impact**: Determines whether explicit or DB2-generated customer number is used

### 5.3. INSERT-CUSTOMER
**Purpose**: Inserts customer record into DB2 CUSTOMER table

**Processing**:
- Checks `LGAC-NCS` flag
- If 'ON': Inserts with explicit customer number from counter
- If 'NO': Inserts with DEFAULT, then retrieves generated number via `IDENTITY_VAL_LOCAL()`
- Handles SQL errors by logging and returning with error code
- Moves assigned customer number to COMMAREA

**SQL Operations**:
- INSERT INTO CUSTOMER (with explicit number or DEFAULT)
- SET :DB2-CUSTOMERNUM-INT = IDENTITY_VAL_LOCAL() (when using DB2 generation)

### 5.4. WRITE-ERROR-MESSAGE
**Purpose**: Logs error information to temporary storage queue

**Processing**:
- Captures SQLCODE in error message
- Obtains current date and time via CICS ASKTIME/FORMATTIME
- Formats error message with timestamp, program name, and SQL information
- Links to [`LGSTSQ`](base/src/lgstsq.cbl) to write error message
- Writes COMMAREA dump (up to 90 bytes) to queue

**Error Information Logged**:
- Date (MM/DD/YYYY format)
- Time (HHMMSS format)
- Program name (LGACDB01)
- Customer number
- SQL request type
- SQLCODE
- COMMAREA contents

## 6. Program Dependencies

### 6.1. Called Programs
- **LGACVS01**: Adds customer record to VSAM KSDS file
  - Called via CICS LINK with COMMAREA length 225
  - Receives customer data including assigned customer number
- **LGACDB02**: Adds customer security record to DB2
  - Called via CICS LINK with CDB2AREA length 32500
  - Receives customer number and initial security settings
- **LGSTSQ**: Writes messages to temporary storage queue
  - Called for error logging
  - Receives formatted error messages

### 6.2. Copybooks
- **LGPOLICY**: Customer and policy data structure definitions
  - Defines `WS-CUSTOMER-LEN` constant
  - Located in [`base/src/lgpolicy.cpy`](base/src/lgpolicy.cpy)
- **LGCMAREA**: Communication area structure
  - Defines all CA-* fields for customer data
  - Included via EXEC SQL INCLUDE
  - Located in [`base/src/lgcmarea.cpy`](base/src/lgcmarea.cpy)
- **SQLCA**: DB2 SQL Communication Area
  - Provides SQLCODE and SQL error information
  - Standard DB2 copybook

### 6.3. External Resources
- **DB2 Database**:
  - Table: CUSTOMER
  - Identity column: CUSTOMERNUMBER (when NCS unavailable)
- **CICS Named Counter**:
  - Counter: GENACUSTNUM
  - Pool: GENA
  - Optional resource (program handles unavailability)
- **Temporary Storage Queue**:
  - Written via LGSTSQ program
  - Contains error logs and COMMAREA dumps

### 6.4. CICS Services
- CICS LINK: Program-to-program communication
- CICS GET COUNTER: Named counter service
- CICS ASKTIME/FORMATTIME: Date/time services
- CICS RETURN: Return to calling program
- CICS ABEND: Abnormal termination

## 7. Constraints and Limitations

### 7.1. COMMAREA Requirements
- **Minimum Length**: Header (18 bytes) + Customer data length (from LGPOLICY)
- **Validation**: Program ABENDs with code 'LGCA' if no COMMAREA received
- **Return Code '98'**: Returned if COMMAREA length insufficient
- **Impact**: Calling programs must provide properly sized COMMAREA

### 7.2. Customer Number Assignment
- **Dual Strategy**: Supports both CICS named counter and DB2 identity column
- **Counter Dependency**: If `GENACUSTNUM` counter unavailable, falls back to DB2 generation
- **No Retry Logic**: Counter failure immediately switches to DB2 generation
- **Implication**: Customer numbers may not be sequential if counter service is intermittent

### 7.3. Transaction Integrity
- **Two-Phase Commit**: Not explicitly implemented
- **Rollback Scenario**: If VSAM update (LGACVS01) fails, DB2 record remains
- **Orphan Records**: Possible if downstream processing fails
- **Recommendation**: Calling program should implement compensating transactions

### 7.4. Error Handling
- **SQL Errors**: Return code '90' with error logging
- **No Automatic Retry**: Program does not retry failed operations
- **Caller Responsibility**: Calling program must handle error codes and decide on retry strategy

### 7.5. Performance Considerations
- **Synchronous Processing**: All operations execute synchronously
- **Multiple Program Links**: Three LINK operations per customer add (LGACVS01, LGACDB02, LGSTSQ)
- **DB2 Commits**: Implicit commit after successful INSERT
- **Throughput Impact**: Each customer add involves multiple program calls and I/O operations

### 7.6. Data Constraints
- **Customer Number**: Must be unique (enforced by DB2 primary key)
- **Field Lengths**: Defined in LGCMAREA copybook
- **No Data Validation**: Program does not validate customer data content (e.g., email format, phone number format)
- **Assumption**: Calling program performs business rule validation

## 8. Error Handling

### 8.1. COMMAREA Validation Errors

#### 8.1.1. Missing COMMAREA
- **Condition**: `EIBCALEN = 0`
- **Action**:
  1. Move ' NO COMMAREA RECEIVED' to error message
  2. Perform WRITE-ERROR-MESSAGE
  3. ABEND with code 'LGCA' (no dump)
- **Recovery**: None - transaction terminates
- **Prevention**: Calling program must always pass COMMAREA

#### 8.1.2. Insufficient COMMAREA Length
- **Condition**: `EIBCALEN < WS-REQUIRED-CA-LEN`
- **Action**:
  1. Set `CA-RETURN-CODE` to '98'
  2. Return to caller
- **Recovery**: Caller must correct COMMAREA size and retry
- **Prevention**: Use correct COMMAREA structure from LGCMAREA copybook

### 8.2. DB2 SQL Errors

#### 8.2.1. INSERT Failure
- **Condition**: `SQLCODE NOT EQUAL 0` after INSERT
- **Action**:
  1. Set `CA-RETURN-CODE` to '90'
  2. Perform WRITE-ERROR-MESSAGE (logs SQLCODE)
  3. Return to caller
- **Common Causes**:
  - Duplicate customer number (primary key violation)
  - DB2 connection failure
  - Table space full
  - Authorization failure
- **Recovery**: Caller must analyze SQLCODE and determine retry strategy
- **Logged Information**: Date, time, program name, customer number, SQL request, SQLCODE

#### 8.2.2. IDENTITY_VAL_LOCAL() Failure
- **Condition**: Error retrieving DB2-generated customer number
- **Action**: Same as INSERT failure (return code '90', error logging)
- **Impact**: Customer record inserted but number not retrieved
- **Recovery**: Manual intervention may be required to identify customer number

### 8.3. CICS Named Counter Errors

#### 8.3.1. Counter Unavailable
- **Condition**: `WS-RESP NOT = DFHRESP(NORMAL)` after GET COUNTER
- **Action**:
  1. Set `LGAC-NCS` to 'NO'
  2. Initialize `DB2-CUSTOMERNUM-INT` to zero
  3. Continue with DB2 identity column generation
- **Recovery**: Automatic fallback to DB2 generation
- **Impact**: Customer numbers switch from counter-based to DB2-generated sequence
- **No Error Logged**: This is treated as normal alternate processing path

### 8.4. Error Logging Mechanism

#### 8.4.1. Error Message Structure
- **ERROR-MSG**: Contains formatted error details
  - `EM-DATE`: Error date (MM/DD/YYYY)
  - `EM-TIME`: Error time (HHMMSS)
  - Program name: 'LGACDB01'
  - `EM-CUSNUM`: Customer number (if available)
  - `EM-SQLREQ`: SQL request type (e.g., ' INSERT CUSTOMER')
  - `EM-SQLRC`: SQLCODE value

#### 8.4.2. COMMAREA Dump
- **CA-ERROR-MSG**: Contains COMMAREA snapshot
  - First 90 bytes of COMMAREA
  - Helps diagnose data-related errors
  - Written separately to temporary storage queue

#### 8.4.3. Logging Process
1. Capture SQLCODE in error message
2. Obtain current timestamp via CICS ASKTIME/FORMATTIME
3. Format error message with all diagnostic information
4. Link to LGSTSQ program to write error message
5. If COMMAREA available, write COMMAREA dump (up to 90 bytes)

### 8.5. Error Code Summary

| Return Code | Meaning | Caller Action |
|-------------|---------|---------------|
| 00 | Success | Continue processing |
| 90 | DB2 INSERT failure | Check error log, analyze SQLCODE, retry or compensate |
| 98 | Invalid COMMAREA length | Correct COMMAREA structure and retry |
| LGCA (ABEND) | No COMMAREA received | Fix calling program logic |

### 8.6. Error Prevention Best Practices

1. **Calling Program Responsibilities**:
   - Always pass properly structured COMMAREA
   - Validate customer data before calling LGACDB01
   - Handle return codes appropriately
   - Implement retry logic for transient errors

2. **Operational Monitoring**:
   - Monitor temporary storage queues for error messages
   - Track SQLCODE patterns to identify systemic issues
   - Monitor CICS named counter availability
   - Alert on ABEND code 'LGCA'

3. **Data Integrity**:
   - Ensure DB2 CUSTOMER table has adequate space
   - Maintain CICS named counter service availability
   - Implement backup and recovery procedures
   - Monitor for orphaned records (DB2 without VSAM)

## 9. Usage Examples

### 9.1. Successful Customer Add (Counter Available)

**Scenario**: Add new customer with CICS named counter service available

**Input COMMAREA**:
```
CA-REQUEST-ID:     '01ACUS'
CA-FIRST-NAME:     'JOHN'
CA-LAST-NAME:      'SMITH'
CA-DOB:            '1985-06-15'
CA-HOUSE-NAME:     'OAK COTTAGE'
CA-HOUSE-NUM:      '42'
CA-POSTCODE:       'SW1A 1AA'
CA-PHONE-MOBILE:   '07700900123'
CA-PHONE-HOME:     '02071234567'
CA-EMAIL-ADDRESS:  'john.smith@example.com'
```

**Processing Flow**:
1. LGACDB01 validates COMMAREA (length OK)
2. Obtains customer number from counter: 1000001
3. Inserts into DB2 CUSTOMER table with CUSTOMERNUMBER=1000001
4. Links to LGACVS01 to add VSAM record
5. Links to LGACDB02 to add security record
6. Returns to caller

**Output COMMAREA**:
```
CA-RETURN-CODE:    '00'
CA-CUSTOMER-NUM:   1000001
(all other fields unchanged)
```

**DB2 Result**:
```sql
SELECT * FROM CUSTOMER WHERE CUSTOMERNUMBER = 1000001;
-- Returns: John Smith, DOB 1985-06-15, etc.
```

### 9.2. Successful Customer Add (Counter Unavailable)

**Scenario**: Add new customer when CICS named counter service is unavailable

**Input COMMAREA**: Same as Example 9.1

**Processing Flow**:
1. LGACDB01 validates COMMAREA (length OK)
2. Attempts to obtain counter - fails (counter unavailable)
3. Sets LGAC-NCS to 'NO'
4. Inserts into DB2 with DEFAULT for CUSTOMERNUMBER
5. DB2 generates customer number: 2000001 (from identity column)
6. Retrieves generated number via IDENTITY_VAL_LOCAL()
7. Links to LGACVS01 and LGACDB02
8. Returns to caller

**Output COMMAREA**:
```
CA-RETURN-CODE:    '00'
CA-CUSTOMER-NUM:   2000001
(all other fields unchanged)
```

**Note**: Customer number sequence differs from counter-based numbering

### 9.3. Error: Duplicate Customer Number

**Scenario**: Attempt to add customer with duplicate number (counter reused)

**Input COMMAREA**: Same as Example 9.1

**Processing Flow**:
1. LGACDB01 validates COMMAREA (length OK)
2. Obtains customer number from counter: 1000001 (already exists)
3. Attempts INSERT into DB2
4. DB2 returns SQLCODE -803 (duplicate key)
5. Sets CA-RETURN-CODE to '90'
6. Logs error message to temporary storage queue
7. Returns to caller without completing VSAM/security updates

**Output COMMAREA**:
```
CA-RETURN-CODE:    '90'
CA-CUSTOMER-NUM:   (unchanged or zero)
```

**Error Log** (via LGSTSQ):
```
12/15/2023 143022 LGACDB01 CNUM=1000001  INSERT CUSTOMER SQLCODE=-0803
COMMAREA=01ACUSJOHN      SMITH     1985-06-15OAK COTTAGE...
```

**Caller Action**: Retry with new counter value or investigate counter synchronization

### 9.4. Error: Invalid COMMAREA Length

**Scenario**: Calling program passes truncated COMMAREA

**Input COMMAREA**: Only 100 bytes (required: 225+ bytes)

**Processing Flow**:
1. LGACDB01 validates COMMAREA
2. Calculates required length: 18 (header) + customer length
3. Compares with EIBCALEN (100 bytes)
4. Length insufficient
5. Sets CA-RETURN-CODE to '98'
6. Returns immediately to caller

**Output COMMAREA**:
```
CA-RETURN-CODE:    '98'
(all other fields unchanged)
```

**Caller Action**: Correct COMMAREA structure and retry

### 9.5. Error: Missing COMMAREA

**Scenario**: Calling program fails to pass COMMAREA

**Processing Flow**:
1. LGACDB01 checks EIBCALEN
2. EIBCALEN = 0 (no COMMAREA)
3. Moves ' NO COMMAREA RECEIVED' to error message
4. Performs WRITE-ERROR-MESSAGE
5. ABENDs with code 'LGCA' (no dump)
6. Transaction terminates

**Error Log** (via LGSTSQ):
```
12/15/2023 143530 LGACDB01  NO COMMAREA RECEIVED
```

**CICS Message**:
```
ASRA ABEND LGCA IN TRANSACTION XXXX
```

**Caller Action**: Fix program logic to always pass COMMAREA

### 9.6. Integration Example: Complete Customer Add Flow

**Scenario**: End-to-end customer add from presentation layer

**Step 1**: User enters customer data in CICS screen (via LGTESTC1)

**Step 2**: LGTESTC1 calls LGACUS01 (business logic layer)
```cobol
EXEC CICS LINK PROGRAM('LGACUS01')
     COMMAREA(WS-COMMAREA)
     LENGTH(225)
END-EXEC
```

**Step 3**: LGACUS01 calls LGACDB01 (this program)
```cobol
EXEC CICS LINK PROGRAM('LGACDB01')
     COMMAREA(DFHCOMMAREA)
     LENGTH(225)
END-EXEC
```

**Step 4**: LGACDB01 processing
- Validates COMMAREA
- Obtains customer number: 1000050
- Inserts into DB2 CUSTOMER table
- Calls LGACVS01 for VSAM update
- Calls LGACDB02 for security record
- Returns with CA-RETURN-CODE='00', CA-CUSTOMER-NUM=1000050

**Step 5**: LGACUS01 checks return code
```cobol
IF CA-RETURN-CODE = '00'
   MOVE 'Customer added successfully' TO RESPONSE-MSG
   MOVE CA-CUSTOMER-NUM TO DISPLAY-CUST-NUM
ELSE
   MOVE 'Customer add failed' TO RESPONSE-MSG
END-IF
```

**Step 6**: LGTESTC1 displays result to user
```
Customer Number: 1000050
Status: Successfully Added
```

**Database State**:
- DB2 CUSTOMER table: New row with CUSTOMERNUMBER=1000050
- VSAM KSDSCUST file: New record with key 1000050
- DB2 security table: New row with customer number 1000050

**Verification Query**:
```sql
SELECT CUSTOMERNUMBER, FIRSTNAME, LASTNAME, EMAILADDRESS
FROM CUSTOMER
WHERE CUSTOMERNUMBER = 1000050;
```

**Result**:
```
CUSTOMERNUMBER  FIRSTNAME  LASTNAME  EMAILADDRESS
1000050         JOHN       SMITH     john.smith@example.com