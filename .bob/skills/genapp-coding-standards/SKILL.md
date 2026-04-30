---
name: genapp-coding-standards
description: Enforces COBOL coding standards for IBM CICS GenApp project. Use when generating code, reviewing code, refactoring, or when user mentions "check standards", "coding conventions", "style guide", or "GenApp standards".
metadata:
  author: GenApp Development Team
  version: 1.0.0
  enforcement-level: moderate
  target-languages: COBOL, JCL, BMS
---

# GenApp COBOL Coding Standards

Enforces coding standards and best practices for the IBM CICS GenApp insurance application project.

## Instructions

### When to Apply This Skill

Use this skill when:

- Generating new COBOL, JCL, or BMS code for GenApp
- Reviewing existing GenApp code
- Refactoring GenApp programs
- Conducting code reviews
- User explicitly requests GenApp standards enforcement

### Enforcement Level: Moderate

- **Critical violations** must be corrected (security, data integrity, CICS compliance)
- **Major violations** should be addressed when practical (naming, structure, documentation)
- **Minor violations** are recommendations (formatting, comment style)
- Document intentional deviations with inline comments

### Step 1: Identify Language and Context

Determine:

- Programming language (COBOL, JCL, BMS)
- Program type (presentation, business logic, data management)
- Whether it's new code or legacy code modification
- Specific standards section to apply

### Step 2: Apply COBOL Standards

#### Program Naming Conventions (CRITICAL)

**Rule:** Program names must be exactly 8 characters, uppercase, following GenApp patterns.

**Patterns:**
- `LGxCyyy` = Customer operations
  - x: A=Add, I=Inquire, U=Update, D=Delete
  - yyy: US=business logic, DB=Db2 data, VS=VSAM data
- `LGxPyyy` = Policy operations (same pattern)
- `LGTESTxy` = Presentation logic
  - x: C=customer, P=policy
  - y: policy type (1=motor, 2=endowment, 3=house, 4=commercial)
- `LGSETUP` = Initialization programs
- `LGxSTyy` = Statistics/utility programs

**Good Example:**
```cobol
       PROGRAM-ID. LGACDB01.
```

**Bad Example:**
```cobol
       PROGRAM-ID. ADDCUST.
```

**Why:** GenApp uses a strict 8-character naming convention for program identification and organization.

#### Variable Naming Conventions (MAJOR)

**Rule:** Variables must use meaningful prefixes indicating scope and purpose.

**Required Prefixes:**
- `WS-` = Working Storage variables
- `CA-` = Commarea fields
- `EM-` = Error Message fields
- `DB2-` = Database host variables
- `D2-` = Db2 area fields (alternative)
- `STSQ-` = Shared TSQ names
- No prefix = Level 77 constants or copybook-defined fields

**Good Example:**
```cobol
       01  WS-HEADER.
           03 WS-EYECATCHER            PIC X(16)
                                        VALUE 'LGACDB01------WS'.
           03 WS-TRANSID               PIC X(4).
           03 WS-TERMID                PIC X(4).
           03 WS-TASKNUM               PIC 9(7).
       
       01  CA-ERROR-MSG.
           03 FILLER                   PIC X(9)  VALUE 'COMMAREA='.
           03 CA-DATA                  PIC X(90) VALUE SPACES.
```

**Bad Example:**
```cobol
       01  HEADER.
           03 EYECATCHER               PIC X(16).
           03 TRANSID                  PIC X(4).
       
       01  ERROR-MSG.
           03 FILLER                   PIC X(9).
           03 DATA                     PIC X(90).
```

**Why:** Prefixes immediately identify variable scope and prevent naming conflicts.

#### Program Documentation (MAJOR)

**Rule:** All programs must have a boxed header comment with copyright, purpose, and description.

**Required Format:**
```cobol
      ******************************************************************
      *                                                                *
      * (C) Copyright IBM Corp. 2011, YYYY                             *
      *                                                                *
      *                    [Program Title]                             *
      *                                                                *
      *   [Detailed description of program purpose]                    *
      *                                                                *
      ******************************************************************
```

