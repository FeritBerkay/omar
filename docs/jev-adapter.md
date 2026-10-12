# Jev topology backend

`team Review[reviewer : Jev]` evaluates a Choice through the TypeSafe API rather
than launching a coding-agent pane. Jev is registered by name/case like other
backends, but is topology-only and is not an executive-assistant backend.

Each prompt writes exactly one string-enum port, and optionally one `string`
port for JSON evidence. The enum can be imported with `type Decision from ...`.
The prompt describes the meaning of the options. The adapter currently uses
option labels as Choice criteria; explicit per-option criteria are future work.

```text
type Decision from "./schemas/decision.json"
team Review[reviewer : Jev]
{
    input request : string
    output out : Decision
    output evidence : string
    prompt reviewer(request) -> out, evidence
    "Choose continue if the request can proceed; stop if it cannot; defer if unsure."
}
main JevReview { review = Review() }
```

Schema: `{ "type": "string", "enum": ["continue", "stop", "defer"] }`.
Set `TYPESAFE_API_KEY` in the launching shell. The model defaults to `jev-latest`;
set `OMAR_JEV_MODEL` to choose another model. Adapter settings survive both
foreground/background session creation. They are captured when the session
starts; changing a client shell does not reconfigure an existing session.
`curl` must be installed; its user curlrc is disabled for this API call.

The decision is a plain string checked against its port enum. Evidence is JSON
serialized into a string: `answers.decision` has `choice`, `probabilities`, and
`confidence`; `context` contains original trigger values. A Rust gate may parse
it with serde_json and apply the workflow's selected-probability threshold.
The adapter does not impose a threshold. Evidence is not an imported object type.

Backend execution returns writes to the VM, which checks the effect contract and
port types before publishing. Native Choice membership/distribution validation
remains in the adapter. Registered topology-only backends get no tmux session.

## Failure policy

`OMAR_JEV_ON_ERROR=fail` is the default. Credentials/configuration errors,
transport errors, non-success HTTP status, timeout, invalid JSON/Choice output,
and oversized responses fail the invocation without publishing a decision.

Set `OMAR_JEV_ON_ERROR=defer` to opt into fallback. The invocation must have a
`defer` enum value AND an evidence string output, checked before the call. On
failure it writes defer with evidence like:

```json
{
  "model": "jev-latest",
  "error": {
    "kind": "rate_limit",
    "message": "Jev rate limit exceeded",
    "http_status": 429
  },
  "context": {"review.request": "original input"}
}
```

Error kinds: configuration, authentication (401/403), rate_limit (429), service
(5xx), http (other non-success status), transport, timeout, invalid_response.
Messages never copy credentials, provider error bodies, or curl stderr. Neither
policy retries automatically. A model-selected defer is a normal Choice result;
a service failure has error evidence and no answers object.

Each HTTP call has a maximum 20-second budget, shortened to the invocation's
declared time limit. A zero budget makes no call. Parent-enforced deadlines terminate
the HTTP process group, including pipe-blocked helpers; curl also gets its own
timeout. Response data is capped at 1 MiB. There is no API-key argument or file:
headers and JSON body are supplied over stdin. Evidence is still a JSON string,
not an imported object schema.

The local experiment launchers opt into defer explicitly. Classification thresholds
remain entirely in the workflow's Rust reaction, independent of transport errors.

## Example

`examples/jev-review.omar` imports `schemas/jev-decision.json` and passes the enum
and evidence to a Rust gate. Its >0.95 threshold is illustrative, not calibrated.
`decider.needs_review` carries the original context and evidence; connect it to a
fallback team when appropriate for the workflow.

```sh
export TYPESAFE_API_KEY
export OMAR_JEV_ON_ERROR=defer  # optional; default is fail
omar run examples/jev-review.omar --input "review.request=Example request" --wait
```

Generated Rust reaction crates enable serde_json when a body references it. The
normal OMAR build caches its sources; reaction builds still use cargo --offline.
