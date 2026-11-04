# IgniteFest 2026 — Red Team Security Assessment

## Overview
A security assessment of the IgniteFest 2026 web portal, conducted using the challenge-provided account and within the defined scope.

## Objectives
- Assess access control and privilege assignment.
- Examine exposure of organizer-only information.
- Review publicly accessible deployment artifacts.
- Document findings with reproducible evidence and remediation recommendations.

## Key Findings
1. Unexpected organizer privileges assigned to the supplied student account.
2. Disclosure of confidential organizer information.
3. Unauthenticated accessibility of the SQL backup resource.

## Repository Structure
- `report/` — Full assessment report.
- `evidence/` — Selected, redacted screenshots supporting findings.
- `findings/` — Technical finding summaries.

## Scope and Limitations
Testing was restricted to the challenge portal and supplied account. The SQL backup was checked using HEAD requests only; its contents were not downloaded or examined. IDOR and mass assignment were not confirmed.

## Disclaimer
This repository documents an authorized educational security assessment. No destructive testing or unauthorized data extraction was performed.
