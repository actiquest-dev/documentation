# Decision Intelligence Layer - Implementation Roadmap

## Purpose and Scope

This document serves as a technical blueprint for implementing the **Decision Intelligence Layer** on top of Membria's **Reasoning Graph**. It translates high-level conceptual frameworks — such as Value of Information (VoI), POMDP belief tracking, and causal sequencing — into concrete software engineering phases, data structures, and algorithms.

This roadmap is designed for engineers, architects, and technical leadership. It provides a transparent, phase-by-phase path to evolve Membria from a knowledge persistence engine into an active decision optimization system.

---

## Overview

Five major components are built incrementally, each providing value on its own.

---

## Phase 1: Foundation (Weeks 1-6)

### 1.1 Extended Decision Object Schema

**What:** Add 6 new fields to Decision records in Graph Memory

**Changes:**
```json
{
  "decision_id": "dec_001",
  "statement": "Use Vendor X",
  "confidence": 0.75,

  // NEW FIELDS
  "value_of_information": {
    "score": 85000.0,      // Float64 in dollars
    "basis": "Revenue impact if vendor wrong",
    "computed_at": "2025-01-21T10:00:00Z",
    "sources": ["dec_001_impact_edge"]
  },
  "belief_state": {
    "state": "consensus",
    "prob_true_consensus": 0.60,
    "prob_surface_only": 0.30,
    "prob_misunderstanding": 0.10,
    "confidence": 0.60,
    "last_updated": "2025-01-21T10:05:00Z",
    "signal_history": ["sig_001", "sig_002"]
  },
  "dependencies": {
    "blocks": ["dec_002", "dec_003"],  // which decisions this blocks
    "blocked_by": [],                  // which decisions block this
    "related": ["dec_004"]              // dependencies (not blocking)
  },
  "predicted_outcome": null,            // Set before execution
  "actual_outcome": null,               // Set after execution
  "calibration_delta": null             // predicted vs actual
}
```

**Implementation:**
- Update DBB Decision data model (Julia struct)
- Add FalkorDB schema updates
- Backward compatible: new fields optional, default to null

**Tech Stack:**
- Language: Julia (for RG components).
- Graph DB: **FalkorDB** (Primary knowledge backbone for low-latency Cypher queries).
- Persistence: Redis-based memory storage via FalkorDB.

**Testing:**
- Unit tests: VoI field can be null, computed, updated.
- Integration tests: Decision creation with and without new fields in FalkorDB.
- Migration tests: Existing nodes in FalkorDB remain accessible.

**Deliverable:** Extended schema, database migrations, test suite

---

### 1.2 Graph Relationships for Dependencies

**What:** Add BLOCKS and DEPENDS_ON relationships to Knowledge Graph

**Changes:**
```
Decision D1 --[BLOCKS]--> Decision D2
Decision D5 --[BLOCKS]--> Decision D1
Decision D7 --[DEPENDS_ON]--> Decision D5
```

**Implementation:**
- Update GraphRAG node/relationship definitions
- Add traversal functions: find_blocking(decision_id), find_blocked(decision_id)
- Implement cycle detection (decide A blocks B blocks A = invalid)

**Tech Stack:**
- Graph DB: **FalkorDB** (utilizing GraphBLAS for high-speed temporal traversals).
- Query Language: **Cypher** (optimized for Membria dependency chains).

**Testing:**
- Cycle detection tests
- Traversal tests (find transitive dependencies)
- Performance tests (query speed for large decision graphs)

**Deliverable:** Graph schema, relationship traversal API, validation

---

### 1.3 Signal Extraction for Belief State

**What:** Extract decision-related signals from communications for belief state tracking

**Changes to DBB:**
```
Signal types extracted from conversations:
- EXPLICIT_AGREEMENT: "+1", "agreed", "approved"
- EXPLICIT_DISAGREEMENT: "-1", "objection", "disagree"
- SILENCE: person present but said nothing
- SKEPTICAL_QUESTION: "how do we know X?", "what about risks?"
- ACTION_TAKEN: "I'll start hiring", "contacted vendor"
- ACTION_DEFERRED: decision made but no action after 48h
- FOLLOW_UP_DISCUSSION: topic reopened after decision

**[NEW] Phase 1.3: Signal Extraction Quality**
- Confidence Score (0.0-1.0)
- Cultural Modifiers (High/Low Context)
- Behavioral Confirmation (Action must follow Silence)

```

