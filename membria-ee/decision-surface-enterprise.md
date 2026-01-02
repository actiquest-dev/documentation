> Role-based decision visibility across teams and organization

## Overview

In Membria Enterprise, the Decision Surface operates as a centralized backend service. All decision capture, storage, and rendering happens server-side. Users access the Decision Surface through a thin web client that displays signals filtered by their role and scope.

Unlike the SMB version with local client-side capture, Enterprise deployments connect directly to corporate communication systems (Slack, Microsoft Teams, email) and process all decision signals on-premise.

---

## Architecture

**Data Sources** (Slack, Teams, Email, Confluence, Jira)

**Backend (on-premise)**  
- DBB Engine — decision capture from all sources  
- Decision Records — immutable append-only store  
- GraphRAG + Ontology — relationships & context  
- Access Layer — role-based filtering  
- DS API — serves signals by scope

**Web UI (thin client)**  
- Decision Surface — renders API response by user role

---

## Roles and permissions

Enterprise Decision Surface supports three role levels with different visibility scopes.

### Individual Contributor (Middle / Senior)

Primary workspace for personal decision tracking.

**Sees:**
- Own decision signals (open loops, commitments, pending decisions)
- Own assumption drift on topics they discussed
- Linked decisions from colleagues (only where they are mentioned or dependent)
- Full reasoning and context for own decisions

**Actions:**
- Confirm, edit, or reject captured decisions
- Add outcomes to own decisions
- Mark decisions as superseded

---

### Director / Team Lead

Aggregated view of team decision health.

**Sees everything IC sees, plus:**
- Team-level signal counts (total open loops, commitments, pending)
- Team members listed by signal load (who has most unresolved items)
- Dependency graph across team (whose decision blocks whom)
- Assumption drift across team topics
- Alerts when team member patterns indicate risk

**Does not see:**
- Reasoning details of other team members' decisions
- Personal notes or context from others' captures

**Actions:**
- View team aggregates and distributions
- Drill down to decision titles and outcomes (not reasoning)
- Receive escalations from team members
- Reassign or redistribute decision ownership

---

### C-level / Owner

Strategic view of organizational decision state.

**Sees everything Director sees, plus:**
- Company-wide signal aggregates
- Cross-team dependencies and blockers
- Strategic topic drift (pricing, hiring, infrastructure)
- Historical patterns (decision velocity, resolution rates)
- Revenue/outcome correlation when linked to business metrics

**Does not see (by default):**
- Individual reasoning details
- Personal decision context

**Exception access:**
- Drill-down into details enabled by trigger (KPI decline, critical escalation)
- Full audit trail available for compliance requirements

**Actions:**
- Monitor organizational decision health
- Receive critical escalations only
- Access audit trail when needed
- Configure alert thresholds

---

## Scope selector

The Decision Surface UI includes a scope toggle that filters the view:

```
Scope: [My] [Team] [Company]
```

Available options depend on user role:

| Role | Available scopes |
|------|------------------|
| IC (Middle/Senior) | My |
| Director | My, Team |
| C-level | My, Team, Company |

---

## Signal visibility matrix

| Signal | IC sees | Director sees | C-level sees |
|--------|---------|---------------|--------------|
| Open Loops | Own (count + details) | Team (count + owners) | Company (count + by team) |
| Commitments | Own (full context) | Team (list, no reasoning) | Company (count + overdue) |
| Decisions Pending | Own (full context) | Team (oldest, blockers) | Cross-team blockers |
| Assumption Drift | Own topics | Team topics | Strategic topics |
| Escalations | From own decisions | From team | Critical only |
| Reasoning | ✅ Own only | ❌ | ❌ (except by trigger) |
| Outcomes | ✅ Own | ✅ Team | ✅ Company |

---

## Data model

Each decision record includes ownership and visibility metadata:

```json
{
  "id": "dec_a1b2c3",
  "owner_id": "user_123",
  "team_id": "team_456",
  "company_id": "company_789",

  "signal_type": "decision_pending",
  "title": "Migrate to new payment provider",
  "created_at": "2026-01-02T10:30:00Z",

  "reasoning": "Current provider has 2.9% fees...",
  "alternatives": ["Stay with current", "Build in-house"],
  "confidence": 75,
  "assumptions": ["Volume stays above 10K/mo"],

  "outcome": null,
  "outcome_at": null,

  "visibility": {
    "reasoning": ["owner"],
    "alternatives": ["owner"],
    "outcome": ["owner", "team", "company"],
    "signal": ["owner", "team", "company"]
  }
}
```

---

## API endpoints

### Get signals

```
GET /api/ds/signals
Authorization: Bearer {token}
Query params:
  - scope: my | team | company
  - period: 7d | 14d | 30d | 90d
  - signal_type: open_loops | commitments | pending | drift | all
```

Response is filtered by user role and requested scope.

### Get signal details

```
GET /api/ds/signals/{id}
Authorization: Bearer {token}
```

Returns full details if user is owner, limited details otherwise.

