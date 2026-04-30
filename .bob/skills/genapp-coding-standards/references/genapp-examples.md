# GenApp COBOL Code Examples

This file contains real examples from the GenApp codebase demonstrating coding standards.

## Complete Program Example: LGACUS01

This is a well-structured business logic program following all GenApp standards:

```cobol
      ******************************************************************
      *                                                                *
      * (C) Copyright IBM Corp. 2011, 2020                             *
      *                                                                *
      *                    ADD Customer                                *
      *                                                                *
      *   Business logic for adding a new customer                     *
      *                                                                *
      ******************************************************************
       IDENTIFICATION DIVISION.
       PROGRAM-ID. LGACUS01.
       ENVIRONMENT DIVISION.
       CONFIGURATION SECTION.
      *
       DATA DIVISION.

       WORKING-STORAGE SECTION.

      *----------------------------------------------------------------*
      * Common defintions                                              *
      *----------------------------------------------------------------*
      * Run time (debug) infomation for this invocation
        01  WS-HEADER.
           03 WS-EYECATCHER            PIC X(16)
                                        VALUE 'LGACUS01------WS'.
           03 WS-TRANSID               PIC X(4).
           03 WS-TERMID                PIC X(4).
           03 WS-TASKNUM               PIC 9(7).
           03 WS-FILLER                PIC X.
           03 WS-ADDR-DFHCOMMAREA      USAGE is POINTER.
           03 WS-CALEN                 PIC S9(4) COMP.

      * Variables for time/date processing
       01  WS-ABSTIME                  PIC S9(8) COMP VALUE +0.
       01  WS-TIME                     PIC X(8)  VALUE SPACES.
       01  WS-DATE                     PIC X(10) VALUE SPACES.

      * Error Message structure
       01  ERROR-MSG.
           03 EM-DATE                  PIC X(8)  VALUE SPACES.
           03 FILLER                   PIC X     VALUE SPACES.
           03 EM-TIME                  PIC X(6)  VALUE SPACES.
           03 FILLER                   PIC X(9)  VALUE ' LGACUS01'.
           03 EM-VARIABLE              PIC X(21) VALUE SPACES.

       01  CA-ERROR-MSG.
           03 FILLER                   PIC X(9)  VALUE 'COMMAREA='.
           03 CA-DATA                  PIC X(90) VALUE SPACES.
      *----------------------------------------------------------------*

      *----------------------------------------------------------------*
      * Definitions required for data manipulation                     *
      *----------------------------------------------------------------*
      * Fields to be used to check that commarea is correct length
       01  WS-COMMAREA-LENGTHS.
           03 WS-CA-HEADER-LEN         PIC S9(4) COMP VALUE +18.
           03 WS-REQUIRED-CA-LEN       PIC S9(4)      VALUE +0.
       77  LGACDB01                    PIC X(8)       VALUE 'LGACDB01'.

      *    Include copybook for defintion of customer details length
           COPY LGPOLICY.
      *----------------------------------------------------------------*

      ******************************************************************
      *    L I N K A G E     S E C T I O N
      ******************************************************************
       LINKAGE SECTION.

       01  DFHCOMMAREA.
             COPY LGCMAREA.

      ******************************************************************
      *    P R O C E D U R E S
      ******************************************************************
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
           MOVE '00' TO CA-NUM-POLICIES
           MOVE EIBCALEN TO WS-CALEN.
           SET WS-ADDR-DFHCOMMAREA TO ADDRESS OF DFHCOMMAREA.

      * check commarea length
           ADD WS-CA-HEADER-LEN TO WS-REQUIRED-CA-LEN
           ADD WS-CUSTOMER-LEN  TO WS-REQUIRED-CA-LEN

      * if less set error return code and return to caller
           IF EIBCALEN IS LESS THAN WS-REQUIRED-CA-LEN
             MOVE '98' TO CA-RETURN-CODE
             EXEC CICS RETURN END-EXEC
           END-IF

      *----------------------------------------------------------------*
      * Call routine to Insert row in DB2 Customer table               *
           PERFORM INSERT-CUSTOMER.
      
      *----------------------------------------------------------------*
      *
           EXEC CICS RETURN END-EXEC.

       MAINLINE-EXIT.
           EXIT.
      *----------------------------------------------------------------*

      *----------------------------------------------------------------*
      * DB2                                                            *
      *----------------------------------------------------------------*
       INSERT-CUSTOMER.

           EXEC CICS LINK Program(LGACDB01)
                Commarea(DFHCOMMAREA)
                LENGTH(32500)
           END-EXEC.
      **********

           EXIT.

      *================================================================*
      * Procedure to write error message to Queues                     *
      *   message will include Date, Time, Program Name, Customer      *
      *   Number, Policy Number and SQLCODE.                           *
      *================================================================*
       WRITE-ERROR-MESSAGE.
      * Obtain and format current time and date
           EXEC CICS ASKTIME ABSTIME(WS-ABSTIME) END-EXEC.
           EXEC CICS FORMATTIME ABSTIME(WS-ABSTIME)
                     DDMMYYYY(EM-DATE)
                     TIME(EM-TIME)
           END-EXEC.
      * Write error message to Temporary Storage Queue
           EXEC CICS WRITEQ TS
                     QUEUE('GENAERRS')
                     FROM(ERROR-MSG)
                     LENGTH(LENGTH OF ERROR-MSG)
           END-EXEC.
           EXIT.
```