**Implementation:**
- LLM-based signal classifier (local or small specialized model)
- Classify each message/comment relating to a decision
- Store signal as Evidence link to Decision
- Enable low-confidence detection (e.g., > 2 skeptical questions = low consensus)

**Tech Stack:**
- Language: Julia + Python (for LLM inference)
- Model: Small specialized classifier or 7B-13B model
- Storage: Signal evidence records linked to Decision

**Testing:**
- Signal extraction accuracy tests (precision/recall on test corpus)
- False positive tests (ensure silence is not confused with agreement)
- Integration tests: signals update decision belief state

**Deliverable:** Signal extraction service, test corpus, initial calibration

---

## Phase 2: Value of Information (Weeks 7-12)

### 2.1 VoI Scoring Algorithm

**What:** Compute Value of Information for each decision

**Inputs needed:**
1. Decision statement + alternatives
2. Estimated impact if wrong: P(wrong) × cost_of_being_wrong
3. Cost to gather more info: time + resources
4. Downstream effects on other decisions

**Algorithm:**
```
VoI(D) = P(current_confidence_sufficient) × E[cost_if_wrong]
       - (P(better_decision_with_info) - P(current_confidence)) × E[cost_if_wrong]
       - cost_of_gathering_info

Simplified:
VoI ≈ (E[cost if wrong without more info] - E[cost if wrong with more info])
    - cost_to_gather_info
```

**Implementation approach:**
1. **Interview stakeholders** (not ML): for each decision type, ask:
   - "What's the cost if we choose wrong?"
   - "How much better decision with more info?"
   - "How much effort to gather more info?"
   - Store as domain heuristics

2. **Encode as rules/templates:**
   ```
   Decision type: "Vendor selection"
   Cost model: "annual_cost × years_commitment"
   Info benefit: "vendor_references_reduce_uncertainty_by_30%"
   Info cost: "2 weeks, 1 person, $5k"

   Result: vendor decisions typically VoI = $50k - $200k
   ```

3. **Compute per decision:**
   - Extract decision type from statement
   - Apply domain heuristic
   - Adjust for actual context (contract length, team size, etc.)

**Tech Stack:**
- Language: Julia
- Storage: VoI domain templates in DB or config file
- Computation: cached field on Decision (compute on save, refresh on demand)

**Testing:**
- Sanity tests: high-cost decisions have high VoI
- Calibration tests: compare VoI estimate to actual impact (long-term)
- Comparison tests: rank decisions by VoI, manually verify ranking makes sense

**Deliverable:** VoI computation service, domain templates, caching layer

### [NEW] Phase 2.3: VoI Calibration Loop
- Track `VoI_predicted` vs `VoI_actual`
- Trigger "Wisdom of Crowds" (3+ estimates) if variance > 30%


---

### 2.2 VoI Integration with Decision Surface

**What:** Surface shows decisions ranked by VoI

**Changes to DS UI:**
```
Open Loops (sorted by VoI DESC):
1. Pricing strategy - VoI: $180k ⭐⭐⭐
   Cost if wrong: $180k/year revenue loss
   Recommendation: PRIORITIZE

2. Tech stack - VoI: $95k ⭐⭐
   Cost if wrong: refactor costs $95k
   Recommendation: PRIORITIZE

3. Meeting format - VoI: $2k ⭐
   Cost if wrong: $2k productivity hit
   Recommendation: CAN DEFER
```

**Implementation:**
- Add VoI score to Decision query results (from Graph Memory)
- DS API returns decisions sorted by value_of_information.score DESC
- UI widget shows VoI with visual indicator (stars or gradient)
- Click to see VoI breakdown (cost components, basis)

**Tech Stack:**
- Backend: GraphRAG query optimization (index VoI for fast sorting)
- Frontend: React component for VoI display + breakdown modal
- API: extend `/decisions` endpoint to support sort_by=voi

**Testing:**
- API tests: VoI sorting works correctly
- UI tests: VoI displayed correctly, click shows breakdown
- Performance tests: VoI sorting doesn't slow down large decision lists

**Deliverable:** DS integration, UI components, API changes

---

## Phase 3: Belief State Tracking (Weeks 13-18)

### [NEW] Phase 3.0: Prior Calibration Period
- 4 weeks silent observation
- Compute team-specific priors
- Learn cultural baseline (e.g., disagreement frequency)


### 3.1 POMDP Belief State Tracking

