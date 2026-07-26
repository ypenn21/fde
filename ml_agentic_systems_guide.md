# ML DOMAIN INTERVIEW: PRODUCTION-GRADE AGENTIC SYSTEMS
## Google Forward Deployed Engineer - Applied AI
## Quick Reference Guide

---

# THE 5 PILLARS

## 1. Modular Orchestration

**Core Pattern:**
```
User Query → Router/Orchestrator → Specialized Agent → Tools → Response
```

**Key Points:**
- Central router classifies intent and delegates to specialist agents
- Each agent has SINGLE responsibility (separation of concerns)
- Agents are composable — can call other agents if needed
- Tools are deterministic functions agents can invoke (APIs, DBs, calculators)

**Architecture Example:**
```
┌─────────────────────────────────────────┐
│              Orchestrator               │
│  (classifies intent, routes, manages)   │
└──────────────┬──────────────────────────┘
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌───────┐ ┌────────┐ ┌─────────┐
│Search │ │ Code   │ │ Summary │
│ Agent │ │ Agent  │ │  Agent  │
└───┬───┘ └───┬────┘ └────┬────┘
    │         │           │
    ▼         ▼           ▼
┌───────┐ ┌────────┐ ┌─────────┐
│Google │ │Python  │ │  LLM    │
│  API  │ │Sandbox │ │  Call   │
└───────┘ └────────┘ └─────────┘
```

**Frameworks:**
- LangGraph — Graph-based agent orchestration
- CrewAI — Role-based multi-agent framework
- AutoGen — Microsoft's multi-agent conversation framework
- LangChain — General-purpose LLM orchestration

**What to Say:**
> "I'd design this with a central orchestrator that maintains conversation state and routes
> to specialized agents. Each agent has a focused capability — one for retrieval, one for
> code execution, one for summarization. This modularity makes testing easier and allows
> us to swap or upgrade individual agents without touching others."

---

## 2. Deterministic Reliability

**Problem:** LLMs are probabilistic — same input can give different outputs.

**Solutions:**

### Structured Outputs
```python
from pydantic import BaseModel

class ToolCall(BaseModel):
    tool_name: str
    parameters: dict
    confidence: float  # 0.0 - 1.0

class AgentResponse(BaseModel):
    thought: str
    action: ToolCall | None
    final_answer: str | None

# Force LLM to return valid schema
response = llm.generate(prompt, response_format=AgentResponse)
```

### Retry Logic
```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
def call_external_api(query: str):
    return api.search(query)
```

### Validation & Fallbacks
```python
def execute_tool(tool_call: ToolCall):
    try:
        result = tools[tool_call.tool_name].run(tool_call.parameters)
        if not validate_result(result):
            raise InvalidResultError()
        return result
    except ToolNotFoundError:
        return "I don't have access to that tool. Let me try another approach."
    except TimeoutError:
        return "The service is slow. Here's what I know from cached data..."
    except Exception as e:
        log_error(e)
        return "I encountered an issue. Could you rephrase your request?"
```

**What to Say:**
> "For reliability, I enforce strict schemas using Pydantic for all LLM outputs. If the
> response doesn't parse, we retry with a clarified prompt. For tool calls, we wrap everything
> in retry logic with exponential backoff, and we have graceful fallbacks when things fail.
> The system should never crash — it should degrade gracefully and communicate clearly."

---

## 3. Context Optimization

**Problem:** LLMs have limited context windows. Long conversations or large documents don't fit.

**Solutions:**

### Memory Compression
```python
class ConversationMemory:
    def __init__(self, max_tokens=4000):
        self.messages = []
        self.summary = ""
        self.max_tokens = max_tokens
    
    def add_message(self, role, content):
        self.messages.append({"role": role, "content": content})
        
        if self.token_count() > self.max_tokens:
            self._compress()
    
    def _compress(self):
        # Keep last 5 messages, summarize the rest
        old_messages = self.messages[:-5]
        self.summary = llm.summarize(self.summary + str(old_messages))
        self.messages = self.messages[-5:]
    
    def get_context(self):
        return f"Previous context: {self.summary}\n\nRecent messages: {self.messages}"
```

