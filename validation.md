# Relief at Source API – Data Validation

Validation is **strict and local on the member-details payload**: NINO, names and date of birth are checked against regexes that mirror the OpenAPI contract, and failures come back as a structured 400 listing each failing field. Everything else is **structural or absent**:

- Path identifiers, file names and Upscan callback data are checked for presence and type only.
- Nothing checks the outbound DES payload against a schema.
- DES responses are parsed leniently, and their status strings are mapped by substring replacement.

The bulk CSV route reuses the API readers but accepts more date formats. It reports each row's errors inside the results file instead of rejecting the file.

## Validation layers at a glance

| # | Layer | Mechanism | Where | On failure |
|---|---|---|---|---|
| 1 | Accept header / API version | Regex `application/vnd\.hmrc\.(\d+\.\d+)\+json` plus allowed list | `SimpleHeaderValidator.validateAccept`, `LookupController.validateVersion` | 406 `ACCEPT_HEADER_INVALID` |
| 2 | Body parsing | Play default `AnyContent` parser, then `body.asJson` | All POST controllers | Malformed JSON: 400 `BAD_REQUEST` (via `RasErrorHandler.onClientError`). Non-JSON content type: 400 (varies by controller, see below) |
| 3 | Auth / identity | Enrolment predicate | `LookupController`, `FileController`, `FileSessionController` | 401 (or 500 for unexpected auth errors) |
| 4 | Member details | Custom `Reads` in `models.JsonReads` + `IndividualDetails.individualDetailsReads` | `LookupController.withValidJson` | 400 `BAD_REQUEST` with an `errors[]` array |
| 5 | Bulk CSV rows | Same readers with `individualDetailsBulkReads` | `ResultsGenerator.createMatchingData` | Errors written into the results CSV row |
| 6 | File-session request | `nonEmptyString` filter on `userId` and `reference` | `CreateFileSessionRequest.reads` | 400 plain text |
| 7 | Upscan callback | `Json.format[UpscanCallbackData]` (shape only) + version param | `FileProcessingController.statusCallback` | 400 empty body; **500** for a JSON body with the wrong shape |
| 8 | Path-userId ownership | Equality with the auth-derived ID | `FileSessionController.fetch/deleteFileSession` | 400 plain text |
| 9 | DES response | `Json.format` for success, then failure, otherwise generic error | `DesConnector.resolveResponse` | Internal failure code; API caller gets 500 |

## Route and path parameters

| Parameter | Rule | Code |
|---|---|---|
| `userId` (callback) | Any string. Not validated and not authenticated | `FileProcessingController.statusCallback` |
| `version` query (callback) | Must be `1.0` or `2.0` (default `2.0`), otherwise 400 | `statusCallback` (`optVersion`) |
| `name` (getFile/remove) | Any string. Not validated. Used as a GridFS `filename` filter and in `Content-Disposition` | `FileController.serveFile`, `remove` |
| `userid` (remove) | **Ignored**: the authenticated ID is used instead | `FileController.remove` |
| `userId` (get/delete file session) | Must equal the auth-derived ID, otherwise 400 | `FileSessionController` |
| `version`, `endpointName`, `*file` (documentation) | No explicit validation; passed to `Assets.at` (spaces in `endpointName` become `-`) | `Documentation` |

## Request headers

| Endpoint | Headers | Validation |
|---|---|---|
| `POST /residency-status` | `Accept` | Must contain `application/vnd.hmrc.1.0+json` or `application/vnd.hmrc.2.0+json`, **and** the version extracted by regex must pass `validateVersion` (`1.0`, or `2.0` when `api-v2_0.enabled`). Otherwise 406. Checked **before** auth. |
| `POST /residency-status` | `Content-Type` | Not checked explicitly. A non-JSON body leads to `asJson = None` and 400 `BAD_REQUEST` |
| `POST /residency-status` | `Authorization` | Auth service, plus OAuth at the API Platform |
| All others | — | Nothing beyond the auth session |

