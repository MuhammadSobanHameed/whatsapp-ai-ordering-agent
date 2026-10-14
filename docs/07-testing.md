# 7. Testing Strategy

[← Back to README](../README.md)

Testing an LLM application has an awkward problem: the most important component doesn't give the same
answer twice. The strategy has three layers, each answering a different question.

## Layer 1: deterministic tests with a fake model

*Does the code around the model do the right thing?*

A fake Claude client returns scripted responses: a tool call, then a text reply. Tests then check what the
code did with them. This covers everything code owns:

- the confirmation parser, including lookalikes that must **not** count as consent;
- the summary validator, given correct and deliberately wrong summaries;
- cart arithmetic, relative quantities ("+1", "-1") and add-on pricing;
- guard chains: a bad reply triggers a correction, and a second bad reply falls back to text
  rendered by code;
- key rotation, rate-limit parking and the circuit breaker;
- phone number formats, currency rendering, and the timing of reminders and delivery checks.

```python
# Illustrative
def test_yes_with_a_change_is_not_consent():
    assert is_pure_confirmation("yes")
    assert is_pure_confirmation("ok bhej do, shukriya")
    assert not is_pure_confirmation("yes but make it 3")
    assert not is_pure_confirmation("yes change the address")
```

## Layer 2: production conversations as regression tests

*Will this bug come back?*

Every production incident became a test built from the real (anonymised) conversation that caused it.
These suites are named after what broke: a dropped request in a multi-item message, a greeting mistaken
for an order, a status question mistaken for a checkout, and so on. There are 35 suites today.

## Layer 3: replaying against the live model and live catalog

*Does the fix actually work with the real model?*

A fake model can only confirm the behaviour you scripted. Before calling a matcher or ordering fix
done, I replay the conversation with the **real model against the real catalog**, using fake phone numbers.

## The rule: tests must never write to production

Early on, one test made a real API call that created a genuine draft order in the ERP. The
abandoned-cart job then found it and sent a reminder to a real phone number.

Since then, **every call that changes data is stubbed** in tests and replay tools, whether it creates,
updates or deletes. Replays use made-up numbers. If a live write is ever really needed, it is cleaned
up straight after.

```python
# Illustrative
@pytest.fixture(autouse=True)
def no_writes(monkeypatch):
    for fn in ("post", "put", "patch", "delete"):
        monkeypatch.setattr(erp_http, fn, _refuse_write)   # reads still hit the real API
```

## What I check before shipping a change

1. The full suite passes.
2. The original failing conversation, replayed live, now behaves correctly.
3. Nearby behaviour still works: the conversations the change could plausibly affect are replayed too.
4. The change is as small as possible. If an existing rule can be corrected instead of adding a new one,
   that's the change.

[← Production incidents](06-production-incidents.md) · [Next: Lessons learned →](08-lessons-learned.md)
