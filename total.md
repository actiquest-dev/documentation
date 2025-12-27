## Membria Reasoning Graph Approach - Full Architecture Overview

This document consolidates the Membria Smart Persistence Layer (Reasoning Graph) approach across all products (CE, SMB, EE). It covers the full system structure, data flows, GraphRAG, and knowledge gap removal via LoRA patching (SkillForge).

---

## 1) What Reasoning Graph Is

Smart Persistence Layer (Reasoning Graph) is the architectural layer that turns transient AI interaction into durable, structured intelligence. It does not store raw conversation as the primary artifact. Instead, it stores decisions, reasoning, assumptions, evidence, and outcomes over time.

Reasoning Graph sits between users and models. It makes intelligence compound instead of reset.

Core Reasoning Graph properties:
- Persistence across chats, tools, and models
- Explicit representation of decisions and reasoning
- Provenance and traceability by default
- Separation between raw input (noise) and structured memory (signal)

---

## 2) System Structure (End-to-End)

Membria is a two-layer system with an orchestration core:

1) Client Runtime Layer (CE and SMB, optional in EE)
- Chat, Decision Surface, Search
- Router / Orchestrator
- Local memory (SQLite + DuckDB)
- GraphRAG index and graph store
- Decision Black Box (DBB)
- Local SLM runtime (when available)

2) Backend Knowledge Layer (shared services)
- Escalation gateway
- LLM Council (multi-model)
- Knowledge Curator (verification + fusion)
- KCG: Knowledge Cache Graph
- Decentralized persistence (Peaq + Arweave or equivalent)

3) Intelligence Expansion Layer
- SkillForge (LoRA lifecycle)
- Domain adapters and expert overlays
- Evaluation and rollback pipeline

---

## 3) Core Data Objects

Reasoning Graph is defined by structured objects, not raw logs:

- Decision: what was decided, by whom, when, and why
- Assumption: explicit belief tied to context
- Evidence: source references and provenance chains
- Outcome: observed result or validation event
- ThoughtUnit: normalized fragment extracted from sources
- Knowledge Artifact: verified answer with citations

These objects are linked over time in the graph and exposed through Decision Surface and search.

---

## 4) Decision Black Box (DBB)

DBB is the capture engine. It detects decision moments and logs them as structured decision objects.

DBB characteristics:
- Continuous and non-intrusive
- Confidence scored
- Captures decisions, not conversations
- Append-only history (no overwrites)
- Correction loop to reduce false positives

DBB output feeds the Decision Surface and the Reasoning Graph memory graph.

---

## 5) Decision Surface (DS)

DS is the visibility layer built on DBB outputs. It shows what matters now:
- Open decisions
- Drift and contradictions
- Precedents and outcomes
- Risks and unresolved assumptions

DS is not a chat view. It is a reasoning and governance surface built on structured memory.

---

## 6) GraphRAG and Structured Memory

GraphRAG is the retrieval spine of Reasoning Graph.

Key mechanics:
- ThoughtUnits are extracted from sources (docs, chats, tools)
- Entities and relations are added to a temporal graph
- Vector embeddings enable similarity search
- Graph traversal provides explainable chain-of-evidence retrieval

GraphRAG enables grounded answers with citations and temporal context.

---

## 7) Escalation and Knowledge Cache

When local reasoning is insufficient, the system escalates:

1) Local memory + GraphRAG
2) Local cache or KCG lookup
3) Council escalation (multi-model synthesis)
4) Curator verification and fusion
5) Write-back to cache and graph

Outputs are cached with provenance. The next query reuses verified knowledge rather than re-generating it.

---

## 8) Knowledge Gap Removal via LoRA Patching (SkillForge)

Reasoning Graph does not rely on static models. It improves expertise over time via LoRA patches.

### What SkillForge Does
SkillForge is the LoRA lifecycle manager. It creates, evaluates, promotes, and rolls back LoRA adapters.

### How LoRAs Are Created
LoRA patches are generated from three sources:

1) Decision outcomes (DBB + results)
- Correct reasoning patterns are reinforced
- Incorrect assumptions are marked as failures

2) Escalation results (Council + Curator)
- Verified answers become training artifacts
- Repeated gaps are detected

3) Explicit domain imports (optional)
- Rules, guidelines, structured references

### How LoRAs Are Applied
The Router evaluates:
- Domain match
- Confidence impact
- Historical success rate

If a LoRA improves confidence, it is applied automatically. The base model remains unchanged.

### SkillForge Safety Rules
- LoRAs never rewrite historical decisions
- LoRA rollback is immediate and side-effect free
- Only verified artifacts are used for training
- If quality degrades, LoRA is removed

Result: knowledge gaps close over time, and escalation frequency drops without model drift.

---

## 9) Product-Specific Reasoning Graph Behavior

### Membria CE (Personal)
- Reasoning Graph is personal and private by default
- Local-first models and memory
- Optional decentralized backend for cache and reuse
- LoRAs are personal and user-controlled

### Membria for SMBs (SaaS)
- Reasoning Graph is shared at the workspace level
- Decisions are captured across team tools
- Team LoRAs scoped to workspace
- Escalation budgets are controlled by workspace policy

### Membria EE (Enterprise)
- Reasoning Graph is enterprise-grade and policy-governed
- Private knowledge domains with explicit boundaries
- Compliance-grade audit trails
- Escalation controlled by policy and approvals
- Deployment on-prem or private cloud

---

## 10) Governance, Security, and Auditability

Reasoning Graph treats reasoning as a governed artifact:
- Explicit access controls and tenancy boundaries
- Append-only audit trails
- Provenance and traceability by default
- Escalation logging and replay

This makes Reasoning Graph suitable for regulated environments and high-stakes decision workflows.

---

## 11) Operational Signals and Metrics

Reasoning Graph behavior is measured by:
- Cache hit rate
- Escalation rate
- Decision capture rate
- Citation coverage
- Latency budgets (local vs escalation)

These metrics tune confidence thresholds and routing policies.

---

## 12) Summary

Membria Reasoning Graph is a persistent reasoning layer built on:
- Decision capture (DBB)
- Visibility and governance (DS)
- GraphRAG memory
- Verified caching (KCG)
- SkillForge LoRA patching for knowledge gap removal

It transforms AI from a stateless responder into a compounding intelligence system that learns from real decisions.