⚠️ **Observation:** `validateVersion` still accepts `1.0` even though `api.versions.v1_0.status = RETIRED`. Blocking it depends on the API Platform. When the API version is missing from audit events, they record `rasApiVersion = MISSING`, but layer 1 means this cannot happen on this route.

## Request bodies

### Residency-status lookup (API)

Reader: `IndividualDetails.individualDetailsReads`, run inside `Try(json.validate(...))`.

| Field | Required | Rule | Error code |
|---|---|---|---|
| `nino` | yes | Must be a JSON string, non-empty, and its **upper-cased** form must match `^((?!(BG\|GB\|KN\|NK\|NT\|TN\|ZZ)\|(D\|F\|I\|Q\|U\|V)[A-Z]\|[A-Z](D\|F\|I\|O\|Q\|U\|V))[A-Z]{2})[0-9]{6}[A-D]?$` | `INVALID_DATA_TYPE`, `MISSING_FIELD`, `INVALID_FORMAT` |
| `firstName`, `lastName` | yes | Must be a JSON string, non-blank after trim, and match `^[a-zA-Z &`\-'^]{1,35}$` | as above |
| `dateOfBirth` | yes | Must be a JSON string, non-blank, match `^\d{4}-\d{2}-\d{2}$`, parse strictly as `uuuu-MM-dd`, and not be after today | `INVALID_DATA_TYPE`, `MISSING_FIELD`, `INVALID_FORMAT`, `INVALID_DATE` |

A missing field produces `error.path.missing`, which `ErrorConverter` maps to `MISSING_FIELD`. The error response looks like this:

```json
{"code":"BAD_REQUEST","message":"Bad Request","errors":[{"code":"INVALID_FORMAT","message":"Invalid format has been used","path":"/nino"}]}
```

A missing or non-JSON body returns `{"code":"BAD_REQUEST","message":"Bad Request"}` with no `errors`.

**Not checked:**
- extra or unknown fields, which are ignored
- names made only of punctuation (e.g. `'-'`) or with leading or trailing spaces
- a plausible minimum age or earliest date of birth
- NINO with embedded spaces, which is rejected rather than normalised

⚠️ **Observation:** `ErrorConverter.getErrorDetails` throws a `MatchError` for any validation key it does not recognise. That would surface as a 500, through `recoverWith { case ex => … }` in the auth block, instead of a 400. The current readers only produce recognised keys, so this is a latent trap for anyone adding a new reader.

**Normalisation (inside the reader):**

| Field | Transformation |
|---|---|
| `nino` | `padTo(9, " ")` then `toUpperCase`. An 8-character NINO without a suffix becomes `XX999999␠` |
| `firstName`, `lastName` | `toUpperCase` (no trimming) |
| `dateOfBirth` | Parsed to `LocalDate`. Sent to DES as `dob` in `yyyy-MM-dd` |
| (derived) | `pensionSchemeOrganisationID` is added from the auth enrolment identifier |

### Bulk CSV rows

Reader: `IndividualDetails.individualDetailsBulkReads`. It has the same rules as the API, plus these differences:

- `dateOfBirth` may be `uuuu-MM-dd`, `dd/MM/uuuu`, `dd-MM-uuuu` or `uuuu/MM/dd` (strict parsing, not in the future).
- Row parsing in `ResultsGenerator.parseString`:
  - The row is split on `,`, with no CSV quoting support.
  - Missing columns are padded with `""`, so they produce `MISSING_FIELD`. Columns after the fourth are ignored.
  - One pair of quotes around the **whole row** is stripped (`removeDoubleQuotes`). Quotes around individual fields are not, so they cause `INVALID_FORMAT`.
- Empty lines are skipped. A header row is not detected: it is processed as data and fails validation.
- The file is decoded as ISO-8859-1.
- Each invalid row is written as `<original row>,<field>-<CODE>,...`, for example `…,nino-INVALID_FORMAT,dateOfBirth-INVALID_DATE`. If `validate` throws, the row gets `INVALID RECORD`.
- There are no limits on file size or row count: the whole file is read with `toList`, and the result file is read back with `Files.readAllBytes`.