**Key Standards Demonstrated:**

1. ✅ Boxed header with copyright and purpose
2. ✅ Program name follows LGxCyyy pattern (LGACUS01 = Add Customer, business logic)
3. ✅ WS-EYECATCHER in WS-HEADER for debugging
4. ✅ Variable prefixes (WS-, CA-, EM-)
5. ✅ Section separator comments
6. ✅ Commarea validation (length check, ABEND on missing)
7. ✅ COPY statements for shared definitions
8. ✅ EXEC CICS commands with END-EXEC
9. ✅ Error handling with TSQ logging
10. ✅ Consistent paragraph structure with EXIT statements

## Program Naming Examples

### Customer Operations
- `LGACDB01` - Add Customer to Db2
- `LGACUS01` - Add Customer business logic
- `LGACVS01` - Add Customer to VSAM
- `LGICDB01` - Inquire Customer from Db2
- `LGICUS01` - Inquire Customer business logic
- `LGICVS01` - Inquire Customer from VSAM
- `LGUCDB01` - Update Customer in Db2
- `LGUCUS01` - Update Customer business logic
- `LGUCVS01` - Update Customer in VSAM

### Policy Operations
- `LGAPDB01` - Add Policy to Db2
- `LGAPOL01` - Add Policy business logic
- `LGAPVS01` - Add Policy to VSAM
- `LGIPDB01` - Inquire Policy from Db2
- `LGIPOL01` - Inquire Policy business logic
- `LGIPVS01` - Inquire Policy from VSAM
- `LGUPDB01` - Update Policy in Db2
- `LGUPOL01` - Update Policy business logic
- `LGDPDB01` - Delete Policy from Db2
- `LGDPOL01` - Delete Policy business logic
- `LGDPVS01` - Delete Policy from VSAM

### Presentation Logic
- `LGTESTC1` - Customer menu presentation
- `LGTESTP1` - Motor policy presentation
- `LGTESTP2` - Endowment policy presentation
- `LGTESTP3` - House policy presentation
- `LGTESTP4` - Commercial property presentation

### Utility Programs
- `LGSETUP` - Initialize counters and TSQs
- `LGASTAT1` - Update transaction counts
- `LGSTSQ` - Write message to TSQ
- `LGWEBST5` - Copy transaction counts to TSQ

## Variable Naming Examples

### Working Storage Variables

