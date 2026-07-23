# FDE MASTER STUDY PLAN

## Google Forward Deployed Engineer — Applied AI

**Profile this plan is built for:** ~4–5 week runway · ~1–2 hrs/day · weakest on **DSA** and **Agentic/ML design**.

> This is the *integrated* plan across all five interview rounds. It deliberately does **not** cover the full 80-problem DSA set in `dsa_problem_set.md` — at 1–2 hrs/day that set is unrealistic. Instead it uses a **high-yield DSA core (~35 problems)** and reinvests the saved time into your weak rounds. Use `dsa_problem_set.md` as the *overflow* list if you get ahead.

---

## Budgeting the time

- **~1.5 hrs/day × 6 days/week × 5 weeks ≈ 45 hrs total.** One rest day per week is built in — protect it.
- **Weekly split (by leverage, given your weak spots):**
  - DSA — ~45% (daily reps; skill decays without them)
  - Agentic/ML design — ~30% (second weak spot; high signal for FDE)
  - Vibe coding — ~10% (overlaps heavily with agentic prep)
  - Googleyness/behavioral — ~10% (front-loaded story-building, then maintenance)
  - Recruiter/"why factor" — ~5% (front-loaded in Week 1, then done)

**Daily shape (90 min):** 10 min warm-up (redo a solved problem) → 60 min primary block → 20 min review + say the solution out loud.

---

## The high-yield DSA core (~35 problems)

Two canonical problems per pattern beats ten. Master these cold; they generalize. LeetCode #s in parentheses.

