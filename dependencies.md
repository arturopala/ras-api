# Relief at Source API – Dependencies

This service is a Play 3 / Scala 3 backend built on HMRC `bootstrap-backend-play-30`. It uses two HTTP clients:

- **`HttpClientV2`** (`HttpClientV2Provider`, enabled by `play.modules.enabled += HttpClientV2Module`). Used for the DES call.
- **`WSHttp`**: a subclass of the legacy `DefaultHttpClient` that adds `buildRequestWithStream(uri)`, which returns a raw streamed `WSResponse`. Used to download Upscan files.

Base URLs come from `ServicesConfig.baseUrl("<service>")`, which reads `microservice.services.<service>.{protocol,host,port}`. Paths are hard-coded in the connectors. Auth and audit are provided by bootstrap: `AuthConnector` is configured by `microservice.services.auth`, and `AuditConnector` sends to `microservice.services.datastream`. Settings are exposed through `AppContext`, a set of lazy config getters.

## Overview

| # | Dependency | Type | Config key | Accessed via |
|---|---|---|---|---|
| 1 | DES – individual residency status | Outbound REST (POST) | `microservice.services.des`, `desauthtoken`, `environment` | `DesConnector` |
| 2 | Upscan file download | Outbound HTTP GET (stream) | none: the URL comes from the callback body | `UpscanConnector`, `RasFileReader` |
| 3 | Auth | Outbound (bootstrap) | `microservice.services.auth` | `AuthorisedFunctions` in `LookupController`, `FileController`, `FileSessionController` |
| 4 | Audit / datastream | Outbound (bootstrap) | `microservice.services.datastream`, `auditing.*` (from `backend.conf`) | `AuditService` → `AuditConnector` |
| 5 | MongoDB | Datastore | `mongodb.uri` | `RasFilesSessionRepository`, `RasFilesRepository` (GridFS), `RasChunksRepository` |
| 6 | Upscan → this service | Inbound callback | n/a | `FileProcessingController.statusCallback` |
| 7 | API Platform | Inbound (definition and documentation) | `api.*`, `api-v2_0.enabled` | `Documentation` |

---

## 1. DES – Individual residency status

- **Base URL**: `baseUrl("des")`. The local default is `http://localhost:9670`.
- **Headers** (built per request in `DesConnector.getResidencyStatus`):

| Header | Value / source |
|---|---|
| `Environment` | config `environment` (local default `local`) |
| `OriginatorId` | constant `DA_RAS` |
| `Content-Type` | `application/json` |
| `authorization` | `Bearer ` + config `desauthtoken` |
| `CorrelationId` | `DesConnector.correlationId`: derived from `hc.requestId` if it matches a UUID-prefix pattern, otherwise a fresh UUID |

| Operation | Method & path | Code | Notes / validation |
|---|---|---|---|
| Get residency status | `POST {des}/individuals/residency-status/` | `DesConnector.getResidencyStatus` → `sendResidencyStatusRequest` → `resolveResponse` | Body: `{nino, firstName, lastName, dob (yyyy-MM-dd), pensionSchemeOrganisationID}`. No schema check before sending. |

**Response handling**: the request uses `execute[HttpResponse]` (the http-verbs `readRaw` reader), so **every** HTTP status comes back as an `HttpResponse`. The body is handled in this order:

1. It is parsed as `ResidencyStatusSuccess`, with fields `nino`, `deathDate?`, `deathDateStatus?`, `deseasedIndicator?`, `currentYearResidencyStatus` and `nextYearResidencyStatus?`.
   - `deseasedIndicator == true` → failure `DECEASED`.
   - `nextYearResidencyStatus` missing and `toggle-feature.allow-no-next-year-status = false` → failure `DO_NOT_RE_PROCESS`, and an audit event `ReliefAtSourceAudit_DES_Response` is sent (`nextYearResidencyStatus=NOT_PRESENT`).
   - Otherwise the status strings are mapped: `Uk` → `otherUKResident`, `Scottish` → `scotResident`, `Welsh` → `welshResident` (v2) or `otherUKResident` (v1).
