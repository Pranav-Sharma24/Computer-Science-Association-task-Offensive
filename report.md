# IgniteFest 2026 — Red Team Security Assessment

**Target:** `https://csa2026.pythonanywhere.com`  
**Assessment window:** 4–5 November 2026 (based on captured browser evidence)  
**Assessor:** `[Your name / team]`  
**Status:** Final draft — add assessor/team details before submission.

## 1. Executive Summary

A limited-scope assessment of the IgniteFest 2026 web portal identified three material security issues:

1. The supplied student account was returned by the login workflow with the role `organizer` and could access organizer-only functionality.
2. The authenticated session could retrieve organizer statistics and a confidential committee briefing.
3. The SQL backup path was accessible to an unauthenticated browser session: a `HEAD` request with credentials omitted returned HTTP 200, and authenticated response headers identified the resource as a downloadable SQL file.

The SQL backup was **not downloaded or inspected**. The assessment confirms unauthenticated access to the resource and its metadata, but does not claim that database records were extracted. The root cause of the unexpected organizer role was not established; account provisioning or backend role assignment requires investigation.

## 2. Scope and Rules of Engagement

- **In-scope target:** IgniteFest 2026 portal at `csa2026.pythonanywhere.com`.
- **Account used:** Only the credentials supplied for the challenge.
- Testing was limited to normal application navigation, browser Network response inspection, client-side JavaScript review, and non-content-retrieving `HEAD` requests to the disclosed backup path.
- No other accounts were used.
- No database dump was downloaded.
- No other users' registration records were deliberately accessed.
- No destructive or state-changing security tests were performed.

## 3. Methodology

1. Authenticated using the challenge-provided student credentials.
2. Inspected login and profile API responses in browser Developer Tools.
3. Reviewed the supplied client-side `app.js` for endpoint references and security-relevant implementation clues.
4. Observed the Organizer Hub, organizer statistics and confidential briefing returned to the authenticated session.
5. Reviewed the registrations page and one ticket record associated with the supplied account.
6. Checked the disclosed backup path using `HEAD`, first in the authenticated browser and then in Incognito with credentials omitted.

## 4. Confirmed Findings

### F-01 — Supplied student account receives organizer privileges

**Severity:** High (preliminary; validate against the intended challenge account configuration)  
**Category:** Broken access control / incorrect privilege assignment; root cause unconfirmed.

**Evidence and observations**

- The login request submitted the email and password only; the client-side login function did not submit a role field.
- The login response returned `role: "organizer"` for the supplied student account.
- The Organizer Hub became available in that session.
- Organizer-only API requests succeeded in that session.
- A detailed registration record associated with the supplied account also showed `attendee_role: "organizer"`.
- A profile update containing only `name`, `phone`, `college` and `bio` returned `role_changed: false`. This test did **not** demonstrate mass assignment.

**Impact**

A user expected to operate as a student was able to reach organizer functionality. This enabled access to organizer statistics and the confidential committee briefing. The extent of any additional administrative permissions was not assessed.

**Reproduction summary**

1. Sign in using only the challenge-provided student credentials.
2. Inspect the login response in Developer Tools → Network.
3. Observe that the returned user role is `organizer`.
4. Observe the Organizer Hub and organizer API requests in the same session.

**Root-cause limitation**

The client-side source and observed responses establish the unexpected role and its effects, but do not establish whether the cause is a database/account-provisioning error, authentication logic, or another backend defect.

**Remediation**

- Correct the account's server-side role assignment.
- Enforce deny-by-default, server-side role-based access control on every organizer endpoint.
- Derive privileges only from trusted server-side records.
- Prevent user-controlled role updates.
- Add automated tests verifying that student accounts cannot access organizer resources.

### F-02 — Confidential organizer briefing disclosed to the supplied account

**Severity:** High (preliminary)  
**Category:** Sensitive information disclosure / broken access control.

**Evidence and observations**

- `GET /api/organizer/stats` returned HTTP 200 and organizer statistics, including event and staff information.
- `GET /api/organizer/briefing` returned HTTP 200 and a document labelled as restricted organizer material.
- The briefing response contained internal operational information and the challenge evaluation flag.

**Impact**

Disclosure of internal event operations information could compromise physical access controls, VIP handling, event security and confidential financial arrangements. Staff information returned by the statistics endpoint should also be treated as sensitive.

**Reproduction summary**

