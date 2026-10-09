# PR #1 Review: Daily to LiveKit/Telnyx Migration and Warm Transfers

**Pinned base:** `5a98a66274322954a0a6255f54650eec6e50c41d`  
**Pinned PR head reviewed:** `f93f7a5f79efdf08c18c583a6a6f7167084b93ae`

## Recommendation: request changes — do not merge

This is a substantial, generally well-scoped migration from Daily to LiveKit with
Telnyx-backed SIP, while retaining the existing single-prompt agent. The PR also
adds a Room1/Room2 warm-transfer lifecycle with rollback and cleanup handling.

The change should not merge yet. The warm-transfer path contains a verified
dependency API mismatch that aborts transfers before a broker is dialed. There
are also material ingress and outbound-call correctness issues that need
resolution and targeted tests.

## Review scope and approach

- Reviewed the public PR and source at the pinned head, concentrating on the
  changed webhook ingress, SIP boundary, bot runner, warm-transfer media,
  orchestration, configuration, and related tests.
- Compared production calls with the exact pinned Pipecat `0.0.95` source and
  checked LiveKit's outbound SIP documentation.
- Did not modify either PR branch, use credentials, or place calls.

## Required review comments

### P0 — Warm transfers fail before broker dialing

**Anchor:** `voice-agent/livekit_transfer_media.py:57-61`

`gate_primary_media()` calls `pause_processing_frames()` and
`resume_processing_frames()` on `transport.input()`. The pinned
`pipecat-ai==0.0.95` `BaseInputTransport` does not provide either method.
Consequently, a warm-transfer attempt raises `AttributeError` at line 59 (or
line 61 on restoration), before hold music starts or the broker is dialed.

The test suite masks the incompatibility: `tests/test_livekit_transfer_media.py:8-11`
adds both methods to a `SimpleNamespace` as `AsyncMock`s. That verifies the
assumption rather than the actual runtime API.

**Requested change:** use a Pipecat 0.0.95-supported input-gating approach and
add an integration-level test that creates or inspects the real installed
LiveKit/Pipecat input transport. Do not mock methods absent from the production
dependency.

**Why this matters:** every warm transfer calls this gate, so this is a release
blocker rather than a fallback-only failure.

### P1 — Webhook deduplication is race-prone and unbounded

**Anchor:** `voice-agent/server.py:112-129`

The webhook handler checks whether an event ID is in `seen_livekit_events`,
awaits `start_agent()`, and only then records the ID. Two concurrent deliveries
of the same event can both pass the initial check and start two bots. The set is
also process-local and retains every accepted ID indefinitely.

`tests/test_server_livekit.py:50-70` covers two sequential deliveries only; it
does not exercise the interleaving that causes duplicate startup.

**Requested change:** atomically reserve the ID before awaiting bot startup;
remove the reservation if startup fails; apply bounded TTL/max-size retention.
Use a shared idempotency store if the deployment uses multiple processes or
needs restart-safe delivery handling. Add concurrent duplicate-delivery,
startup-failure cleanup, expiry, and bounded-retention tests.

Illustrative control flow:

```python
if event_id in in_flight_or_seen:
    return duplicate()
in_flight_or_seen.add(event_id)
try:
    await start_agent(...)
except Exception:
    in_flight_or_seen.discard(event_id)
    raise
```

### P1 — Outbound caller identity and selected tenant can diverge

**Anchors:** `voice-agent/server.py:144-166`,
`voice-agent/livekit_telephony_service.py:100-110`

`/outbound-call` accepts `from_phone` and stores it as `AgentRequest.bot_phone`.
The bot uses that value to look up the organisation and build the call context
(`bot.py:230-233`). However, the LiveKit SIP request always uses the configured
`LIVEKIT_SIP_OUTBOUND_NUMBER` as its actual caller ID, not `from_phone`.

A holder of the outbound API key can therefore request the context for one
number while originating the PSTN call from another. This can associate a call
with the wrong organisation and present the wrong business context to the
carrier.

**Requested change:** reject a `from_phone` that differs from the configured
caller ID, or explicitly authorize requested caller IDs and pass the same
validated value to LiveKit. Add tests for mismatch rejection and authorized
matching caller IDs.

### P1 — Endpoint returns 202 but blocks until the carrier answers

**Anchors:** `voice-agent/server.py:148-176`,
`voice-agent/livekit_telephony_service.py:100-110`

The endpoint awaits `dial_phone()` before returning `202`. The SIP request
unconditionally uses `wait_until_answered=True`, which LiveKit documents as
waiting for pickup and raising on call failure. Therefore a no-answer or reject
can keep the HTTP request open through the ringing period and return an error
instead of promptly acknowledging call initiation.

Waiting for answer is appropriate for the broker-consultation path, but the
public asynchronous endpoint needs a separate policy.

**Requested change:** make waiting for answer an explicit `dial_phone()` option.
Keep it enabled for a warm-transfer broker, and either disable it for
`/outbound-call` or document and enforce a bounded synchronous contract. Add
delayed-answer, rejection, and no-answer tests.

## Test and validation assessment

- The PR's GitHub Actions `functional-tests` job is recorded as successful for
  this head. Its log is not publicly available in the review environment, so
  the success status can be validated but the PR's claimed numerical test totals
  cannot be independently verified.