**Good Example:**
```cobol
      ******************************************************************
      *                                                                *
      * (C) Copyright IBM Corp. 2011, 2021                             *
      *                                                                *
      *                    ADD Customer Details                        *
      *                                                                *
      *   To add customer's name, address and date of birth to the     *
      *  DB2 customer table creating a new customer entry.             *
      *                                                                *
      ******************************************************************
```

**Bad Example:**
```cobol
      * Add customer program
      * Created: 2021-01-15
```

**Why:** Consistent documentation format aids maintenance and legal compliance.

#### Program Structure (MAJOR)

**Rule:** Programs must follow the standard 4-division structure with consistent organization.

**Required Structure:**
1. IDENTIFICATION DIVISION
   - PROGRAM-ID (8 characters, uppercase)
2. ENVIRONMENT DIVISION
   - CONFIGURATION SECTION (even if empty)
3. DATA DIVISION
   - WORKING-STORAGE SECTION
     - Common definitions (WS-HEADER with eyecatcher)
     - Variables grouped by purpose with comment separators
     - Copybooks (COPY statements)
   - LINKAGE SECTION
     - DFHCOMMAREA with COPY statements
4. PROCEDURE DIVISION
   - MAINLINE SECTION
   - Named paragraphs for specific functions

**Good Example:**
```cobol
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
        01  WS-HEADER.
           03 WS-EYECATCHER            PIC X(16)
                                        VALUE 'LGACUS01------WS'.
      *----------------------------------------------------------------*
      * Definitions required for data manipulation                     *
      *----------------------------------------------------------------*
        01  WS-COMMAREA-LENGTHS.
           03 WS-CA-HEADER-LEN         PIC S9(4) COMP VALUE +18.
      *    Include copybook for defintion of customer details length
           COPY LGPOLICY.
      *----------------------------------------------------------------*
       LINKAGE SECTION.
        01  DFHCOMMAREA.
              COPY LGCMAREA.
       PROCEDURE DIVISION.
      *----------------------------------------------------------------*
       MAINLINE SECTION.
           INITIALIZE WS-HEADER.
           PERFORM INSERT-CUSTOMER.
           EXEC CICS RETURN END-EXEC.
       MAINLINE-EXIT.
           EXIT.
```

#### Eyecatcher Pattern (MAJOR)

**Rule:** Every program must have a WS-EYECATCHER in WS-HEADER for debugging.

**Format:** `'[PROGRAM-ID]------WS'` (16 characters total)

**Good Example:**
```cobol
        01  WS-HEADER.
           03 WS-EYECATCHER            PIC X(16)
                                        VALUE 'LGACDB01------WS'.
```

**Why:** Eyecatchers help identify program instances in memory dumps during debugging.

#### CICS Command Standards (CRITICAL)

**Rule:** All CICS commands must use EXEC CICS...END-EXEC format with proper error handling.

**Required Patterns:**

1. **LINK Commands:**
```cobol
       EXEC CICS LINK Program(LGACDB01)
            Commarea(DFHCOMMAREA)
            LENGTH(32500)
       END-EXEC.
```

2. **ABEND with Custom Codes:**
```cobol
       EXEC CICS ABEND ABCODE('LGCA') NODUMP END-EXEC
```

3. **Return Statements:**
```cobol
       EXEC CICS RETURN END-EXEC.
```

**Bad Example:**
```cobol
       CALL 'LGACDB01' USING DFHCOMMAREA.
```

**Why:** EXEC CICS commands are required for CICS program communication and control.

#### Commarea Validation (CRITICAL)

**Rule:** All programs must validate commarea length before processing.

**Required Pattern:**
```cobol
      * If NO commarea received issue an ABEND
           IF EIBCALEN IS EQUAL TO ZERO
               MOVE ' NO COMMAREA RECEIVED' TO EM-VARIABLE
               PERFORM WRITE-ERROR-MESSAGE
               EXEC CICS ABEND ABCODE('LGCA') NODUMP END-EXEC
           END-IF

      * initialize commarea return code to zero
           MOVE '00' TO CA-RETURN-CODE
           MOVE EIBCALEN TO WS-CALEN.

      * check commarea length
           ADD WS-CA-HEADER-LEN TO WS-REQUIRED-CA-LEN
           ADD WS-CUSTOMER-LEN  TO WS-REQUIRED-CA-LEN

      * if less set error return code and return to caller
           IF EIBCALEN IS LESS THAN WS-REQUIRED-CA-LEN
             MOVE '98' TO CA-RETURN-CODE
             EXEC CICS RETURN END-EXEC
           END-IF
```