2. If that fails, the body is parsed as `ResidencyStatusFailure {code, reason}`, and the DES `code` is passed through unchanged.
3. If that also fails, the result is failure `INTERNAL_SERVER_ERROR`.

The `recover` block maps exceptions to failure codes:

| Exception | Failure code |
|---|---|
| `BadRequestException`, `RequestTimeoutException` | `DO_NOT_RE_PROCESS` |
| `NotFoundException`, `UpstreamErrorResponse(403)` | `STATUS_UNAVAILABLE` |
| `UpstreamErrorResponse(429)` | `TOO_MANY_REQUESTS` |
| `UpstreamErrorResponse(503)` | `SERVICE_UNAVAILABLE` |
| any other exception | `INTERNAL_SERVER_ERROR` |

⚠️ **Observation:** `readRaw` never raises for 4xx or 5xx statuses. I confirmed this from the bytecode of `http-verbs-play-30_3` 15.8.0: `readRaw` simply returns the response. So the `BadRequestException`, `NotFoundException` and `UpstreamErrorResponse(403/429/503)` branches only run for transport-level exceptions. In practice, mapping a DES error depends entirely on the **body `code`** matching the configured status strings (`STATUS_UNAVAILABLE`, `TOO_MANY_REQUESTS`, `SERVICE_UNAVAILABLE`, …). A DES 429 or 503 whose body has a different code ends up as a 500 to API callers.

⚠️ **Observation:** `getResidencyStatus` builds a fresh `HeaderCarrier()` and doesn't use the caller's, so `hc.requestId` is always empty. The `CorrelationId` is therefore always a fresh random UUID and is never linked to the inbound request ID.

**Retry**:
- Retry is enabled when `toggle-feature.retry-enabled` is on, or when `toggle-feature.bulk-retry-enabled` is on and the request is a bulk request (both default to `false`).
- Up to `request-retry-limit` attempts are made (default 3), with `Thread.sleep(retry-delay)` between them (default 700 ms).
- `STATUS_UNAVAILABLE`, `DECEASED` and `DO_NOT_RE_PROCESS` are never retried.
- Any `DO_NOT_RE_PROCESS` remaining at the end is rewritten as `INTERNAL_SERVER_ERROR`.

⚠️ **Observation:** `Thread.sleep` blocks a thread of the shared execution context.

**Callers**:
- `LookupController.getResidencyStatus` (API).
- `ResultsGenerator.fetchResult` (bulk). It calls `Await.result(..., 20.seconds)` for each CSV row, so rows are processed one at a time.

---

## 2. Upscan file download

- **Base URL**: none. The connector does a `GET` on the `downloadUrl` taken from the Upscan callback body.
- **Headers/auth**: none. `buildRequest(uri, Seq())` sends no headers.

| Operation | Method & path | Code | Notes |
|---|---|---|---|
| Stream the uploaded CSV | `GET {downloadUrl}` | `UpscanConnector.getUpscanFile`, used by `RasFileReader.readFile` | The body is turned into an `InputStream` and read as ISO-8859-1 lines. Any failure throws `RuntimeException("Error Streaming file from Upscan service")`. |

⚠️ **Observation:** the URL comes from an **unauthenticated** callback body and is not checked against an allow-list of Upscan hosts. Anyone who can reach the callback route can make the service fetch an arbitrary URL (an SSRF risk) and write a results file for any `userId`.

⚠️ **Observation:** `getUpscanFile` creates a new `ActorSystem()` on every call, and `RasFileReader` creates one by default. Neither is ever terminated, so resources leak.

---

## 3. Auth

`AuthorisedFunctions` with `AuthConnector`, from the bootstrap `AuthModule`. The predicate is always:

`AuthProviders(GovernmentGateway) and (Enrolment("HMRC-PSA-ORG") or Enrolment("HMRC-PP-ORG") or Enrolment("HMRC-PODS-ORG") or Enrolment("HMRC-PODSPP-ORG"))`

with the retrieval `authorisedEnrolments`.

| Guard | Used by | Requires | Retrieves / derives | On failure |
|---|---|---|---|---|
| Inline `authorised(...).retrieve(authorisedEnrolments)` + `getEnrolmentIdentifier` | `LookupController.getResidencyStatus`, `FileController.serveFile`, `FileController.remove` | GG plus one of the 4 enrolments | The first identifier value of the first matching enrolment = `userId` | `InsufficientEnrolments` → 401 `UNAUTHORIZED`; `NoActiveSession` → 401 `INVALID_CREDENTIALS`; other → 500 |
| `FileSessionController.isAuthorised` | create, fetch and delete file session | same | `enrolments.head.identifiers.head.value` (no key filter) | any `AuthorisationException` → 401 with the exception message |
| none | `FileProcessingController.statusCallback`, `Documentation.*`, health | — | — | — |

The public endpoint also sits behind API Platform OAuth (scope `read:ras`), and `controllers.confidenceLevel = 300` is set in config.

---

## 4. Audit

`AuditService.audit(auditType, path, auditData)` builds a `DataEvent`:
- `auditSource` is `appName`.
- The tags come from `hc.toAuditTags`.
- The detail is `hc.toAuditDetails() ++ auditData`.
- It is sent with `AuditConnector.sendEvent` (fire-and-forget).

| auditType | Sent from | Detail keys |
|---|---|---|
| `ReliefAtSourceResidency` | `LookupController.auditResponse` (API) | `successfulLookup`, `CYStatus`, `NextCYStatus`?, `reason`?, `userIdentifier`, `requestSource=API`, `rasApiVersion` (`v1`/`v2`/`MISSING`), `nino`? |
| `ReliefAtSourceResidency` | `ResultsGenerator.auditResponse` (bulk) | `nino`, `fileId`, `userIdentifier`, `requestSource=FE_BULK`, `CYStatus`, `NextCYStatus`, `successfulLookup`, `reason` (blank values removed) |
| `ReliefAtSourceAudit_DES_Response` | `DesConnector.resolveResponse` | `userId`, `nino`, `nextYearResidencyStatus=NOT_PRESENT`, path `PATH_NOT_DEFINED` |
| `FileDeletion` | `FileController.remove` | `userIdentifier`, `fileName`, `chunkDeletionSuccess`, `reason`? |

Bootstrap's own request auditing is enabled only for `LookupController`, where `needsAuditing = true`.

---

## 5. Datastore (MongoDB)

| Collection | Repository | Key | Expiry | Purpose |
|---|---|---|---|---|
| `fileSessions` | `RasFilesSessionRepository` (`MongoCacheRepository`, `SimpleCacheId`) | `_id = userId`; data key `fileSession` | `mongodb.fileSessionsCacheTTLDays` (default `4.days`) via the hmrc-mongo cache TTL | One `FileSession` per user: `userFile` (the Upscan callback data), `resultsFile` metadata, `uploadTimeStamp`, `fileMetadata` |
| `resultsFiles.files` / `resultsFiles.chunks` (GridFS bucket `resultsFiles`) | `RasFilesRepository` (`gridFSG`) | `filename` = Upscan `reference`; metadata `{reference, fileId, userId, contentType=text/csv}` | TTL index `lastUpdatedIndex` on `uploadDate`, `expireAfterSeconds = results.expiry.time` (default 3600). It is dropped and recreated on **every save** (`GridFsTTLIndexing.checkAndEnsureTTL`) | Results CSVs |
| `resultsFiles.chunks` | `RasChunksRepository` | `files_id` | none | Orphaned-chunk clean-up and chunk removal on delete |
| `rasFileStore` | `RasFilesRepository` (declared `collectionName`) | — | — | Declared but not used by any query. All file operations use the GridFS bucket |

