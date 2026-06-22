# Contribution #1: feat: Respect RateLimit headers in default REST backoff implementation

**Contribution Number:** 1
**Student:** Nazib Irfan Khan  
**Issue:** https://github.com/meltano/sdk/issues/2012  
**Status:** Phase III Complete

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
  - `backoff_wait_generator()` — currently `backoff.expo(factor=2)`; header-blind. **This is the method I changed** (the `backoff` library sends the raised exception into this generator on each retry, which makes header-aware waits possible here).
  - `backoff_runtime()` — a related generator-based alternative; considered but not needed for this approach.
  - `request_decorator()` — wires up `backoff.on_exception`, catching `RetriableAPIError`, timeouts, connection errors.
  - `validate_response()` — raises `RetriableAPIError` (with the `response` attached) on 429/5xx.
- **`singer_sdk/exceptions.py`** — `RetriableAPIError` carries the failed `response`, which is how the backoff layer can read headers off the throttled response.
- **Tests** — `tests/core/rest/` (or equivalent) for the REST stream backoff behavior.
- **Docs** — the SDK docs page describing custom backoff, if the default behavior changes.

---

## Reproduction Process

### Environment Setup

I cloned my fork to `c:\dev\sdk` and added the original repo as an `upstream` remote so I can pull in the latest `meltano/sdk` changes:

```
git clone https://github.com/Nazib65/sdk.git
git remote add upstream https://github.com/meltano/sdk.git
```

The SDK's `docs/CONTRIBUTING.md` specifies the toolchain: **uv** (dependency/venv manager), **pre-commit**, and **nox** (Python 3.10–3.14). Setup steps:

```
uv tool install pre-commit
uv tool install nox
uv sync --all-groups --all-extras   # builds the venv + installs all deps
pre-commit install                  # enable lint/format hooks on commit
```

`uv sync` provisioned a managed **CPython 3.14.4** virtualenv and installed 135 packages (the editable `singer-sdk`, `pytest`, and `python-backoff` 2.3.1 — the library at the core of this issue).

**Challenge:** system Python wasn't on my PATH (Windows pointed `python` at the Microsoft Store alias). This turned out not to be a blocker — `uv` downloads and manages its own Python, so `uv sync` provisioned 3.14.4 automatically without a separate Python install.

**Verification:** the existing REST backoff tests pass on a clean checkout —

```
uv run pytest tests/core/rest/test_failure.py -q
# 13 passed in 0.19s
```

### Steps to Reproduce

This is a missing-feature gap, so I demonstrated it by inspecting the existing code path (plus a real downstream failure) rather than by triggering a crash inside the SDK:

1. On upstream `main`, `RESTStream.backoff_wait_generator()` is literally `return backoff.expo(factor=2)` — it never inspects the response, so `X-RateLimit-Reset` / `Retry-After` headers are ignored.
2. `validate_response()` turns a 429 into a `RetriableAPIError` **with the response attached**, but the retry wait is computed purely from the exponential generator — the header value is never read. So a tap waits an arbitrary exponential delay instead of the exact reset window the server advertised.
3. Real-world evidence of the impact: downstream PR [Matatika/tap-outbrain#61](https://github.com/Matatika/tap-outbrain/pull/61) shows a tap had to hand-write header-aware backoff because the SDK has no default — and that workaround crashed with `AttributeError: 'NoneType' object has no attribute 'status_code'` when a connection-level error (e.g. `RemoteDisconnected`) arrived with **no `.response`**. That proves both (a) the SDK doesn't respect these headers by default, and (b) any default implementation must guard against retriable errors that carry no response.

### Reproduction Evidence

- **Working branch (my fork):** https://github.com/Nazib65/sdk/tree/feat/respect-ratelimit-headers
- **Baseline behavior:** before my change, `backoff_wait_generator()` returned `backoff.expo(factor=2)` (upstream `singer_sdk/streams/rest.py`) — header-blind.
- **Downstream crash showing the need + the edge case:** [Matatika/tap-outbrain#61](https://github.com/Matatika/tap-outbrain/pull/61).
- **Maintainer confirmation of the design:** @edgarrmondragon — *"respect RateLimit headers when they're available, and fall back to exponential backoff when they're not."*
- **My findings:** the gap is structural — the wait generator had no access to the response/exception, so it physically couldn't read a header. The `backoff` library, however, already *sends* the raised exception into the wait generator on each retry; the fix taps into that to read `.response` headers (with a `None`-response guard for connection errors/timeouts), falling back to exponential otherwise.

---

## Solution Approach

Per the maintainer (@ReubenFrankel, Oct 2025), the agreed direction is to **implement `X-RateLimit`-aware backoff within the existing backoff mechanism** — *not* to undertake the larger refactor to `requests`/urllib3 native retries, which is being deferred because of open questions about `RetriableAPIError` compatibility.

### Analysis

This is missing functionality, not a bug. The original `backoff_wait_generator()` simply returned `backoff.expo(factor=2)` and **ignored the exception that the `backoff` library sends into the generator before each retry**. Because the wait was computed without ever looking at the response, the server's `X-RateLimit-Reset` / `Retry-After` headers had no effect. The key realization: `backoff` already feeds the raised exception into the wait generator via `.send()`, and `RetriableAPIError` already carries the failed `response` — so a header-aware default is achievable by making the generator *read* that exception, as long as it guards against retriable errors that have no response (connection resets, timeouts).

### Proposed Solution

Make the default backoff header-aware while keeping exponential backoff as the fallback. The implementation adds/changes three methods in `singer_sdk/streams/rest.py`:

1. **`backoff_wait_generator()`** — rewritten as a stateful generator. It primes an internal `backoff.expo(factor=2)` fallback, then on each retry receives the thrown exception, reads `.response` (guarding for `None`), and asks `get_wait_time_from_response()` for a header-based wait — falling back to the exponential value when there's no usable header or no response.
2. **`get_wait_time_from_response(response)`** (new) — reads standard headers in precedence order: `Retry-After` (RFC 9110 — seconds *or* an HTTP date), then `X-RateLimit-Reset` (IETF RateLimit draft — interpreted as seconds until reset). Returns `None` when neither is usable. This is the clean override point for APIs with non-standard rate-limit headers.
3. **`_parse_retry_after()`** (new, static) — parses a numeric seconds value or an HTTP date, treats a timezone-naive date as UTC, and clamps the result to `>= 0`.

This stays **backwards-compatible**: taps that override `backoff_wait_generator()` keep working unchanged, and `get_wait_time_from_response()` gives taps with bespoke headers a smaller, targeted override point. (Note: `X-RateLimit-Reset` is interpreted as delta-seconds per the IETF draft; epoch-timestamp variants would be handled by overriding `get_wait_time_from_response()`.) Design confirmed by @edgarrmondragon: *"respect RateLimit headers when available, fall back to exponential when not."*

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The default REST backoff ignored `X-RateLimit-*` / `Retry-After` headers and always used exponential backoff. It should respect those headers when present and fall back to exponential otherwise — within the existing `backoff` mechanism, and without breaking taps that customize backoff.

**Match:** The codebase already had the patterns needed:
- The `backoff` library `.send()`s the raised exception into the wait generator on each retry — the hook I exploited.
- `RetriableAPIError` already stores the `response`, so headers are reachable.
- `validate_response()` already classifies 429/5xx as retriable (and connection errors/timeouts are retried with no response — the case to guard).
- `email.utils.parsedate_to_datetime` (stdlib) parses HTTP-date `Retry-After` values.

**Plan:**
1. Rewrite `backoff_wait_generator()` to read the per-retry exception and delegate to a header helper, with a primed exponential fallback.
2. Add `get_wait_time_from_response()` to read `Retry-After` then `X-RateLimit-Reset`.
3. Add `_parse_retry_after()` to handle numeric vs. HTTP-date values, UTC-naive dates, and clamping.
4. Add unit tests for header-present, precedence, clamping, unparsable, HTTP-date, and tz-naive cases, plus a generator-level fallback test.

**Implement:** Branch — https://github.com/Nazib65/sdk/tree/feat/respect-ratelimit-headers
- `2c996ee` feat(taps): Respect rate-limit headers in the default REST backoff
- `dbd417d` test(taps): Cover clamped and unparsable rate-limit header waits
- `f17ce32` test(taps): Cover Retry-After dates without timezone info

**Review:** Self-review against the SDK contributing guide — `pre-commit` hooks installed and run, `nox` / `pytest` green, full type hints, docstrings with `.. versionadded::`, and backwards compatibility preserved.

**Evaluate:** Unit tests in `tests/core/rest/test_failure.py` assert the wait equals the header value (e.g. `Retry-After: 30` → `30`) and that an unparsable header falls back to exponential (→ `2`). Full suite stays green.

---

## Testing Strategy

All tests added in `tests/core/rest/test_failure.py`. Full file: **26 passed in 0.57s** (13 pre-existing + 13 new).

### Unit Tests — `get_wait_time_from_response` (parametrized, all passing)

- [x] `Retry-After: 30` → `30.0` (`retry-after-seconds`)
- [x] `X-RateLimit-Reset: 45` → `45.0` (`ratelimit-reset-seconds`)
- [x] `Retry-After` takes precedence over `X-RateLimit-Reset` → `30.0` (`retry-after-precedence`)
- [x] Zero/negative values clamped to `0.0` (`retry-after-zero`, `retry-after-negative-clamped`, `ratelimit-reset-zero`, `ratelimit-reset-negative-clamped`)
- [x] Unparsable header → `None` (fall back to exponential) (`retry-after-unparsable`, `ratelimit-reset-unparsable`)
- [x] No rate-limit headers → `None` (`no-headers`)
- [x] `Retry-After` as an HTTP date → seconds-from-now (`test_get_wait_time_from_response_http_date`)
- [x] `Retry-After` HTTP date **without** timezone → interpreted as UTC (`test_get_wait_time_from_response_naive_http_date`)

### Generator-level Test — `test_backoff_wait_generator_respects_headers_and_falls_back` (passing)

Drives the rewritten `backoff_wait_generator()` across a realistic exception sequence:

- [x] 429 with `Retry-After: 30` → waits `30` (header honored)
- [x] 429 with unparsable header → `2` (exponential fallback)
- [x] `requests.exceptions.ConnectionError` (no `.response`) → `4` (**no crash** — the tap-outbrain#61 edge case)
- [x] 429 with no headers → `8` (exponential sequence continues: 2 → 4 → 8)

### Manual / Verification

- Ran the existing suite on a clean checkout before changes: `uv run pytest tests/core/rest/test_failure.py -q` → 13 passed.
- After implementing: `uv run pytest tests/core/rest/test_failure.py -v` → **26 passed in 0.57s**.
- `X-RateLimit-Reset` is interpreted as **delta-seconds** (per the IETF draft). Epoch-timestamp variants are intentionally left to a per-tap override of `get_wait_time_from_response()` rather than guessed at automatically.

---

## Implementation Notes

### Progress

Implemented the default header-aware backoff and its test suite. The work landed in three commits: the feature first, then two follow-up commits hardening the test coverage (clamped/unparsable values, then timezone-naive HTTP dates). All 26 tests in the REST failure suite pass; pre-commit hooks (Ruff, formatting, type checks) run clean on commit.

### Code Changes

- **Files modified:**
  - `singer_sdk/streams/rest.py` (+97 / −5) — rewrote `backoff_wait_generator()`; added `get_wait_time_from_response()` and the static `_parse_retry_after()` helper; imported `datetime`/`timezone` and `email.utils.parsedate_to_datetime`.
  - `tests/core/rest/test_failure.py` (+103) — added the parametrized `get_wait_time_from_response` tests, two HTTP-date tests, and the generator-level fallback test.
- **Key commits** (branch [`feat/respect-ratelimit-headers`](https://github.com/Nazib65/sdk/tree/feat/respect-ratelimit-headers)):
  - [`2c996ee`](https://github.com/Nazib65/sdk/commit/2c996ee8b3dec55d80bba1ee0caa93254eee12e0) — feat(taps): Respect rate-limit headers in the default REST backoff
  - [`dbd417d`](https://github.com/Nazib65/sdk/commit/dbd417d8a9e0701dd680972ce0f006b1b7870758) — test(taps): Cover clamped and unparsable rate-limit header waits
  - [`f17ce32`](https://github.com/Nazib65/sdk/commit/f17ce3283b438dd4415ff491f9a675fa0ef8313c) — test(taps): Cover Retry-After dates without timezone info
- **Approach decisions:**
  - **Used `backoff_wait_generator()`, not `backoff_runtime()`.** The `backoff` library already `.send()`s the raised exception into the wait generator on each retry, so I could read the response there directly — avoiding the `backoff_runtime` raise/suppress handshake and falling back to exponential cleanly.
  - **Guarded against a missing `.response`.** Connection-level retriable errors (e.g. `RemoteDisconnected`, timeouts) carry no response. Reading `error.response.status_code` blindly is exactly what crashed [tap-outbrain#61](https://github.com/Matatika/tap-outbrain/pull/61); my generator checks for `None` and falls back to exponential.
  - **`Retry-After` precedence over `X-RateLimit-Reset`.** `Retry-After` is the more specific, standardized retry directive (RFC 9110).
  - **Small, targeted override point.** `get_wait_time_from_response()` is its own method so taps with non-standard headers (or epoch-based reset values) can override just that, not the whole generator — preserving backwards compatibility.
  - **Stdlib for date parsing.** Used `email.utils.parsedate_to_datetime` for HTTP-date `Retry-After`, treating timezone-naive dates as UTC and clamping negative waits to `0`.

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