```cobol
      * Standard WS-HEADER structure
       01  WS-HEADER.
           03 WS-EYECATCHER            PIC X(16)
                                        VALUE 'LGACDB01------WS'.
           03 WS-TRANSID               PIC X(4).
           03 WS-TERMID                PIC X(4).
           03 WS-TASKNUM               PIC 9(7).
           03 WS-FILLER                PIC X.
           03 WS-ADDR-DFHCOMMAREA      USAGE is POINTER.
           03 WS-CALEN                 PIC S9(4) COMP.

      * Response codes and counters
       01  WS-RESP                   PIC S9(8) COMP.
       01  WS-Item-Count             PIC S9(4) Comp.
       01  WS-Cust-High              Pic S9(10).
       01  WS-Cust-Low               Pic S9(10).

      * Time and date variables
       01  WS-ABSTIME                  PIC S9(8) COMP VALUE +0.
       01  WS-TIME                     PIC X(8)  VALUE SPACES.
       01  WS-DATE                     PIC X(10) VALUE SPACES.

      * Commarea length validation
       01  WS-COMMAREA-LENGTHS.
           03 WS-CA-HEADER-LEN         PIC S9(4) COMP VALUE +18.
           03 WS-REQUIRED-CA-LEN       PIC S9(4)      VALUE +0.
```

### Commarea Variables

```cobol
       01  CA-ERROR-MSG.
           03 FILLER                   PIC X(9)  VALUE 'COMMAREA='.
           03 CA-DATA                  PIC X(90) VALUE SPACES.

      * From LGCMAREA copybook
       01  DFHCOMMAREA.
           03 CA-REQUEST-ID            PIC X(6).
           03 CA-RETURN-CODE           PIC X(2).
           03 CA-CUSTOMER-NUM          PIC 9(10).
           03 CA-FIRST-NAME            PIC X(10).
           03 CA-LAST-NAME             PIC X(20).
```

### Error Message Variables

```cobol
       01  ERROR-MSG.
           03 EM-DATE                  PIC X(8)  VALUE SPACES.
           03 FILLER                   PIC X     VALUE SPACES.
           03 EM-TIME                  PIC X(6)  VALUE SPACES.
           03 FILLER                   PIC X(9)  VALUE ' LGACDB01'.
           03 EM-VARIABLE.
             05 FILLER                 PIC X(6)  VALUE ' CNUM='.
             05 EM-CUSNUM              PIC X(10)  VALUE SPACES.
             05 EM-SQLREQ              PIC X(16) VALUE SPACES.
             05 FILLER                 PIC X(9)  VALUE ' SQLCODE='.
             05 EM-SQLRC               PIC +9(5) USAGE DISPLAY.
```

### Database Variables

```cobol
       01  DB2-OUT-INTEGERS.
           03 DB2-CUSTOMERNUM-INT   PIC S9(9) COMP.

       01  CDB2AREA.
           03 D2-REQUEST-ID            PIC X(6).
           03 D2-RETURN-CODE           PIC 9(2).
           03 D2-CUSTOMER-NUM          PIC 9(10).
           03 D2-CUSTSECR-PASS         PIC X(32).
           03 D2-CUSTSECR-COUNT        PIC X(4).
           03 D2-CUSTSECR-STATE        PIC X.
```

### TSQ and Counter Variables

```cobol
       01  STSQ.
         03  STSQ-NAME                 PIC X(8) Value 'GENACNTL'.

       01  GENAcount                 PIC X(16) Value 'GENACUSTNUM'.
       01  GENApool                  PIC X(8)  Value 'GENA'.
```

## CICS Command Examples

### LINK Commands

```cobol
      * Link to database program
       EXEC CICS LINK Program(LGACDB01)
            Commarea(DFHCOMMAREA)
            LENGTH(32500)
       END-EXEC.

      * Link to VSAM program
       EXEC CICS LINK Program(LGACVS01)
            Commarea(DFHCOMMAREA)
            LENGTH(225)
       END-EXEC.

      * Link to business logic
       EXEC CICS LINK PROGRAM('LGICUS01')
            COMMAREA(COMM-AREA)
            LENGTH(32500)
       END-EXEC.
```

### Error Handling Commands