**Why:** Prevents data corruption and program failures from invalid commarea data.

#### Copybook Usage (MAJOR)

**Rule:** Use COPY statements for shared definitions. GenApp standard copybooks:

- `LGCMAREA` - Commarea structure (required in LINKAGE SECTION)
- `LGPOLICY` - Policy definitions and lengths
- `SSMAP` - BMS map definitions

**Good Example:**
```cobol
       WORKING-STORAGE SECTION.
      *    Include copybook for defintion of customer details length
           COPY LGPOLICY.
      *----------------------------------------------------------------*
       LINKAGE SECTION.
        01  DFHCOMMAREA.
              COPY LGCMAREA.
```

**Why:** Ensures consistency across programs and simplifies maintenance.

#### Error Handling (CRITICAL)

**Rule:** All programs must implement consistent error handling with logging.

**Required Pattern:**
```cobol
       01  ERROR-MSG.
           03 EM-DATE                  PIC X(8)  VALUE SPACES.
           03 FILLER                   PIC X     VALUE SPACES.
           03 EM-TIME                  PIC X(6)  VALUE SPACES.
           03 FILLER                   PIC X(9)  VALUE ' [PROGRAM-ID]'.
           03 EM-VARIABLE              PIC X(21) VALUE SPACES.

       WRITE-ERROR-MESSAGE.
      * Obtain and format current time and date
           EXEC CICS ASKTIME ABSTIME(WS-ABSTIME) END-EXEC.
           EXEC CICS FORMATTIME ABSTIME(WS-ABSTIME)
                     DDMMYYYY(EM-DATE)
                     TIME(EM-TIME)
           END-EXEC.
      * Write error message to TSQ
           EXEC CICS WRITEQ TS
                     QUEUE('GENAERRS')
                     FROM(ERROR-MSG)
                     LENGTH(LENGTH OF ERROR-MSG)
           END-EXEC.
```

**Why:** Consistent error logging aids troubleshooting and system monitoring.

#### Comment Style (MINOR)

**Rule:** Use consistent comment formatting for section separators and inline comments.

**Section Separators:**
```cobol
      *----------------------------------------------------------------*
      * Section description                                            *
      *----------------------------------------------------------------*
```

**Inline Comments:**
```cobol
      * Single line comment explaining next statement
           MOVE VALUE TO FIELD.
```

**Multi-line Comments:**
```cobol
      * First line of explanation
      *  continuation with leading space
      *  another continuation
```

### Step 3: Apply JCL Standards

#### JCL Job Documentation (MAJOR)

**Rule:** All JCL jobs must have header comments explaining purpose, inputs, and outputs.

**Good Example:**
```jcl
//*
//* (C) Copyright IBM Corp. 2011, 2021
//*
//*-------------------------------------------------------------------*
//*  INVOKE THE COBOL COMPILE                                         *
//*-------------------------------------------------------------------*
```

#### JCL Naming (MAJOR)

**Rule:** Job names and step names must be descriptive and follow GenApp patterns.

**Patterns:**
- `@COBOL` - Compilation jobs
- `@DB2xxx` - Db2-related jobs (BIND, CRE, DEL)
- `@ASMMAP` - Assembly jobs
- `@Cxxx` - CICS definition jobs

### Step 4: Cross-Cutting Concerns

#### Security (CRITICAL)

**Rules:**
1. No hardcoded credentials in source code
2. Use environment variables or external configuration
3. Validate all input data before processing
4. Use CICS security features (ABEND codes, return codes)

**Bad Example:**
```cobol
       77 WS-CS-PASSWORD               PIC X(16) Value 'NewPass'.
```

**Note:** The above example exists in legacy code but should not be replicated in new code.

#### Performance (MAJOR)

**Rules:**
1. Initialize variables once at program start
2. Use COMP for numeric fields in calculations
3. Minimize CICS LINK calls where possible
4. Use appropriate commarea lengths (avoid oversized commareas)

