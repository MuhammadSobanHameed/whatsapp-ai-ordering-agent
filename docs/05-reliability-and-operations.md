# 5. Reliability & Operations

[← Back to README](../README.md)

A chat bot is unforgiving to operate. Customers wait on every reply, WhatsApp re-sends messages it
thinks failed, and every outside dependency (the LLM, speech-to-text, the ERP, Redis) will
eventually be slow or down. This section covers how the system copes.

## LLM API: one retry policy, in one place

Several API keys form a pool. A rate-limit error (HTTP 429) **parks** that key for as long as the
response's `Retry-After` header says, and the request moves straight to the next key.

```python
# Illustrative
clients = [Anthropic(api_key=k, max_retries=0) for k in keys]   # max_retries=0 is deliberate

def call_llm(**req):
    for client in pool.available():          # each key gets at most one try per cycle
        try:
            return client.messages.create(**req)
        except RateLimitError as e:
            pool.park(client, seconds=retry_after(e))
        except APIStatusError as e:
            if e.status_code in (500, 529):  # server error / overloaded: brief, try another key
                continue
            raise
    raise AllKeysBusy()
```

`max_retries=0` is the important line. The SDK has its own retry loop *underneath* this code. Left at
its default, it caught every 429 itself and **slept** until the `Retry-After` time had passed, so the
key-rotation code never even saw the error. One production message took **101 seconds** that way.
Retries now happen in exactly one place: the one that can switch keys.

Speech-to-text has its **own** key pool, so a rate limit from one provider never parks the other
provider's keys.

## Caches that fail safe

The rule: **a cache that's down must look exactly like a cache that's empty.** A cache that can take
the bot down is worse than having no cache.

**Redis (conversation history)**
- It is off unless configured, so a deployment without Redis behaves exactly as it did before Redis was added.
- Socket timeouts are about half a second. A hung Redis must not add seconds to a customer's reply.
- A **circuit breaker** stops trying for a while after a few failures, so not every message pays the
  connect timeout.
- Every miss *or* error falls back to the ERP's memory API, which also receives every write. Redis can
  be wiped at any time without losing a conversation.
- The key's expiry is the session window itself. If the key exists, the session is live, with no
  timestamp maths and no clock difference to worry about.

**Tenant credentials and catalog**
- Cached for a few minutes, so a busy shop doesn't hit the ERP on every message.
- Old cached data is used **only when the ERP is having an outage** (5xx errors or timeouts).
  If the ERP answers *"not found"*, or the credentials come back blank, the cache entry is dropped at
  once. The shop has removed or changed something, and the bot must stop using the old values
  right away. See [incident #6](06-production-incidents.md#6-a-removed-number-kept-answering).

```python
# Illustrative
def get_credentials(tenant_id):
    try:
        resp = erp.get(f"/whatsapp-setup/{tenant_id}", timeout=10)
    except (Timeout, ConnectionError):
        return cache.get_stale(tenant_id)          # outage: keep serving
    if resp.status_code >= 500:
        return cache.get_stale(tenant_id)          # outage: keep serving
    if resp.status_code == 404 or not usable(resp.json()):
        cache.drop(tenant_id)                      # removed or changed: stop now
        return None
    return cache.put(tenant_id, parse(resp.json()))
```

## Authentication with the ERP

When the ERP started requiring bearer tokens, the bot had been sending no credentials. Almost every
request failed, and for a while it was answering customers only from old cached data.

The auth module now:
- signs in once and refreshes using the refresh token, so the password isn't sent again;
- reads the expiry time from the **token's own `exp` claim**, not from a local timer, because the
  server's clock and ours don't agree;
- uses one token for all tenants, with each request limited to a tenant by its tenant ID.

## Webhook safety

- **Signature verification:** each webhook body is checked with HMAC-SHA256 against that tenant's
  app secret, compared with `hmac.compare_digest` so the check can't leak the signature through timing.
- **Duplicate removal:** WhatsApp re-sends webhooks it thinks failed. The IDs of processed messages are
  kept for a while and saved to disk **atomically** (written to a temporary file, then renamed), so a crash
  can't leave a half-written file that loses every ID.
- **Fast acknowledgement:** the webhook returns quickly, and the slow work happens after it has replied.

## Idempotency

Each order carries an idempotency key. When the cart is confirmed in place, it keeps the same key and
the same order number, so retries and re-sent webhooks can't create a second order.

## Cost tracking

Tokens are counted per *message*, but a business is billed per *order*. Usage is added up per
conversation and reported once, when the order is placed. Voice-note seconds are counted separately,
because speech-to-text is billed by the second and no token counter would notice it. The running
totals are saved to disk, because losing them would mean under-billing, and unlike a cache they
can't be rebuilt.

## Operations

- Docker Compose (bot + Redis), with Nginx TLS in front.
- Size-capped container logs, plus HTTP access logs to tell "never delivered" apart from "received
  and dropped".
- Configuration comes from environment variables, so behaviour like cache expiry, cache time-to-live,
  session length and status names can be adjusted per deployment without code changes.

[← Catalog search](04-catalog-search.md) · [Next: Production incidents →](06-production-incidents.md)