### Update outcome

```
POST /api/ds/signals/{id}/outcome
Authorization: Bearer {token}
Body: { "outcome": "Completed migration, fees now 2.1%", "status": "resolved" }
```

Only owner can update. Outcome is appended, original record unchanged.

---

## Aggregation rules

When viewing Team or Company scope, signals are aggregated:

**Counts:**
- Total signals by type
- Distribution by owner (Director) or by team (C-level)

**Alerts:**
- Pattern detection: "3 missed commitments in 14 days"
- Bottleneck detection: "5 decisions blocked by same dependency"
- Drift detection: "Vocabulary changed on topic X across 4 team members"

**No aggregation of:**
- Reasoning content
- Personal notes
- Confidence scores (remain individual)

---

## Trigger-based access

C-level users can access detailed reasoning only when a trigger condition is met:

| Trigger | Access granted |
|---------|----------------|
| KPI decline > 2 weeks | Drill-down to related decisions |
| Critical escalation | Full context of escalated decision |
| Compliance audit request | Full audit trail with timestamps |
| Direct request + approval | Time-limited access to specific record |

All access is logged for audit purposes.

---

## Privacy principles

1. **Reasoning is private by default** — only the decision owner sees their own reasoning and alternatives
2. **Outcomes are team-visible** — results can be shared without exposing thought process
3. **Signals are company-visible** — everyone knows a decision exists, not how it was made
4. **Aggregation over exposure** — higher roles see patterns, not individual details
5. **Trigger-based exceptions** — detailed access requires justification and is logged

---

## AI output principles

Decision Surface uses LLMs to analyze patterns and generate insights. To ensure trust and accuracy, all AI outputs follow strict epistemic guardrails.

### Evidence-based outputs

LLM never makes claims without traceable evidence.

**Required format:**
```
[Conclusion] → [Evidence: quotes, dates, counts]
```

**Example — correct:**
> "@ivanov raised concerns about timeline 3 times in the last 14 days. 2 commitments were missed. Pattern suggests risk on Project X."

**Example — incorrect:**
> "Ivanov is sabotaging the project."

### Prohibited language

LLM outputs must not include:
- Psychological inference ("believes", "feels", "is afraid")
- Personality typing ("is a blocker", "toxic", "passive-aggressive")
- Unsupported predictions ("will fail", "won't deliver")

### Required hedging

All pattern descriptions must include context:
- "Based on last 14 days..."
- "In conversations captured by DBB..."
- "Pattern suggests..." (not "This means...")

---

## Tensions detection

Decision Surface can identify implicit disagreements within teams — situations where team members act on different assumptions without explicit discussion.

### How it works

The system analyzes decision records across team members for:

1. **Topic bimodality** — same topic, divergent approaches
2. **Assumption drift** — vocabulary changes over time without new decision
3. **Contradictory commitments** — overlapping decisions with different directions

### Detection method

Using seeded k-means clustering on decision embeddings:
- Bimodality coefficient > 0.555 indicates split in team approach
- Alert generated for Director when detected
- No individual attribution — only topic and pattern shown

### Example alert

```
⚠️ Tension detected: "Q2 pricing strategy"

Team shows bimodal pattern:
- Cluster A (3 members): language suggests premium positioning
- Cluster B (2 members): language suggests volume discount approach

No explicit decision captured resolving this difference.
Recommendation: surface for team discussion.
```

### Privacy preservation

Tensions detection shows:
- Topic where tension exists
- Cluster sizes (not names)
- Pattern description

Does not show:
- Individual reasoning
- Who is in which cluster (unless escalated)

---

## Reproducibility and audit

All algorithmic operations use deterministic methods for compliance and audit requirements.

### Seeded randomness

- All clustering operations use fixed seeds derived from input data
- Identical input always produces identical output
- Seeds are logged with each analysis run

### Audit trail

Every signal, alert, and pattern includes:
- Timestamp of detection
- Algorithm version
- Input hash
- Seed used
- Output hash

This allows auditors to verify that any past analysis can be reproduced exactly.

### Versioning

- LLM prompts are version-controlled
- Algorithm changes are logged with effective dates
- Historical analyses reference the version used at the time

---

## Integration with enterprise systems

Decision Surface connects to existing tools without changing workflows:

| Source | What DBB captures |
|--------|-------------------|
| Slack / Teams | Commitment phrases, decision announcements, blockers |
| Email | Approvals, sign-offs, escalations |
| Jira / Linear | Status changes linked to decisions |
| Confluence / Notion | Decision documents, assumption changes |
| Calendar | Decision meetings, review deadlines |

Users continue working in their normal tools. Decision Surface shows what matters without requiring new habits.

---

## Summary

Enterprise Decision Surface provides:

- **For ICs:** Personal decision workspace with full context
- **For Directors:** Team health monitoring without micromanagement
- **For C-level:** Strategic visibility with privacy-respecting drill-down

Same UI, same signals, different scope — filtered by role, aggregated by level, private by default.
