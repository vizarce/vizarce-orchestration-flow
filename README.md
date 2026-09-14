# VIZARCE AI Orchestration Flow

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Mule](https://img.shields.io/badge/Mule-4.9.x-blue.svg)](https://www.mulesoft.com/)

A Mule 4 integration application demonstrating the
[`vizarce-connector`](https://github.com/vizarce/vizarce-connector) in a realistic
enterprise orchestration scenario. Three HTTP endpoints:

- **`POST /orchestrate`** — the main flow: composes a song via VIZARCE behind a
  Circuit Breaker and Scatter-Gather, then writes the result to Salesforce as a
  `VIZARCE_Song_Request__c` Custom Object record.
- **`POST /demo/generate-lyrics`** and **`POST /demo/generate-structure`** — simple
  demo flows, each calling one VIZARCE operation directly with no orchestration
  wrapping, for quick manual testing of that operation in isolation.

The main flow combines four integration patterns:

- **Scatter-Gather** — parallel, independent calls to VIZARCE (`composeSong` and,
  when an artist reference is supplied, `buildArtistDNA`), merged into one result.
- **Until Successful** — retries the compose call on transient failure with a fixed
  backoff.
- **Circuit Breaker** — an Object-Store-backed breaker around the whole VIZARCE call:
  after 3 consecutive failures it trips open and fails fast (no further VIZARCE calls)
  for a 30-second cooldown, then allows a single half-open trial request.
- **DataWeave transform to an enterprise schema, then a real Salesforce write** — the
  merged VIZARCE result is reshaped into a Salesforce Custom Object-style payload
  (`VIZARCE_Song_Request__c`) and written to Salesforce via `salesforce:create`
  (Basic Authentication — see [Salesforce Integration](#salesforce-integration)).

## Endpoints

### POST /orchestrate

The main flow. Composes a song (and, optionally, an Artist DNA profile) via VIZARCE,
then creates a `VIZARCE_Song_Request__c` record in Salesforce.

```bash
curl -X POST http://localhost:8081/orchestrate \
  -H "Content-Type: application/json" \
  -d '{
    "concept": "a song about gravity and distance",
    "genreTag": "future-house",
    "bpm": "112",
    "musicalKey": "A minor",
    "artistName": "some artist"
  }'
```

`artistName` is optional — omit it to skip the Artist DNA route.

Responses:

| Status | Body                                                                                                            | When                                                                           |
|--------|-------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------|
| 200    | `{ status: "SUCCESS", VIZARCE_Song_Request__c: { ... }, salesforceRecordId: "...", salesforceSuccess: true }`     | Compose (and Artist DNA, if requested) and the Salesforce write both succeeded   |
| 502    | `{ status: "FAILED", errorType: "VIZARCE:..." \| "SALESFORCE:...", message: "..." }`                              | VIZARCE call failed (after retries, where applicable), or the Salesforce write failed |
| 503    | `{ status: "CIRCUIT_OPEN", message: "..." }`                                                                      | Circuit breaker is open — VIZARCE was not called at all                          |

### POST /demo/generate-lyrics

A simple demo flow — calls `vizarce:generate-lyrics` directly, with no
Scatter-Gather/Circuit Breaker/Salesforce wrapping, for easy manual testing of that
one operation in isolation.

```bash
curl -X POST http://localhost:8081/demo/generate-lyrics \
  -H "Content-Type: application/json" \
  -d '{
    "concept": "a song about gravity and distance",
    "genreTag": "future-house"
  }'
```

Responses:

| Status | Body                                                              | When                          |
|--------|--------------------------------------------------------------------|--------------------------------|
| 200    | Raw `vizarce:generate-lyrics` result                              | Call succeeded                 |
| 502    | `{ status: "FAILED", errorType: "VIZARCE:...", message: "..." }`  | VIZARCE call failed            |

### POST /demo/generate-structure

A simple demo flow — calls `vizarce:generate-structure` directly, with no
Scatter-Gather/Circuit Breaker/Salesforce wrapping, for easy manual testing of that
one operation in isolation. **Note:** the `validSectionNames` list this demo sends is
a representative hardcoded subset, not VIZARCE's full ~56-entry canonical
section-name taxonomy (that list lives client-side in VIZARCE's frontend, not
duplicated server-side — a real integration would need to fetch/maintain it itself).

```bash
curl -X POST http://localhost:8081/demo/generate-structure \
  -H "Content-Type: application/json" \
  -d '{
    "concept": "a song about gravity and distance"
  }'
```

Responses:

| Status | Body                                                              | When                          |
|--------|--------------------------------------------------------------------|--------------------------------|
| 200    | Raw `vizarce:generate-structure` result                           | Call succeeded                 |
| 502    | `{ status: "FAILED", errorType: "VIZARCE:...", message: "..." }`  | VIZARCE call failed            |

## Architecture

```
HTTP Listener (POST /orchestrate)
  → parse request into flow variables
  → check-circuit-breaker (sub-flow, reads Object Store)
  → Choice:
      ├─ breaker OPEN  → 503 fallback, VIZARCE never called
      └─ breaker CLOSED/half-open →
            Try
              ├─ Scatter-Gather
              │    ├─ route 0: Until Successful → vizarce:compose-song
              │    └─ route 1: Choice → vizarce:build-artist-dna (if artistName given) | null
              ├─ record-circuit-success (sub-flow, clears Object Store keys)
              ├─ DataWeave transform → VIZARCE_Song_Request__c shape
              ├─ salesforce:create → writes the VIZARCE_Song_Request__c record
              ├─ DataWeave transform → merges salesforceRecordId/salesforceSuccess into the response
              └─ error-handler (on-error-continue, type="ANY") → record-circuit-failure, 502 response
                   (catches VIZARCE:* and SALESFORCE:* errors alike)

HTTP Listener (POST /demo/generate-lyrics)
  → parse request into flow variables
  → vizarce:generate-lyrics
  → error-handler (on-error-continue, type="ANY") → 502 response

HTTP Listener (POST /demo/generate-structure)
  → parse request into flow variables
  → vizarce:generate-structure (hardcoded representative validSectionNames subset)
  → error-handler (on-error-continue, type="ANY") → 502 response
```

## Salesforce Integration

The main flow (`/orchestrate`) writes each successfully-composed song request to
Salesforce as a `VIZARCE_Song_Request__c` Custom Object record, via
`salesforce:create`
([mule-salesforce-connector](https://www.mulesoft.com/exchange/com.mulesoft.connectors/mule-salesforce-connector/)
10.16.0).

**Authentication:** Basic Authentication (username + password + security token) —
`salesforce:sfdc-config` / `salesforce:basic-connection`. This requires no Connected
App setup, which is why it's used here for a self-contained demo. **This is fine for
demo/sandbox use, but is explicitly not what MuleSoft recommends for production** —
their own guidance is that OAuth (JWT Bearer or Authorization Code, via
`salesforce:oauth-jwt-connection` / `salesforce:oauth-connection`) is the
production-appropriate choice, since it avoids storing a raw username/password/token
combination and supports token revocation, scoped access, and connected-app-level
audit logging.

**Getting a Security Token:** Salesforce Setup → search "Reset My Security Token" (or
My Personal Information → Reset My Security Token) → click **Reset Security Token**.
Salesforce emails the new token to the address on the user's profile. It's required
in addition to the password for API logins from an untrusted IP range (most sandbox
and developer accounts don't have IP restrictions relaxed, so this is usually
mandatory).

**Configuration:** set `salesforce.username`, `salesforce.password`, and
`salesforce.securityToken` in `src/main/resources/local.properties` (see
[Running locally](#running-locally)). The connector's Basic Auth provider
concatenates password + security token automatically — do not append the token to
the password value yourself.

**Response shape:** on success, `/orchestrate`'s 200 response merges
`salesforceRecordId` (the new record's Salesforce ID) and `salesforceSuccess` (`true`)
into the `VIZARCE_Song_Request__c`-shaped payload. A Salesforce-side failure (bad
credentials, a malformed record, connectivity) is caught by the same error handler
that catches VIZARCE failures and returned as a 502 with `errorType` prefixed
`SALESFORCE:` instead of `VIZARCE:`.

## Known limitation — `<until-successful>` retry scoping

Mule 4's `<until-successful>` retries on **any** unhandled error inside its scope; it
has no built-in mechanism to retry only on a specific error type (here,
`VIZARCE:RATE_LIMITED`) while failing fast on others (`VIZARCE:CONNECTIVITY`,
`VIZARCE:API_ERROR`, `VIZARCE:INVALID_RESPONSE`). This flow accepts that
simplification for its first version — it retries on any transient compose failure,
which is a reasonable default since VIZARCE's `/compose` calls are safe to retry.

The idiomatic workaround for genuinely selective retry-by-error-type is a hand-rolled
retry sub-flow: a recursive `flow-ref` with an explicit `<choice>` on
`error.errorType`, an attempt counter passed as a variable, and a `<raise-error>` to
signal "do not retry" for non-retryable types. Left as a documented follow-up rather
than implemented here, to keep this first version's flow readable.

## Known Anypoint Studio IDE limitations (this environment)

Two separate, confirmed Studio-side issues were hit while building this project.
Neither is a defect in this code, and **CLI Maven builds (`mvn clean package`,
`mvn clean install`) are unaffected by either** — both are specific to Studio's own
tooling.

**(a) `PackagingType`/`MULE_EXTENSION` gap — affects the connector project, not this
application.** On at least one tested Anypoint Studio installation (build 7.24–7.25,
mule-packager 4.7.0–4.9.1), the bundled
`org.mule.tools.api.packager.packaging.PackagingType` enum does not include a
`MULE_EXTENSION` constant. Studio's built-in Mule Project importer — which only
recognizes `mule-application`, `mule-domain`, `mule-domain-bundle`, and `mule-policy`
packaging — cannot open a `mule-extension`-packaged project (like `vizarce-connector`)
as a first-class Studio project with a design canvas. This is a real, confirmed gap in
that Studio build, not a misconfiguration — `mule-extension` is MuleSoft's own
documented packaging type for custom Mule SDK connectors. It does not affect this
application: `vizarce-orchestration-flow` is a standard `mule-application` and opens
fine in Studio.

**(b) Custom connector icons not rendered on the design canvas — a separate,
MuleSoft-confirmed open platform issue.** Even once `vizarce-connector` is added as a
dependency and its operations are usable and functional in this application, Studio's
design canvas displays its operations with a generic/default icon rather than the
connector's own branding. This is a known, MuleSoft-acknowledged gap in how Studio's
canvas resolves custom Mule SDK connector icons — distinct from the `PackagingType`
gap above (that one blocks opening the connector project itself; this one only affects
icon rendering for a connector consumed by an application, which otherwise works
correctly). Functionally inert: the operations execute correctly regardless of which
icon Studio draws for them.

## Prerequisites

- Anypoint Studio 7.19+ / Mule Runtime 4.9.0
- The [`vizarce-connector`](https://github.com/vizarce/vizarce-connector) project
  built and installed to your local `.m2` first (`mvn clean install` in that project)
- A VIZARCE API key
- A Salesforce username/password/security token (see
  [Salesforce Integration](#salesforce-integration))

## Running locally

1. Edit `src/main/resources/local.properties`, replacing the placeholders with a real
   VIZARCE API key and Salesforce username/password/security token.
2. `mvn clean package`
3. Run the application in Anypoint Studio, or:
   ```bash
   mvn -Dmule.env=local mule:run
   ```
4. `POST` to `http://localhost:8081/orchestrate`, `http://localhost:8081/demo/generate-lyrics`,
   or `http://localhost:8081/demo/generate-structure` — see [Endpoints](#endpoints)
   above for request/response shapes and curl examples for all three.

## Testing

```bash
mvn clean test
```

Runs the MUnit suite (`src/test/munit/ai-orchestration-flow-test-suite.xml`):
five tests covering the circuit breaker sub-flows' state transitions in isolation
(closed → open on 3rd failure, open → half-open after the 30s cooldown, success
clears state), plus one happy-path test of the main flow with both VIZARCE connector
operations mocked via `munit-tools:mock-when` — so the suite never makes a real
network call.

Not yet covered (documented as a next step, not silently skipped): the circuit-open
fail-fast branch of the main flow, a failure-path test asserting
`record-circuit-failure` actually gets invoked when `vizarce:compose-song` errors, the
Salesforce write path (not mocked in the current happy-path test), and the two demo
flows.

## License

MIT — see [LICENSE](LICENSE).