### RAG (Retrieval Augmented Generation)
```python
# Indexing
def index_documents(docs: list[str]):
    embeddings = embedding_model.encode(docs)
    vector_db.upsert(embeddings, docs)

# Retrieval
def retrieve_context(query: str, top_k=5):
    query_embedding = embedding_model.encode(query)
    results = vector_db.search(query_embedding, top_k=top_k)
    return results

# Generation
def generate_response(query: str):
    context = retrieve_context(query)
    prompt = f"Context: {context}\n\nQuestion: {query}"
    return llm.generate(prompt)
```

**Vector DBs to Know:**
- Pinecone — Managed, easy to scale
- Weaviate — Open source, GraphQL API
- Chroma — Lightweight, good for prototyping
- FAISS — Facebook's library, great for local/fast search
- Qdrant — Rust-based, performant

**What to Say:**
> "For context optimization, I use a tiered approach. Recent messages stay in full,
> older conversation gets progressively summarized. For knowledge bases, I use RAG —
> embed documents into a vector DB like Pinecone, then retrieve only the top-k most
> relevant chunks for each query. This keeps context focused and costs down."

---

## 4. Operational Guardrails

**Problem:** Agents can loop infinitely, take dangerous actions, or burn through API costs.

**Solutions:**

### Circuit Breakers
```python
class AgentRunner:
    def __init__(self, max_iterations=10, max_tokens=10000, timeout_seconds=60):
        self.max_iterations = max_iterations
        self.max_tokens = max_tokens
        self.timeout = timeout_seconds
    
    def run(self, task: str):
        iterations = 0
        tokens_used = 0
        start_time = time.time()
        
        while not task_complete:
            # Circuit breaker checks
            if iterations >= self.max_iterations:
                return "Reached maximum iterations. Here's my best answer so far..."
            
            if tokens_used >= self.max_tokens:
                return "Reached token budget. Summarizing what I found..."
            
            if time.time() - start_time > self.timeout:
                return "Taking too long. Here's a partial response..."
            
            # Execute step
            result, tokens = self.execute_step()
            tokens_used += tokens
            iterations += 1
```

### Human-in-the-Loop
```python
DANGEROUS_ACTIONS = ["delete", "purchase", "send_email", "execute_code", "modify_production"]

def execute_action(action: Action):
    if action.type in DANGEROUS_ACTIONS:
        # Pause and ask for confirmation
        approved = request_human_approval(
            action=action,
            reason="This action has side effects",
            timeout_minutes=5
        )
        if not approved:
            return "Action cancelled by user."
    
    return action.execute()
```

### Cost Controls
```python
class CostTracker:
    def __init__(self, budget_per_session=1.00):  # $1 max
        self.budget = budget_per_session
        self.spent = 0.0
    
    def track_call(self, model: str, input_tokens: int, output_tokens: int):
        cost = calculate_cost(model, input_tokens, output_tokens)
        self.spent += cost
        
        if self.spent >= self.budget:
            raise BudgetExceededError(f"Session budget of ${self.budget} exceeded")
```

**What to Say:**
> "Production agents need strict guardrails. I implement circuit breakers that cap iterations,
> tokens, and wall-clock time. Any destructive action — deletes, purchases, external sends —
> requires explicit human approval. We also track costs per session and per user to prevent
> runaway spending. The agent should be safe to leave running without supervision."

---

## 5. Systematic Evaluation

**Problem:** "Vibe checks" don't scale. Need automated, reproducible quality measurement.

**Solutions:**

### Golden Datasets
```python
GOLDEN_TESTS = [
    {
        "input": "What's the capital of France?",
        "expected": "Paris",
        "criteria": ["factually_correct", "concise"]
    },
    {
        "input": "Summarize this 10-page document",
        "expected_criteria": ["captures_main_points", "under_500_words", "no_hallucinations"],
        "reference_doc": "doc_123.pdf"
    },
]

def run_golden_tests():
    results = []
    for test in GOLDEN_TESTS:
        output = agent.run(test["input"])
        score = evaluate(output, test)
        results.append({"test": test["input"], "score": score, "output": output})
    return results
```