**What:** Maintain probability distribution over decision consensus using Partially Observable Markov Decision Process logic.


**States:**
- `true_consensus`: team genuinely agreed
- `surface_agreement`: people said "yes" but don't believe it
- `misunderstanding`: people understood differently

**Bayesian filtering:**
```
Prior: P(state | evidence_history)

Likelihood model (learned or hand-crafted):
- P(explicit_agreement | true_consensus) = 0.95
- P(explicit_agreement | surface_agreement) = 0.90
- P(explicit_agreement | misunderstanding) = 0.40

- P(silence | true_consensus) = 0.05
- P(silence | surface_agreement) = 0.30
- P(silence | misunderstanding) = 0.40

- P(skeptical_question | true_consensus) = 0.10
- P(skeptical_question | surface_agreement) = 0.40
- P(skeptical_question | misunderstanding) = 0.80

- P(action_taken | true_consensus) = 0.90
- P(action_taken | surface_agreement) = 0.30
- P(action_taken | misunderstanding) = 0.20

Posterior: P(state | new_evidence) ∝ P(new_evidence | state) × P(state)
```

**Implementation:**
1. Initialize belief uniform: P(state) = 1/3 for each state

2. For each signal, update:
   ```
   beliefs = beliefs ∝ signal_likelihood × beliefs  // Bayes rule
   beliefs = beliefs / sum(beliefs)                 // normalize
   ```

3. Store belief state on Decision.belief_state

4. Confidence = max(beliefs) - if < 0.60, alert

**Tech Stack:**
- Language: Julia (for numerical computation)
- Numerics: Distributions.jl for Beta/Categorical distributions
- Storage: belief state object as JSON in DB

**Testing:**
- Likelihood model calibration: does likelihood match reality?
- Update tests: given signals S1, S2, S3, belief updates correctly
- Convergence tests: does belief stabilize with more evidence?
- False positive tests: ensure no alert if consensus actually high

**Deliverable:** Bayesian filtering implementation, likelihood model, tests

---

### 3.2 Signal-to-Belief Integration

**What:** Wire signal extraction → belief updates

**Pipeline:**
1. Signal extracted by 1.3 (EXPLICIT_AGREEMENT, SILENCE, etc.)
2. Signal stored as Evidence linked to Decision
3. On signal arrival, trigger belief update
4. If confidence drops below threshold, add alert to DS

**Implementation:**
- Hook in DBB: after signal extraction, call update_belief_state(decision, signal)
- Real-time updates: use pub/sub or change event listener
- Batch updates: periodic refresh of belief state from signal history

**Tech Stack:**
- Message queue: Kafka or similar for event streaming
- OR: database triggers/callbacks for simpler deployments

**Testing:**
- Integration tests: signal extraction → belief update → DS alert
- End-to-end test: add skeptical question, observe belief confidence drop

**Deliverable:** Signal-to-belief pipeline, alerting logic

---

### 3.3 Decision Surface - Hidden Disagreement Alerts

**What:** DS shows alert when decision appears made but consensus confidence is low

**UI changes:**
```
Decision: "Hire 5 engineers in Q2"
Status: ✓ APPROVED

Consensus Confidence: 40% ⚠️
  - True consensus: 40%
  - Surface agreement only: 35%
  - Misunderstanding: 25%

Signal history:
  ✓ 3 people agreed
  ⓘ 1 person silent
  ❓ 2 skeptical questions
  ✗ 0 action items started

Recommendation: Resurface for explicit discussion before execution
```

**Implementation:**
- Add belief_state to Decision query results
- DS shows confidence as visual indicator (green/yellow/red)
- Click to see signal breakdown
- Alert triggers notification to decision owner if confidence < threshold

**Testing:**
- UI displays confidence correctly
- Alert triggers on low confidence
- Hidden disagreement is discovered in post-mortem (calibration)

**Deliverable:** DS belief state visualization, alerting, decision owner notifications

---

## Phase 4: Assumption Calibration & Learning (Weeks 19-26)

### 4.1 Assumption Outcome Tracking

**What:** Link outcomes back to assumptions, update belief distributions

**Schema changes:**
```
Assumption {
  assumption_id,
  statement: "Vendor X is reliable",
  confidence: 0.75,  // initial
  prior_belief: Beta(α, β),

  outcomes: [
    {
      decision_id: "dec_001",
      outcome: "success",    // success | failure | mixed
      recorded_at: timestamp,
      impact: "Vendor delivered on time"
    }
  ]
}
```

