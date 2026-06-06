# Contribution #1: feat: Respect RateLimit headers in default REST backoff implementation

**Contribution Number:** 1
**Student:** Nazib Irfan Khan  
**Issue:** https://github.com/meltano/sdk/issues/2012  
**Status:** Phase I Complete

---

## Why I Chose This Issue

I chose this issue because the Meltano Singer SDK is a real, widely-used Python framework that hundreds of data "taps" and "targets" are built on, so improving its default behavior helps every connector that talks to a rate-limited REST API — not just one app. The problem is also very well-scoped and easy to explain: when an API replies with the standard `X-RateLimit-*` headers, the SDK currently ignores them and just backs off exponentially, instead of waiting the exact amount of time the server tells it to. Knowing what "done" looks like makes it a good fit for a focused 3–4 week contribution.

It matches my skills and learning goals well. The fix lives entirely in the Python backend (`singer_sdk/streams/rest.py`) around HTTP retry/backoff logic — no frontend — which is exactly the kind of focused backend work I want more experience in. I'll get to learn how a mature SDK structures retry handling with the `backoff` library, how HTTP rate-limit headers (the IETF `RateLimit` draft) work in practice, and how to keep a change backwards-compatible in a framework other people extend. The issue is labelled `good first issue` and `Accepting Pull Requests`, and the maintainer (@ReubenFrankel) has already replied to confirm the direction, so I'm starting with a clear green light. I am very much excited to work here as this experience is pretty new to me and hope to enjoy throughout the process.

---

## Understanding the Issue

### Problem Description

The [IETF RateLimit headers draft](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/) defines a set of response headers — `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset` — that an API uses to tell a client how many requests it has left and when the limit resets. Taps built with the Singer SDK around REST APIs that send these headers would benefit from a default `RESTStream` backoff that understands and respects them — namely `X-RateLimit-Reset`, so the tap waits exactly until the window resets instead of guessing.

### Expected Behavior

When a request is throttled (e.g. HTTP 429) and the response carries `X-RateLimit-Reset` (or the related `Retry-After` header), the SDK's **default** backoff should wait the duration indicated by that header before retrying, out of the box, without each tap author having to write custom backoff code. When no such header is present, it should fall back to the current exponential backoff.

### Current Behavior

The default backoff ignores response headers entirely. `RESTStream.backoff_wait_generator()` returns `backoff.expo(factor=2)` — a fixed exponential schedule with jitter — regardless of what the server says. `validate_response()` raises `RetriableAPIError` for 429 and 5xx responses (and `FatalAPIError` for other 4xx), but the retry wait is computed purely from the exponential generator, so a tap may wait too long or retry too early relative to the server's actual reset window.

### Affected Components

Based on reading the codebase (to be confirmed during Phase II):

- **`singer_sdk/streams/rest.py`** — the `RESTStream` class and its retry/backoff machinery:
  - `backoff_wait_generator()` — currently `backoff.expo(factor=2)`; header-blind.
  - `backoff_runtime()` — a generator-based alternative that *can* derive wait time from the raised exception (and thus the response). This is the natural hook for header-aware waits.
  - `request_decorator()` — wires up `backoff.on_exception`, catching `RetriableAPIError`, timeouts, connection errors.
  - `validate_response()` — raises `RetriableAPIError` (with the `response` attached) on 429/5xx.
- **`singer_sdk/exceptions.py`** — `RetriableAPIError` carries the failed `response`, which is how the backoff layer can read headers off the throttled response.
- **Tests** — `tests/core/rest/` (or equivalent) for the REST stream backoff behavior.
- **Docs** — the SDK docs page describing custom backoff, if the default behavior changes.

---

## Reproduction Process

### Environment Setup

[Phase II — notes on cloning my fork and setting up the Meltano SDK dev environment (uv / hatch, pre-commit, pytest), including any challenges and how I solved them]

### Steps to Reproduce