⚠️ **Observation:** the input row is echoed into the results CSV without escaping. A cell that starts with `=`, `+`, `-` or `@` is passed through, so a spreadsheet could execute it as a formula (CSV/formula injection).

### File session (create)

Reader: `CreateFileSessionRequest.reads`. Both `userId` and `reference` must be non-empty strings.

| Failure | Response |
|---|---|
| Not JSON | 400 `Missing or invalid json body` (plain text) |
| `JsError` | 400 `Json could not be parsed. Errors: …` |

⚠️ **Observation:** `userId` comes from the body and is **not compared** with the authenticated enrolment ID. An authenticated user can therefore create or overwrite another organisation's session. `reference` is validated but never stored; it only appears in logs.

### Upscan callback

Reader: `UpscanCallbackData.formats` (`Json.format`).

- Required: `reference: String` and `fileStatus: String`.
- Optional: `downloadUrl`, `uploadDetails {uploadTimestamp: Instant, checksum, fileMimeType, fileName, size: Int}` and `failureDetails {failureReason, message}`.
- There is no check on the URL format or host, the `fileStatus` value, the MIME type, the size or the checksum.

| Input | Result |
|---|---|
| Unsupported `version` | 400 (empty body) |
| No JSON body | 400 |
| JSON that fails `validate` (`JsError`) | **500**. `withValidJson` only matches `Success(JsSuccess)` and `Failure`, so `Success(JsError)` throws a `scala.MatchError` synchronously inside `Action.async`. Play sends it to `RasErrorHandler.onServerError` → 500 `INTERNAL_SERVER_ERROR`. No unit test covers this case. |
| `fileStatus = READY` with `downloadUrl` | 200. Processing starts asynchronously and the session is updated with the callback data |
| `fileStatus = READY` without `downloadUrl` | 200. Processing doesn't start and the session isn't updated |
| `fileStatus = FAILED` | 200. The session is updated |
| Any other `fileStatus` | 200. Only a warning is logged |

## External validation services

DES is the member-matching authority:
- The service checks only the format of the member details. DES decides whether the NINO, name and date of birth match.
- A non-match leads to a 403 `STATUS_UNAVAILABLE` (API) or `cannot_provide_status` (CSV).
- The service fails **closed**: any unexpected DES response becomes an error, never a default status. (`allow-default-ruk` only applies before 2018-04-06, so it is dormant.)

## Outbound payload validation

- No JSON Schema files are bundled for DES, and the DES request is not validated before it is sent.
- The payload is built from the `IndividualDetails` already validated by the reader, plus the auth-derived organisation ID.
- On failure, logs contain the `userId` only, not the NINO or names. The NINO does appear in audit events.
- The only schemas in the repository are the public OpenAPI files (`resources/public/api/conf/{1.0,2.0}/application.yaml`). They are served as documentation and never loaded for validation.

## Upstream response validation

| Response | Validation | Failure result |
|---|---|---|
| DES success body | `Json.format[ResidencyStatusSuccess]`: `nino` and `currentYearResidencyStatus` are required strings | Falls through to failure parsing |
| DES failure body | `Json.format[ResidencyStatusFailure]`: `code` and `reason` are required | Neither parse succeeds → `INTERNAL_SERVER_ERROR` |
| Residency status values | `String.replace("Uk"→…, "Scottish"→…, "Welsh"→…)`, with no enum check | ⚠️ Unknown or unexpected values pass through to the API response, outside the contract's `scotResident/welshResident/otherUKResident` enum |
| Missing next-year status | `allow-no-next-year-status` toggle (default false) | `DO_NOT_RE_PROCESS` → 500 + audit. ⚠️ This applies all year round, not only in the Jan–Apr window when the forecast is returned |
| HTTP status | **Not inspected**, because `execute[HttpResponse]` uses `readRaw` | The mapping depends on the body `code`. Unknown DES codes go to 500 for the API, and into the CSV **verbatim** |
| Upscan download | None: the stream is read as text lines | Exception → no session update (see below) |