### LLM-as-Judge
```python
JUDGE_PROMPT = """
Rate the following response on these criteria (1-5 each):
1. Relevance: Does it answer the question?
2. Accuracy: Is the information correct?
3. Clarity: Is it easy to understand?
4. Completeness: Does it fully address the query?

Question: {question}
Response: {response}

Return JSON: {"relevance": X, "accuracy": X, "clarity": X, "completeness": X, "reasoning": "..."}
"""

def llm_judge(question: str, response: str) -> dict:
    judgment = gpt4.generate(JUDGE_PROMPT.format(question=question, response=response))
    return json.loads(judgment)
```

### Evaluation Pipeline
```python
def nightly_eval_pipeline():
    # 1. Run all golden tests
    results = run_golden_tests()
    
    # 2. Calculate aggregate metrics
    metrics = {
        "success_rate": sum(r["passed"] for r in results) / len(results),
        "avg_latency": mean(r["latency"] for r in results),
        "avg_cost": mean(r["cost"] for r in results),
        "avg_judge_score": mean(r["judge_score"] for r in results),
    }
    
    # 3. Compare to baseline
    regression = detect_regression(metrics, previous_metrics)
    
    # 4. Alert if needed
    if regression:
        send_alert(f"Regression detected: {regression}")
    
    # 5. Log to dashboard
    log_metrics(metrics)
```

**Key Metrics:**
- Task success rate (binary: did it complete the task?)
- Factual accuracy (via ground truth comparison or LLM judge)
- Latency (p50, p95, p99)
- Cost per query
- User satisfaction (thumbs up/down, explicit ratings)

**What to Say:**
> "We move beyond vibe checks with systematic evaluation. We maintain golden datasets —
> curated test cases with expected outputs. Every PR runs against these. For subjective
> quality, we use LLM-as-judge with GPT-4 scoring on relevance, accuracy, and completeness.
> Nightly pipelines track metrics over time and alert on regressions. This catches issues
> before users do."

---

# AGENTIC FRAMEWORK EXPERTISE

