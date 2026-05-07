# LGICVS01 - Customer Inquire (Random Customer Retrieval)

## 1. Program Purpose

LGICVS01 is a CICS-VSAM utility program that retrieves a random customer number from the VSAM KSDS (Key Sequenced Data Set) customer file. The program serves as a test data generator and inquiry support tool, using a pseudo-random number generator seeded by the CICS task number to select a customer within a configurable range. It maintains customer number range boundaries in a temporary storage queue (TSQ) named GENACNTL, initializing the queue if it doesn't exist and updating it as needed. The program can be invoked either as a transaction (displaying results on screen) or via CICS LINK (returning results in COMMAREA), making it flexible for both interactive testing and programmatic use in automated test scenarios.

## 2. Program Inputs

### 2.1. Invocation Method Detection
The program determines how it was called by examining CICS execution context:

#### 2.1.1. Transaction Invocation
- **Detection**: `WS-STARTCODE(1:1) = 'D'` OR `WS-Invokeprog = Spaces`
- **Input Source**: CICS RECEIVE command
- **Input Structure**: `WS-RECV` (80 bytes)
  - `WS-RECV-TRANID`: Transaction ID (5 bytes)
  - `WS-RECV-DATA`: Input data (74 bytes)
- **Processing**: Sets `WS-FLAG` to 'R' (Receive mode)

#### 2.1.2. Program LINK Invocation
- **Detection**: `WS-STARTCODE(1:1) NOT = 'D'` AND `WS-Invokeprog NOT = Spaces`
- **Input Source**: DFHCOMMAREA
- **Input Structure**: `COMMA-DATA` (84 bytes)
  - `Comma-Data-H`: Header text (14 bytes)
  - `Comma-Data-High`: High customer number (10 digits)
  - Filler: 60 bytes
- **Processing**: Sets `WS-FLAG` to 'C' (Commarea mode)

### 2.2. CICS Execution Interface Block (EIB)
- **EIBTASKN**: Task number (used as random seed for customer selection)
- **EIBCALEN**: Length of COMMAREA (used to determine invocation method)

### 2.3. CICS System Information
- **WS-SYSID**: System ID (obtained via CICS ASSIGN SYSID)
- **WS-STARTCODE**: Start code (obtained via CICS ASSIGN STARTCODE)
- **WS-Invokeprog**: Invoking program name (obtained via CICS ASSIGN Invokingprog)

### 2.4. Temporary Storage Queue (GENACNTL)
The program reads customer range boundaries from TSQ:

#### 2.4.1. Queue Structure
- **Queue Name**: `GENACNTL` (with 4-byte extension `STSQ-EXT`)
- **Item 1**: Control header ('**** GENAPP CNTL')
- **Item 2**: Low customer boundary ('LOW CUSTOMER=nnnnnnnnnn')
- **Item 3**: High customer boundary ('HIGH CUSTOMER=nnnnnnnnnn')

#### 2.4.2. Default Values (if queue doesn't exist)
- **WS-Cust-Low**: 0001000001
- **WS-Cust-High**: 0001000001

### 2.5. VSAM KSDS Customer File
- **File Name**: `KSDSCUST`
- **Access Method**: GTEQ (Greater Than or Equal) read
- **Key Field**: Customer number (10 bytes)
- **Record Length**: 225 bytes (stored in `F82`)

## 3. Program Outputs

### 3.1. Screen Display Output (Transaction Mode)
When invoked as a transaction (`WS-FLAG = 'R'`):

**Output Format**:
```
HIGH CUSTOMER=nnnnnnnnnn
```

**CICS Command**:
```cobol
EXEC CICS SEND TEXT FROM(WRITE-MSG-H)
     WAIT
     ERASE
     LENGTH(24)
     FREEKB
END-EXEC
```

**Display Fields**:
- `WRITE-MSG-H`: 'HIGH CUSTOMER=' (14 bytes)
- `WRITE-MSG-High`: Selected customer number (10 digits)

### 3.2. COMMAREA Output (LINK Mode)
When invoked via LINK (`WS-FLAG = 'C'`):

**Output Structure** (COMMA-DATA):
- `COMMA-Data-H`: 'HIGH CUSTOMER=' (14 bytes)
- `COMMA-Data-High`: Selected customer number (10 digits)
- Remaining bytes: Spaces

**Return to Caller**: Via CICS RETURN with updated COMMAREA

