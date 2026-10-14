# 8. Lessons Learned

[← Back to README](../README.md)

What I'd tell anyone putting an LLM in front of real money and real customers.

### 1. Let the model understand; let code decide

The model was consistently excellent at reading intent: mixed languages, typos, voice transcripts,
half-finished sentences. It was consistently unreliable at arithmetic, exact prices and keeping
track of what had actually happened. Once anything involving money or an irreversible action was
moved into code, a whole class of bugs disappeared, and the prompt got *shorter*.

### 2. Check output, don't just instruct

"Never invent products" in a prompt lowers the error rate. A check that compares every product name
in the reply with the catalog brings it close to zero. Prompts set the default behaviour; checks
enforce the rules. When a check fails, a precise correction ("the cart says Rs 480, you wrote
Rs 450") fixes the reply in one short lap.

### 3. Fixed procedures belong in code

Checkout has a fixed order: details, instructions, summary, confirmation. Asking the model to follow
it gave *mostly* correct flows. Enforcing the step in code and letting the model write each message
gave correct flows every time, and the conversation still sounded natural.

### 4. Most bugs are two truths disagreeing

Two clocks, two retry loops, two copies of an address, two order records for one sale. Looking for "what
else thinks it knows this?" found the root cause faster than reading the code that failed.

### 5. Prefer fixing a rule over adding one

Many fixes changed or replaced existing logic instead of adding new code. For example, the cart expiry
reused the reminder record that already existed, rather than adding a timer. A small change in the right
place is easier to review, and less likely to break something three features away.

### 6. Fallbacks need to know *why* they're falling back

"Use cached data when the lookup fails" is right for an outage and wrong for a deletion. Every
fallback should separate "unavailable" from "gone".

### 7. Cost is a design concern

Prompt caching, stopping early once a result is certain, smaller lap caps for corrections, and
detecting repeated tool calls each came from looking at what a single message actually cost. With
an LLM, an unnecessary round trip costs money as well as time.

### 8. Your test suite is made of your incidents

The most valuable tests weren't written up front. They came from production conversations that broke
something. And tests have to be **unable** to write to production, not just unlikely to.

---

[← Testing strategy](07-testing.md) · [Back to README](../README.md)