Using the supplied account's authenticated session, inspect requests to `/api/organizer/stats` and `/api/organizer/briefing` in Developer Tools → Network and record their status codes and response classifications.

**Remediation**

- Apply server-side authorization checks to each organizer endpoint.
- Return only the minimum fields required by the interface.
- Keep security codes and sensitive operational details out of broadly accessible API responses.
- Audit access logs and rotate any real credentials or access codes that may have been exposed.

### F-03 — SQL backup resource accessible without session credentials

**Severity:** High (preliminary; contents and data sensitivity not assessed)  
**Category:** Sensitive file exposure / deployment misconfiguration  
**Affected path:** `/backups/fest_backup_2026.sql`

**Evidence and observations**

- The client-side `app.js` contained a developer note disclosing the backup path.
- An authenticated `HEAD` request returned HTTP 200.
- Response headers identified:
  - `Content-Type: application/sql`
  - `Content-Disposition: attachment; filename="fest_backup_2026.sql"`
  - `Content-Length: 5793`
- A separate Incognito request used `method: "HEAD"` and `credentials: "omit"` and returned `STATUS: 200`.

**Impact**

An unauthenticated client can reach a resource presented by the server as a SQL backup attachment. If the file contains production or staging data, it could expose personal information, credentials, tokens or other sensitive records. The file contents were not retrieved, so the presence of any particular data type is not asserted.

**Reproduction summary**

1. Open the portal in an Incognito session without logging in.
2. Open Developer Tools → Console.
3. Execute a same-origin `HEAD` request to `/backups/fest_backup_2026.sql` with credentials omitted.
4. Observe HTTP 200.
5. Compare with the authenticated response headers identifying the resource as a SQL attachment.

**Remediation**

- Remove database backups from the web root and deployment artifacts.
- Store backups in private, access-controlled storage.
- Deny public access at the web-server layer.
- Ensure deployment pipelines remove staging artifacts.
- Review access logs to determine whether the exposed resource was accessed.
- If exposure of secrets or personal data is confirmed, rotate credentials and follow the incident-response process.

**Evidence limitation:** A `HEAD` request confirms resource availability and metadata, not file contents. No GET/download or database inspection was performed.

## 5. Additional Observations — Not Confirmed Vulnerabilities

- **IDOR lead:** `app.js` requests `GET /api/registrations/${regId}` for detailed ticket records. Only a record associated with the supplied account was inspected. No cross-user access test was performed; IDOR is unconfirmed.
- **Mass-assignment lead:** The profile handler submits `name`, `phone`, `college` and `bio`. The observed response reported `role_changed: false`. No role modification was demonstrated; mass assignment is unconfirmed.
- **Client-side restrictions:** The source contains a client-side closed-event registration warning and a client-side organizer-role view check. The assessment did not establish whether corresponding backend controls can be bypassed.
- **Potential DOM injection:** Some API-derived values are inserted using `innerHTML`. No attacker-controlled payload was demonstrated; exploitability is unverified.

## 6. Challenge Flag

The flag recovered from the confidential briefing response is stored separately in [`flag.txt`](flag.txt).

The flag wording suggests intended themes, but it is not treated as independent proof of IDOR or mass assignment.

## 7. Conclusion

The assessment confirmed that the supplied student account received organizer-level access and could retrieve restricted organizer information. It also confirmed that the SQL backup path responded successfully to a request made without session credentials. These issues warrant prompt review of account-role provisioning, server-side authorization and backup deployment controls.

IDOR, mass assignment and closed-event enforcement were not confirmed and are not represented as demonstrated vulnerabilities.

## 8. Evidence Index

The `evidence/` directory contains selected screenshots supporting the findings. Personal information and sensitive operational details have been redacted in the report copies where applicable.

| File | Purpose |
|---|---|
| `E01_organizer_hub_and_flag.png` | Organizer Hub and recovered flag |
| `E02_organizer_stats_response.png` | Organizer statistics response |
| `E03_briefing_response_redacted.png` | Briefing response; sensitive operational detail redacted |
| `E04_profile_update_role_unchanged_redacted.png` | Profile update response; phone value redacted |
| `E05_backup_HEAD_status.png` | Authenticated backup HEAD request |
| `E06_backup_response_headers.png` | SQL attachment response headers |
| `E07_incognito_unauthenticated_HEAD_200.png` | Incognito request with credentials omitted returning HTTP 200 |
| `E08_registration_record_redacted.png` | Registration record associated with supplied account; phone value redacted |