**Encryption**: none at the application level. Values are stored as plain JSON/BSON.

⚠️ **Observation:** `fetchFile` and `removeFile` take a `userId` but query only by `filename`. The `metadata.userId` stored with the file is never checked, so any authenticated pension user who knows (or guesses) a file reference can download or delete another organisation's results file.

⚠️ **Observation:** `FileController.remove` calls `chunksRepo.removeChunk(new ObjectId(fileName))`. The file name is the Upscan reference, not the GridFS `_id`, so this nearly always fails to parse (and is audited as `chunkDeletionSuccess=false`). It is also redundant, because `GridFSBucket.delete` already removes the chunks.

⚠️ **Observation:** the TTL index applies only to `resultsFiles.files`, so chunks expire only if the clean-up job runs. `DataCleansingService.removeOrphanedChunks()` runs once, eagerly, at application start-up when `remove-chunks-data-exercise.enabled = true` (default `true`). It loads all chunk `files_id`s and removes those whose parent file no longer exists.

⚠️ **Observation:** the default results-file TTL (`results.expiry.time` = 3600 s, i.e. 1 hour) contradicts the comment on `fileSessionsCacheTTLDays` ("3 days time to download processed file"). This needs checking against the environment config, which is not in this repository.

---

## 6. Inbound callers and callbacks

| Endpoint | Caller | Auth |
|---|---|---|
| `POST /residency-status` (context `/individuals/relief-at-source`) | Third-party software via the API Platform | OAuth `read:ras`, plus GG enrolment check |
| `POST /ras-api/file-processing/status/:userId?version=` | Upscan callback (the callback URL with the `userId` is registered by the frontend when it initiates the upload) | **None** |
| `GET /ras-api/file/getFile/:name`, `DELETE /ras-api/file/remove/:name/:userid` | RAS frontend | GG enrolment |
| `POST /create-file-session`, `GET /get-file-session/:userId`, `DELETE /delete-file-session/:userId` | RAS frontend | GG enrolment |
| `GET /api/definition`, `/api/documentation/:version/:endpointName`, `/api/conf/:version/*file` | API Platform | None |

The service does not register any webhooks itself. The Upscan callback URL is supplied by the frontend.

---

## 7. Libraries

| Library | Version | Backs |
|---|---|---|
| `bootstrap-backend-play-30` | 10.8.0 | Auth client, audit, `HttpClientV2`, `ServicesConfig`, JSON error handler, metrics |
| `http-verbs-play-30` (transitive) | 15.8.0 (resolved) | `HttpClientV2`, `DefaultHttpClient`, `HttpReads` |
| `hmrc-mongo-play-30` | 2.13.0 | `PlayMongoRepository`, `MongoCacheRepository`, GridFS via the Mongo Scala driver |
| `domain-play-30` | 13.0.0 | Declared, but not imported anywhere in `app/` |
| Play WS (`ws`) | Play 3.0.10 plugin | `WSHttp` streaming download |
| Scala | 3.3.7 | — |

---

## 8. Local development

- `sbt "run 9669"` (`PlayKeys.playDefaultPort := 9669`). Dependencies are started with `sm2 --start RAS_ALL`.
- Local defaults in `application.conf`:
  - auth: `localhost:8500`
  - datastream: `localhost:8100`
  - DES: `localhost:9670` (a RAS stub in the `RAS_ALL` profile)
  - Mongo: `mongodb://localhost:27017/ras-api`
- `desauthtoken` and `environment` hold placeholder values locally. Environment-specific values live outside this repository.
- `microservice.services.des.Environment` and `microservice.services.des.authorization` are present in config but **unused**: the code reads the top-level `environment` and `desauthtoken` keys instead.
- Integration tests (`it/`) use WireMock (`WireMockServerHelper`) for auth and Upscan.