**Good Example:**
```cobol
       01  WS-RESP                   PIC S9(8) COMP.
       01  LastCustNum               PIC S9(8) COMP.
```

#### Maintainability (MAJOR)

**Rules:**
1. Keep paragraphs focused on single responsibility
2. Use meaningful paragraph names (verb-noun format)
3. Limit paragraph length to ~50 lines
4. Use PERFORM for repeated logic

**Good Example:**
```cobol
       INSERT-CUSTOMER.
           EXEC CICS LINK Program(LGACDB01)
                Commarea(DFHCOMMAREA)
                LENGTH(32500)
           END-EXEC.
           EXIT.
```

### Step 5: Report Findings

When reviewing code, structure your response as:

```
## GenApp Standards Review Results

**Overall Compliance:** [Excellent|Good|Needs Improvement|Poor]

### Critical Issues (Must Fix)
- [Issue with line number and explanation]

### Major Issues (Should Fix)
- [Issue with line number and explanation]

### Minor Issues (Recommendations)
- [Issue with line number and explanation]

### Positive Observations
- [What's done well]
```

### Step 6: Provide Corrected Code (if requested)

When user asks for corrections:

1. Show the problematic section with line numbers
2. Explain what violates GenApp standards
3. Provide corrected version following GenApp patterns
4. Explain why the correction aligns with standards

## Review Checklist

Use this checklist when reviewing GenApp code:

**Critical (Must Fix):**
- [ ] Program name follows LGxCyyy/LGxPyyy pattern (8 chars)
- [ ] CICS commands use EXEC CICS...END-EXEC format
- [ ] Commarea validation implemented
- [ ] Error handling with ABEND codes present
- [ ] No hardcoded credentials

**Major (Should Fix):**
- [ ] Variable prefixes used (WS-, CA-, EM-, DB2-)
- [ ] Boxed header comment with copyright
- [ ] Standard 4-division structure
- [ ] WS-EYECATCHER present in WS-HEADER
- [ ] Copybooks used for shared definitions (LGCMAREA, LGPOLICY)
- [ ] Section separator comments present
- [ ] Meaningful paragraph names

**Minor (Recommendations):**
- [ ] Consistent comment formatting
- [ ] Inline comments for complex logic
- [ ] Proper indentation (7-column COBOL format)
- [ ] COMP usage for numeric calculations

## Troubleshooting

### Issue: Legacy Code Doesn't Meet Standards

**Cause:** Code written before standards were formalized

**Solution:**
- Apply standards to new code and modifications
- Document deviations in legacy code with comments
- Refactor incrementally during maintenance
- Prioritize critical issues (security, CICS compliance)

### Issue: Copybook Not Found

**Cause:** COPY statement references missing copybook

**Solution:**
- Verify copybook exists in `base/src/` directory
- Check JCL SYSLIB concatenation includes source directory
- Standard GenApp copybooks: LGCMAREA, LGPOLICY, SSMAP

### Issue: Program Name Conflicts with Pattern

**Cause:** New program doesn't fit LGxCyyy/LGxPyyy pattern

**Solution:**
- Extend pattern with new letter codes if needed
- Document new pattern in this skill
- Ensure 8-character limit maintained
- Get architectural approval for new patterns

## References

For detailed information, see:

- `references/genapp-examples.md` - Additional code examples from GenApp
- `base/Architecture.md` - GenApp architecture documentation
- `base/Reference.md` - Complete program reference

## Notes

**3-Tier Architecture:**
All GenApp programs follow presentation → business logic → data management pattern using EXEC CICS LINK PROGRAM. Maintain this separation when creating new programs.

**Two-Phase Commit:**
Customer/policy add operations update Db2 first, then VSAM. VSAM failure triggers Db2 rollback. Maintain this pattern for data integrity.

**Commarea Standard:**
All programs use 32500-byte commarea for maximum flexibility. Smaller sizes acceptable for specific programs.

---

**Usage:** Reference this skill by saying "Apply GenApp coding standards" or "Review this code using GenApp standards."

**Maintenance:** Update this skill as GenApp standards evolve. Version control changes and communicate updates to the team.