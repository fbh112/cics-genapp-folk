# AGENTS.md

This file provides guidance to agents when working with code in this repository.

## Workspace Type

**Type:** IBM CICS Transaction Server for z/OS  
**Languages:** IBM Enterprise COBOL for z/OS, JCL, REXX, BMS

## Build Commands

All build jobs are in [`base/cntl/`](base/cntl/) and must be customized before use:
- **Compile COBOL:** Submit `@COBOL` JCL job (compiles all programs to *userid*.GENAPP.LOAD)
- **Assemble BMS maps:** Submit `@ASMMAP` JCL job (RC=4 is expected)
- **Bind Db2:** Submit `@DB2BIND` JCL job
- **Create Db2 objects:** Submit `@DB2CRE` JCL job
- **Delete Db2 database:** Submit `@DB2DEL` JCL job (for rebuild)

## Testing

Manual testing via 3270 terminal:
1. Run `LGSE` transaction to initialize control tables
2. Run `SSC1` for customer menu (inquire: option 1, add: option 2)
3. Run `SSP1`-`SSP4` for policy menus (motor, endowment, house, commercial)

No automated test framework exists.

## Architecture Patterns

**3-Tier Design:** All programs follow presentation → business logic → data management pattern using `EXEC CICS LINK PROGRAM`.

**Program Naming Convention:**
- `LGxCyyy` = Customer operations (x: A=Add, I=Inquire, U=Update, D=Delete; yyy: US=business, DB=Db2, VS=VSAM)
- `LGxPyyy` = Policy operations (same pattern)
- `LGTESTxy` = Presentation logic (x: C=customer, P=policy; y: policy type 1-4)

**Shared Copybooks:** Programs use `COPY LGCMAREA` (commarea structure) and `COPY LGPOLICY` (policy definitions).

**Two-Phase Commit:** Customer/policy adds update Db2 first, then VSAM. VSAM failure triggers Db2 rollback.

## COBOL Compiler Options

From [`base/cntl/cobol.jcl`](base/cntl/cobol.jcl:44-50):
```
NODYNAM
RENT
APOST
CICS
CODEPAGE(<DB2CCSID>)
```

## Technical Documentation Location

**Documentation Directory:** [`base/`](base/)

**COBOL Program Documentation Mapping:**

| Program Name | Documentation File(s) | Description/Purpose |
|--------------|----------------------|---------------------|
| LGACDB01.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Add customer details to Db2 database |
| LGACUS01.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Add customer business logic |
| LGACVS01.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Add customer details to VSAM file |
| LGAPDB01.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Add policy details to Db2 database |
| LGAPOL01.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Add policy business logic |
| LGAPVS01.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Add policy details to VSAM file |
| LGICDB01.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Retrieve customer details from Db2 database |
| LGICUS01.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Inquire customer business logic |
| LGICVS01.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Retrieve customer record from VSAM file |
| LGIPDB01.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Retrieve policy record from Db2 database |
| LGIPOL01.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Inquire policy business logic |
| LGIPVS01.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Retrieve policy record from VSAM file |
| LGUCDB01.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Update Db2 policy record |
| LGUPOL01.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Update policy business logic |
| LGDPDB01.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Delete policy record from Db2 database |
| LGDPOL01.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Delete policy business logic |
| LGDPVS01.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Delete policy record from VSAM file |
| LGSETUP.cbl | [Testing.md](base/Testing.md), [Reference.md](base/Reference.md) | Initialize counters and temporary storage queues |
| LGTESTC1.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Presentation logic for customer menu |
| LGTESTP1.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Motor policy presentation logic |
| LGTESTP2.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Endowment policy presentation logic |
| LGTESTP3.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | House policy presentation logic |
| LGTESTP4.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Commercial property policy presentation logic |
| LGASTAT1.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Update transaction counts by using named counters |
| LGSTSQ.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Write a message to a temporary storage queue |
| LGWEBST5.cbl | [Architecture.md](base/Architecture.md), [Reference.md](base/Reference.md) | Copy business transaction counts from named counters to TSQ |

**Auto-Update Rules:**
1. When modifying COBOL code, check if related documentation exists in the mapping table
2. If documentation is out of sync, prompt user to update documentation
3. Always verify documentation accuracy after significant code changes
4. When analyzing a COBOL program, reference its mapped documentation for context
5. Update the mapping table when new programs or documentation are added

## Data Dictionary

**Status:** Configured

**Location:** `bobz/DD.json`

The data dictionary contains variable descriptions and business context for COBOL programs in this workspace. It provides semantic meaning for cryptic mainframe variable names, enabling better code comprehension and modernization.

**Current Coverage:**
- LGTESTC1 program: 15 variables documented
- LGICUS01 program: 15 variables documented (14 new, 1 shared with LGTESTC1)

**Usage:**
- Always reference the data dictionary when analyzing or modifying COBOL code
- The data dictionary is automatically read alongside COBOL files for context
- To update or extend the data dictionary, use the `data-dictionary-management` skill

## Coding Standards

**Status:** Coding standards skill available

Coding standards are enforced through the `genapp-coding-standards` skill located at `.bob/skills/genapp-coding-standards/`.

**Enforcement Level:** Moderate (critical violations required, minor ones recommended)

**To apply standards:**
- Reference the skill when generating code: "Apply GenApp coding standards"
- Use during code review: "Review this code using GenApp standards"
- The skill automatically activates when you mention "check standards", "coding conventions", or "GenApp standards"

**Key Standards:**
- Program naming: 8-character pattern (LGxCyyy/LGxPyyy)
- Variable prefixes: WS-, CA-, EM-, DB2-
- CICS command format: EXEC CICS...END-EXEC
- Commarea validation required
- Error handling with ABEND codes
- Boxed header documentation

See `.bob/skills/genapp-coding-standards/SKILL.md` for complete standards and `.bob/skills/genapp-coding-standards/references/genapp-examples.md` for code examples.