| Pattern | Problems (LeetCode #) |
|---|---|
| Arrays & Hashing | Two Sum (1), Group Anagrams (49), Top K Frequent (347), Product of Array Except Self (238) |
| Two Pointers | Valid Palindrome (125), 3Sum (15), Container With Most Water (11) |
| Sliding Window | Best Time to Buy/Sell (121), Longest Substring w/o Repeating (3), Longest Repeating Char Replacement (424), Minimum Window Substring (76) |
| Stack | Valid Parentheses (20), Min Stack (155), Daily Temperatures (739) |
| Binary Search | Binary Search (704), Search in Rotated Sorted Array (33), Koko Eating Bananas (875) |
| Linked Lists | Reverse Linked List (206), Merge Two Sorted Lists (21), Linked List Cycle (141), Remove Nth From End (19) |
| Trees | Max Depth (104), Invert (226), Level Order/BFS (102), Validate BST (98), Lowest Common Ancestor (236), Kth Smallest in BST (230) |
| Heaps | Kth Largest (215), Find Median from Data Stream (295) |
| Graphs | Number of Islands (200), Clone Graph (133), Course Schedule (207) |
| Tries | Implement Trie (208) |
| DP | Climbing Stairs (70), House Robber (198), Coin Change (322), Longest Increasing Subsequence (300), Word Break (139) |

**Rule:** if stuck >20 min → hints; >30 min → study the solution, then re-implement from scratch the next day. Every solved problem gets re-derived once, 2–3 days later, from memory.

---

## Week-by-week

### Week 1 — Foundations + get the "why factor" done
*Theme: array/string patterns, and clear the recruiter-story overhead early so it stops nagging you.*

- **DSA (daily, ~50 min):** Arrays & Hashing → Two Pointers → Sliding Window (rows 1–3 of the core table). ~1 problem/day.
- **Recruiter prep (2 short sessions):** Write and rehearse answers from `README.md` — "What interests you in this role?", "Why FDE / why Google?", and prepare **1 agentic-model story + 1 classical-ML story**. Draft 4–5 questions to ask the recruiter. Finish this by end of Week 1.
- **Notebook:** type the Data Structures + Two Pointers + Sliding Window templates from memory once.

### Week 2 — DSA core patterns + start Agentic pillars
*Theme: your two weak rounds now run in parallel.*

- **DSA (daily, ~40 min):** Stack → Binary Search → Linked Lists (core table).
- **Agentic/ML (2 sessions, ~40 min each):** Read `ml_agentic_systems_guide.md` Pillars **1 (Modular Orchestration)** and **2 (Deterministic Reliability)**. For each, be able to whiteboard the pattern *and* state the failure it prevents.
- **Notebook:** Binary Search + Linked List templates from memory.

### Week 3 — Trees/Graphs + finish Agentic pillars + first mock
*Theme: the heaviest DSA topics land with the heaviest system-design content.*

- **DSA (daily, ~45 min):** Trees → Heaps → Graphs (core table).
- **Agentic/ML (2 sessions):** Pillars **3 (Context Optimization / RAG)**, **4 (Guardrails)**, **5 (Evaluation)**. Prepare a 2-minute verbal answer to "How would you design a production agent for \<customer use case\>?" using the "Putting it all together" section.
- **Tokenomics (1 session, ~40 min):** Read `tokenomics_cost_optimization.md` — memorize the **cost ladder** (right-size → cache → cut tokens → batch → semantic-cache → budgets) and rehearse the **60-second cost answer**. "How do you keep inference costs from skyrocketing at scale?" is a near-guaranteed FDE follow-up; know the three traps (tiktoken, output-token cost, semantic-cache staleness) cold.
- **Mock #1 (end of week):** 45 min, out loud, no IDE — 1 medium DSA problem + talk through 1 agentic design prompt.

### Week 4 — DP, Vibe coding, Googleyness
*Theme: close the remaining gaps.*

- **DSA (daily, ~40 min):** Tries (light) → Dynamic Programming core. DP is where most people are shaky — spend the extra reps here.
- **Vibe coding (2 sessions):** Read `vibe_coding_mastery_guide.md` — the **4 competencies** and the **15 failure modes** (know the Security + Reliability categories cold). Do one hands-on rep: take an ambiguous prompt, decompose it, generate code with an AI tool, then *validate/critique* it out loud.
- **Googleyness (2 sessions):** Build **6–8 STAR stories** (see the framework below) covering: leadership/ownership, conflict, failure, ambiguity, customer impact, and a technical trade-off. Reuse the recruiter stories from Week 1.

### Week 5 — Polish, mocks, rest
- **Day 1–2:** Re-do every DSA problem you flagged as hard, from scratch. Re-type all notebook templates from memory.
- **Day 3:** **Mock #2** — fresh problems (Pramp / interviewing.io / a friend). Include one system-design + one behavioral question.
- **Day 4:** Light review — read guides + templates, rehearse STAR stories aloud, prep interviewer questions.
- **Day 5:** **Rest.** Logistics check (video/audio, quiet space), good sleep.

---

## Googleyness / STAR quick-framework

*(No dedicated file exists yet — capture your stories here or in a new `googleyness_notes.md`.)*

Structure every story as **S**ituation → **T**ask → **A**ction → **R**esult, ~2 min, with a quantified result. Cover these prompts:

- A time you **led without authority** / drove a decision.
- A **conflict** with a teammate or stakeholder and how you resolved it.
- A **failure** and what you changed afterward.
- Working under **ambiguity** with incomplete requirements (very FDE-relevant).
- A **customer/user-facing** win where you translated a fuzzy need into a solution.
- A hard **technical trade-off** you made and why.

FDE tilt: emphasize customer empathy, shipping under constraints, and communicating with non-engineers — that's the "forward deployed" part.

---

## Daily checklist

- [ ] Warm-up: re-derive one solved problem (10 min)
- [ ] Primary block for today's topic (60 min)
- [ ] Say today's solution / design out loud as if interviewing (10 min)
- [ ] Log what tripped you up (1 line)
- [ ] End of week: one mock or one weak-area re-do

---

## If you fall behind

Cut in this order: (1) Tries + hardest DP, (2) vibe-coding hands-on rep (reading is enough), (3) drop from 2 mocks to 1 (keep Mock #2). **Never** cut the daily DSA warm-up or the STAR stories — those are cheap and high-signal.
