# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is a **content repository**, not a software project. It's a personal knowledge base documenting preparation for the **Google Forward Deployed Engineer (FDE) — Applied AI** interview. There is no build system, dependency manifest, test suite, or application to run. Work here almost always means editing Markdown or the notebook's content, not shipping code.

## Files and how they relate

The four content files map to the interview's components (see `README.md` for the full breakdown):

- **`python_dsa_patterns.ipynb`** — The centerpiece for the DSA round. A single large code cell organized into numbered `SECTION`s (Data Structures, Two Pointers, Sliding Window, Binary Search, Linked Lists, Trees, Graphs, and beyond). Each function is a reusable *template* meant to be memorized, annotated with time/space complexity and "Use when" comments. The README claims these patterns cover ~80% of DSA questions.
- **`dsa_problem_set.md`** — A 25-day study plan pairing the notebook's patterns with specific LeetCode problems in tables (columns: Problem / Difficulty / Pattern / LeetCode #). This is the *practice* companion to the notebook's *reference*.
- **`ml_agentic_systems_guide.md`** — System-design reference for the ML round, structured around "THE 5 PILLARS" of production agentic systems (Modular Orchestration, Deterministic Reliability, Context Optimization, Operational Guardrails, Systematic Evaluation).
- **`vibe_coding_mastery_guide.md`** — Guide for the "vibe coding" round, structured around four core competencies and a catalog of common AI-coding failure modes.

`README.md` is the top-level index tying these to interview stages; keep it in sync when adding or renaming files.

## Conventions to preserve when editing

- **Notebook**: keep the `# ===...===` banner + `SECTION N:` heading style. Every template carries a complexity note and a "Use when" comment — match that when adding functions. Type hints use modern builtins (`list[int]`, not `List[int]`).
- **Guides**: heavy use of `# AI generates:` / `# Should be:` contrast blocks (vibe guide) and runnable code snippets inside Markdown fences. Preserve these paired-example patterns.
- **Problem set**: keep the Markdown-table format and the LeetCode number column; it's how the plan cross-references problems.

## Verifying notebook code

There's no test harness. To sanity-check a template you add or change, extract and run the single code cell:

```bash
python3 -c "import json; print(''.join(json.load(open('python_dsa_patterns.ipynb'))['cells'][1]['source']))" | python3 -
```