1. [Phase II — build/use a minimal RESTStream against a mock API that returns 429 with an `X-RateLimit-Reset` header]
2. [Phase II — observe the SDK's wait time before retrying]
3. [Phase II — confirm the wait is driven by exponential backoff, NOT by the reset header]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [Retry log lines showing the wait time vs. the header value]
- **My findings:** [What I discovered during reproduction]

---

## Solution Approach

Per the maintainer (@ReubenFrankel, Oct 2025), the agreed direction is to **implement `X-RateLimit`-aware backoff within the existing backoff mechanism** — *not* to undertake the larger refactor to `requests`/urllib3 native retries, which is being deferred because of open questions about `RetriableAPIError` compatibility.

### Analysis

This is missing functionality, not a bug. The reason headers are ignored is structural: `backoff_wait_generator()` produces wait times but has **no access to the response/exception**, so it physically can't read a header. The SDK already exposes `backoff_runtime()`, which *is* given the raised exception (and `RetriableAPIError` already carries the `response`). So the pieces exist — the default path just doesn't use them to consult `X-RateLimit-Reset`.

### Proposed Solution

Make the default backoff header-aware: when a retriable response carries `X-RateLimit-Reset` (and/or `Retry-After`), derive the wait from that header; otherwise fall back to the existing exponential generator. Implementation-wise this means routing the default retry through a runtime/value function (via `backoff_runtime()` or an equivalent hook) that:

1. Reads the `response` off the `RetriableAPIError`.
2. Parses `X-RateLimit-Reset` (handling the two real-world encodings — IETF delta-seconds vs. a Unix epoch timestamp that some APIs like GitHub use), and `Retry-After` as a complementary case.
3. Returns the computed wait, clamped sensibly (non-negative, capped to avoid pathological values).
4. Falls back to `backoff_wait_generator()` when no usable header is present.

This must stay **backwards-compatible**: taps that already override `backoff_wait_generator()` should keep working unchanged. (Open design question to confirm with the maintainer: whether header-aware backoff is on by default or behind an opt-in, and exactly which headers/encodings to support — I'll confirm in the issue/PR thread before finalizing.)

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The default REST backoff ignores `X-RateLimit-*` headers and always uses exponential backoff. We want it to respect `X-RateLimit-Reset` (and `Retry-After`) when present, falling back to exponential otherwise, within the existing `backoff` mechanism.

**Match:** The codebase already has the patterns needed:
- `backoff_runtime()` exists precisely to compute waits from the raised exception.
- `RetriableAPIError` already stores the `response`, so headers are reachable.
- `validate_response()` already classifies 429/5xx as retriable.
- The `backoff` library supports runtime/value-based waits alongside `backoff.expo`.

**Plan:**
1. Add a helper that extracts a wait duration from a response's `X-RateLimit-Reset` / `Retry-After` headers (handling delta-seconds vs. epoch, missing/invalid values).
2. Wire the default retry path to use that helper, falling back to `backoff_wait_generator()` when no header is usable.
3. Preserve overridability so existing custom `backoff_wait_generator()` implementations are unaffected.
4. Add unit tests covering header-present, header-absent, and malformed-header cases.
5. Update docs describing the new default behavior.

**Implement:** [Link to your branch/commits here as you work — Phase III]

**Review:** [Self-review against the SDK's contributing guide: pre-commit/lint passes, `pytest` green, type hints, changelog/docs updated, backwards compatibility preserved — Phase III]

**Evaluate:** Unit tests + a manual run of a RESTStream against a mock 429 + `X-RateLimit-Reset` response, confirming the retry waits for the header-specified duration (and still falls back to exponential when the header is absent).

---

## Testing Strategy

### Unit Tests (planned — to implement in Phase III)

- [ ] 429 response **with** `X-RateLimit-Reset` (delta-seconds) → backoff waits the header-specified duration.
- [ ] 429 response **with** `X-RateLimit-Reset` as a Unix epoch timestamp → wait computed correctly relative to "now".
- [ ] 429 response **with** `Retry-After` and no `X-RateLimit-Reset` → respects `Retry-After`.
- [ ] 429/5xx response with **no** rate-limit headers → falls back to exponential `backoff_wait_generator()` (existing behavior unchanged).
- [ ] Malformed/negative/empty header value → safe fallback, no crash.
- [ ] A tap that **overrides** `backoff_wait_generator()` still behaves as before (backwards compatibility).

### Integration Tests (planned)

- [ ] End-to-end RESTStream run against a mock API that throttles then recovers, asserting total wait aligns with the reset header.

### Manual Testing

[Phase III — results of running a minimal tap against a mock rate-limited endpoint, with retry log output]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- 2026-06-04 (approx.): @ReubenFrankel confirmed to start with `X-RateLimit`-aware backoff within the existing mechanism rather than the native-retries refactor; shared a Slack thread from the original tap-auth0 work as context.
- [Date]: [Further feedback received]
- [Date]: [How I addressed it]

**Status:** Awaiting PR (Phase II/III in progress)

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- Issue #2012 — https://github.com/meltano/sdk/issues/2012
- Meltano Singer SDK repo — https://github.com/meltano/sdk
- `RESTStream` backoff implementation — `singer_sdk/streams/rest.py` (`backoff_wait_generator`, `backoff_runtime`, `request_decorator`, `validate_response`)
- `RetriableAPIError` / `FatalAPIError` — `singer_sdk/exceptions.py`
- IETF RateLimit headers draft — https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/
- Python `backoff` library docs — https://github.com/litl/backoff
- [Add the tap-auth0 Slack thread link and SDK contributing docs as you use them in Phase II–III]
