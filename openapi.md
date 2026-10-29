# Relief at Source API – HTTP API

The OpenAPI 3.0.3 spec is in [`openapi.json`](openapi.json). It was generated from an endpoint table built by reading `conf/prod.routes` (which includes `app.routes`, `definition.routes` and bootstrap's `health.Routes`), the controllers, the Play JSON readers and writers, and `RasErrorHandler`. It covers **11 operations on 11 paths**: 10 from the repository's routes files plus bootstrap's `/ping/ping`. Locally the service runs on port 9669 (`sbt "run 9669"`). Only `POST /residency-status` is published to the API Platform, as `POST /individuals/relief-at-source/residency-status`. Its public contract is `resources/public/api/conf/2.0/application.yaml`. The other routes are internal (RAS frontend, Upscan, API Platform).

## Conventions that apply to every endpoint

| Concern | Behaviour |
|---|---|
| Authentication | User-facing endpoints call the auth service: `AuthProviders(GovernmentGateway)` and one of the enrolments `HMRC-PSA-ORG`, `HMRC-PP-ORG`, `HMRC-PODS-ORG`, `HMRC-PODSPP-ORG`. The organisation ID is that enrolment's first identifier. The Upscan callback, `/api/*` and `/ping/ping` are unauthenticated. On the API Platform, OAuth (scope `read:ras`) sits in front of the public endpoint. |
| Request body parsing | Play's default `AnyContent` parser, then `body.asJson`. With a JSON `Content-Type`, malformed JSON is rejected **before the controller** with 400 `{"code":"BAD_REQUEST","message":"Bad Request"}` (`RasErrorHandler.onClientError`). A non-JSON content type reaches the controller as `asJson = None`, and each controller returns its own 400. |
| Parameter binding | All path parameters are `String`. Nothing makes a request fail with 400 before the controller runs, apart from malformed JSON. The callback's `version` query parameter defaults to `2.0`, and the controller rejects anything other than `1.0`/`2.0` with 400. |
| Accept header | Only `POST /residency-status` checks it, **before** auth. It must contain `application/vnd.hmrc.1.0+json` or `application/vnd.hmrc.2.0+json`, and the version must be allowed (`2.0` only while `api-v2_0.enabled`). Otherwise the response is 406 `ACCEPT_HEADER_INVALID`. |
| Error bodies | The lookup and results-file endpoints use JSON `{code, message}` (plus `errors[]` for field validation). The file-session endpoints return **plain-text** 400/401/404/500 bodies. The callback returns empty 200 and 400 bodies. |
| Unhandled exceptions | `RasErrorHandler.onServerError` → 500 `{"code":"INTERNAL_SERVER_ERROR","message":"Internal server error"}`. |
| Unknown routes | 404 `{"code":"NOT_FOUND","message":"Resource Not Found"}`. |
| Size limits | No application limits. Play's default in-memory body buffer applies (`play.http.parser.maxMemoryBuffer` is not overridden), and larger bodies get 413 from the framework. |
| Auditing | `ReliefAtSourceResidency` for each lookup outcome (including the NINO and `rasApiVersion`), and `FileDeletion` for each results-file removal. Bootstrap's implicit request auditing also applies. |

## Access-controlled endpoints

| Endpoint | Check | Failure |
|---|---|---|
| `POST /residency-status` | GG + pension enrolment | 401 JSON `UNAUTHORIZED` (insufficient enrolments) or `INVALID_CREDENTIALS` (no session). Any other auth error: 500 JSON |
| `GET /ras-api/file/getFile/{name}` | GG + pension enrolment. ⚠️ The file is looked up by `name` only | as above |
| `DELETE /ras-api/file/remove/{name}/{userid}` | GG + pension enrolment. `userid` is ignored | as above |
| `POST /create-file-session` | GG + pension enrolment. ⚠️ the body's `userId` is not compared with the caller | 401 plain text (the exception message) for any `AuthorisationException`. Anything else: 500 JSON via the error handler |
| `GET /get-file-session/{userId}` | GG + pension enrolment, and path `userId` must equal the caller's organisation ID | 401 plain text; 400 plain text on mismatch |
| `DELETE /delete-file-session/{userId}` | as above | as above |

The file-session guard takes the identifier from `enrolments.head`, without filtering by enrolment key. The other guards take it from the first enrolment whose key matches one of the four pension keys.

## Version routing (residency status)

| Accept version | Condition | Behaviour |
|---|---|---|
| `1.0` | always accepted locally (`api.versions.v1_0.status = RETIRED`; blocking is left to the API Platform) | Welsh residents are reported as `otherUKResident` |
| `2.0` | `api-v2_0.enabled = true` | `welshResident` is returned |
| anything else / missing | — | 406 |

The callback's `version` query parameter selects the same v1.0/v2.0 mapping for bulk results.

## Back-end services

- **DES** (`microservice.services.des`): `POST /individuals/residency-status/`, called by the residency-status lookup and, for each CSV row, during bulk processing.
- **Auth** (`microservice.services.auth`): enrolment checks for every user-facing endpoint.
- **Upscan**: the results-file download uses the `downloadUrl` from the callback, during asynchronous bulk processing.
- **MongoDB**: `fileSessions` (file sessions) and the GridFS bucket `resultsFiles` (results CSVs).
- **Datastream/audit**: audit events.

## Endpoints with non-standard behaviour

- **`POST /residency-status`**:
  - The next-year forecast is returned only from 1 January to 5 April, and removed the rest of the year.
  - "Not matched" and "deceased" both return the same 403 `STATUS_UNAVAILABLE`.
  - DES outcomes map to 429 `TOO_MANY_REQUESTS`, 503 `SERVER_ERROR` or 500.
  - ⚠️ **Observation:** the 429 body is written with the explicitly imported `errorResponseWrites`, giving `{code, message}`. The `TooManyRequestsResponse` companion writer would give `{"tooManyRequestsStatus": …}`. Scala 3 picks the lexical given first, but no controller test pins the 429 body.
  - Field validation failures return 400 with `errors[]`. A missing or non-JSON body returns 400 without `errors`.
- **`POST /ras-api/file-processing/status/{userId}`**:
  - The callback is unauthenticated, and `userId` comes from the URL that Upscan was given.
  - Once accepted, it always returns 200 with an empty body. Bulk processing runs asynchronously, and its failures are never reported back to the caller.
  - ⚠️ **Observation:** a JSON body that does not fit the callback shape returns **500**, not 400. `withValidJson` doesn't match `Success(JsError)`, so a `MatchError` is thrown inside the action.
- **`GET /ras-api/file/getFile/{name}`**:
  - The file is streamed (`HttpEntity.Streamed`) as `application/csv`.
  - `name` is put **unescaped** into `Content-Disposition: attachment; filename="<name>"`.
  - Repository failures return 500 with an **empty** body.
- **`DELETE /ras-api/file/remove/{name}/{userid}`**:
  - The `userid` path segment is ignored.
  - Chunk removal is attempted only when `name` parses as a Mongo ObjectId. The outcome is audited, but it never changes the response status: 200 empty if the file was removed, otherwise 500 empty.
- **`POST /create-file-session`**: returns 201 with an empty body. 400, 401 and 500 bodies are plain text.
- **`GET /api/definition`**: the JSON is rendered from a Twirl `.txt` template, so `Content-Type` is `text/plain`.
- **`GET /api/documentation/{version}/{endpointName}`**: there is no `public/api/documentation` directory in `resources/`, so this route always returns 404. The 200 in the spec is theoretical.
- **`GET /api/conf/{version}/{file}`**: `*file` spans several segments. It is served by Play `Assets.at` from `/public/api/conf/<version>/`, for example `2.0/application.yaml` or `common/overview.md`.
- **`GET /ping/ping`**: provided by bootstrap's `health.Routes`, not by a routes file in the repository.

## Endpoint catalogue

### Residency status

| Method | Path | Purpose | Auth | Back end |
|---|---|---|---|---|
| POST | `/residency-status` | Look up a member's residency status | Government Gateway + one of HMRC-PSA-ORG / HMRC-PP-ORG / HMRC-PODS-ORG / HMRC-PODSPP-ORG | `des` → `POST /individuals/residency-status/` |

### Bulk file processing

| Method | Path | Purpose | Auth | Back end |
|---|---|---|---|---|
| POST | `/ras-api/file-processing/status/{userId}` | Upscan upload-status callback | None (unauthenticated) | — (served locally) |

### Results files

| Method | Path | Purpose | Auth | Back end |
|---|---|---|---|---|
| GET | `/ras-api/file/getFile/{name}` | Download a results CSV | Government Gateway + pension enrolment (as lookup) | — (served locally) |
| DELETE | `/ras-api/file/remove/{name}/{userid}` | Delete a results CSV | Government Gateway + pension enrolment (as lookup) | — (served locally) |

### File sessions

| Method | Path | Purpose | Auth | Back end |
|---|---|---|---|---|
| POST | `/create-file-session` | Create a file session for an upload | Government Gateway + pension enrolment | — (served locally) |
| GET | `/get-file-session/{userId}` | Fetch the caller's file session | Government Gateway + pension enrolment; path userId must equal the enrolment identifier | — (served locally) |
| DELETE | `/delete-file-session/{userId}` | Delete the caller's file session | Government Gateway + pension enrolment; path userId must equal the enrolment identifier | — (served locally) |

### API publishing

| Method | Path | Purpose | Auth | Back end |
|---|---|---|---|---|
| GET | `/api/definition` | API Platform definition | None | — (served locally) |
| GET | `/api/documentation/{version}/{endpointName}` | Endpoint XML documentation (legacy) | None | — (served locally) |
| GET | `/api/conf/{version}/{file}` | Published OpenAPI spec and docs | None | — (served locally) |

### Platform

| Method | Path | Purpose | Auth | Back end |
|---|---|---|---|---|
| GET | `/ping/ping` | Health check (bootstrap health.Routes) | — | — (served locally) |