**Implementation:**
1. When decision outcome is recorded, update related assumptions
   ```
   decision = Decision(statement="Use Vendor X")
   assumptions = decision.assumptions

   for assumption in assumptions:
     if "Vendor X" in assumption.statement:
       update_assumption_belief(assumption, outcome)
   ```

2. Update Beta distribution:
   ```
   Outcome("success"): Beta(α, β) → Beta(α+1, β)
   Outcome("failure"): Beta(α, β) → Beta(α, β+1)
   ```

3. Recompute confidence from Beta distribution:
   ```
   confidence = E[Beta(α, β)] = α / (α + β)
   ```

**Tech Stack:**
- Language: Julia + Distributions.jl
- Storage: outcomes array on Assumption node

**Testing:**
- Beta update correctness
- Confidence computation accuracy
- Integration: decision outcome → assumption belief update

**Deliverable:** Assumption outcome tracking, Beta belief updates

**[NEW] Stratified Updates:** Weight outcomes by recency and confounder match


---

### 4.2 Calibration Analytics

**What:** Aggregate calibration data across decisions, detect patterns

**Analytics queries:**
```
Team Calibration Report (Product team):
- 20 decisions over 6 months
- Predicted confidence: [75%, 82%, 70%, 88%, ...] (avg 76%)
- Actual success rate: 78%
- Calibration gap: -2% (slightly overconfident)
- Well-calibrated, slight bias in risky decisions

Decision Type Calibration (Hiring):
- 15 hiring decisions
- Predicted confidence: avg 82%
- Actual success rate: 52% (hires lasted > 6 months)
- Calibration gap: +30% (significantly overconfident)
- Pattern: team overestimates hiring success

Individual Calibration (CTO):
- 10 infrastructure decisions
- Predicted: avg 80%, Actual: 85%
- Well-calibrated, slightly underconfident
- Pattern: CTO's estimates are trustworthy in infrastructure
```

**Implementation:**
1. Query: all decisions with outcomes in time range
   ```
   SELECT decision WHERE status="executed" AND actual_outcome NOT NULL
   ```

2. Aggregate predictions vs actuals:
   - Group by: team, decision_type, individual, date_range
   - Compute: avg(predicted_confidence), success_rate(actual_outcome)
   - Compute: calibration_gap = predicted - actual

3. Detect patterns:
   - If consistent bias (e.g., > 20% gap), flag as systematic bias
   - Generate LoRA candidate: "hiring_confidence_reducer"

**Tech Stack:**
- Language: Julia or Python
- DB queries: GraphRAG or Cypher
- Analysis: DataFrames.jl or pandas

**Testing:**
- Correctness of aggregation queries
- Accuracy of calibration gap calculation
- Pattern detection heuristics

**Deliverable:** Calibration analytics service, dashboards

---

### 4.3 LoRA Candidate Generation

**What:** Automatic proposal of LoRA adapters based on calibration patterns

**Process:**
1. Detect systematic bias:
   ```
   IF (calibration_gap > 20% AND decision_count > 10)
   THEN create_LoRA_candidate(
     domain: decision_type,
     issue: "overconfident" | "underconfident",
     magnitude: calibration_gap,
     sample_size: decision_count
   )
   ```

2. Generate LoRA Justification Record:
   ```
   LoRA Candidate: hiring_decision_confidence_reducer

   Reason: Systematic overconfidence in hiring decisions
   - Sample: 15 hiring decisions over 6 months
   - Predicted avg confidence: 82%
   - Actual success rate: 52%
   - Gap: +30%

   Expected effect:
   - Apply 0.70× multiplier to hiring confidence scores
   - Expected new gap: ~10% (acceptable range)
   - Risk: May undersell good hiring prospects

   Recommendation: Approve for canary rollout
   ```

3. Present to human approval:
   - DS shows: "LoRA candidate ready for review: hiring_confidence_reducer"
   - Reviewer can: Approve for canary → Promote → Monitor rollback

**Tech Stack:**
- Language: Julia
- Workflow: async job (cron or event-triggered)
- Storage: LoRA candidates in DB

**Testing:**
- Candidate generation correctness
- Justification record completeness
- Workflow integration with LoRA approval process

**Deliverable:** LoRA candidate generation, justification records

---

## Phase 5: Decision Sequencing Optimizer (Weeks 27-32)

