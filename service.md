# Relief at Source API – Summary

## Purpose

The Relief at Source (RAS) API tells pension scheme administrators and practitioners which part of the UK a scheme member is resident in for income tax purposes: Scotland, Wales or the rest of the UK. Administrators need this to claim relief at source on members' pension contributions at the correct rate of tax. The service does not keep a register of members. It checks the member's details against HMRC's records each time, through HMRC's head-of-duty system for residency status.

The service has no UI and is used in two ways:

- **Single lookups through the public API**: third-party pension software calls the API through the HMRC API Platform, using a user-restricted OAuth token (scope `read:ras`). Version 2.0 is current. Version 1.0 is marked retired and, per the README, is being removed.
- **Bulk lookups for the RAS frontend**: an administrator uploads a CSV of members through the companion frontend, and the file passes through the platform's file-upload (virus-scanning) service. This service receives the upload callback, looks up every row, builds a results CSV, stores it for a limited time and serves it back for download.

## Main Capabilities

### 1. Single residency-status lookup (public API)

- The caller sends a member's National Insurance number, first name, last name and date of birth.
- The member's details are checked for format before any upstream call is made: NINO pattern, letters-only names up to 35 characters, an ISO date of birth that is not in the future.
- The service returns the residency status for the **current tax year**. Between 1 January and 5 April it also returns a **forecast for the next tax year**. For the rest of the year, any forecast is removed.
- The statuses are `scotResident`, `welshResident` and `otherUKResident`. In version 1.0, Welsh residents are reported as `otherUKResident`.
- Business outcomes:
  - **Member not matched, or member deceased**: the same "cannot provide a residency status" response (403) is returned in both cases, so the API does not reveal that the member has died.
  - **Upstream throttled**: 429.
  - **Upstream unavailable**: 503.
  - **Anything else unexpected**: 500.
- If the upstream system leaves out the next-year forecast during the forecast window, the lookup fails as an internal error (unless a toggle allows it) and an audit event records it.

### 2. Bulk residency-status lookup (file processing)

- The file-upload service calls this service back when a user's file is ready or has failed.
- **Ready**: the file is downloaded and read line by line. Each row (NINO, first name, last name, date of birth) is validated with the same rules as the API, except that it accepts four date formats. Each valid row is then looked up.
- The results CSV repeats each input row and adds the status columns. It has a header row with one or two tax-year columns, depending on the time of year. Validation errors (such as `nino-INVALID_FORMAT`) are written in place of a status, and so are business failures: `cannot_provide_status` and `problem-getting-status`.
- Bulk lookups can be retried automatically with a delay when a feature toggle is on. "Not matched", "deceased" and "do not reprocess" outcomes are never retried.
- **Failed**: the failure is recorded against the user's file session.

### 3. File session tracking

- A file session is one record per user (pension scheme organisation). It tracks the upload reference, the upload's status, the results-file metadata and when the upload started.
- The frontend creates, fetches and deletes the session. Sessions expire automatically (4 days by default).

### 4. Results-file download and removal

- An authenticated pension user can download the results CSV by file name, or delete it.
- Stored results files expire automatically after a configurable period (1 hour by default).
- Deleting a file also tries to remove its data chunks. At start-up, a housekeeping job can remove orphaned chunks left behind from earlier deletions.

### 5. API publishing

- The service publishes its API definition (context `individuals/relief-at-source` and version statuses) and its OpenAPI documentation to the API Platform.

## Cross-cutting Concerns

- **Authentication and authorisation**:
  - Every user-facing endpoint needs a Government Gateway login holding one of four pension enrolments: pension scheme administrator, pension practitioner, or either of their scheme-administration-system (PODS) equivalents.
  - The pension scheme organisation ID is taken from that enrolment and used as the user identity.
  - The upload status callback is unauthenticated.
- **Auditing**:
  - Each lookup is audited: API and bulk lookups, successful and failed, with the NINO, user ID, statuses, reason and API version.
  - File deletions are audited, and so are upstream responses that are missing the next-year status.
- **Validation and transformation**:
  - Names and the NINO are upper-cased. The NINO is padded to 9 characters.
  - Upstream status values (`Uk`, `Scottish`, `Welsh`) are mapped to the API's vocabulary.
  - Internal status codes are translated to user-facing CSV wording.
- **Data protection**:
  - Results files and file sessions are short-lived, enforced by TTL indexes.
  - Logs refer to users by organisation ID, not by NINO.
  - Audit events contain NINOs.
  - Nothing is encrypted at the application level.
- **Feature toggles**:
  - allow a missing next-year status
  - default the current year to rest-of-UK (pre-2018 only, so effectively dormant)
  - retry for all lookups, or for bulk lookups only
  - enable API v2.0
  - run the orphaned-chunk clean-up at start-up
- **Metrics**: timers for lookups and file processing, and counters for error responses.

## Dependencies (generic)

| Kind | Role |
|---|---|
| Identity/authorisation service | Checks the Government Gateway session and pension enrolments, and supplies the organisation ID |
| HMRC residency-status system of record (via the integration gateway) | Matches the member and returns the current-year and next-year residency status and the deceased indicator |
| File-upload / virus-scanning service | Holds uploaded CSVs. Calls back with the status and a download URL |
| Document database (MongoDB) | Stores file sessions (cache) and results files (GridFS), both with TTLs |
| Audit / data-stream service | Receives audit events for lookups and deletions |
| API Platform | Publishes the API definition and documentation, and handles OAuth in front of the public endpoint |

## In One Sentence

The RAS API tells pension scheme administrators, one member at a time or by bulk CSV, whether each member pays income tax as a Scottish, Welsh or rest-of-UK resident, so that the right rate of relief is claimed on their pension contributions.
