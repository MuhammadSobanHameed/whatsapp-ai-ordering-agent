# 2. AI Agent Design & Guardrails

[← Back to README](../README.md)

The core design question was **how much to trust the model.** An LLM is excellent at working out
what someone means from "bhai 2 wali biryani aur woh green chutney bhi". It is not trustworthy at
arithmetic, at remembering exact prices, or at not claiming things that didn't happen.

So responsibility is split down the middle:

| The model decides | Code decides |
|---|---|
| What the customer is asking for | Which catalog item that maps to, and its real ID and price |
| Which tool to call | Whether the call is allowed right now (for example, no placement before a summary) |
| How to phrase the reply, in the customer's language | Every number in the reply: line prices, subtotal, tax, total |
| When to ask a clarifying question | Whether an order is actually placed |
| | The final receipt, which code renders from the saved order |

## Tools

The model gets 15 tools, grouped by job:

- **Catalog:** search items (by name, or by need, such as "something for a cough"), list categories
- **Cart:** add, set quantities (absolute or relative: "+1", "make it 3"), remove, clear
- **Add-ons:** choose add-ons, change add-ons
- **Checkout:** set delivery details, add order instructions, save (place) order
- **After-sales:** order status, order history, cancel order, confirm delivery

The tool descriptions are written as **contracts, not hints**. Each one says when to call it, when *not*
to, and what the result means. For example, the cart tools take a *product name and quantity*,
never a price. A price field in a tool's input would only be an invitation to make one up.

```python
# Illustrative tool definition
{
    "name": "add_to_cart",
    "description": (
        "Add products the customer asked for. Use names exactly as returned by "
        "search_items. Never pass a price: the price comes from the catalog."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "items": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "product_name": {"type": "string"},
                        "quantity": {"type": "string", "description": "e.g. '2', '+1', '-1'"},
                    },
                    "required": ["product_name", "quantity"],
                },
            }
        },
        "required": ["items"],
    },
}
```

Inside the tool, the name is resolved against the catalog again. If it doesn't resolve to exactly one
real item, the tool **refuses and explains why** in its result, and the model has to ask the customer.

## The agent loop

```python
# Illustrative: the shape of the loop, not the production code
def run(system, messages, tools, max_laps=10, stop_when=None):
    seen_calls = Counter()
    withhold_tools = False

    for lap in range(max_laps):
        resp = call_llm(system, messages, tools=None if withhold_tools else tools)
        if resp.stop_reason != "tool_use":
            return text_of(resp)

        results = []
        for call in tool_calls(resp):
            key = (call.name, canonical_json(call.input))
            seen_calls[key] += 1
            if seen_calls[key] > 2:
                # The model is retrying a call that was already refused.
                # Take the tools away so it has to answer in words.
                withhold_tools = True
            results.append(execute(call))

        messages += [assistant_turn(resp), tool_results_turn(results)]
        if stop_when and stop_when():
            return ""   # e.g. the order was saved: code renders the receipt
    return fallback_reply()
```

Three details came straight out of production logs:

1. **Detecting identical repeated calls.** When a tool refused an action, the model sometimes "retried"
   with exactly the same call again, until it hit the iteration cap. That was ten paid requests for one
   message, and it emptied the API key pool for the next customer. Now an identical call made a third
   time takes the tools away, so the model has to reply in words.
2. **Stopping early once the order is saved.** The receipt is always built by code from the saved
   order. Without `stop_when`, the model spent one more round trip (~1.5 s and ~9k tokens) writing a
   paraphrase that was then thrown away.
3. **A cap with room for multi-item orders.** Each product takes about two laps (search, then add). A
   three-product message needed seven laps and failed on the original cap of six.

## Guardrails: checking the reply before the customer sees it

Once the model has finished, its draft reply goes through a set of **deterministic validators** that
compare it with the real cart and catalog:

| Guard | Catches |
|---|---|
| Invented products | Item names in the reply that don't exist in this shop's catalog |
| Invented sizes or variants | "Large", "family size" and so on when the item has no such option |
| Misquoted prices | A price in the reply that doesn't match the catalog |
| Summary check | Lines, quantities, add-ons, subtotal, tax and total that disagree with the cart; a name or address that disagrees with what the customer said |
| Misstated cart | "Added X" when X was never added; add-ons listed on the wrong line |
| False placement claim | "Your order has been placed!" when no order was saved |
| Premature summary | A final summary before the checkout questions were answered |
| Dropped request | The customer asked for three things and only two were handled |

If a guard fails, the reply is **not sent**. Instead the agent adds an *operator note*, a short
instruction from the code that says exactly what is wrong, and runs a short correction loop:

```python
# Illustrative
problems = validate_summary(cart, reply)
if problems:
    messages.append(operator_note(
        "Your summary is wrong. The cart has: 2 x Chicken Biryani @ Rs 480 = Rs 960. "
        "Subtotal Rs 960, GST Rs 154, Total Rs 1,114. Rewrite the summary with these figures."
    ))
    reply = run(system, messages, tools, max_laps=3)   # a short lap: everything it needs is in context
```

The correction loop is capped at **3 laps** on purpose. A correction already has every catalog result
it needs in the transcript, so it needs one lap to act and one to answer. If it got the usual cap, a guard
firing on an already expensive turn could double its cost.

If correction still fails, the reply falls back to text **rendered by code** from the real cart. It is
less conversational, but it is always correct.

## Consent: when does "yes" mean "place the order"?

Placing an order can't be undone, so the rule is strict. **The whole message has to be a
confirmation.** "Yes" placed inside "yes but change the address" must not place anything.

```python
# Illustrative
CONFIRM_WORDS = {"yes", "y", "ok", "okay", "ji", "haan", "han", "theek hai", "done",
                 "confirm", "place it", "bhej do", "chalo", "sahi", ...}
FILLER = {"please", "bhai", "shukriya", "thanks", "jee", ...}

def is_pure_confirmation(text: str) -> bool:
    words = normalise(text)            # lowercase, strip emoji/punctuation, collapse repeats ("okkk")
    rest = remove_phrases(words, CONFIRM_WORDS)
    if rest == words:                  # no confirmation word at all
        return False
    return all(w in FILLER for w in rest)   # anything else = a change request, not consent
```

The confirmation vocabulary was widened from real chats, for example "g", "k", "acha", "zaroor" and
"send it". Every addition went in with a test that also checks lookalikes **don't** match.

## Checkout as a fixed sequence

The model is good at conversation and bad at following a strict procedure. So the checkout steps are
**enforced in code**, and the model only writes each message:

1. Ask for name and address, if they aren't known yet.
2. Ask whether the customer wants to add special instructions.
3. Show **one** full summary: items, add-ons, subtotal, tax, total, name, address and instructions.
4. Place the order on a pure confirmation, then send a receipt rendered by code.

Food requests that arrive while step 2 is open ("add a raita too") are recognised as cart changes,
not saved as instructions.

## Prompt caching

Every request has a large fixed part (rules plus 15 tool schemas) and a small part that changes (the
cart, any placed order and the saved address). The system prompt is split so that only the fixed part
is cached:

```python
# Illustrative
system = [
    {"type": "text", "text": STATIC_RULES, "cache_control": {"type": "ephemeral", "ttl": "1h"}},
    {"type": "text", "text": render_cart_and_state(session)},   # changes every turn, not cached
]
# The conversation history only grows, so a second breakpoint goes on its last message:
messages[-1]["content"][-1]["cache_control"] = {"type": "ephemeral", "ttl": "1h"}
```

- The fixed part is **identical for every customer of the same shop**, so one customer's cache write is
  read by the next.
- A cart change in the uncached part never invalidates the cached part.
- The code checks the usage numbers in each response and logs a warning if the cache stops being read.
  A cache that silently stops working shows up only on the monthly bill.

[← Architecture](01-architecture.md) · [Next: Order lifecycle →](03-order-lifecycle.md)