### 5.1 Dependency Graph & Blocking Analysis

**What:** Analyze decision dependencies, compute critical path

**Data structure:**
```
Decision D1 ← statement: "Tech stack"
  blocks: [D2, D3, D4]

Decision D5 ← statement: "Budget"
  blocks: [D1, D6]

Decision D7 ← statement: "Vendor"
  blocks: [D8]

Dependency graph:
D5 → D1 → [D2, D3, D4]
D5 → D6
D7 → D8
```

**Algorithm - Critical Path Method (CPM):**
1. Topological sort: find valid orderings
2. Compute earliest start time (EST) for each decision:
   ```
   EST(D) = max(EST(predecessors)) + duration(predecessor)
   ```
3. Compute latest start time (LST):
   ```
   LST(D) = min(LST(successors)) - duration(D)
   ```
4. Critical path: decisions where EST = LST (no slack)
5. Total project time: EST(last_decision)

**Implementation:**
```julia
function critical_path(decisions::Vector{Decision})
  # Build dependency graph
  graph = build_dependency_graph(decisions)

  # Topological sort
  ordered = topological_sort(graph)

  # Compute EST
  est = Dict()
  for d in ordered
    est[d.id] = maximum(est[p.id] for p in predecessors(d))
  end

  # Compute LST
  lst = Dict()
  for d in reverse(ordered)
    lst[d.id] = minimum(lst[s.id] for s in successors(d))
  end

  # Critical path
  critical = [d for d in decisions if est[d.id] == lst[d.id]]
  total_time = est[last_decision]

  return (critical_path=critical, total_time=total_time, est=est, lst=lst)
end
```

**Tech Stack:**
- Language: Julia
- Graph library: Graphs.jl
- Storage: decision dependency graph from Knowledge Graph

**Testing:**
- CPM correctness on test graphs
- Critical path identification accuracy
- Performance on large decision graphs

**Deliverable:** CPM algorithm, implementation, tests

---

### 5.2 Uncertainty-Aware Sequence Recommendation (Priority Queue)

**What:** Rank valid next decisions using a Priority Queue based on VoI, Uncertainty, and Blocking power.

**Why:** MCTS was found to be overkill for typical decision graph sizes (10-50 nodes). A deterministic Heuristic Priority Queue provides 90% of the benefit with O(N log N) complexity.

**Score Function:**
```
Score(D) = (VoI(D) * Urgency(D)) / Duration(D) * UncertaintyMultiplier(D)
```
Where:
- `Urgency`: 1.0 if on critical path, < 1.0 otherwise.
- `UncertaintyMultiplier`: Higher if resolving D reduces global entropy (e.g. unlocks many branches).

**Algorithm:**
```julia
function recommend_sequence(decisions, dependencies):
  # 1. Identify "Ready Set" (unblocked decisions)
  ready_queue = PriorityQueue()
  
  # 2. Score and Enqueue
  for d in decisions:
    if is_unblocked(d):
      score = calculate_heuristic_score(d)
      enqueue(ready_queue, d, score)
      
  recommendation = []
  
  # 3. Greedy Selection loop
  while !isempty(ready_queue):
    best = dequeue!(ready_queue)
    push!(recommendation, best)
    
    # Simulate completion
    for successor in successors(best):
      remove_dependency(successor, best)
      if is_unblocked(successor):
         score = calculate_heuristic_score(successor)
         enqueue(ready_queue, successor, score)
         
  return recommendation
end
```

**Implementation:**
1. Implement scoring heuristics (VoI, Critical Path flag).
2. Maintain live Priority Queue of blocked/unblocked states.
3. Return simple ordered list.

**Tech Stack:**
- Language: Julia (DataSructures.jl).
- Input: Graph Memory nodes.

**Testing:**
- Compare "Naive FIFO" vs "Priority Queue" total value realized over time.
- Verify critical path items get higher priority.

**Deliverable:** Priority Queue Sequencing Engine.

---

### 5.3 Sequencing Integration with Decision Surface

**What:** DS shows recommended decision sequence and expected speedup