```cobol
      * ABEND with custom code
       EXEC CICS ABEND ABCODE('LGCA') NODUMP END-EXEC.

      * Syncpoint rollback
       Exec CICS Syncpoint Rollback End-Exec.

      * Return to caller
       EXEC CICS RETURN END-EXEC.
```

### TSQ Operations

```cobol
      * Write to error queue
       EXEC CICS WRITEQ TS
            QUEUE('GENAERRS')
            FROM(ERROR-MSG)
            LENGTH(LENGTH OF ERROR-MSG)
       END-EXEC.

      * Delete TSQ
       Exec CICS DeleteQ TS Queue(STSQ-ERRS)
            Resp(WS-RESP)
       End-Exec.

      * Read from TSQ
       EXEC CICS READQ TS
            QUEUE(STSQ-NAME)
            INTO(READ-MSG)
            LENGTH(F82)
            ITEM(1)
            RESP(WS-RESP)
       END-EXEC.
```

### Time and Date Commands

```cobol
      * Get current time
       EXEC CICS ASKTIME ABSTIME(WS-ABSTIME) END-EXEC.

      * Format time and date
       EXEC CICS FORMATTIME ABSTIME(WS-ABSTIME)
            DDMMYYYY(EM-DATE)
            TIME(EM-TIME)
       END-EXEC.
```

### BMS Map Commands

```cobol
      * Send map to terminal
       EXEC CICS SEND MAP ('SSMAPC1')
            FROM(SSMAPC1O)
            MAPSET ('SSMAP')
            ERASE
       END-EXEC.

      * Receive map from terminal
       EXEC CICS RECEIVE MAP('SSMAPC1')
            INTO(SSMAPC1I) ASIS
            MAPSET('SSMAP')
       END-EXEC.
```

### Handle AID and Conditions

```cobol
      * Handle attention keys
       EXEC CICS HANDLE AID
            CLEAR(CLEARIT)
            PF3(ENDIT)
       END-EXEC.

      * Handle conditions
       EXEC CICS HANDLE CONDITION
            MAPFAIL(ENDIT)
       END-EXEC.
```

## Comment Style Examples

### Section Separators

```cobol
      *----------------------------------------------------------------*
      * Common defintions                                              *
      *----------------------------------------------------------------*

      *----------------------------------------------------------------*
      * Definitions required for data manipulation                     *
      *----------------------------------------------------------------*

      *----------------------------------------------------------------*
      * Definitions required by SQL statement                          *
      *   DB2 datatypes to COBOL equivalents                           *
      *     SMALLINT    :   PIC S9(4) COMP                             *
      *     INTEGER     :   PIC S9(9) COMP                             *
      *     DATE        :   PIC X(10)                                  *
      *     TIMESTAMP   :   PIC X(26)                                  *
      *----------------------------------------------------------------*
```

### Inline Comments

```cobol
      * initialize working storage variables
           INITIALIZE WS-HEADER.
      * set up general variable
           MOVE EIBTRNID TO WS-TRANSID.
           MOVE EIBTRMID TO WS-TERMID.
           MOVE EIBTASKN TO WS-TASKNUM.

      * If NO commarea received issue an ABEND
           IF EIBCALEN IS EQUAL TO ZERO
               MOVE ' NO COMMAREA RECEIVED' TO EM-VARIABLE
               PERFORM WRITE-ERROR-MESSAGE
               EXEC CICS ABEND ABCODE('LGCA') NODUMP END-EXEC
           END-IF

      * initialize commarea return code to zero
           MOVE '00' TO CA-RETURN-CODE
```

### Paragraph Documentation

```cobol
      *================================================================*
      * Procedure to write error message to Queues                     *
      *   message will include Date, Time, Program Name, Customer      *
      *   Number, Policy Number and SQLCODE.                           *
      *================================================================*
       WRITE-ERROR-MESSAGE.
```

## Anti-Patterns to Avoid

### ❌ Bad: Cryptic Variable Names