### 3.3. Temporary Storage Queue Updates
The program writes/updates the GENACNTL TSQ with current range boundaries:

#### 3.3.1. Queue Item 1 (Control Header)
- **Content**: `WRITE-MSG-E` = '**** GENAPP CNTL' (20 bytes)
- **Written When**: Queue doesn't exist (`WS-FLAG-TSQE = 'Y'`)

#### 3.3.2. Queue Item 2 (Low Customer)
- **Content**: `WRITE-MSG-L` = 'LOW CUSTOMER=' + `WRITE-MSG-LOW` (23 bytes)
- **Written When**: Low customer not found in queue (`WS-FLAG-TSQL = 'Y'`)

#### 3.3.3. Queue Item 3 (High Customer)
- **Content**: `WRITE-MSG-H` = 'HIGH CUSTOMER=' + `WRITE-MSG-HIGH` (24 bytes)
- **Written When**: High customer not found in queue (`WS-FLAG-TSQH = 'Y'`)

### 3.4. VSAM File Access
- **Operation**: READ with GTEQ (Greater Than or Equal)
- **Purpose**: Retrieve customer record at or after random customer number
- **Result**: `CA-AREA` populated with customer data (225 bytes)
- **Customer Number**: Extracted from `CA-Customer-Num` (first 10 bytes of record)

## 4. Processing Logic

### 4.1. Initialization Phase

#### 4.1.1. Environment Detection
1. Clear `WS-RECV` area
2. Obtain system ID via `CICS ASSIGN SYSID`
3. Obtain start code via `CICS ASSIGN STARTCODE`
4. Obtain invoking program name via `CICS ASSIGN Invokingprog`

#### 4.1.2. Invocation Mode Determination
```cobol
IF WS-STARTCODE(1:1) = 'D' OR WS-Invokeprog NOT = Spaces
   MOVE 'C' To WS-FLAG          (Commarea mode)
   MOVE COMMA-DATA TO WS-COMMAREA
   MOVE EIBCALEN TO WS-RECV-LEN
ELSE
   EXEC CICS RECEIVE INTO(WS-RECV)
        LENGTH(WS-RECV-LEN)
   END-EXEC
   MOVE 'R' To WS-FLAG          (Receive mode)
   MOVE WS-RECV-DATA TO WS-COMMAREA
   SUBTRACT 5 FROM WS-RECV-LEN
END-IF
```