**UI changes:**
```
Decision Sequence Optimizer

Current state:
- 5 pending decisions
- 3 blocked decisions
- Estimated completion time: 6 weeks

Naive approach (FIFO):
→ D1: Tech stack (1 week)
  → D5: Budget (blocks D1, currently waiting)
  ✗ [Blocked, wasting 1 week]

Recommended sequence (Priority Queue):
→ D5: Budget (1 week) [Score: 95 - Critical Path]
→ D1: Tech stack (1 week) [Score: 88 - High VoI]
→ [D2, D3, D4 in parallel: 2 weeks]
→ D7: Vendor (1 week)

Expected completion: 3 weeks (50% faster)

Critical path: D5 → D1 → [D2-D4]
Non-critical: D7 (can start anytime)
```

**Implementation:**
- GraphRAG query: fetch all pending/blocked decisions + dependencies
- Compute recommendation: call sequencing_recommender()
- Format response: sequence with explanation
- DS API: `/decisions/recommend-sequence`
- UI: timeline view of recommended sequence

**Testing:**
- E2E test: query decisions, compute sequence, display in UI
- Correctness test: compare recommendation to manual analysis
- Performance test: does it complete in < 1 second for 20 decisions?

**Deliverable:** DS integration, API, UI timeline component

---

## Phase 6: Integration & Validation (Weeks 33-38)

### 6.1 End-to-End Testing

**Scenario 1: Complete Decision Lifecycle**
1. User makes decision in Slack: "Use Vendor X (75% confidence)"
2. DBB captures, creates Decision record
3. VoI computed: $85k (decision is high-impact)
4. Belief state initialized: 60% consensus (some people silent)
5. DS shows: "Vendor X selected (VoI=$85k, consensus=60%)"
6. 6 months later: vendor succeeds
7. Outcome recorded, assumption belief updated
8. Calibration report: CTO well-calibrated in vendor decisions
9. No LoRA needed (already accurate)

**Scenario 2: Low Consensus Alert**
1. Decision: "Launch product in Q2"
2. DBB captures: "approved" → confidence 0.75
3. Signals arrive:
   - "+1 from PM"
   - Silence from VP Eng
   - "What about tech debt?" from engineer
4. Belief updated: 40% true consensus
5. DS alert: "Low confidence in decision. Recommend discussion."
6. Team discusses, explicitly resolves concerns
7. Follow-up message: "Approved, tech debt handled"
8. Belief updated: 85% consensus
9. Proceed without risk

**Scenario 3: Sequencing Optimization**
1. 5 decisions pending: Budget, Tech stack, Vendor, Hiring, Data strategy
2. Dependencies: Budget blocks Tech stack and Vendor; Tech stack blocks Hiring
3. Current order would take 6 weeks (lots of blocking)
4. Sequencing recommends: Budget → Tech stack → Hiring (+ Data in parallel)
5. Expected time: 3 weeks
6. Team follows recommendation
7. Tracks actual time, validates recommendation

**Testing:**
- Run scenarios in test environment
- Measure accuracy of recommendations
- Gather qualitative feedback (did it help?)

---

### 6.2 Pilot Program

**Target:** 1-2 early adopter teams (internal or friendly customer)

**Scope:**
- Team size: 5-20 people
- Decision frequency: 1-5 decisions/week
- Duration: 2-3 months

**Metrics:**
- VoI ranking: do high-VoI decisions get resolved faster?
- Belief state: how many hidden disagreements detected?
- Calibration: are predictions accurate?
- Sequencing: actual time vs recommendation?
- Adoption: % of decisions with intelligence signals enabled

**Deliverable:** Pilot report, lessons learned, customer testimonial

---

## [NEW] Phase 7: Debiasing Intervention Engine (Weeks 39-44)

**(Powered by "System 2" Resonance Detection)**

### 7.1 Intervention Library
- Anchoring → "Consider 3 alternatives"
- Confirmation → "Devil's Advocate" prompt
- Overconfidence → Pre-mortem exercise
- Sunk Cost → "Fresh Start" reframe

### 7.2 Intervention Delivery
- Triggered by **Resonance Score > 0.6** (see Architecture Appendix D)
- Inline prompts in Decision Surface
- Required fields before decision confirmation

### 7.3 Intervention Calibration
- A/B test interventions
- Disable ineffective triggers per team

---

### 6.3 Production Hardening

**What:** Ensure reliability, performance, security for production

**Checklist:**
- [ ] Error handling: graceful degradation if VoI calc fails
- [ ] Performance: caching of VoI, belief state updates
- [ ] Monitoring: track signal extraction accuracy, belief stability
- [ ] Backups: daily backup of decision intelligence data
- [ ] Audit: all computations logged and reproducible
- [ ] Privacy: belief state and signals private to team
- [ ] Rollback: disable feature without data loss

