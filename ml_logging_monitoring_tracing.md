# ML DOMAIN INTERVIEW: TRACING, DEBUGGING & MONITORING AGENTS
## Google Forward Deployed Engineer - Applied AI
## Quick Reference Guide

---

# WHY THIS IS HARD (AND WHY THEY ASK)

Agents are the **worst-case observability problem**: non-deterministic, multi-step,
tool-calling, and expensive. A single user request can fan out into a dozen LLM calls,
retrieval hits, and tool invocations — and any one of them can silently degrade quality
without throwing an error. "It gave a bad answer" is not a stack trace.

Traditional app monitoring (is the server up? what's the 500 rate?) doesn't catch the
failures that matter for agents: a subtly wrong tool argument, a retrieval that returned
irrelevant chunks, a prompt that drifted, a loop that burned $4 before giving up. **You
can't debug what you can't see** — so the FDE question is always "how do you know your
deployed agent is actually working?"

This guide is structured around **THE 3 PILLARS OF OBSERVABILITY** (logs, metrics,
traces) adapted for agents, then debugging workflow, then production monitoring.

---

# THE 3 PILLARS OF OBSERVABILITY

| Pillar | Question it answers | Agent example |
|--------|--------------------|--------------------|
| **Logs** | *What happened at this exact step?* | The full prompt, tool args, and raw completion for one LLM call |
| **Metrics** | *What's the aggregate health over time?* | p95 latency, cost/query, tool-error rate, task success rate |
| **Traces** | *What was the full path of one request?* | The tree of every LLM call, tool call, and sub-agent for a single user turn |

**One-liner to say:** "Logs are the *what*, metrics are the *how much*, traces are the
*why* — traces stitch the individual log lines into the causal chain of a single request."

---

## 1. Tracing — the causal tree of a request

**Core idea:** a **trace** represents one end-to-end request. It's a tree of **spans**,
where each span is one unit of work (an LLM call, a tool call, a retrieval, a sub-agent).
Spans nest — a parent orchestrator span contains child spans for each sub-agent it calls.
This is exactly the mental model from distributed systems (OpenTelemetry), applied to agents.

**Anatomy of a span:**
```
trace_id: abc123                          # one per user request
└── span: orchestrator            (2.4s, $0.03)
    ├── span: retrieve_context    (0.2s)  input=query, output=[5 chunks]
    ├── span: llm_call:planner    (1.1s, 1,200 tok)  prompt=..., completion=...
    └── span: sub_agent:coder     (1.0s, $0.02)
        ├── span: llm_call        (0.6s, 900 tok)
        └── span: tool:run_tests  (0.4s)  input=code, output="2 failed"
```

**What each span should capture:**
- `input` / `output` (the actual prompt and completion — the most important part)
- `latency`, `token counts` (prompt/completion), `cost`
- `model`, `temperature`, and other params
- `status` (ok / error) and any exception
- `metadata`: user_id, session_id, agent_name, tool_name

### Manual instrumentation (framework-agnostic)
```python
from opentelemetry import trace
tracer = trace.get_tracer("agent")

def llm_call(prompt: str, model: str):
    with tracer.start_as_current_span("llm_call") as span:
        span.set_attribute("gen_ai.request.model", model)
        span.set_attribute("gen_ai.prompt", prompt)          # capture input
        response = client.generate(prompt, model=model)
        span.set_attribute("gen_ai.completion", response.text)
        span.set_attribute("gen_ai.usage.prompt_tokens", response.usage.input)
        span.set_attribute("gen_ai.usage.completion_tokens", response.usage.output)
        return response
```
> Note: `gen_ai.*` are the **OpenTelemetry GenAI semantic conventions** — the emerging
> standard for LLM spans. Name-dropping this signals you know the vendor-neutral layer,
> not just one SaaS dashboard.

### Auto-instrumentation (what you actually use in practice)
Most teams don't hand-roll spans — the tracing SDK wraps the framework. One decorator or
callback captures the whole tree:
```python
# LangSmith — set env vars, tracing is automatic for LangChain/LangGraph
# LANGCHAIN_TRACING_V2=true, LANGCHAIN_API_KEY=...

# Langfuse — decorator-based, framework-agnostic
from langfuse.decorators import observe

@observe()                       # auto-creates a span, nests under the current trace
def my_agent_step(query: str):
    return llm_call(query, "gemini-2.0-flash")
```

**ADK note (Google stack):** ADK emits **OpenTelemetry** traces natively and integrates
with **Cloud Trace / Vertex AI**, and the ADK web dev UI shows a per-event trace view
(each LLM call, tool call, and agent transfer) for local debugging. Say this for the
Google-specific answer.

**What to Say:**
> "I instrument the agent so every request produces a trace — a tree of spans for each LLM
> call, tool call, and sub-agent, capturing the actual prompt, completion, tokens, latency,
> and cost. I lean on OpenTelemetry's GenAI conventions so the data is vendor-neutral, and
> surface it in something like LangSmith, Langfuse, or Cloud Trace. When someone reports a
> bad answer, I pull that trace and see exactly which step went wrong instead of guessing."

---

## 2. Metrics — aggregate health over time

**Problem:** a trace explains *one* request. Metrics tell you whether the system is
healthy across *millions* — and catch regressions before users complain.

**The metrics that matter for agents (four buckets):**

| Bucket | Metrics |
|--------|---------|
| **Latency** | p50 / p95 / p99 end-to-end; time-to-first-token; per-step latency |
| **Cost** | tokens in/out per request; $/request; $/user/day; cost by model |
| **Reliability** | tool-error rate; LLM error/timeout rate; retry rate; loop-cap-hit rate |
| **Quality** | task success rate; hallucination/groundedness score; user thumbs up/down |

**Why p95/p99, not average:** LLM latency is long-tailed — a few requests that retry or
loop drag the tail badly. Averages hide the users who are actually suffering.

```python
# Emit metrics per request (StatsD / Prometheus / Cloud Monitoring style)
def record_request_metrics(trace):
    metrics.histogram("agent.latency_ms", trace.duration_ms, tags=[f"agent:{trace.name}"])
    metrics.histogram("agent.cost_usd", trace.cost, tags=[f"model:{trace.model}"])
    metrics.increment("agent.requests_total", tags=[f"status:{trace.status}"])
    if trace.hit_iteration_cap:
        metrics.increment("agent.loop_cap_hit")     # early smell of runaway agents
```

**Golden signals repurposed for agents:** latency, traffic, errors, saturation — plus two
agent-specific ones: **cost** and **quality**. Those last two are what make agent
monitoring different from ordinary service monitoring.

**What to Say:**
> "Beyond standard latency and error rates, I track two things ordinary services don't:
> cost per request and a running quality signal. I alert on p95 latency, cost-per-request
> creeping up, tool-error rate, and how often the agent hits its iteration cap — that last
> one is an early warning that something upstream broke and the agent is flailing."

---

## 3. Quality Monitoring in Production — the hard part

**Problem:** metrics and traces tell you the system *ran*; they don't tell you the answer
was *good*. In prod you usually have no ground truth. This is the piece that separates a
real answer from a naive one.

**Techniques (weakest → strongest signal):**

- **Implicit feedback:** thumbs up/down, copy/regenerate clicks, conversation abandonment,
  "did the user rephrase?" — cheap, noisy, always-on.
- **Online LLM-as-judge:** sample X% of prod traffic, run an LLM judge on
  (input, output) for relevance/groundedness. Async, so it doesn't add user latency.
- **Guardrail checks inline:** cheap validators on every response (schema valid? contains
  a citation? PII leaked? refusal when it shouldn't?) — fast enough to run synchronously.
- **Drift detection:** track the *distribution* of inputs and outputs over time. A shift
  in input topics or output length/embedding often precedes a quality drop.

```python
# Async LLM-as-judge on sampled prod traffic (see ml_agentic_systems_guide.md Pillar 5)
def sample_and_judge(trace, sample_rate=0.05):
    if random.random() < sample_rate:
        score = llm_judge(trace.input, trace.output)   # relevance/accuracy/groundedness
        metrics.histogram("agent.quality_score", score["overall"])
        if score["overall"] < 3:
            alert_low_quality(trace)                    # capture for review
```

> This connects directly to **Systematic Evaluation (Pillar 5)** in the agentic guide:
> *offline* eval on golden datasets gates deploys; *online* eval on sampled traffic catches
> what golden sets missed. Same LLM-as-judge machinery, two deployment points.

**What to Say:**
> "In prod I don't have ground truth, so I layer signals: implicit feedback like thumbs and
> regenerates, inline guardrail checks on every response, and an async LLM-as-judge on a
> sample of traffic so it doesn't add latency. I also watch input/output drift, because a
> distribution shift usually shows up before the quality metrics crater."

---

# DEBUGGING WORKFLOW

Monitoring tells you *something* is wrong; debugging finds *what*. The workflow:

**1. Reproduce from the trace.** A good trace is a recording — replay the exact inputs.
Log a random seed and pin `temperature=0` in debug runs to make behavior deterministic.

**2. Walk the span tree to the failure.** Agent bugs almost always live in one of these:
```
Bad final answer
├── Was the RIGHT sub-agent/tool chosen?      → routing / planning bug
├── Were the tool ARGUMENTS correct?          → the classic silent failure
├── Did the tool RETURN good data?            → tool/API bug (not the LLM's fault)
├── Did retrieval fetch RELEVANT context?     → RAG bug (check retrieved chunks)
└── Given good inputs, was the reasoning bad? → prompt / model bug
```
Most "the LLM is dumb" reports are actually bad tool args or bad retrieved context —
**the trace tells you which**, so you fix the real cause instead of over-tuning the prompt.

**3. Common agent failure modes (name these — they're interview gold):**

| Failure mode | Symptom in the trace | Fix |
|--------------|----------------------|-----|
| Wrong tool selection | Router picked the wrong specialist | Better tool descriptions / routing prompt |
| Malformed tool args | Tool span errors or returns garbage | Structured outputs + validation (Pillar 2) |
| Context loss | Later spans "forget" earlier facts | Memory/summarization bug (Pillar 3) |
| Retrieval miss | Retrieved chunks irrelevant to query | Re-rank, chunking, query rewriting |
| Infinite loop | Span tree hits iteration cap | Circuit breaker fired (Pillar 4) — find *why* it looped |
| Cost spike | One trace with 10× the tokens | Runaway loop or context bloat |
| Hallucination | Output not grounded in tool/RAG results | Citations + groundedness check |

**4. Structured logging (so logs are queryable, not `print`):**
```python
import structlog
log = structlog.get_logger()

log.info("tool_call",
    trace_id=trace_id, agent="coder", tool="run_tests",
    args={"file": "x.py"}, status="error", error="timeout", latency_ms=412)
# → one JSON line you can filter: "show all timed-out run_tests calls for user 42 today"
```
Correlate everything by `trace_id` / `session_id` so a single ID pulls the whole story.

**What to Say:**
> "My first move is to pull the trace and replay it — I log seeds and run debug at
> temperature zero for determinism. Then I walk the span tree: was the right tool chosen,
> were the arguments right, did the tool return good data, was retrieval relevant? Nine
> times out of ten a 'dumb model' turns out to be a bad tool argument or an irrelevant
> retrieved chunk, and the trace points straight at it."

---

# THE TOOL LANDSCAPE

| Tool | What it is | Note |
|------|-----------|------|
| **OpenTelemetry** | Vendor-neutral tracing standard (+ GenAI conventions) | The layer everything else builds on — know this |
| **LangSmith** | LangChain's tracing/eval/monitoring platform | Auto-traces LangChain/LangGraph; strong eval integration |
| **Langfuse** | Open-source LLM observability | Framework-agnostic, decorator-based, self-hostable |
| **Arize Phoenix** | Open-source LLM tracing + eval | OTel-native; strong on drift & embeddings analysis |
| **W&B Weave** | Weights & Biases' LLM tracing | Good if already using W&B for experiments |
| **Google Cloud Trace / Vertex** | GCP-native tracing + monitoring | **ADK emits OTel → Cloud Trace natively — the Google answer** |
| **Prometheus + Grafana** | Metrics + dashboards | The standard for the metrics pillar |

**Interview move:** "I'd instrument against **OpenTelemetry** so I'm not locked into one
vendor, then send traces to whatever the customer already runs — LangSmith or Langfuse for
LLM-specific views, Cloud Trace on GCP. The instrumentation layer stays the same; only the
backend changes." That's a very FDE (meet-the-customer-where-they-are) answer.

---

# PUTTING IT ALL TOGETHER

**Sample System Design Answer — "How would you monitor a deployed support agent?"**

"I'd build observability in three layers:

**Tracing:** Every ticket produces one trace — a span tree covering the router, each
specialist sub-agent, retrieval, and tool calls, each span capturing the prompt,
completion, tokens, latency, and cost. Instrumented via OpenTelemetry so it's portable,
surfaced in LangSmith or Cloud Trace.

**Metrics:** Dashboards for p95/p99 latency, cost per ticket, tool-error rate, and
iteration-cap-hit rate. Alerts fire when cost-per-ticket or error rate crosses a threshold,
or when p95 latency regresses.

**Quality:** No ground truth in prod, so I layer signals — thumbs up/down, inline guardrail
checks (valid schema, has citation, no PII), and an async LLM-as-judge on 5% of traffic for
groundedness. Low-scoring traces get captured for human review and folded back into the
golden eval set.

**Debugging loop:** When quality dips, I pull the offending traces, replay them
deterministically, and walk the span tree to the failing step — usually a bad tool argument
or an irrelevant retrieval, not the model itself. The fix gets a regression test in the
golden set so it can't recur."

This ties observability back to the agentic guide's five pillars: traces expose failures in
**Orchestration (1)**, tool errors map to **Reliability (2)**, context loss to **Context
Optimization (3)**, loop-caps to **Guardrails (4)**, and online judging extends
**Evaluation (5)** into production.

---

# QUICK REFERENCE: WHAT TO INSTRUMENT

| Layer | Capture |
|-------|---------|
| **Every LLM call** | model, params, full prompt, full completion, prompt/completion tokens, latency, cost |
| **Every tool call** | tool name, input args, output, status, latency, error |
| **Every retrieval** | query, retrieved chunk IDs + scores, count |
| **Every request** | trace_id, session_id, user_id, total cost, total latency, final status |
| **Every sub-agent** | agent name, parent span, its own token/cost subtotal |

---

# QUESTIONS THEY MIGHT ASK

1. "A user says the agent gave a wrong answer. Walk me through debugging it."
   → Pull the trace, replay deterministically, walk the span tree (routing → tool args →
   tool output → retrieval → reasoning) to isolate the failing step.

2. "How do you monitor quality in production without ground truth?"
   → Implicit feedback + inline guardrail checks + async LLM-as-judge on sampled traffic +
   drift detection.

3. "What's the difference between logs, metrics, and traces?"
   → Logs = what happened at a step; metrics = aggregate health over time; traces = the
   causal path of one request. Correlate all three by trace_id.

4. "Why p95/p99 instead of average latency?"
   → LLM latency is long-tailed (retries, loops); averages hide the suffering tail.

5. "How do you keep tracing from leaking PII or blowing up storage?"
   → Redact/scrub prompt+completion fields, sample traces (keep 100% of errors, sample the
   rest), set retention limits, hash user IDs.

6. "How would you catch a prompt or model change that quietly hurt quality?"
   → Online LLM-as-judge on sampled traffic + drift detection on output distribution, gated
   by offline golden-set eval before the change ships.

---

# YOUR EXPERIENCE TO HIGHLIGHT

Based on your past projects (Prophet forecasting, clinical dashboards, data pipelines):

- "I've built monitoring dashboards before — for clinical/forecasting pipelines I tracked
  data freshness, error rates, and drift, and alerted on regressions. The same instincts
  apply to agents; the metrics just shift to tokens, cost, and groundedness."

- "I'm used to debugging multi-stage pipelines where a failure three steps back surfaces as
  a bad final output — tracing an agent's span tree is the same discipline of following the
  data to the real root cause."

- "I've worked with structured, queryable logging and correlation IDs, which is exactly
  what makes agent traces debuggable at scale."