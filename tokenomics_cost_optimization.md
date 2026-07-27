# TOKENOMICS & COST OPTIMIZATION
## Forward Deployed Engineer - Applied AI

> **Why this matters for FDE:** You deploy AI systems *at customer sites*, where a working
> demo that costs $0.02/query can quietly become a $40,000/month bill at production scale.
> "How do you keep this from skyrocketing in inference costs?" is a near-guaranteed follow-up
> in the Agentic & ML System Design round. This file is the deep dive behind **Pillar 4 →
> Cost Controls** in `ml_agentic_systems_guide.md`.

---

## THE MENTAL MODEL: What You Actually Pay For

You are billed **per token, split into input and output**, and **output is 3-5x more
expensive than input**. Everything below flows from that one asymmetry.

```
cost = (input_tokens  × input_price_per_token)
     + (output_tokens × output_price_per_token)
```

**Current Claude pricing (per 1M tokens, as of 2026-07):**

| Model            | Model ID          | Input $/1M | Output $/1M | Use when |
|------------------|-------------------|-----------|-------------|----------|
| Claude Haiku 4.5 | `claude-haiku-4-5`| $1.00     | $5.00       | Classification, routing, extraction, high-volume simple tasks |
| Claude Sonnet 5  | `claude-sonnet-5` | $3.00*    | $15.00*     | The default workhorse — most agentic steps, RAG synthesis |
| Claude Opus 4.8  | `claude-opus-4-8` | $5.00     | $25.00      | Hard reasoning, planning, final-answer quality |
| Claude Fable 5   | `claude-fable-5`  | $10.00    | $50.00      | Most demanding long-horizon reasoning only |

*Sonnet 5 has intro pricing of $2/$10 per 1M through 2026-08-31.

**The one number to memorize:** Opus output is **5x** Sonnet-intro output and **5x** Haiku
output. Routing a task down one tier is the single biggest lever you have.

---

## THE COST LADDER (cheapest → most involved)

Optimize in this order. Each rung is roughly a step more effort for a step less savings.

1. **Right-size the model** (route simple work to Haiku) — up to ~80% off per call
2. **Prompt caching** (reuse a stable prefix) — up to ~90% off input on cache hits
3. **Cut the tokens** (shorter context, capped output) — linear, compounding savings
4. **Batch** (non-urgent work via Batch API) — flat 50% off
5. **Semantic caching** (skip the model entirely for repeat questions) — 100% off on hits
6. **Budgets + circuit breakers** (bound the worst case) — caps blast radius, not average cost

---

## 1. MODEL RIGHT-SIZING & ROUTING

The most common cost mistake is using one big model for everything.

```python
# AI generates: one expensive model for every call
def handle(query: str) -> str:
    return call_model("claude-opus-4-8", query)   # $5/$25 for "is this spam?"

# Should be: route by task difficulty; escalate only when needed
def route_model(task_type: str) -> str:
    cheap = {"classify", "extract", "route", "summarize_short"}
    hard  = {"plan", "multi_step_reason", "final_synthesis"}
    if task_type in cheap:
        return "claude-haiku-4-5"     # $1/$5
    if task_type in hard:
        return "claude-opus-4-8"      # $5/$25
    return "claude-sonnet-5"          # $3/$15 default workhorse
```

**Escalation pattern (cheap-first, verify, retry high):** run Haiku, and only if a
confidence check or validator fails do you re-run on Opus. Most traffic never touches the
expensive model, but hard cases still get top quality.

```python
def answer_with_escalation(query: str) -> str:
    draft = call("claude-haiku-4-5", query)
    if is_confident(draft):          # e.g. schema-valid, no "I'm not sure", passes checks
        return draft
    return call("claude-opus-4-8", query)   # pay for quality only when cheap tier fails
```

**What to Say:**
> "I don't pick one model for the whole system — I tier by task. Classification, routing, and
> extraction go to Haiku; the agent's reasoning steps to Sonnet; only the hard planning or the
> final customer-facing answer touches Opus. For borderline cases I use cheap-first escalation:
> run the small model, and only re-run on the big one when a validator fails. In practice most
> traffic never hits the expensive tier, so the average cost per request drops by more than half
> with no quality loss on the cases that matter."

---

## 2. PROMPT CACHING — the highest-leverage single change

If every request repeats a big stable chunk — a system prompt, a tool schema, a policy doc,
few-shot examples, a retrieved document reused across turns — you're **paying full input price
to re-send identical tokens.** Prompt caching stores that prefix server-side.

**Economics:**
- **Cache write:** 1.25x base input price (5-min TTL) or 2x (1-hour TTL) — a one-time premium.
- **Cache read:** ~0.1x base input price — i.e. **~90% off** for every subsequent hit.
- Break-even is ~2 hits; anything reused more than that is almost pure savings.