**Testing:**
- Load test: 1000 decisions with continuous signal updates
- Failover test: service restart, data integrity check
- Security test: unauthorized user cannot see other team's decisions

**Deliverable:** Production deployment checklist, runbooks

---

## Architecture Summary

```
┌────────────────────────────────────────────────────┐
│ Reasoning Graph - Extended                         │
├────────────────────────────────────────────────────┤
│                                                    │
│ Phase 1: Foundation                               │
│  ├─ Extended Decision schema                      │
│  ├─ Graph dependencies (BLOCKS relationship)      │
│  └─ Signal extraction for belief state            │
│                                                    │
│ Phase 2: Value of Information                     │
│  ├─ VoI computation                               │
│  ├─ Domain heuristics/templates                   │
│  └─ DS integration (ranking)                      │
│                                                    │
│ Phase 3: Belief State Tracking                    │
│  ├─ Bayesian filtering (Discrete State Filter)    │
│  ├─ Signal-to-belief pipeline                     │
│  └─ Hidden disagreement alerts                    │
│                                                    │
│ Phase 4: Assumption Calibration                   │
│  ├─ Outcome tracking                              │
│  ├─ Calibration analytics                         │
│  └─ LoRA candidate generation                     │
│                                                    │
│ Phase 5: Decision Sequencing                      │
│  ├─ Critical Path Method (CPM)                    │
│  ├─ Priority Queue sequence optimizer             │
│  └─ DS timeline visualization                     │
│                                                    │
│ Phase 6: Integration & Validation                 │
│  ├─ E2E testing                                   │
│  ├─ Pilot program                                 │
│  └─ Production hardening                          │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## Timeline & Resources

| Phase | Duration | Team Size | Key Deliverables |
|-------|----------|-----------|------------------|
| 1 | 6 weeks | 2 engineers | Schema, migrations, signals |
| 2 | 6 weeks | 1 engineer + domain experts | VoI algorithm, DS integration |
| 3 | 6 weeks | 1-2 engineers | Bayesian filtering, alerts |
| 4 | 8 weeks | 1 engineer + ML engineer | Calibration analytics, LoRA pipeline |
| 5 | 6 weeks | 1 engineer + algorithms expert | CPM, MCTS, visualization |
| 6 | 6 weeks | 2 engineers + QA | Testing, pilot, hardening |
| **Total** | **~38 weeks** | **2-4 engineers avg** | **Complete Decision Intelligence** |

---

## Success Metrics

| Metric | Target | Method |
|--------|--------|--------|
| High-VoI decisions resolved 2x faster | 2.0x speedup | A/B test old vs new DS |
| Hidden disagreements detected before execution | >80% of potential issues | Post-mortem analysis |
| Team calibration improves | 10-15% improvement in prediction accuracy | 6-month lookback |
| Decision sequence recommendations accepted | >70% adoption | User surveys + tracking |
| System reliability | >99.5% uptime | Monitoring dashboards |

---

## Risk Mitigation

| Risk | Mitigation |
|------|-----------|
| VoI computation too complex or inaccurate | Start with simple heuristics, validate with domain experts |
| Belief state updates too frequent/noisy | Use debouncing, minimum signal confidence threshold |
| Sequencing recommendations not followed | Provide strong evidence, pilot with skeptical teams |
| LoRA overfitting to team biases | Require eval dataset, canary rollout, instant rollback |
| Privacy concerns with decision tracking | Encrypt at rest, RBAC, audit logs, transparent data use |

---

## Dependencies & Blockers

- **Reasoning Graph must be deployed**: decision capture working, graph memory functional
- **GraphRAG queries stable**: decision retrieval, dependency queries fast enough
- **Domain expertise available**: for VoI templates, calibration heuristics
- **LoRA infrastructure in place**: for Phase 4 (LoRA training, canary, rollback)

---

## Post-Launch Roadmap

After full implementation:

1. **Continuous learning**: calibration improves over time as more data accumulates
2. **Domain specialization**: separate LoRA adapters for different decision types
3. **Team-specific models**: personalize confidence scoring per team/domain
4. **Predictive alerts**: predict risky decisions before execution
5. **Explainable AI**: explain why specific decision flagged/prioritized
6. **Integration with workflows**: trigger actions on high-VoI decisions (escalate, notify, etc.)