> Job requirement: *"Experience implementing multi-agent systems using frameworks
> (e.g., LangGraph, CrewAI, or Google's ADK) and complex patterns like ReAct,
> self-reflection, and hierarchical delegation."*

This section tests three things: (1) do you know the **frameworks** and when to reach
for each, (2) can you implement the **reasoning patterns** that make agents work, and
(3) have you wrestled with the **production realities** (delegation, state, failure).

---

## The Frameworks

Key mental model: frameworks differ mainly in **abstraction level** — how much control
they give you vs. how much they do for you.

### LangGraph — state machines for agents

Models an agent as a **graph**: nodes are functions (LLM calls, tools, logic), edges are
transitions, and a shared **State** object flows through it. Cycles are allowed — that's
what makes it agentic (loop "think → act → observe" until done).

- **Mental model:** a directed graph with cycles.
- **Why it wins:** explicit control over flow, built-in persistence/checkpointing (pause,
  resume, time-travel), native human-in-the-loop via interrupts.
- **Use when:** you need fine-grained control over control flow, durable state, or human
  approval steps.

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

class AgentState(TypedDict):
    messages: Annotated[list, operator.add]   # reducer: appends
    next_step: str

def agent_node(state): ...      # calls LLM, decides action
def tool_node(state): ...       # executes tool

def should_continue(state):     # conditional edge
    return "tools" if state["next_step"] == "act" else END

graph = StateGraph(AgentState)
graph.add_node("agent", agent_node)
graph.add_node("tools", tool_node)
graph.add_conditional_edges("agent", should_continue)
graph.add_edge("tools", "agent")            # cycle: back to reasoning
graph.set_entry_point("agent")
app = graph.compile(checkpointer=...)        # persistence
```

**Quotable:** "LangGraph makes the agent's control flow a first-class, inspectable object
instead of hiding it inside a prompt loop."

### CrewAI — role-based teams

Higher-level. Define **agents with roles** (goal, backstory, tools) and **tasks**, then a
**Crew** runs them sequentially or hierarchically.

- **Mental model:** a company org chart — each agent is a "coworker" with a job description.
- **Why it wins:** extremely fast to prototype; role/goal/backstory maps to how humans divide work.
- **Trade-off:** less control than LangGraph; the "magic" can obscure what's happening.

```python
from crewai import Agent, Task, Crew, Process

researcher = Agent(role="Researcher", goal="Find accurate data",
                   backstory="Expert analyst...", tools=[search_tool])
writer = Agent(role="Writer", goal="Write clear summaries", backstory="...")

crew = Crew(agents=[researcher, writer],
            tasks=[research_task, write_task],
            process=Process.hierarchical)    # a manager agent delegates
```

### Google ADK (Agent Development Kit) — your differentiator

Google's open-source framework (~2025, powers Agentspace / Vertex AI Agent Builder).
**Given this is a Google FDE role, know this one.**

- **Mental model:** code-first, hierarchical agents that compose. Distinguishes `LlmAgent`
  (reasoning) from **workflow agents** (`SequentialAgent`, `ParallelAgent`, `LoopAgent`)
  that give deterministic orchestration.
- **Selling points:** first-class multi-agent hierarchy, tight Gemini/Vertex integration,
  built-in evaluation, and the **A2A (Agent-to-Agent) protocol** for cross-agent
  communication.

```python
from google.adk.agents import LlmAgent, SequentialAgent

greeter = LlmAgent(name="greeter", model="gemini-2.0-flash", instruction="...")
executor = LlmAgent(name="executor", model="gemini-2.0-flash", tools=[...])
pipeline = SequentialAgent(name="root", sub_agents=[greeter, executor])
```

**Interview move:** "MCP standardizes how agents talk to *tools*; A2A standardizes how
agents talk to *each other*. ADK is built around both." Signals you're current.

### Framework Comparison

| Framework | Abstraction | Best for | Control |
|-----------|-------------|----------|---------|
| LangGraph | Low (graph) | Complex flows, state, HITL | High |
| CrewAI | High (roles) | Fast prototyping, clear team roles | Low |
| Google ADK | Medium | Google stack, hierarchical multi-agent | Medium-High |
| AutoGen | Medium | Conversational multi-agent | Medium |

---

## The Patterns

### ReAct — Reason + Act

The foundational agent loop: LLM alternates **Thought** (reasoning) and **Action** (tool
call), reads the **Observation**, repeats. Interleaves chain-of-thought with tool use.

```
Thought: I need the user's order status. I should query the order DB.
Action: get_order_status(order_id="12345")
Observation: {"status": "shipped", "eta": "2 days"}
Thought: I now have what I need.
Answer: Your order shipped and arrives in ~2 days.
```

- **Why it matters:** grounds reasoning in real data (reduces hallucination), makes
  decisions traceable.
- **Framework mapping:** it's literally the cycle in a LangGraph graph (agent → tool →
  agent). Modern implementations use native **structured/function-calling** rather than
  parsing free-text "Action:" strings — more reliable (ties to Pillar 2).

**"Doesn't the LLM do this automatically?" — the key distinction**

Sort of, but no — and this is exactly the trap that separates people who've *used*
ChatGPT from people who've *built* agents.

*What the LLM does on its own:* a modern tool-calling model has been trained to produce
reasoning ("I should look up the order status") **and** emit a structured tool call
(`get_order_status(order_id="12345")`) instead of hallucinating the answer. So the
**Thought** and the **Action-proposal** feel automatic. That part is real.

*What the LLM does NOT do:* it cannot actually run the tool. An LLM is a function
`tokens → tokens` — it has no ability to hit a database, call an API, or execute code.
When it "calls" a tool, it just outputs a *request* and then **stops**. Something outside
the model must:

1. **Parse** the tool-call request,
2. **Execute** the real function,
3. **Feed the result (Observation) back** into the context,
4. **Re-invoke** the model so it can continue.

That execute → observe → re-invoke loop, repeated until the model emits a final answer,
**is** ReAct — and it lives in your harness / the SDK's tool-runner / the framework
(a LangGraph cycle, CrewAI's executor), **not** inside the LLM.

```
┌─── the LLM does this ───┐   ┌──── YOU / the framework do this ────┐
  Thought + tool request  →   parse → run real tool → get result
         ▲                                                  │
         └──────────── re-invoke with Observation ──────────┘
```

- **Why this matters:** the loop is *yours* to control — that's where all the real
  engineering lives (retries, timeouts, the iteration cap so it doesn't loop forever,
  error handling when a tool fails). If ReAct were truly automatic inside the LLM, you'd
  have nowhere to put the guardrails from Pillars 2 and 4.
- **What actually changed:** early ReAct (2022) had you prompt the model to output literal
  `Thought:`/`Action:`/`Observation:` text and regex-parse it — brittle. Native
  function-calling replaced that string-parsing. The *pattern* is the same; the *plumbing*
  got reliable.

**What to Say:**
> "The model is trained to emit structured tool calls, but it can't execute anything — it
> just proposes. My runtime executes the tool, injects the observation back into context,
> and re-invokes the model. That execute-observe loop is ReAct, and it's where the real
> engineering lives: retries, timeouts, and the iteration cap so it doesn't loop forever."

### Self-Reflection — the agent critiques its own work

Agent generates output, a **critic step** evaluates it, corrections feed back. Name-drop:
**Reflexion** (verbal self-feedback stored in memory) and **Self-Refine** (iterative
refinement).

```
Generate → Critique ("Is this correct? What's missing?") → Revise → repeat until good enough
```

- **Concrete example:** code agent writes code → runs tests → reads failures → fixes. The
  test output *is* the reflection signal — the strongest kind (grounded, not self-graded).
- **Watch-out to raise unprompted:** reflection costs tokens/latency and can loop. Cap
  iterations (circuit breaker — Pillar 4). Reflection grounded in an *external* signal
  (tests, validators, tool results) beats pure LLM self-judgment, which is overconfident.

### Hierarchical Delegation — manager + workers

A top-level **orchestrator/manager agent** decomposes a task, delegates subtasks to
**specialist sub-agents**, then synthesizes results. This is Pillar 1 (Modular Orchestration).

```
                 ┌─────────────┐
                 │   Manager   │  decomposes & routes
                 └──────┬──────┘
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   Research Agent   Code Agent     Writer Agent
```

- **Why hierarchical > flat:** each sub-agent gets a clean, focused context (context
  isolation), you can test/swap them independently, and a specialist prompt beats one
  giant do-everything prompt.
- **Key design decisions:**
  - *Delegation as tool-calling:* manager treats sub-agents as callable tools
    (`call_research_agent(query)`).
  - *Context passing:* pass only what each child needs, not the whole history.
  - *Result aggregation:* manager reconciles/merges outputs (and handles disagreement).
  - *Failure isolation:* one sub-agent failing shouldn't crash the crew — degrade gracefully.

**Cost reality (worth quoting):** Anthropic's multi-agent research system beat a single
agent — but at ~15× the token cost. So use multi-agent only when the task truly
parallelizes or needs separated contexts; a single well-tooled ReAct agent is often
cheaper and sufficient. This cost-awareness is exactly the pragmatic FDE instinct they want.

---

## What to Say (30-second synthesis)

> "I'd start with the simplest thing that works — usually a single **ReAct** agent with
> good tools, because multi-agent adds real token and latency cost. When the task genuinely
> decomposes, I move to **hierarchical delegation**: an orchestrator routing to specialist
> sub-agents, each with an isolated context. I'd build it in **LangGraph** when I need
> explicit control over state and human-in-the-loop, or **CrewAI** for fast role-based
> prototyping — and **ADK** on the Google stack, where A2A and Gemini integration are
> first-class. For quality, I add **self-reflection** loops grounded in real signals like
> test results, always bounded by iteration limits. Then it's the production pillars:
> structured outputs, retries, circuit breakers, and eval against golden datasets."

---

## Questions They Might Ask

1. "When would you NOT use a multi-agent system?"
   → Sequential/single-context tasks; multi-agent multiplies cost and adds coordination
   failure modes. Start single-agent, split only when justified.

2. "How do agents pass information in a hierarchy?"
   → Shared state object (LangGraph) or sub-agent-as-tool with explicit I/O; pass minimal
   necessary context to keep each window focused.

3. "How do you stop a self-reflection loop from running forever?"
   → Iteration cap + "good enough" threshold + ground the critique in an external signal
   so it converges.

4. "LangGraph vs CrewAI?"
   → Control vs. speed. Graph/state-machine control vs. role-based rapid prototyping.

5. "What's A2A / MCP?"
   → MCP = agent↔tool standard; A2A = agent↔agent standard. ADK is built around both.

---

# PUTTING IT ALL TOGETHER

**Sample System Design Answer:**

"If I were building a customer support agent for an e-commerce platform, here's how I'd approach it:

**Orchestration:** Central router classifies tickets into categories — order status, returns, product questions, complaints. Each routes to a specialized agent with access to relevant tools (order DB, return policy docs, product catalog).

**Reliability:** All agent outputs use Pydantic schemas. Tool calls have retry logic with exponential backoff. If the order API is down, we gracefully tell the user we're checking and will follow up.

**Context:** Customer history is stored in a vector DB. When a ticket comes in, we retrieve relevant past interactions and order history. Long conversations get summarized every 10 turns.

**Guardrails:** Refund approvals over $100 require human review. Agents can't access other customers' data. We cap each session at 20 iterations and $0.50 in API costs.

**Evaluation:** We have 500 golden tickets with labeled correct resolutions. Nightly evals measure resolution accuracy, customer satisfaction scores, and average handle time. Any regression >3% blocks deployment."

---

# QUICK REFERENCE: TOOLS & FRAMEWORKS

| Category | Tools |
|----------|-------|
| Orchestration | LangGraph, LangChain, CrewAI, AutoGen, Google ADK |
| Agent Protocols | MCP (agent↔tool), A2A (agent↔agent) |
| Vector DBs | Pinecone, Weaviate, Chroma, FAISS, Qdrant |
| Structured Output | Pydantic, Instructor, OpenAI function calling |
| Observability | LangSmith, Weights & Biases, Arize |
| Evaluation | RAGAS, DeepEval, custom LLM-as-judge |

---

# QUESTIONS THEY MIGHT ASK

1. "How would you handle an agent that keeps looping?"
   → Circuit breakers, iteration limits, loop detection

2. "How do you ensure the agent doesn't hallucinate?"
   → RAG with citations, confidence scores, fact-checking tools

3. "How would you evaluate if your agent is improving?"
   → Golden datasets, LLM-as-judge, A/B testing, user feedback metrics

4. "What happens when an external API fails?"
   → Retry with backoff, fallback to cached data, graceful error messages

5. "How do you manage costs at scale?"
   → Token budgets per session, caching frequent queries, smaller models for simple tasks

---

# YOUR EXPERIENCE TO HIGHLIGHT

Based on your past projects (Prophet forecasting, clinical dashboards, data pipelines):

- "In my forecasting pipeline, I built similar reliability patterns — retry logic, 
   validation checks, graceful degradation when data sources were unavailable."

- "I understand production systems. I've built pipelines that need to run reliably
   without human intervention, with proper error handling and monitoring."

- "I've worked with structured data and schemas extensively, which translates directly
   to enforcing structured outputs from LLMs."

Connect your experience to these pillars. You have more relevant background than you might think.