#### 4.1.3. Default Range Initialization
- Set `WS-Cust-Low` to 0001000001
- Set `WS-Cust-High` to 0001000001
- Set all TSQ flags to 'Y' (assume queue items don't exist)

### 4.2. Temporary Storage Queue Processing

#### 4.2.1. Queue Enqueue (Serialization)
```cobol
EXEC CICS ENQ Resource(STSQ-NAME)
             Length(Length Of STSQ-NAME)
END-EXEC
```
**Purpose**: Prevent concurrent updates to GENACNTL queue

#### 4.2.2. Queue Read and Parse
1. Attempt to read first item from GENACNTL queue
2. If successful (`WS-RESP = DFHRESP(NORMAL)`):
   - Set `WS-FLAG-TSQE` to Space (queue exists)
   - Loop through remaining items using NEXT option
   - For each item:
     - If message starts with 'LOW CUSTOMER':
       - Extract customer number to `WS-Cust-Low`
       - Set `WS-FLAG-TSQL` to Space
     - If message starts with 'HIGH CUSTOMER':
       - Extract customer number to `WS-Cust-High`
       - Set `WS-FLAG-TSQH` to Space
3. If queue doesn't exist:
   - Keep default values (0001000001)
   - Keep all flags as 'Y'

#### 4.2.3. Queue Update (Write Missing Items)
1. Prepare output messages with current range values
2. If `WS-FLAG-TSQE = 'Y'`: Write control header
3. If `WS-FLAG-TSQL = 'Y'`: Write low customer boundary
4. If `WS-FLAG-TSQH = 'Y'`: Write high customer boundary

#### 4.2.4. Queue Dequeue (Release Lock)
```cobol
EXEC CICS DEQ Resource(STSQ-NAME)
             Length(Length Of STSQ-NAME)
END-EXEC
```

### 4.3. Random Customer Number Generation

#### 4.3.1. Random Number Calculation
```cobol
Compute WS-Random-Number = Function Integer((
          Function Random(EIBTASKN) *
            (ws-cust-high - ws-cust-low)) +
               WS-Cust-Low)
```

**Algorithm**:
1. Use COBOL RANDOM function with EIBTASKN as seed
2. Multiply by range size (high - low)
3. Convert to integer
4. Add low boundary
5. Result: Random number between WS-Cust-Low and WS-Cust-High

**Example**:
- Low: 1000001, High: 1000100, Task: 12345
- Random(12345) = 0.456 (example)
- 0.456 * (1000100 - 1000001) = 0.456 * 99 = 45.144
- Integer(45.144) = 45
- 45 + 1000001 = 1000046

#### 4.3.2. Store Random Number
```cobol
Move WS-Random-Number to WRITE-MSG-HIGH
```

### 4.4. VSAM Customer Record Retrieval

#### 4.4.1. GTEQ Read Operation
```cobol
Exec CICS Read File('KSDSCUST')
          Into(CA-AREA)
          Length(F82)
          Ridfld(WRITE-MSG-HIGH)
          KeyLength(F10)
          RESP(WS-RESP)
          GTEQ
End-Exec
```

**Purpose**: Find customer record at or after random number

**GTEQ Behavior**:
- If exact match exists: Returns that record
- If no exact match: Returns next higher key
- If random number > highest key: Returns NOTFND

#### 4.4.2. Extract Actual Customer Number
```cobol
If WS-RESP = DFHRESP(NORMAL)
   Move CA-Customer-Num to WRITE-MSG-HIGH
End-if
```

**Result**: `WRITE-MSG-HIGH` contains actual customer number from VSAM

### 4.5. Output Generation

#### 4.5.1. Transaction Mode Output
```cobol
If WS-FLAG = 'R' Then
   EXEC CICS SEND TEXT FROM(WRITE-MSG-H)
        WAIT
        ERASE
        LENGTH(24)
        FREEKB
   END-EXEC
```

**Display**: "HIGH CUSTOMER=nnnnnnnnnn" on terminal screen

#### 4.5.2. LINK Mode Output
```cobol
Else
   Move Spaces To COMMA-Data
   Move Write-Msg-H To COMMA-Data-H
   Move Write-Msg-High To COMMA-Data-High
End-If
```

**Return**: Updated COMMAREA with customer number

### 4.6. Program Termination
```cobol
EXEC CICS RETURN
END-EXEC
```

## 5. Program Paragraphs

### 5.1. MAINLINE
**Purpose**: Main program flow control (only paragraph in program)

**Processing Sequence**:
1. Initialize working storage
2. Obtain CICS environment information (SYSID, STARTCODE, Invokingprog)
3. Determine invocation mode (transaction vs. LINK)
4. Receive input (via RECEIVE or COMMAREA)
5. Set default customer range (0001000001 to 0001000001)
6. Initialize TSQ flags
7. Enqueue GENACNTL resource
8. Read and parse GENACNTL TSQ
9. Update TSQ with missing items
10. Dequeue GENACNTL resource
11. Generate random customer number
12. Read VSAM customer file with GTEQ
13. Extract actual customer number
14. Output result (screen or COMMAREA)
15. Return to caller

**Note**: This program uses a single paragraph structure rather than multiple paragraphs

### 5.2. A-EXIT
**Purpose**: Exit point (not actively used)

**Code**:
```cobol
A-EXIT.
    EXIT.
    GOBACK.
```

**Note**: This paragraph is defined but never executed via PERFORM. The program exits via CICS RETURN before reaching this code.

## 6. Program Dependencies

### 6.1. CICS Services
- **CICS ASSIGN**: Obtains system information
  - SYSID: System identifier
  - STARTCODE: Transaction start code
  - Invokingprog: Name of calling program
- **CICS RECEIVE**: Receives terminal input (transaction mode)
- **CICS ENQ/DEQ**: Serializes access to GENACNTL TSQ
- **CICS READQ TS**: Reads temporary storage queue
- **CICS WRITEQ TS**: Writes temporary storage queue
- **CICS READ**: Reads VSAM file with GTEQ option
- **CICS SEND TEXT**: Sends output to terminal (transaction mode)
- **CICS RETURN**: Returns to caller

### 6.2. VSAM Files
- **KSDSCUST**: Customer KSDS file
  - Key: Customer number (10 bytes)
  - Record length: 225 bytes
  - Access: Random read with GTEQ
  - Purpose: Source of customer data

### 6.3. Temporary Storage Queues
- **GENACNTL**: Customer range control queue
  - Purpose: Stores low/high customer number boundaries
  - Structure: 3 items (header, low, high)
  - Scope: Shared across tasks
  - Serialization: ENQ/DEQ on queue name

### 6.4. COBOL Intrinsic Functions
- **RANDOM(seed)**: Generates pseudo-random number (0.0 to 1.0)
  - Seed: EIBTASKN (ensures different sequence per task)
- **INTEGER(value)**: Converts decimal to integer
  - Used to convert random decimal to customer number

### 6.5. External Dependencies
None - This is a self-contained utility program

### 6.6. Calling Programs
This program can be called by:
- **Interactive transactions**: Direct transaction invocation
- **Test programs**: Via CICS LINK for automated testing
- **Inquiry programs**: To obtain random customer for testing

## 7. Constraints and Limitations

### 7.1. Customer Range Constraints

#### 7.1.1. Default Range
- **Initial Low**: 0001000001
- **Initial High**: 0001000001
- **Implication**: If TSQ doesn't exist, only one customer number is possible
- **Resolution**: TSQ must be initialized with actual customer range

#### 7.1.2. Range Persistence
- **Storage**: Temporary storage queue (GENACNTL)
- **Volatility**: TSQ may be cleared during CICS restart
- **Impact**: Range resets to default after CICS cold start
- **Recommendation**: Initialize TSQ as part of CICS startup procedures

#### 7.1.3. Range Updates
- **Manual Update Required**: Program doesn't automatically detect customer file changes
- **Stale Range Risk**: If customers added/deleted, range may be outdated
- **Consequence**: May generate customer numbers outside actual file range
- **Mitigation**: Periodically update GENACNTL TSQ with current boundaries

### 7.2. Random Number Generation Constraints

#### 7.2.1. Pseudo-Random Algorithm
- **Seed**: EIBTASKN (CICS task number)
- **Predictability**: Same task number produces same sequence
- **Distribution**: Uniform distribution within range
- **Not Cryptographically Secure**: Suitable for testing, not security

#### 7.2.2. Task Number Reuse
- **CICS Behavior**: Task numbers wrap around after reaching maximum
- **Impact**: Same random sequences may repeat
- **Consequence**: Not suitable for applications requiring true randomness

### 7.3. VSAM File Access Constraints

#### 7.3.1. GTEQ Read Behavior
- **Exact Match**: Returns exact customer if exists
- **No Match**: Returns next higher customer number
- **Gap Handling**: If random number falls in gap, returns next valid customer
- **End of File**: If random number > highest key, returns NOTFND

#### 7.3.2. File Availability
- **No Error Handling**: Program doesn't handle VSAM file unavailability
- **NOTFND Response**: If VSAM read fails, `WRITE-MSG-HIGH` retains random number (not actual customer)
- **Caller Impact**: Caller receives non-existent customer number
- **Recommendation**: Add error handling for VSAM read failures

### 7.4. Concurrency Constraints

#### 7.4.1. TSQ Serialization
- **ENQ/DEQ**: Serializes access to GENACNTL queue
- **Scope**: Single CICS region only
- **Multi-Region**: No serialization across CICS regions
- **Impact**: Concurrent updates possible in multi-region environment

#### 7.4.2. Performance Impact
- **ENQ Duration**: Held during entire TSQ read/update cycle
- **Contention**: High-volume concurrent access may cause delays
- **Recommendation**: Keep TSQ operations minimal

### 7.5. Input/Output Constraints

#### 7.5.1. Transaction Mode
- **Input Length**: Fixed 80 bytes
- **Output Length**: Fixed 24 bytes
- **Screen Format**: Simple text display
- **No Formatting**: No screen maps or formatted displays

#### 7.5.2. LINK Mode
- **COMMAREA Length**: 84 bytes expected
- **No Validation**: Program doesn't validate COMMAREA structure
- **Caller Responsibility**: Caller must provide correct COMMAREA format

### 7.6. Data Integrity Constraints

#### 7.6.1. No Customer Validation
- **Assumption**: All customers in range exist in VSAM file
- **Reality**: Gaps may exist due to deletions
- **Consequence**: May return customer numbers for deleted records
- **Mitigation**: Use GTEQ to find next valid customer

#### 7.6.2. No Range Validation
- **No Bounds Checking**: Program doesn't verify low < high
- **Invalid Range**: If low > high, random calculation produces unexpected results
- **Recommendation**: Validate range when updating GENACNTL TSQ

## 8. Error Handling

### 8.1. CICS ASSIGN Errors

#### 8.1.1. SYSID Assignment Failure
- **Condition**: `WS-RESP NOT = DFHRESP(NORMAL)` after ASSIGN SYSID
- **Action**: None - program continues with spaces in `WS-SYSID`
- **Impact**: Minimal - SYSID only used for informational purposes
- **Recovery**: Automatic - program continues normally

#### 8.1.2. STARTCODE Assignment Failure
- **Condition**: `WS-RESP NOT = DFHRESP(NORMAL)` after ASSIGN STARTCODE
- **Action**: None - program continues with spaces in `WS-STARTCODE`
- **Impact**: May incorrectly determine invocation mode
- **Recovery**: None - may cause incorrect input processing

#### 8.1.3. Invokingprog Assignment Failure
- **Condition**: `WS-RESP NOT = DFHRESP(NORMAL)` after ASSIGN Invokingprog
- **Action**: None - program continues with spaces in `WS-Invokeprog`
- **Impact**: May incorrectly determine invocation mode
- **Recovery**: None - may cause incorrect input processing

**Recommendation**: Add explicit error handling for ASSIGN failures

### 8.2. CICS RECEIVE Errors

#### 8.2.1. Terminal Input Error
- **Condition**: `WS-RESP NOT = DFHRESP(NORMAL)` after RECEIVE
- **Action**: None - program continues with spaces in `WS-RECV`
- **Impact**: No input data available for processing
- **Recovery**: None - program continues with default values
- **Consequence**: Uses default customer range (0001000001)

**Recommendation**: Add error handling to detect and report RECEIVE failures

### 8.3. Temporary Storage Queue Errors

#### 8.3.1. Queue Read Failure (Queue Doesn't Exist)
- **Condition**: `WS-RESP NOT = DFHRESP(NORMAL)` after first READQ TS
- **Action**: Treated as normal - queue will be created
- **Processing**:
  - Keep default range values (0001000001)
  - Keep all TSQ flags as 'Y'
  - Write all three queue items
- **Recovery**: Automatic - queue created with default values

#### 8.3.2. Queue Read Failure (Other Errors)
- **Condition**: IOERR, NOTAUTH, etc.
- **Action**: None - program continues as if queue doesn't exist
- **Impact**: Uses default range, may overwrite existing queue
- **Recovery**: None - requires manual intervention

**Recommendation**: Distinguish between QIDERR (queue doesn't exist) and other errors

#### 8.3.3. Queue Write Failure
- **Condition**: `WS-RESP NOT = DFHRESP(NORMAL)` after WRITEQ TS
- **Action**: None - program continues
- **Impact**: Queue not updated, range information lost
- **Recovery**: None - queue remains incomplete
- **Consequence**: Next execution may use incorrect range

**Recommendation**: Add error handling for WRITEQ failures

### 8.4. VSAM File Errors

#### 8.4.1. File Not Found
- **Condition**: `WS-RESP = DFHRESP(NOTFND)` after READ
- **Action**: None - program continues
- **Impact**: `WRITE-MSG-HIGH` retains random number (not actual customer)
- **Recovery**: None - caller receives non-existent customer number
- **Consequence**: Caller may attempt to process invalid customer

#### 8.4.2. File Unavailable
- **Condition**: `WS-RESP = DFHRESP(FILENOTFOUND)` or DISABLED
- **Action**: None - program continues
- **Impact**: Same as NOTFND - invalid customer number returned
- **Recovery**: None - requires VSAM file to be available

#### 8.4.3. I/O Error
- **Condition**: `WS-RESP = DFHRESP(IOERR)`
- **Action**: None - program continues
- **Impact**: Invalid customer number returned
- **Recovery**: None - may require VSAM file recovery

**Recommendation**: Add comprehensive VSAM error handling

### 8.5. Error Handling Summary

#### 8.5.1. Current Error Handling Approach
- **Philosophy**: Continue processing despite errors
- **Assumption**: Errors are rare and non-critical
- **Benefit**: Program always completes
- **Risk**: May return invalid or unexpected results

#### 8.5.2. Error Detection Gaps
1. No validation of CICS ASSIGN results
2. No validation of RECEIVE results
3. No distinction between TSQ error types
4. No VSAM error handling
5. No error logging or reporting

#### 8.5.3. Recommended Enhancements
1. **Add Error Logging**: Write errors to temporary storage queue
2. **Validate Critical Operations**: Check RESP codes for ASSIGN, RECEIVE, READ
3. **Return Error Indicators**: Add error flag to COMMAREA output
4. **Distinguish Error Types**: Handle QIDERR differently from other TSQ errors
5. **VSAM Error Handling**: Detect and report file access failures

#### 8.5.4. Error Code Recommendations

Suggested return codes for enhanced error handling:

| Condition | Suggested Action | Return Indicator |
|-----------|------------------|------------------|
| ASSIGN failure | Log error, use defaults | Warning flag |
| RECEIVE failure | Log error, use defaults | Warning flag |
| TSQ QIDERR | Create queue (current behavior) | Normal |
| TSQ other error | Log error, use defaults | Error flag |
| VSAM NOTFND | Log warning, return random number | Warning flag |
| VSAM IOERR | Log error, return error indicator | Error flag |
| VSAM DISABLED | Log error, return error indicator | Error flag |

## 9. Usage Examples

### 9.1. Transaction Mode - Interactive Inquiry

**Scenario**: User executes transaction to get random customer number

**Execution**:
```
CICS Transaction: ICVS
```

**Processing Flow**:
1. CICS starts transaction, calls LGICVS01
2. Program detects transaction mode (`WS-STARTCODE = 'TD'`)
3. Executes CICS RECEIVE (no input expected)
4. Reads GENACNTL TSQ:
   - LOW CUSTOMER=1000001
   - HIGH CUSTOMER=1000500
5. Generates random number using task 12345:
   - Random(12345) * (1000500 - 1000001) + 1000001 = 1000234
6. Reads VSAM KSDSCUST with GTEQ for key 1000234
7. Finds exact match, customer 1000234 exists
8. Displays on terminal:
   ```
   HIGH CUSTOMER=1000234
   ```
9. Returns to CICS

**User View**:
```
HIGH CUSTOMER=1000234
```

### 9.2. LINK Mode - Programmatic Call

**Scenario**: Test program needs random customer for automated testing

**Calling Program Code**:
```cobol
01  TEST-COMMAREA.
    03 FILLER              PIC X(14).
    03 CUST-NUMBER         PIC 9(10).
    03 FILLER              PIC X(60).

EXEC CICS LINK PROGRAM('LGICVS01')
     COMMAREA(TEST-COMMAREA)
     LENGTH(84)
END-EXEC

IF CUST-NUMBER > 0
   DISPLAY 'Random customer: ' CUST-NUMBER
   * Use customer number for testing
ELSE
   DISPLAY 'Error: No customer returned'
END-IF
```

**Processing Flow**:
1. Calling program links to LGICVS01 with 84-byte COMMAREA
2. LGICVS01 detects LINK mode (`WS-Invokeprog NOT = Spaces`)
3. Copies COMMAREA to working storage
4. Reads GENACNTL TSQ (range: 1000001 to 1000500)
5. Generates random number: 1000378
6. Reads VSAM with GTEQ for 1000378
7. Finds customer 1000378
8. Updates COMMAREA:
   ```
   COMMA-Data-H:    'HIGH CUSTOMER='
   COMMA-Data-High: 1000378
   ```
9. Returns to caller

**Caller Receives**:
```
CUST-NUMBER: 1000378
```

### 9.3. First Execution - TSQ Initialization

**Scenario**: First time program runs, GENACNTL TSQ doesn't exist

**Processing Flow**:
1. Program attempts to read GENACNTL TSQ
2. Receives QIDERR (queue doesn't exist)
3. Uses default range:
   - WS-Cust-Low: 0001000001
   - WS-Cust-High: 0001000001
4. Sets all TSQ flags to 'Y'
5. Writes three items to GENACNTL:
   - Item 1: '**** GENAPP CNTL'
   - Item 2: 'LOW CUSTOMER=0001000001'
   - Item 3: 'HIGH CUSTOMER=0001000001'
6. Generates random number: 0001000001 (only possible value)
7. Reads VSAM for customer 0001000001
8. Returns customer 0001000001

**TSQ After Execution**:
```
Item 1: **** GENAPP CNTL
Item 2: LOW CUSTOMER=0001000001
Item 3: HIGH CUSTOMER=0001000001
```

**Note**: TSQ must be manually updated with actual customer range for meaningful results

### 9.4. TSQ Update - Adding High Customer Boundary

**Scenario**: TSQ exists with header and low customer, but missing high customer

**Initial TSQ State**:
```
Item 1: **** GENAPP CNTL
Item 2: LOW CUSTOMER=1000001
```

**Processing Flow**:
1. Program reads GENACNTL TSQ
2. Finds header: Sets `WS-FLAG-TSQE` to Space
3. Finds low customer: Sets `WS-FLAG-TSQL` to Space, `WS-Cust-Low` = 1000001
4. Doesn't find high customer: `WS-FLAG-TSQH` remains 'Y'
5. Uses default high: `WS-Cust-High` = 0001000001
6. Writes missing high customer item:
   ```
   Item 3: HIGH CUSTOMER=0001000001
   ```
7. Continues with random customer selection

**TSQ After Execution**:
```
Item 1: **** GENAPP CNTL
Item 2: LOW CUSTOMER=1000001
Item 3: HIGH CUSTOMER=0001000001
```

**Issue**: High customer is default value, not actual maximum

**Resolution**: Manually update TSQ with correct high customer value

### 9.5. GTEQ Behavior - Customer Gap

**Scenario**: Random number falls in gap between customers

**Customer File State**:
```
Customer 1000100: John Smith
Customer 1000105: Jane Doe
Customer 1000110: Bob Johnson
(Customers 1000101-1000104 and 1000106-1000109 deleted)
```

**Processing Flow**:
1. Random number generated: 1000103
2. VSAM READ with GTEQ for key 1000103
3. No exact match found
4. GTEQ returns next higher key: 1000105
5. Program extracts customer number: 1000105
6. Returns customer 1000105 (Jane Doe)

**Result**: Customer 1000105 returned instead of 1000103

**Benefit**: Always returns valid customer, even with gaps

### 9.6. GTEQ Behavior - Random Number Beyond Range

**Scenario**: Random number exceeds highest customer in file

**Customer File State**:
```
Highest customer: 1000500
```

**Processing Flow**:
1. Random number generated: 1000600 (beyond range)
2. VSAM READ with GTEQ for key 1000600
3. No record at or above 1000600
4. VSAM returns NOTFND
5. Program doesn't update `WRITE-MSG-HIGH`
6. Returns original random number: 1000600

**Result**: Invalid customer number 1000600 returned

**Issue**: Caller receives non-existent customer

**Prevention**: Ensure GENACNTL TSQ high customer matches actual file maximum

### 9.7. Integration Example - Automated Test Suite

**Scenario**: Test suite needs random customers for inquiry testing

**Test Program**:
```cobol
PERFORM 10 TIMES
   EXEC CICS LINK PROGRAM('LGICVS01')
        COMMAREA(TEST-COMMAREA)
        LENGTH(84)
   END-EXEC
   
   MOVE COMMA-Data-High TO TEST-CUST-NUM
   
   * Test customer inquiry with random customer
   EXEC CICS LINK PROGRAM('LGICUS01')
        COMMAREA(INQUIRY-COMMAREA)
        LENGTH(225)
   END-EXEC
   
   IF CA-RETURN-CODE = '00'
      ADD 1 TO SUCCESS-COUNT
   ELSE
      ADD 1 TO FAILURE-COUNT
   END-IF
END-PERFORM

DISPLAY 'Tests: ' SUCCESS-COUNT ' passed, ' FAILURE-COUNT ' failed'
```

**Execution Results**:
```
Test 1: Customer 1000234 - PASS
Test 2: Customer 1000456 - PASS
Test 3: Customer 1000123 - PASS
Test 4: Customer 1000389 - PASS
Test 5: Customer 1000267 - PASS
Test 6: Customer 1000445 - PASS
Test 7: Customer 1000178 - PASS
Test 8: Customer 1000334 - PASS
Test 9: Customer 1000412 - PASS
Test 10: Customer 1000201 - PASS

Tests: 10 passed, 0 failed
```

**Benefits**:
- Automated testing with varied customer data
- No hardcoded customer numbers
- Tests different customers each run
- Validates inquiry functionality across customer range