```cobol
       01  CR.
           05  ID              PIC 9(8).
           05  NM              PIC X(50).
           05  BAL             PIC S9(9)V99 COMP-3.
```

### ✅ Good: Descriptive Variable Names with Prefixes

```cobol
       01  WS-CUSTOMER-RECORD.
           05  WS-CUST-ID              PIC 9(8).
           05  WS-CUST-NAME            PIC X(50).
           05  WS-CUST-BALANCE         PIC S9(9)V99 COMP-3.
```

### ❌ Bad: Missing Error Handling

```cobol
       EXEC CICS LINK Program(LGACDB01)
            Commarea(DFHCOMMAREA)
       END-EXEC.
```

### ✅ Good: Proper Error Handling

```cobol
       EXEC CICS LINK Program(LGACDB01)
            Commarea(DFHCOMMAREA)
            LENGTH(32500)
       END-EXEC.
       
       IF CA-RETURN-CODE > 0
           PERFORM WRITE-ERROR-MESSAGE
           EXEC CICS ABEND ABCODE('LGCA') NODUMP END-EXEC
       END-IF.
```

### ❌ Bad: No Commarea Validation

```cobol
       PROCEDURE DIVISION.
           MOVE '00' TO CA-RETURN-CODE.
           PERFORM PROCESS-REQUEST.
```

### ✅ Good: Proper Commarea Validation

```cobol
       PROCEDURE DIVISION.
           IF EIBCALEN IS EQUAL TO ZERO
               MOVE ' NO COMMAREA RECEIVED' TO EM-VARIABLE
               PERFORM WRITE-ERROR-MESSAGE
               EXEC CICS ABEND ABCODE('LGCA') NODUMP END-EXEC
           END-IF
           
           MOVE '00' TO CA-RETURN-CODE
           MOVE EIBCALEN TO WS-CALEN.
           
           ADD WS-CA-HEADER-LEN TO WS-REQUIRED-CA-LEN
           ADD WS-CUSTOMER-LEN  TO WS-REQUIRED-CA-LEN
           
           IF EIBCALEN IS LESS THAN WS-REQUIRED-CA-LEN
             MOVE '98' TO CA-RETURN-CODE
             EXEC CICS RETURN END-EXEC
           END-IF.
```

## JCL Examples

### Compilation Job Structure

```jcl
//GENACOMP JOB 241901,'COB GENERATE',NOTIFY=&SYSUID,CLASS=A,MSGCLASS=H,
//       REGION=128M
//*
//* (C) Copyright IBM Corp. 2011, 2021
//*
//*-------------------------------------------------------------------*
//DB2PROC  PROC
//*-------------------------------------------------------------------*
//*  INVOKE THE COBOL COMPILE                                         *
//*-------------------------------------------------------------------*
//*
//COBL     EXEC PGM=IGYCRCTL,PARM='OPTFILE'
//STEPLIB  DD DSN=<COBOLHLQ>.SIGYCOMP,DISP=SHR
//         DD DSN=<CICSHLQ>.SDFHLOAD,DISP=SHR
//         DD DSN=<DB2HLQ>.SDSNEXIT,DISP=SHR
//         DD DSN=<DB2HLQ>.SDSNLOAD,DISP=SHR
//SYSOPTF  DD DATA,DLM=@@
 NODYNAM
 RENT
 APOST
 CICS
 CODEPAGE(<DB2CCSID>)
@@
//SYSPRINT DD SYSOUT=*
```

### Program Compilation Steps

```jcl
//LGACDB01 EXEC DB2PROC,MEM=LGACDB01
//LGACUS01 EXEC DB2PROC,MEM=LGACUS01
//LGACVS01 EXEC DB2PROC,MEM=LGACVS01
```

## Summary

These examples demonstrate the GenApp coding standards in practice. When writing new code or reviewing existing code, use these patterns as reference for:

- Consistent program structure
- Proper naming conventions
- Error handling patterns
- CICS command usage
- Comment formatting
- Variable organization

All new GenApp code should follow these established patterns to maintain consistency and quality across the codebase.