⚠️ **Observation:** if the Upscan download or read fails, `processFile`'s `onComplete` does nothing (`inputFileData.isSuccess` is false). The session keeps the READY callback data with no results file and no FAILED status, so the user is left waiting.

## Cache and opaque stores

- `fileSessions` stores whatever `FileSession` is written: the callback data as received, and the results metadata. Nothing is re-validated when it is read.
- GridFS results files are stored as written. Files are looked up only by `filename`; the `metadata.userId` is not checked.

## Error mapping summary

| Source | Exception/value | HTTP to caller |
|---|---|---|
| Accept header | missing or invalid version | 406 `ACCEPT_HEADER_INVALID` |
| Body parser | malformed JSON | 400 `BAD_REQUEST` |
| Lookup body | non-JSON | 400 `BAD_REQUEST` |
| Lookup body | `JsError` | 400 `BAD_REQUEST` + `errors[]` |
| Lookup body | exception thrown during `validate` or `onSuccess` | 500 `INTERNAL_SERVER_ERROR` |
| Auth | `InsufficientEnrolments` | 401 `UNAUTHORIZED` (lookup, file) / 401 plain text (session) |
| Auth | `NoActiveSession` | 401 `INVALID_CREDENTIALS` (lookup, file) / 401 plain text (session) |
| Auth | other exception | 500 |
| DES | `deseasedIndicator=true` | 403 `STATUS_UNAVAILABLE` (same as not matched) |
| DES | body code `STATUS_UNAVAILABLE`, or 403/404 exception | 403 `STATUS_UNAVAILABLE` |
| DES | body code `TOO_MANY_REQUESTS`, or 429 exception | 429 `TOO_MANY_REQUESTS` |
| DES | body code `SERVICE_UNAVAILABLE`, or 503 exception | 503 `SERVER_ERROR` |
| DES | missing next-year status, 400 exception, timeout, unparseable body, unknown code | 500 `INTERNAL_SERVER_ERROR` |
| Lookup future | any `Throwable` | 500 (audited with no NINO) |
| Callback | bad version or no JSON | 400 |
| Callback | wrongly shaped JSON | 500 (`MatchError`) |
| File session | userId mismatch | 400 |
| File session | not found | 404 plain text |
| File session | repository failure | 500 plain text |
| File download | not found | 404 `NOT_FOUND` |
| File download | repository exception | 500 (empty) |
| File remove | `removeFile` false or exception | 500 (empty) |
| Unknown route | — | 404 `NOT_FOUND` |

## Recommendations

1. Handle `Success(JsError)` in `FileProcessingController.withValidJson` so that a malformed callback returns 400 instead of 500.
2. Compare `CreateFileSessionRequest.userId` with the authenticated ID, or drop it from the body and use the auth ID. Return 403 instead of 400 on ownership mismatch for fetch and delete.
3. Filter GridFS look-ups by `metadata.userId` in `fetchFile` and `removeFile`.
4. Validate the callback `downloadUrl` against the expected Upscan host(s), and consider authenticating the callback route.
5. Map DES responses by HTTP status (for example `execute[Either[UpstreamErrorResponse, HttpResponse]]`) as well as by body code. Treat unknown DES codes as an internal error rather than passing them through to CSV output.
6. Check DES residency values against an explicit enum (`Uk`, `Scottish`, `Welsh`). Treat anything else as an internal error.
7. Escape CSV formula characters when echoing input rows. Consider limits on file size and row count, and a way to detect header rows.
8. Mark the session FAILED when the Upscan download or read fails.
9. Make `ErrorConverter` total: fall back to a generic 400 code instead of throwing `MatchError`.
