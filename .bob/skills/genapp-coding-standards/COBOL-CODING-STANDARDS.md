# IBM CICS GenApp COBOL Coding Standards

This document defines the COBOL coding standards for the IBM CICS GenApp project, derived from analysis of existing codebase patterns.

## Table of Contents

1. [Program Structure](#program-structure)
2. [Naming Conventions](#naming-conventions)
3. [Data Division Standards](#data-division-standards)
4. [Procedure Division Standards](#procedure-division-standards)
5. [CICS Command Standards](#cics-command-standards)
6. [SQL Standards](#sql-standards)
7. [Error Handling](#error-handling)
8. [Documentation Standards](#documentation-standards)
9. [Code Organization](#code-organization)

---

## 1. Program Structure

### 1.1 Standard Program Template

All COBOL programs MUST follow this structure:

```cobol
      ******************************************************************
      *                                                                *
      * (C) Copyright IBM Corp. 2011, 2020                             *
      *                                                                *
      *                    [Program Title]                             *
      *                                                                *
      *   [Program Description]                                        *
      *                                                                *
      ******************************************************************
       IDENTIFICATION DIVISION.
       PROGRAM-ID. [8-char-name].
       ENVIRONMENT DIVISION.
       CONFIGURATION SECTION.
      *
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       [Working storage definitions]
       
       LINKAGE SECTION.
       [Linkage section definitions]
       
       PROCEDURE DIVISION.
       [Procedure code]
```

### 1.2 Division Requirements

- **IDENTIFICATION DIVISION**: Required, contains PROGRAM-ID
- **ENVIRONMENT DIVISION**: Required with CONFIGURATION SECTION
- **DATA DIVISION**: Required with WORKING-STORAGE and LINKAGE sections
- **PROCEDURE DIVISION**: Required, contains all executable code

### 1.3 SQL Programs

Programs using SQL MUST include `PROCESS SQL` as the first line:

```cobol
       PROCESS SQL
      ******************************************************************
```

---

## 2. Naming Conventions

### 2.1 Program Names

**Format**: `LGxCyyy` or `LGxPyyy` (8 characters exactly)

**Pattern Breakdown**:
- `LG` = Application prefix (GenApp)
- `x` = Operation type:
  - `A` = Add
  - `I` = Inquire
  - `U` = Update
  - `D` = Delete
  - `T` = Test/Presentation
  - `S` = Setup/Utility
  - `W` = Web
- `C` or `P` = Entity type:
  - `C` = Customer
  - `P` = Policy
- `yyy` = Implementation layer:
  - `US01` = Business logic (US = User Service)
  - `DB01`/`DB02` = Database layer (DB = Database)
  - `VS01` = VSAM layer (VS = VSAM)
  - `EST` = Presentation logic (EST = Estimate/Test)
  - `ETUP` = Setup utility
  - `TSQ` = Temporary Storage Queue utility
  - `STAT1` = Statistics utility
  - `EBST5` = Web statistics

**Examples**:
- `LGACUS01` = Add Customer User Service (business logic)
- `LGACDB01` = Add Customer Database (DB2 operations)
- `LGICUS01` = Inquire Customer User Service
- `LGTESTC1` = Test Customer presentation logic
- `LGSETUP` = Setup utility

### 2.2 Variable Naming

#### Working Storage Variables

**Prefix Standards**:
- `WS-` = Working Storage variable
- `DB2-` = DB2 host variable
- `EM-` = Error Message field
- `CA-` = Communication Area field
- `D2-` = DB2 area field

**Examples**:
```cobol
01  WS-HEADER.
    03 WS-EYECATCHER            PIC X(16).
    03 WS-TRANSID               PIC X(4).
    03 WS-TERMID                PIC X(4).

01  DB2-OUT-INTEGERS.
    03 DB2-CUSTOMERNUM-INT      PIC S9(9) COMP.

01  ERROR-MSG.
    03 EM-DATE                  PIC X(8).
    03 EM-TIME                  PIC X(6).
```

#### Level Numbers

- `01` = Top-level group or elementary item
- `03`, `05`, `07` = Subordinate group items (use odd numbers)
- `77` = Independent elementary item
- `88` = Condition name

### 2.3 Copybook Names

**Format**: `LGxxxxxx` (8 characters)

**Examples**:
- `LGCMAREA` = Communication Area structure
- `LGPOLICY` = Policy definitions

### 2.4 Section and Paragraph Names

**Standards**:
- Use descriptive, hyphenated names in UPPERCASE
- End utility paragraphs with `-EXIT`
- Use verb-noun format

**Examples**:
```cobol
MAINLINE SECTION.
MAINLINE-EXIT.
    EXIT.

INSERT-CUSTOMER.
    [code]
    EXIT.

WRITE-ERROR-MESSAGE.
    [code]
    EXIT.
```

---

## 3. Data Division Standards

### 3.1 Working Storage Section

#### 3.1.1 Standard Header Block

Every program MUST include this standard header:

```cobol
WORKING-STORAGE SECTION.

*----------------------------------------------------------------*
* Common defintions                                              *
*----------------------------------------------------------------*
* Run time (debug) infomation for this invocation
  01  WS-HEADER.
     03 WS-EYECATCHER            PIC X(16)
                                  VALUE '[PROGNAME]------WS'.
     03 WS-TRANSID               PIC X(4).
     03 WS-TERMID                PIC X(4).
     03 WS-TASKNUM               PIC 9(7).
     03 WS-FILLER                PIC X.
     03 WS-ADDR-DFHCOMMAREA      USAGE is POINTER.
     03 WS-CALEN                 PIC S9(4) COMP.
```

#### 3.1.2 Time/Date Variables

Standard time/date processing variables:

```cobol
* Variables for time/date processing
 01  WS-ABSTIME                  PIC S9(8) COMP VALUE +0.
 01  WS-TIME                     PIC X(8)  VALUE SPACES.
 01  WS-DATE                     PIC X(10) VALUE SPACES.
```

#### 3.1.3 Error Message Structure

Standard error message structure:

```cobol
* Error Message structure
 01  ERROR-MSG.
     03 EM-DATE                  PIC X(8)  VALUE SPACES.
     03 FILLER                   PIC X     VALUE SPACES.
     03 EM-TIME                  PIC X(6)  VALUE SPACES.
     03 FILLER                   PIC X(9)  VALUE ' [PROGNAME]'.
     03 EM-VARIABLE              PIC X(21) VALUE SPACES.
```

For DB2 programs, include SQL-specific error fields:

```cobol
 01  ERROR-MSG.
     03 EM-DATE                  PIC X(8)  VALUE SPACES.
     03 FILLER                   PIC X     VALUE SPACES.
     03 EM-TIME                  PIC X(6)  VALUE SPACES.
     03 FILLER                   PIC X(9)  VALUE ' [PROGNAME]'.
     03 EM-VARIABLE.
       05 FILLER                 PIC X(6)  VALUE ' CNUM='.
       05 EM-CUSNUM              PIC X(10)  VALUE SPACES.
       05 EM-SQLREQ              PIC X(16) VALUE SPACES.
       05 FILLER                 PIC X(9)  VALUE ' SQLCODE='.
       05 EM-SQLRC               PIC +9(5) USAGE DISPLAY.
```

#### 3.1.4 Commarea Error Message

```cobol
 01  CA-ERROR-MSG.
     03 FILLER                   PIC X(9)  VALUE 'COMMAREA='.
     03 CA-DATA                  PIC X(90) VALUE SPACES.
```

### 3.2 Linkage Section

#### 3.2.1 Standard DFHCOMMAREA

```cobol
LINKAGE SECTION.

 01  DFHCOMMAREA.
       COPY LGCMAREA.
```

For SQL programs:

```cobol
LINKAGE SECTION.

 01  DFHCOMMAREA.
     EXEC SQL
       INCLUDE LGCMAREA
     END-EXEC.
```

### 3.3 PICTURE Clauses

**Standards**:
- Use `PIC` (not `PICTURE`)
- Numeric: `PIC 9(n)` or `PIC S9(n) COMP`
- Alphanumeric: `PIC X(n)`
- Edited: `PIC +9(5)` for signed display

**Examples**:
```cobol
03 WS-CUSTOMER-NUM          PIC 9(10).
03 WS-RESP                  PIC S9(8) COMP.
03 WS-NAME                  PIC X(20).
03 EM-SQLRC                 PIC +9(5) USAGE DISPLAY.
```

---

## 4. Procedure Division Standards

### 4.1 MAINLINE Section Structure

Every program MUST follow this standard mainline pattern:

```cobol
PROCEDURE DIVISION.

*----------------------------------------------------------------*
 MAINLINE SECTION.

*----------------------------------------------------------------*
* Common code                                                    *
*----------------------------------------------------------------*
* initialize working storage variables
     INITIALIZE WS-HEADER.
* set up general variable
     MOVE EIBTRNID TO WS-TRANSID.
     MOVE EIBTRMID TO WS-TERMID.
     MOVE EIBTASKN TO WS-TASKNUM.
*----------------------------------------------------------------*

*----------------------------------------------------------------*
* Process incoming commarea                                      *
*----------------------------------------------------------------*
* If NO commarea received issue an ABEND
     IF EIBCALEN IS EQUAL TO ZERO
         MOVE ' NO COMMAREA RECEIVED' TO EM-VARIABLE
         PERFORM WRITE-ERROR-MESSAGE
         EXEC CICS ABEND ABCODE('LGCA') NODUMP END-EXEC
     END-IF

* initialize commarea return code to zero
     MOVE '00' TO CA-RETURN-CODE
     MOVE EIBCALEN TO WS-CALEN.
     SET WS-ADDR-DFHCOMMAREA TO ADDRESS OF DFHCOMMAREA.

* check commarea length
     [commarea validation logic]

*----------------------------------------------------------------*
* [Main processing logic]
*----------------------------------------------------------------*
     [business logic]

     EXEC CICS RETURN END-EXEC.

 MAINLINE-EXIT.
     EXIT.
*----------------------------------------------------------------*
```

### 4.2 Commarea Validation

**Standard Pattern**:

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

### 4.3 Paragraph Structure

**Standards**:
- Each paragraph MUST end with `EXIT`
- Use descriptive names
- Include section comments

**Example**:

```cobol
*================================================================*
* Procedure to write error message to Queues                     *
*   message will include Date, Time, Program Name, Customer      *
*   Number, Policy Number and SQLCODE.                           *
*================================================================*
 WRITE-ERROR-MESSAGE.
* Obtain and format current time and date
     EXEC CICS ASKTIME ABSTIME(WS-ABSTIME)
     END-EXEC
     EXEC CICS FORMATTIME ABSTIME(WS-ABSTIME)
               MMDDYYYY(WS-DATE)
               TIME(WS-TIME)
     END-EXEC
     MOVE WS-DATE TO EM-DATE
     MOVE WS-TIME TO EM-TIME
* Write output message to TDQ
     EXEC CICS LINK PROGRAM('LGSTSQ')
               COMMAREA(ERROR-MSG)
               LENGTH(LENGTH OF ERROR-MSG)
     END-EXEC.
     [additional error handling]
     EXIT.
```

### 4.4 Control Flow

**Standards**:
- Use `PERFORM` for paragraph execution
- Use `GO TO` sparingly (only for error handling or menu navigation)
- Use `EVALUATE` for multi-way branching (not nested IFs)

**Example**:

```cobol
EVALUATE ENT1OPTO
  WHEN '1'
      [option 1 logic]
  WHEN '2'
      [option 2 logic]
  WHEN OTHER
      [default logic]
END-EVALUATE.
```

---

## 5. CICS Command Standards

### 5.1 Command Format

**Standards**:
- Use `EXEC CICS` ... `END-EXEC` format
- Indent command parameters
- One parameter per line for readability
- Always include `RESP` for error handling where applicable

**Example**:

```cobol
EXEC CICS LINK PROGRAM('LGACUS01')
          COMMAREA(COMM-AREA)
          LENGTH(32500)
END-EXEC
```

### 5.2 Common CICS Commands

#### 5.2.1 LINK Command

```cobol
EXEC CICS LINK Program(LGACDB01)
     Commarea(DFHCOMMAREA)
     LENGTH(32500)
END-EXEC.
```

#### 5.2.2 SEND MAP Command

```cobol
EXEC CICS SEND MAP ('SSMAPC1')
          FROM(SSMAPC1O)
          MAPSET ('SSMAP')
          ERASE
END-EXEC.
```

#### 5.2.3 RECEIVE MAP Command

```cobol
EXEC CICS RECEIVE MAP('SSMAPC1')
          INTO(SSMAPC1I) ASIS
          MAPSET('SSMAP')
END-EXEC.
```

#### 5.2.4 RETURN Command

```cobol
EXEC CICS RETURN
END-EXEC.

* Or with TRANSID and COMMAREA:
EXEC CICS RETURN
     TRANSID('SSC1')
     COMMAREA(COMM-AREA)
END-EXEC.
```

#### 5.2.5 ABEND Command

```cobol
EXEC CICS ABEND ABCODE('LGCA') NODUMP END-EXEC
```

#### 5.2.6 HANDLE AID/CONDITION

```cobol
EXEC CICS HANDLE AID
          CLEAR(CLEARIT)
          PF3(ENDIT)
END-EXEC.

EXEC CICS HANDLE CONDITION
          MAPFAIL(ENDIT)
END-EXEC.
```

#### 5.2.7 Time/Date Commands

```cobol
EXEC CICS ASKTIME ABSTIME(WS-ABSTIME)
END-EXEC
EXEC CICS FORMATTIME ABSTIME(WS-ABSTIME)
          MMDDYYYY(WS-DATE)
          TIME(WS-TIME)
END-EXEC
```

#### 5.2.8 Named Counter Commands

```cobol
Exec CICS Get Counter(GENAcount)
              Pool(GENApool)
              Value(LastCustNum)
              Resp(WS-RESP)
End-Exec.

Exec CICS Define Counter(GENAcount)
              Pool(GENApool)
              Value(LastCustNum)
              Resp(WS-RESP)
End-Exec.
```

#### 5.2.9 Temporary Storage Queue Commands

```cobol
Exec CICS ReadQ TS Queue(STSQ-NAME)
          Into(READ-MSG)
          Resp(WS-RESP)
          Item(1)
End-Exec.

EXEC CICS WRITEQ TS QUEUE(STSQ-NAME)
          FROM(WRITE-MSG)
          RESP(WS-RESP)
          NOSUSPEND
          LENGTH(24)
END-EXEC
```

#### 5.2.10 ENQ/DEQ Commands

```cobol
EXEC CICS ENQ Resource(STSQ-NAME)
              Length(Length Of STSQ-NAME)
END-EXEC.

EXEC CICS DEQ Resource(STSQ-NAME)
              Length(Length Of STSQ-NAME)
END-EXEC.
```

---

## 6. SQL Standards

### 6.1 SQL Declaration

Programs using SQL MUST include:

```cobol
*----------------------------------------------------------------*
*    DB2 CONTROL
*----------------------------------------------------------------*
* SQLCA DB2 communications area
     EXEC SQL
         INCLUDE SQLCA
     END-EXEC.
```

### 6.2 Host Variables

**Standards**:
- Prefix with `DB2-`
- Group related variables
- Use COBOL-to-DB2 type mappings

**Type Mappings**:
```cobol
*----------------------------------------------------------------*
* Definitions required by SQL statement                          *
*   DB2 datatypes to COBOL equivalents                           *
*     SMALLINT    :   PIC S9(4) COMP                             *
*     INTEGER     :   PIC S9(9) COMP                             *
*     DATE        :   PIC X(10)                                  *
*     TIMESTAMP   :   PIC X(26)                                  *
*----------------------------------------------------------------*
```

**Example**:

```cobol
* Host variables for output from DB2 integer types
 01  DB2-OUT-INTEGERS.
     03 DB2-CUSTOMERNUM-INT   PIC S9(9) COMP.

* Host variables for input to DB2 integer types
 01  DB2-IN-INTEGERS.
     03 DB2-CUSTOMERNUMBER-INT   PIC S9(9) COMP.
```

### 6.3 SQL Statement Format

**Standards**:
- Use `EXEC SQL` ... `END-EXEC`
- Indent SQL keywords
- Align column names
- Use host variable prefix `:`

**INSERT Example**:

```cobol
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
```

**SELECT Example**:

```cobol
EXEC SQL
    SELECT FIRSTNAME,
           LASTNAME,
           DATEOFBIRTH,
           HOUSENAME,
           HOUSENUMBER,
           POSTCODE,
           PHONEMOBILE,
           PHONEHOME,
           EMAILADDRESS
    INTO  :CA-FIRST-NAME,
          :CA-LAST-NAME,
          :CA-DOB,
          :CA-HOUSE-NAME,
          :CA-HOUSE-NUM,
          :CA-POSTCODE,
          :CA-PHONE-MOBILE,
          :CA-PHONE-HOME,
          :CA-EMAIL-ADDRESS
    FROM CUSTOMER
    WHERE CUSTOMERNUMBER = :DB2-CUSTOMERNUMBER-INT
END-EXEC.
```

### 6.4 SQLCODE Handling

**Standard Pattern**:

```cobol
Evaluate SQLCODE
  When 0
    MOVE '00' TO CA-RETURN-CODE
  When 100
    MOVE '01' TO CA-RETURN-CODE
  When -913
    MOVE '01' TO CA-RETURN-CODE
  When Other
    MOVE '90' TO CA-RETURN-CODE
    PERFORM WRITE-ERROR-MESSAGE
    EXEC CICS RETURN END-EXEC
END-Evaluate.
```

**Error Handling**:

```cobol
IF SQLCODE NOT EQUAL 0
  MOVE '90' TO CA-RETURN-CODE
  PERFORM WRITE-ERROR-MESSAGE
  EXEC CICS RETURN END-EXEC
END-IF
```

---

## 7. Error Handling

### 7.1 Return Codes

**Standard Return Codes**:
- `'00'` = Success
- `'01'` = No data found / Not found
- `'90'` = SQL/Database error
- `'98'` = Invalid commarea length

### 7.2 ABEND Codes

**Standard ABEND Code**:
- `'LGCA'` = No commarea received (with NODUMP)

### 7.3 Error Message Routine

Every program MUST include this standard error handling paragraph:

```cobol
*================================================================*
* Procedure to write error message to Queues                     *
*   message will include Date, Time, Program Name, Customer      *
*   Number, Policy Number and SQLCODE.                           *
*================================================================*
 WRITE-ERROR-MESSAGE.
* Save SQLCODE in message (if applicable)
     MOVE SQLCODE TO EM-SQLRC
* Obtain and format current time and date
     EXEC CICS ASKTIME ABSTIME(WS-ABSTIME)
     END-EXEC
     EXEC CICS FORMATTIME ABSTIME(WS-ABSTIME)
               MMDDYYYY(WS-DATE)
               TIME(WS-TIME)
     END-EXEC
     MOVE WS-DATE TO EM-DATE
     MOVE WS-TIME TO EM-TIME
* Write output message to TDQ
     EXEC CICS LINK PROGRAM('LGSTSQ')
               COMMAREA(ERROR-MSG)
               LENGTH(LENGTH OF ERROR-MSG)
     END-EXEC.
* Write 90 bytes or as much as we have of commarea to TDQ
     IF EIBCALEN > 0 THEN
       IF EIBCALEN < 91 THEN
         MOVE DFHCOMMAREA(1:EIBCALEN) TO CA-DATA
         EXEC CICS LINK PROGRAM('LGSTSQ')
                   COMMAREA(CA-ERROR-MSG)
                   LENGTH(LENGTH OF CA-ERROR-MSG)
         END-EXEC
       ELSE
         MOVE DFHCOMMAREA(1:90) TO CA-DATA
         EXEC CICS LINK PROGRAM('LGSTSQ')
                   COMMAREA(CA-ERROR-MSG)
                   LENGTH(LENGTH OF CA-ERROR-MSG)
         END-EXEC
       END-IF
     END-IF.
     EXIT.
```

### 7.4 RESP Code Checking

**Standard Pattern**:

```cobol
Exec CICS [Command]
     Resp(WS-RESP)
End-Exec.

If WS-RESP Not = DFHRESP(NORMAL)
   [error handling]
End-If.
```

---

## 8. Documentation Standards

### 8.1 Program Header

**Required Format**:

```cobol
      ******************************************************************
      *                                                                *
      * (C) Copyright IBM Corp. 2011, 2020                             *
      *                                                                *
      *                    [Program Title]                             *
      *                                                                *
      *   [Detailed program description]                               *
      *   [Additional description lines as needed]                     *
      *                                                                *
      ******************************************************************
```

### 8.2 Section Comments

**Format**:

```cobol
*----------------------------------------------------------------*
* [Section description]                                          *
*----------------------------------------------------------------*
```

**For major sections**:

```cobol
*================================================================*
* [Major section description]                                    *
*   [Additional details]                                         *
*   [More details]                                               *
*================================================================*
```

### 8.3 Inline Comments

**Standards**:
- Use `*` in column 7 for full-line comments
- Align comments with code indentation
- Use descriptive comments for complex logic
- Comment CICS commands to explain purpose

**Examples**:

```cobol
* initialize working storage variables
     INITIALIZE WS-HEADER.
* set up general variable
     MOVE EIBTRNID TO WS-TRANSID.

* If NO commarea received issue an ABEND
     IF EIBCALEN IS EQUAL TO ZERO
         MOVE ' NO COMMAREA RECEIVED' TO EM-VARIABLE
         PERFORM WRITE-ERROR-MESSAGE
         EXEC CICS ABEND ABCODE('LGCA') NODUMP END-EXEC
     END-IF
```

### 8.4 Copybook Documentation

Copybooks MUST include:
- Copyright notice
- Purpose description
- Usage instructions
- Field descriptions

**Example**:

```cobol
      ******************************************************************
      *                                                                *
      * (C) Copyright IBM Corp. 2011, 2020                             *
      *                                                                *
      *               COPYBOOK for COMMAREA structure                  *
      *                                                                *
      *   This commarea can be used for all functions                  *
      *                                                                *
      ******************************************************************
```

---

## 9. Code Organization

### 9.1 File Organization

**Standard Structure**:
```
base/
├── src/           # COBOL source programs
│   ├── lg*.cbl    # Main programs
│   ├── *.cpy      # Copybooks
│   └── *.bms      # BMS maps
├── cntl/          # JCL control files
├── data/          # Test data
└── exec/          # REXX scripts
```

### 9.2 Program Layering

**3-Tier Architecture**:

1. **Presentation Layer** (`LGTESTxx`)
   - User interface logic
   - Screen handling
   - Input validation

2. **Business Logic Layer** (`LGxCUS01`, `LGxPOL01`)
   - Business rules
   - Orchestration
   - Commarea validation

3. **Data Access Layer** (`LGxCDB01`, `LGxCVS01`)
   - Database operations
   - VSAM operations
   - Data persistence

### 9.3 Copybook Usage

**Standards**:
- Use `COPY` for COBOL copybooks
- Use `EXEC SQL INCLUDE` for SQL copybooks
- Place COPY statements in appropriate sections

**Examples**:

```cobol
* In WORKING-STORAGE:
     COPY LGPOLICY.

* In LINKAGE SECTION:
 01  DFHCOMMAREA.
       COPY LGCMAREA.

* For SQL programs:
 01  DFHCOMMAREA.
     EXEC SQL
       INCLUDE LGCMAREA
     END-EXEC.
```

### 9.4 Code Reuse

**Standards**:
- Use LINK for program-to-program calls
- Use COPY for common data structures
- Use standard utility programs:
  - `LGSTSQ` for logging
  - `LGASTAT1` for statistics
  - `LGSETUP` for initialization

---

## 10. Formatting Standards

### 10.1 Indentation

**Standards**:
- Use consistent indentation (typically 5 spaces per level)
- Align PICTURE clauses vertically
- Align VALUE clauses vertically

**Example**:

```cobol
 01  WS-HEADER.
     03 WS-EYECATCHER            PIC X(16)
                                  VALUE 'LGACUS01------WS'.
     03 WS-TRANSID               PIC X(4).
     03 WS-TERMID                PIC X(4).
     03 WS-TASKNUM               PIC 9(7).
```

### 10.2 Line Length

- Maximum 72 characters for code
- Columns 1-6: Sequence numbers (optional)
- Column 7: Indicator area
- Columns 8-72: Area A and Area B

### 10.3 Spacing

**Standards**:
- One blank line between major sections
- No blank lines within paragraph logic (use comments instead)
- Space around operators in conditions

**Example**:

```cobol
IF EIBCALEN IS EQUAL TO ZERO
    MOVE ' NO COMMAREA RECEIVED' TO EM-VARIABLE
    PERFORM WRITE-ERROR-MESSAGE
    EXEC CICS ABEND ABCODE('LGCA') NODUMP END-EXEC
END-IF
```

### 10.4 Case Conventions

**Standards**:
- COBOL reserved words: UPPERCASE
- Variable names: Mixed case or UPPERCASE
- Program names: UPPERCASE
- Comments: Mixed case for readability

---

## 11. Best Practices

### 11.1 Initialization

**Always initialize**:
- Working storage structures with `INITIALIZE`
- Return codes to `'00'`
- Counters and accumulators

```cobol
INITIALIZE WS-HEADER.
MOVE '00' TO CA-RETURN-CODE
```

### 11.2 Data Validation

**Always validate**:
- EIBCALEN (commarea length)
- Commarea structure
- Input data before processing

### 11.3 Resource Management

**Standards**:
- Use ENQ/DEQ for shared resources
- Use NOSUSPEND for non-critical writes
- Clean up temporary storage queues

### 11.4 Transaction Management

**Standards**:
- Use SYNCPOINT ROLLBACK on errors
- Implement two-phase commit for multi-resource updates
- Update DB2 before VSAM (rollback capability)

**Example**:

```cobol
IF CA-RETURN-CODE > 0
  Exec CICS Syncpoint Rollback End-Exec
  GO TO NO-ADD
END-IF
```

### 11.5 Performance

**Standards**:
- Use COMP for numeric fields in calculations
- Minimize LINK calls
- Use appropriate LENGTH values
- Avoid unnecessary data movement

---

## 12. Compliance Checklist

Use this checklist to verify compliance with coding standards:

### Program Structure
- [ ] Copyright header present
- [ ] Program description included
- [ ] Standard divisions present
- [ ] PROCESS SQL for DB2 programs

### Naming
- [ ] Program name follows LGxCyyy/LGxPyyy pattern
- [ ] Variables use standard prefixes (WS-, DB2-, EM-, CA-)
- [ ] Paragraphs use descriptive names
- [ ] Sections end with -EXIT

### Data Division
- [ ] WS-HEADER structure present
- [ ] Time/date variables defined
- [ ] ERROR-MSG structure present
- [ ] CA-ERROR-MSG structure present
- [ ] DFHCOMMAREA in LINKAGE SECTION

### Procedure Division
- [ ] MAINLINE section follows standard pattern
- [ ] Commarea validation implemented
- [ ] INITIALIZE WS-HEADER executed
- [ ] EIB fields captured
- [ ] Return codes set appropriately

### CICS Commands
- [ ] EXEC CICS ... END-EXEC format used
- [ ] Parameters properly indented
- [ ] RESP codes checked where applicable
- [ ] ABEND uses 'LGCA' code

### SQL (if applicable)
- [ ] SQLCA included
- [ ] Host variables prefixed with DB2-
- [ ] SQLCODE evaluated
- [ ] Error handling implemented

### Error Handling
- [ ] WRITE-ERROR-MESSAGE paragraph present
- [ ] Return codes follow standards
- [ ] ABEND handling implemented
- [ ] Error messages logged

### Documentation
- [ ] Program header complete
- [ ] Section comments present
- [ ] Complex logic commented
- [ ] Copybook documentation included

### Code Quality
- [ ] Consistent indentation
- [ ] Proper spacing
- [ ] No hardcoded values (use VALUE clauses)
- [ ] EXIT at end of paragraphs

---

## Revision History

| Version | Date | Author | Description |
|---------|------|--------|-------------|
| 1.0 | 2026-06-08 | Bob (AI Assistant) | Initial standards document derived from GenApp codebase analysis |

---

## References

- IBM Enterprise COBOL for z/OS Documentation
- CICS Transaction Server for z/OS Documentation
- IBM Db2 for z/OS Documentation
- GenApp Architecture Documentation (base/Architecture.md)
- GenApp Reference Documentation (base/Reference.md)