```python
# AI generates: 8K-token system prompt re-sent at full price every call
messages = [{"role": "user", "content": SYSTEM_8K + user_query}]

# Should be: mark the stable prefix as cacheable; pay ~10% on every reuse
response = client.messages.create(
    model="claude-sonnet-5",
    system=[{
        "type": "text",
        "text": SYSTEM_8K,                       # policy, tools, few-shot — stable across calls
        "cache_control": {"type": "ephemeral"},  # cache this prefix
    }],
    messages=[{"role": "user", "content": user_query}],  # only this varies
    max_tokens=1024,
)
```

**Rules that make or break caching:**
- The cached prefix must be **byte-identical** and **at the front**. Put stable content first
  (system prompt → tools → docs), volatile content (the user's turn) last. One changed token
  early busts the whole cache.
- Great fit: chatbots with a fixed persona, RAG over a hot document set, agents re-sending the
  same tool definitions every loop iteration.

**What to Say:**
> "The biggest quick win is almost always prompt caching. In a customer deployment the system
> prompt, tool definitions, and policy docs are identical on every call — often thousands of
> tokens. Caching that prefix means we pay full price once, then about a tenth of the input
> price on every subsequent request. The catch is ordering: stable content has to sit at the
> front, byte-for-byte identical, with the user's input last. That one change routinely cuts
> input cost by 80-90% on chat and RAG workloads."

---

## 3. CUT THE TOKENS (context in, output out)

Caching makes repeated tokens cheap; this makes sure you never send junk tokens at all.

### Input side — stop stuffing context
```python
# AI generates: dump the whole knowledge base into every prompt
context = "\n".join(all_documents)          # 50K tokens, mostly irrelevant

# Should be: retrieve only the top-k relevant chunks (this is why RAG exists)
context = "\n".join(retrieve_top_k(query, k=5))   # ~2K tokens, all relevant
```
- Retrieve, don't stuff. Top-k relevant chunks beat the whole corpus on both cost *and* quality.
- Summarize/compact long agent histories instead of re-sending the full transcript each turn
  (see the `ContextManager` in Pillar 3 of the ML guide).
- Trim tool-result payloads — return the 3 fields the model needs, not a 4KB JSON blob.

### Output side — the expensive side
Output tokens cost 3-5x input, so bounding them is high-value:
```python
response = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=512,                 # hard ceiling: never pay for a runaway essay
    messages=[{"role": "user",
               "content": "Reply with ONLY a JSON object: {label, confidence}"}],
)
```
- Set `max_tokens` deliberately — it's a **cost ceiling**, not just a safety limit.
- Ask for structured/terse output ("JSON only", "one word", "≤3 bullets"). Prose is tokens
  you pay for and then parse away.

### Always count tokens with the real tokenizer
```python
# AI generates: guess with a rule of thumb, or use tiktoken (an OpenAI tokenizer)
n = len(text.split())                     # wrong
n = len(tiktoken_encoding.encode(text))   # WRONG for Claude — undercounts 15-20%

# Should be: use Anthropic's count_tokens endpoint before you send
n = client.messages.count_tokens(
    model="claude-sonnet-5",
    messages=[{"role": "user", "content": text}],
).input_tokens
```
> **Interview trap:** never estimate Claude costs with `tiktoken` — it's OpenAI's tokenizer and
> undercounts Claude tokens by ~15-20%, so your budget math comes out low. Use `count_tokens`.

---

## 4. BATCH THE NON-URGENT WORK

Anything that doesn't need a real-time answer — nightly evals, bulk classification, embedding a
document backlog, offline enrichment — should go through the **Batch API for a flat 50% discount**
on both input and output (async, results typically within the hour).

```python
# AI generates: loop 100K docs through the live endpoint at full price + rate limits
for doc in docs:
    classify(doc)                    # full price, synchronous, hammers rate limits

# Should be: submit as one batch — 50% off, no rate-limit babysitting
batch = client.messages.batches.create(requests=[
    {"custom_id": doc.id,
     "params": {"model": "claude-haiku-4-5", "max_tokens": 128,
                "messages": [{"role": "user", "content": classify_prompt(doc)}]}}
    for doc in docs
])
```
Rule of thumb: **if a human isn't waiting on the response, batch it.** Batch + Haiku + caching
stack multiplicatively on offline pipelines.

---

## 5. SEMANTIC CACHING — skip the model entirely

Prompt caching discounts a call; semantic caching *eliminates* it. If a new query is
semantically close to one you've already answered, return the stored answer for **$0**.

```python
# Should be: check a cache of past (embedding → answer) before calling the model
def answer(query: str) -> str:
    q_emb = embed(query)
    hit = vector_cache.search(q_emb, threshold=0.95)   # cosine similarity
    if hit:
        return hit.answer                              # $0, ~0 latency
    ans = call("claude-sonnet-5", query)
    vector_cache.add(q_emb, query, ans)
    return ans
```
- Huge for support bots / FAQ-style traffic where the same questions recur constantly.
- **Caveat to raise proactively:** set the similarity threshold high (~0.95) and scope the cache
  by tenant/context, or you'll serve a confidently wrong cached answer to a subtly different
  question. Cache invalidation when the underlying data changes is the hard part.

---

## 6. BUDGETS, METERING & CIRCUIT BREAKERS

Right-sizing lowers *average* cost; budgets bound the *worst case* — a looping agent, an abusive
user, or a runaway workflow. This is the `CostTracker` from Pillar 4, filled in with real pricing.

```python
class BudgetExceededError(Exception):
    pass

# Prices per 1M tokens → per-token, split input/output (output is the expensive side)
PRICING = {  # (input_per_token, output_per_token)
    "claude-haiku-4-5": (1.00 / 1e6, 5.00 / 1e6),
    "claude-sonnet-5":  (3.00 / 1e6, 15.00 / 1e6),
    "claude-opus-4-8":  (5.00 / 1e6, 25.00 / 1e6),
    "claude-fable-5":   (10.00 / 1e6, 50.00 / 1e6),
}

def calculate_cost(model: str, input_tokens: int, output_tokens: int) -> float:
    """The helper the CostTracker in Pillar 4 calls."""
    in_price, out_price = PRICING[model]
    return input_tokens * in_price + output_tokens * out_price

class CostTracker:
    """Extends Pillar 4's CostTracker: per-session AND per-user budgets."""
    def __init__(self, session_budget=1.00, user_budget=10.00):
        self.session_budget = session_budget
        self.user_spent: dict[str, float] = {}
        self.session_spent = 0.0

    def track_call(self, user_id: str, model: str,
                   input_tokens: int, output_tokens: int) -> float:
        cost = calculate_cost(model, input_tokens, output_tokens)
        self.session_spent += cost
        self.user_spent[user_id] = self.user_spent.get(user_id, 0.0) + cost

        if self.session_spent >= self.session_budget:
            raise BudgetExceededError(f"Session budget ${self.session_budget} exceeded")
        if self.user_spent[user_id] >= self.user_budget:  # stop one user draining the account
            raise BudgetExceededError(f"User {user_id} exceeded ${self.user_budget}")
        return cost
```

- **Meter before you optimize.** Log tokens + cost tagged by `user_id`, `feature`, and `model`
  on *every* call. You can't cut what you can't see, and it's the first thing to show a customer.
- Pair budgets with the **circuit breakers** from Pillar 4 (max iterations, max tokens,
  wall-clock timeout) so an agent loop fails safe instead of billing forever.

**What to Say:**
> "I meter cost per call tagged by user, feature, and model, so we can see exactly where spend
> goes. On top of that I set hard budgets — per session and per user — plus the circuit breakers
> from the guardrails layer. If an agent loops or a single user goes abusive, it trips the budget
> and fails gracefully instead of running up an unbounded bill. That's what lets a customer leave
> the system running in production without watching it."

---

## PUTTING IT TOGETHER: the 60-second cost answer

> "First I meter everything — cost per call by user, feature, and model — because you optimize
> what you measure. Then I work a ladder. **Right-size the model:** Haiku for classification and
> routing, Sonnet for the agent's steps, Opus only for hard reasoning, with cheap-first
> escalation. **Prompt caching** on the stable system prompt, tools, and docs — that's usually
> an 80-90% cut on input alone. **Trim tokens** on both ends: top-k retrieval instead of dumping
> the corpus, and a hard `max_tokens` cap since output is the pricey side. **Batch** anything
> offline for 50% off, and **semantic-cache** repeat questions to skip the model entirely. Then
> **budgets and circuit breakers** bound the worst case. Stacked, that's routinely a 5-10x cost
> reduction versus the naive 'one big model, full context every call' baseline — with no user-
> facing quality loss."

---

## QUICK-REFERENCE: lever → savings → cost

| Lever | Typical savings | What it costs you |
|-------|-----------------|-------------------|
| Right-size / route to smaller model | up to ~80% per call | A router + confidence check; quality risk if under-routed |
| Prompt caching (stable prefix) | up to ~90% off input on hits | Careful prompt ordering; ~1.25-2x write premium |
| Top-k retrieval vs full context | linear in tokens cut | A retriever / vector store |
| `max_tokens` cap + terse output | 3-5x-weighted (output side) | Risk of truncation if set too low |
| Batch API (offline work) | flat 50% | Async only — not for real-time |
| Semantic caching | 100% on cache hits | Stale/wrong-answer risk; invalidation logic |
| Per-session/user budgets | bounds worst case | Must handle the fail-gracefully path |

**Three traps interviewers listen for:**
1. Estimating Claude cost with `tiktoken` → use `count_tokens`.
2. Forgetting **output tokens cost 3-5x input** → cap and shorten output first.
3. Semantic caching without a tight threshold + invalidation → confidently wrong answers.