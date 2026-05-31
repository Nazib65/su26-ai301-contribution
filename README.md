# Contribution [#]: [FR] Required serials

**Contribution Number:** 1
**Student:** Nazib Irfan Khan  
**Issue:** https://github.com/inventree/InvenTree/issues/11258  
**Status:** Phase I Complete

---

## Why I Chose This Issue

I chose this issue because InvenTree is real inventory-management software used by actual teams to track physical stock, so a fix here has tangible impact rather than being a toy exercise. The problem is also concrete and easy to explain: trackable parts currently *allow* serial numbers but can't *require* them, which lets stock be received with missing serials. Knowing what "done" looks like up front makes it a good fit for a focused 3–4 week contribution.

It matches my skills and learning goals well. The core of the fix lives in the Python/Django backend — adding a field to the Part model, a database migration, serializer changes, and validation logic when receiving stock — which is exactly the area I want more experience in. It also reaches into the React frontend for a toggle in the part settings, so I get full-stack exposure in one contribution. I hope to learn how a mature Django project structures models, migrations, and validation, and how backend rules surface in the UI. I am very much excited to work here as this experience is pretty new to me and hope to enjoy throughout the process.

---

## Understanding the Issue

### Problem Description

Parts may be trackable, which allows but does not require serial numbers to be assigned to stock items. In my use of inventree, almost all trackable parts (whether serials are assigned by "us" or by an external manufacturer) should always have a serial number assigned - but currently this cannot be enforced which can lead to missing serials and mistakes when receiving stock.

### Expected Behavior

A maintainer should be able to mark a part so that serial numbers are **mandatory**. When stock for such a part is created or received without a serial number, InvenTree should reject the operation with a clear validation error — the same way it already rejects duplicate serials or non-integer quantities for trackable parts.

### Current Behavior

A trackable part accepts stock items with an empty/null serial. The validation in `stock/models.py` (`clean()` / `validate_unique()`) only enforces rules *when a serial is present* (quantity must be 1, serial must be unique). Nothing forces a serial to be present in the first place, so stock can be received un-serialized with no warning.

### Affected Components

Based on reading the codebase (to be confirmed during Phase II reproduction):

- **`src/backend/InvenTree/part/models.py`** — the `Part` model, where `trackable` is defined alongside `component`, `assembly`, `salable`, `virtual`. The new `require_serial` field would live here. Also home to `validate_serial_number()`, `get_latest_serial_number()`, `get_next_serial_number()`.
- **`src/backend/InvenTree/part/migrations/`** — a new auto-generated migration for the added field.
- **`src/backend/InvenTree/part/serializers.py`** — expose the new field in the Part API.
- **`src/backend/InvenTree/stock/models.py`** — `StockItem.clean()` / `validate_unique()` / `serializeStock()`, where serial-related validation already lives; the "serial required" check would be added here so it fires on create and on receiving stock.
- **`src/backend/InvenTree/stock/serializers.py`** — the stock receive/create serializers, where a missing serial should surface as a field error.
- **Frontend (React, `src/frontend/src/`)** — the Part edit form needs a toggle for the new field; exact file paths to confirm in Phase II.
- **Tests** — `part/test_*.py` and `stock/test_*.py`.

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach
Make trackable functionally ternary: in addition to "no" and "optional" (the current behavior of trackable = true), add a "required" state. This would enforce serials when stock items are added or received.

This could be implemented as a second boolean "require serial" which is only respected when trackable=true.

### Analysis

This isn't a bug — it's missing functionality. The root cause is that "trackable" encodes only two states (off / on-but-optional). InvenTree's serial validation is written defensively: it constrains a serial *if one exists*, but never asserts that one *must* exist. So there is simply no field to express "required" and no check that reads such a field. The fix is additive: introduce the missing state and the missing assertion.

### Proposed Solution

Add a `require_serial` BooleanField to the `Part` model (default `False`) that is only meaningful when `trackable=True`. Surface it through the Part serializer and a toggle in the React part-edit form. Then add one validation rule to `StockItem` (in `clean()`/serializer validation) that raises a `ValidationError` when the part has `require_serial=True` and no serial is provided — covering both manual stock creation and receiving stock against a purchase order. This is the lower-risk of the two options in the issue (a second boolean) because it leaves the existing `trackable` field and all current behavior untouched, and only activates new behavior when a user opts in.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Trackable parts allow but don't require serials. We need an opt-in "serial required" rule that blocks creating/receiving stock for such a part without a serial.

**Match:** The codebase already has the exact patterns I need:
- Boolean part flags (`trackable`, `assembly`, `salable`) defined together in `part/models.py` — model the new field the same way.
- Serial-conditional validation already exists in `stock/models.py` (`clean()` raises `ValidationError` when a serialized item's quantity ≠ 1). I add a sibling rule there.
- Existing migrations in `part/migrations/` show the project's migration style.

**Plan:**
1. Add `require_serial = models.BooleanField(default=False, ...)` to `Part` in `part/models.py`.
2. Generate the migration via `invoke migrate` / `makemigrations`.
3. Add the field to the Part serializer in `part/serializers.py`.
4. Add validation in `StockItem.clean()` (and/or the stock create/receive serializer) raising `ValidationError` when `self.part.require_serial` is true and no serial is set.
5. Add a toggle for the field in the React part-edit form.
6. Add unit tests in `part/` and `stock/` covering the new rule.

**Implement:** [Link to your branch/commits here as you work — Phase III]

**Review:** [Self-review against InvenTree CONTRIBUTING.md: migrations committed, `invoke dev.test` passes, pre-commit/linters clean, frontend translations updated if needed — Phase III]

**Evaluate:** Backend unit tests + manual test: create a part with `require_serial=True`, attempt to receive stock without a serial, confirm it's rejected; with a serial, confirm it succeeds.

---

## Testing Strategy

### Unit Tests (planned — to implement in Phase III)

- [ ] Creating a StockItem for a part with `require_serial=True` and **no** serial raises `ValidationError`.
- [ ] Creating a StockItem for the same part **with** a valid serial succeeds.
- [ ] A part with `require_serial=False` (default) keeps existing behavior — stock with no serial is accepted.
- [ ] `require_serial=True` has no effect when `trackable=False` (or is disallowed/ignored, per maintainer guidance).
- [ ] The new field round-trips through the Part API serializer (read + write).

### Integration Tests (planned)

- [ ] Receiving stock against a purchase order without a serial is blocked when `require_serial=True`.
- [ ] The React part-edit form shows and persists the new toggle.

### Manual Testing

[To be filled in Phase III — results of manually creating a required-serial part and attempting to receive stock with and without a serial]

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
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

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

- Issue #11258 — https://github.com/inventree/InvenTree/issues/11258
- InvenTree `Part` model — `src/backend/InvenTree/part/models.py` (`trackable` field + serial helper methods)
- InvenTree `StockItem` model — `src/backend/InvenTree/stock/models.py` (existing serial validation in `clean()` / `validate_unique()` / `serializeStock()`)
- InvenTree contributor docs — https://docs.inventree.org/en/latest/develop/contributing/
- [Add migration/Django docs and any Stack Overflow / discussion links you use during Phase II–III]
