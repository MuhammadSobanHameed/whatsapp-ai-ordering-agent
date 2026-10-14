# 6. Production Incidents

[← Back to README](../README.md)

These are real bugs, almost all found in live traffic. They're written up in a blameless *symptom → root
cause → fix → lesson* format. Most of them have the same shape: **two parts of the system disagreed
about the truth.** Two clocks, two copies of state, two retry loops. The lasting fix was nearly always
to pick one source of truth, rather than add another patch on top.

---

## 1. A 101-second reply

**Symptom:** At busy times, some customers waited well over a minute for a reply. One message took
101 seconds.

**Root cause:** The app rotates between API keys when one is rate-limited. But the Anthropic SDK has its
own retry loop *underneath* the app. With its default settings, it caught every 429 and slept until the
`Retry-After` time had passed, so the app's rotation code never ran.

**Fix:** Set the SDK's `max_retries=0`, so retrying is handled only by the code that can switch keys.

**Lesson:** Retries should happen in exactly one layer. Two retry loops stacked on each other give the
worst of both.

---

## 2. The same order placed twice

**Symptom:** A customer received two identical orders a minute apart.

**Root cause:** The order had saved with placeholder delivery details, and the cart was cleared. The
customer then sent their name and address. On the next turn, the model saw an empty cart and someone
giving delivery details, which looks exactly like the start of a new order. So it rebuilt the cart and
placed it again. Nothing in the system remembered that an order had just been placed.

**Fix:** A short-lived record of the order just placed: its ID and a fingerprint of the items. It is
saved to disk so a restart can't lose it, and it expires with the session. The next turn knows "an
order was just placed" and treats new details as an update to that order.

**Lesson:** When state is cleared after an action, something must still record that the action happened.

---

## 3. The bot went blind when the ERP added authentication

**Symptom:** Almost every ERP request suddenly failed (157 of 161 in one window). For a while the bot
kept working from cached data, then fell over as the caches expired.

**Root cause:** The ERP began requiring bearer tokens on every route, and the bot sent none.

**Fix:** An auth module that signs in, refreshes with the refresh token, and reads the expiry time
from the JWT's own `exp` claim rather than a local timer. The server's clock and ours disagreed by
enough to matter.

**Lesson:** Graceful degradation can hide a total failure. Old cached data kept the bot looking healthy
while it was actually broken, so cache fallbacks need alerts too.

---

## 4. The backend clock was hours in the future

**Symptom:** Abandoned-cart reminders went out many hours late, or not at all.

**Root cause:** The reminder timed a cart's age from the order timestamp the ERP returned. That
timestamp came back several hours *in the future*, so new carts looked like they had a negative age for
half a day.

**Fix:** Every age-based rule (the reminder, the 24-hour messaging window, the delivery check) now uses
one clock this system controls: the time of the customer's last message. I reported the timestamp shift
to the backend team.

**Lesson:** If you don't own a clock, don't base your timing on it.

---

## 5. One message emptied the whole key pool

**Symptom:** One customer's message was followed by other customers getting "I'm a bit busy right now".

**Root cause:** A tool refused an action and explained why. The model "retried" with exactly the same
call, again and again, until the iteration cap. That was ten ~5k-token requests for one message, and
they used up every key's rate limit.

**Fix:** The loop counts each call by its tool name and exact arguments. A third identical call takes
the tools away, so the model has to answer in words. Correction loops also get a smaller lap cap.

**Lesson:** An agent loop needs to detect when it is going round in circles, not just a maximum number of laps.

---

## 6. A removed number kept answering

**Symptom:** After a shop's WhatsApp credentials were removed in the portal, the bot kept replying from
that number until the container was restarted.

**Root cause:** The credentials cache fell back to old cached data on **any** failed lookup, including
"not found". It was built to survive ERP outages, but it couldn't tell "the ERP is down" apart from "this
was deliberately removed".

**Fix:** Old cached data is used only on real outages (5xx errors or timeouts). A "not found" response or
blank credentials drop the cache entry straight away. The number-to-tenant mapping got the same rule.

**Lesson:** A fallback must be able to tell "unavailable" from "gone". Treating them the same turns a
safety net into a stale copy that never goes away.

---

## 7. A reply to the reminder found an empty cart

**Symptom:** A customer replied to the abandoned-cart reminder ("yes, send it") and was told their cart
was empty.

**Root cause:** Two clocks. The reminder went out after 20 minutes of silence, and the cart expired 30
minutes after the customer's last message. A customer who replied 15 minutes after the reminder was
answering about a cart that had already been deleted.

**Fix:** No new timer was added. The cart's expiry check now also looks at the reminder record that
already existed: a reminded cart stays alive for a full window from the **reminder**. If a cart really has
expired, the customer gets an apology instead of a confusing "your cart is empty".

**Lesson:** If two rules describe the same customer's journey, they have to share a clock.

---

## 8. Gaps in the shop's order numbers

**Symptom:** The shop's order numbers had gaps, and drafts appeared to "use up" numbers.

**Root cause:** Carts are saved as Draft orders, and the ERP gives every new order a number. The order
was placed by creating a **new** order and deleting the draft, so each placed order used two numbers.

**Fix:** Placing the order now updates the draft **in place**. It keeps the same order ID, number and
idempotency key, and just changes the status to *Order Received*. Add-ons are sent explicitly on every
line (`"[]"` for none), because the update endpoint had dropped them in the past.

**Lesson:** An entity that changes state should stay the same record. Creating a new one and deleting
the old one spreads one thing across two IDs.

---

## 9. The address in the summary wasn't the address that got saved

**Symptom:** A customer approved a summary that included their full address, plot number and all,
but the saved order had a shorter address.

**Root cause:** The summary was written by the model, while the saved address came from separately
stored delivery details. Nothing checked that the two matched. What the customer approved and what
got saved were two separate copies.

**Fix:** The summary check, which already verified items and totals, now also checks the name and address
in the summary against the stored details. When the summary shows an address the customer actually typed,
that address is saved.

**Lesson:** What the customer approves has to be exactly what you save. Check it, don't assume it.

---

## 10. A UK shop quoting prices in rupees

**Symptom:** A shop in the UK showed its prices as "Rs 6".

**Root cause:** The currency symbol was hard-coded from when there was only one market.

**Fix:** The currency is read from each tenant's settings (an ISO code), mapped to the symbol customers
expect, and stored in a `ContextVar` for the duration of each message. An unknown currency code is shown
as the code itself ("XAF 500"). It looks odd but is never misleading, unlike defaulting to a symbol that
means a different currency.

---

## 11. "Yes" to "anything else?" could add a paid extra (caught in design)

**Risk:** When add-ons were added, a test conversation showed that a customer answering "yes" to
"anything else?" could be charged for an add-on offered a few messages earlier.

**Root cause:** The model linked the later "yes" to the older add-on question. Nothing recorded which
question was actually still open.

**Fix:** A ledger of add-on questions that are still **waiting for an answer**. A bare "yes" can only
accept an add-on whose question is currently open. The model may *suggest* that an add-on was accepted,
but the ledger decides.

**Lesson:** Consent is tied to a specific question. Record which question was asked.

[← Reliability & operations](05-reliability-and-operations.md) · [Next: Testing strategy →](07-testing.md)
