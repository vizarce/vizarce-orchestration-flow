# VIZARCE AI Orchestration Flow

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Mule](https://img.shields.io/badge/Mule-4.9.x-blue.svg)](https://www.mulesoft.com/)

A Mule 4 integration flow demonstrating the [`vizarce-connector`](https://github.com/vizarce/vizarce-connector)
in a realistic enterprise orchestration scenario, combining four integration patterns:

- **Scatter-Gather** — parallel, independent calls to VIZARCE (`composeSong` and, when
  an artist reference is supplied, `buildArtistDNA`), merged into one result.
- **Until Successful** — retries the compose call on transient failure with a fixed
  backoff.
- **Circuit Breaker** — an Object-Store-backed breaker around the whole VIZARCE call:
  after 3 consecutive failures it trips open and fails fast (no further VIZARCE calls)
  for a 30-second cooldown, then allows a single half-open trial request.
- **DataWeave transform to an enterprise schema** — the merged VIZARCE result is
  reshaped into a Salesforce Custom Object-style payload (`VIZARCE_Song_Request__c`),
  ready to hand to a real `salesforce:create` operation.

## Endpoint

```
POST /orchestrate
Content-Type: application/json

{
  "concept": "a song about gravity and distance",
  "genreTag": "future-house",
  "bpm": "112",
  "musicalKey": "A minor",
  "artistName": "some artist"   // optional — omit to skip the Artist DNA route
}
```

Responses:

| Status | Body                                                              | When                                                    |
|--------|--------------------------------------------------------------------|----------------------------------------------------------|
| 200    | `{ status: "SUCCESS", VIZARCE_Song_Request__c: { ... } }`          | Compose (and Artist DNA, if requested) both succeeded    |
| 502    | `{ status: "FAILED", errorType: "VIZARCE:...", message: "..." }`  | VIZARCE call failed (after retries, where applicable)    |
| 503    | `{ status: "CIRCUIT_OPEN", message: "..." }`                      | Circuit breaker is open — VIZARCE was not called at all  |

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
              │    └─ route 1: Choice → vizarce:build-artist-d-n-a (if artistName given) | null
              ├─ record-circuit-success (sub-flow, clears Object Store keys)
              ├─ DataWeave transform → VIZARCE_Song_Request__c shape
              └─ error-handler (on-error-continue) → record-circuit-failure, 502 response
```

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

## Prerequisites

- Anypoint Studio 7.19+ / Mule Runtime 4.9.0
- The [`vizarce-connector`](https://github.com/vizarce/vizarce-connector) project
  built and installed to your local `.m2` first (`mvn clean install` in that project)
- A VIZARCE API key

## Running locally

1. Edit `src/main/resources/local.properties`, replacing the placeholder with a real
   VIZARCE API key.
2. `mvn clean package`
3. Run the application in Anypoint Studio, or:
   ```bash
   mvn -Dmule.env=local mule:run
   ```
4. `POST` to `http://localhost:8081/orchestrate` with the JSON body shown above.

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
fail-fast branch of the main flow, and a failure-path test asserting
`record-circuit-failure` actually gets invoked when `vizarce:compose-song` errors.

## Known Anypoint Studio IDE limitation (this environment)

On at least one tested Anypoint Studio installation (build 7.24–7.25, mule-packager
4.7.0–4.9.1), the bundled `org.mule.tools.api.packager.packaging.PackagingType` enum
does not include a `MULE_EXTENSION` constant. This means Studio's built-in Mule
Project importer — which only recognizes `mule-application`, `mule-domain`,
`mule-domain-bundle`, and `mule-policy` packaging — cannot open a
`mule-extension`-packaged project (like `vizarce-connector`) as a first-class Studio
project with a design canvas. This is a real, confirmed gap in that Studio build, not
a misconfiguration — `mule-extension` is MuleSoft's own documented packaging type for
custom Mule SDK connectors.

This does not affect building or consuming the connector: `mvn clean install` on the
connector succeeds cleanly via the command line regardless, and this
orchestration-flow application (a standard `mule-application`) is fully supported by
Studio's importer. The open question still being investigated is why this
application's own design canvas doesn't yet render the VIZARCE connector's operations
with their proper icon/branding — likely related to a separate Studio-internal
dependency-resolution cache, distinct from the `PackagingType` gap above.

## License

MIT — see [LICENSE](LICENSE).