- The repository README describes mocked functional tests and correctly notes
  that they do not validate real LiveKit/Telnyx provisioning, calls, recordings,
  or provider behaviour.
- Local test execution could not be performed because the supplied workspace is
  empty and is not a Git checkout. No attempt was made to use credentials or
  external telephony resources.
- I checked for repository-wide agent guidance and Python project/lint
  configuration at the root and `voice-agent/`; no `AGENTS.md` or `pyproject.toml`
  was present at those locations. No additional established coding-rule breach
  was found. The API-surface mock in the P0 test is still a material test-quality
  issue.

### Requested reproducible validation evidence

Please append the following to the PR description, a follow-up commit message,
or the CI summary after running them against this exact head. Include tool
versions, the complete commands, and pass/fail totals rather than only a
statement that tests passed:

```bash
cd voice-agent
python -m pytest ../shared/tests -v --tb=short
python -m pytest tests -v --tb=short
python -m ruff check .
python -m black --check .
```

`ruff` and `black` are not repository-mandated tools in the inspected files, so
their output is supplementary consistency evidence rather than a newly imposed
style gate. If either tool is not part of the development image, record the
version used to install it before running the command. The review environment
does not contain the checkout, so these commands could not be executed here.

## LiveKit/Telnyx integration-practice assessment

### Practices the implementation follows

- **Webhook authenticity before routing:** `livekit_ingress.py:65-85` requires
  the authorization header and sends the exact raw UTF-8 request body to the
  LiveKit verifier before parsing. This follows LiveKit's signed-webhook model,
  including the body-hash validation performed by `WebhookReceiver`.
- **Inbound-SIP contract validation:** `livekit_ingress.py:108-145` restricts
  accepted events to SIP `participant_joined` events and requires room,
  participant, call, trunk, dispatch-rule, and dialed-number identifiers. That
  is a good defensive boundary and makes call/provider correlation practical.
- **Warm-transfer lifecycle controls:** the orchestrator serializes transfer and
  handoff attempts, waits for an answered broker and Room2 readiness, verifies
  arrival in Room1, and has bounded timeout/cleanup paths. This aligns with the
  agent-assisted warm-transfer model in LiveKit's telephony guidance.
- **Provider configuration:** the application keeps LiveKit API credentials and
  SIP identifiers in typed configuration, while the documentation keeps Telnyx
  credentials in provider setup rather than application environment examples.

### Gaps requiring release qualification or follow-up

- **Provider integration has not been qualified:** LiveKit recommends a real
  development call covering inbound/outbound trunk routing, dispatch-rule
  matching, SIP participant attributes, provider-side logs, and failure paths.
  Offline tests cannot establish those properties. This is an unresolved release
  qualification requirement, not a claim that the code is already broken.
- **Failure-path acceptance evidence is missing:** LiveKit specifically calls
  for pre-answer reject/no-answer and mid-call-disconnect checks. Add the output
  of those development-environment tests once authorized; test the transfer
  failure/restore path as well as successful inbound and outbound calls.
- **Operational observability:** add telemetry around the interval from primary
  media gating to the hold publisher's first audio frame. The current ordering
  gates caller media before `start_hold()` completes, so a provider or join
  delay can create an audible silence. This is a non-blocking reliability and
  UX recommendation, not a verified defect.
- **Participant-ID semantics after move:** confirm with the pinned LiveKit
  version whether a SIP participant SID is preserved after `MoveParticipant`.
  The code retains the Room2 participant ID to process later Room1 disconnects.
  If IDs can change across rooms, a broker departure can be missed. Add a test
  with different source/destination IDs unless the provider contract guarantees
  preservation.

## Non-blocking operational risk / unresolved question

`voice-agent/bot.py:632-646` permits only one active bot task per BotRunner
process. A second call receives HTTP 409, which ingress converts to 503. The PR
description identifies the runner as intentionally single-call, so this is not
counted as a defect within the stated scope. Before production rollout, confirm
that deployment supplies one isolated runner per concurrent call, or replace
the global task with room/call-keyed task management.

## Evidence links

- [PR overview and stated validation](https://github.com/e3-solutions/Frontline-Assignment/pull/1)
- [Pinned warm-transfer media code](https://github.com/e3-solutions/Frontline-Assignment/blob/f93f7a5f79efdf08c18c583a6a6f7167084b93ae/voice-agent/livekit_transfer_media.py#L57)
- [Pinned transfer-media test](https://github.com/e3-solutions/Frontline-Assignment/blob/f93f7a5f79efdf08c18c583a6a6f7167084b93ae/voice-agent/tests/test_livekit_transfer_media.py#L8)
- [Pipecat 0.0.95 BaseInputTransport source](https://raw.githubusercontent.com/pipecat-ai/pipecat/v0.0.95/src/pipecat/transports/base_input.py)
- [Pinned ingress/outbound handler](https://github.com/e3-solutions/Frontline-Assignment/blob/f93f7a5f79efdf08c18c583a6a6f7167084b93ae/voice-agent/server.py#L98)
- [LiveKit outbound-call behaviour](https://docs.livekit.io/telephony/making-calls/outbound-calls/)
- [LiveKit webhook verification](https://docs.livekit.io/intro/basics/rooms-participants-tracks/webhooks-events/)
- [LiveKit telephony test guidance](https://docs.livekit.io/telephony/testing/)
- [LiveKit warm-transfer overview](https://docs.livekit.io/telephony/features/transfers/)
