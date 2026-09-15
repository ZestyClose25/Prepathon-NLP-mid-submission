# Agentic Customer 360 — Mid-Term Submission

## 1. Preliminary Research Log

The problem is an **ambient agentic Customer 360 system** that continuously consumes asynchronous customer events, maintains customer memory, detects meaningful changes, and decides whether a bounded intervention is required.

The main areas researched / being researched are:

| Topic | What we are taking from the research | Reference |
|---|---|---|
| Ambient agents | Background/event-driven agents rather than prompt-driven chatbots | https://www.langchain.com/blog/introducing-ambient-agents |
| Agent memory | Separation of working, episodic and semantic memory | https://www.mongodb.com/resources/basics/artificial-intelligence/agent-memory |
| Streaming RAG | Incremental vector updates and retrieval freshness | https://arxiv.org/pdf/2501.14101, https://medium.com/aingineer/a-complete-guide-to-implementing-streaming-rag-4e86bc0bb994 |
| HITL | Human approval for costly, sensitive or ambiguous actions | https://medium.com/@tahirbalarabe2/human-in-the-loop-agentic-systems-explained-db9805dbaa86 |
| Guardrails | Deterministic hard stops, PII protection and tool boundaries | https://bigid.com/blog/agentic-ai-guardrails/ |
| Explainability | Evidence-backed explanations generated as part of the decision | https://ieeexplore.ieee.org/stamp/stamp.jsp?arnumber=11620927 |


### LLM Chats

- https://chatgpt.com/share/6aa986ef-d534-83e9-a2f4-6175b777f50c — Ambient Agent
- https://chatgpt.com/share/6aa9873e-650c-83ee-9237-307dc5c1e953 — Memory & State Management
- https://chatgpt.com/share/6aa987fd-5108-83ee-8be2-9da4161efab0 — Understanding the PS better
 - https://youtu.be/O6ryuSpqdOw - Parallel workflows in LangGraph-CampusX
---

# 2. Preliminary System Architecture

## 2.1 High-Level Flow

```text
Live Event Stream
       ↓
Continuous Ingestion
       ↓
Validation + PII Protection + Deduplication
       ↓
Event-Time Processing
       ↓
Working Memory / Customer State Board
       ↓
Manager
       ↓
┌───────────────────────────────────────────┐
│          AMBIENT AGENTIC LAYER            │
│                                           │
│ Life-Event Agent                          │
│ Investment Analyst Agent                  │
│ KYC / Compliance Agent                    │
│ Sentiment / Support Agent                 │
│ User / Engagement Agent                   │
│                                           │
│        ↓                                  │
│ Synthesis / Correlation Agent             │
│        ↓                                  │
│ Debate / Arbitration if disagreement      │
│        ↓                                  │
│ Action Agent                              │
│        ↓                                  │
│ Critique Agent                            │
└────────────────────┬──────────────────────┘
                     ↓
            Deterministic Guardrails
                     ↓
                HITL Checkpoint
                     ↓
          Final Action / No Action
                     ↓
       Trace + Audit + Explanation
                     ↓
             Checkpoint JSON
```

## 2.2 Architecture Choice

The proposed system is a **hybrid ambient-agent architecture**.

The deterministic layer handles operations where correctness and safety should not depend on an LLM:

- continuous ingestion;
- event ordering;
- checkpointing and replay;
- state updates;
- PII protection;
- access control;
- deterministic hard stops;
- tool permissions;
- audit/trace storage.

The agentic layer handles interpretation and correlation:

- life-event inference;
- investment analysis;
- KYC/compliance reasoning;
- support/sentiment analysis;
- engagement analysis;
- synthesis;
- action selection;
- critique.

A **Manager** decides whether a meaningful trigger warrants invoking the agentic layer, avoiding unnecessary multi-agent execution for every raw event.

---

# 3. Memory Architecture

The system uses three memory types.

### Working / Short-Term Memory

Stores the customer's current state:

- recent events;
- financial state;
- transfers/payments;
- investments;
- credit;
- usage;
- support;
- risks;
- current life-event inferences;
- agent findings;
- current synthesis;
- current action.

This is maintained as a structured per-customer **state board**.

### Episodic Memory

Stores important customer-specific episodes:

```text
episode
→ situation
→ evidence
→ agent findings
→ action
→ outcome
→ effectiveness
```

Examples include financial stress, churn risk, fraud alerts, life events, credit needs and previous product opportunities.

### Semantic Memory

Stores shared knowledge:

- policies;
- eligibility rules;
- risk/compliance rules;
- product knowledge;
- agent guidelines;
- general behavior patterns.

**Storage approach:** episodic and semantic memory will use a **structured database**, while semantic/textual retrieval will additionally use a **vector database**.

When a new support ticket, CRM note or policy document arrives, it will be embedded and incrementally added/updated in the vector index.

---

# 4. Out-of-Order Event Handling

The dataset distinguishes between:

```text
event_time      = when the event actually happened
ingestion_time  = when the system received it
```

The system therefore reasons using **event time**.

For a late event:

```text
Late Event
    ↓
Find latest checkpoint ≤ event_time
    ↓
Rollback customer state
    ↓
Insert late event in correct temporal position
    ↓
Replay events from checkpoint
    ↓
Recompute state / triggers
    ↓
Continue from corrected state
```

This prevents late events from corrupting rolling aggregates or customer-state inference.

---

# 5. Multi-Agent Coordination

The specialist agents operate independently where possible and publish **structured findings** to the shared customer state board.

```text
Life Event ───────┐
Investment ───────┤
KYC ──────────────┤
Support ──────────┤ → Synthesis → Action → Critique
Engagement ───────┘
```

The system uses different coordination patterns for different situations:

- **Parallel/swarm:** independent signal analysis.
- **Explicit handoffs:** structured findings passed between stages.
- **Agent debate:** when agents produce conflicting interpretations.
- **Round robin:** iterative refinement where multiple perspectives are useful.
- **Critique-refiner:** Action Agent proposes; Critique Agent checks and forces refinement when needed.

### Disagreement example

```text
Engagement Agent:
"Usage is falling → churn risk."

Investment Agent:
"Investment activity is increasing → customer value/engagement is growing."

                    ↓

             Debate / Arbitration
                    ↓
        resolved interpretation
          OR escalation to HITL
```

The system will not silently choose one agent's output.

---

# 6. Safety, HITL & Explainability

### Guardrails

The system will enforce:

- **PII protection** — masking/redaction/tokenization where unnecessary.
- **Deterministic hard stops** — high-risk conditions cannot be overridden by an LLM.
- **Safe tool boundaries** — agents can draft actions without necessarily being allowed to execute them.
- **Data-layer access control** — agents can only query data they are authorized to access.

### HITL

A real approval checkpoint will occur before costly, sensitive or customer-facing actions.

The human can:

```text
approve
reject
modify
escalate
ask "why?"
```

Ambiguous or conflicting cases can be escalated even when the monetary value is low.

### Explainability

Every decision will retain:

```text
triggering events
→ relevant memories
→ agent findings
→ retrieved policies/documents
→ proposed action
→ critique
→ guardrail result
→ HITL decision
```

The explanation is generated from these recorded evidence references as part of the decision process, rather than reconstructed afterward.

---

# 7. Traceability

For each decision we intend to retain:

- event IDs;
- memory versions;
- retrieval queries/results;
- agent prompts and versions;
- structured agent outputs;
- tool calls and outputs;
- handoffs;
- debate/arbitration results;
- critique results;
- guardrail decisions;
- HITL approval/rejection/modification;
- final action.

This allows a reviewer to reconstruct **why the system acted and exactly what information it used**.

---

# 8. Proposed Approach

I propose a **hybrid ambient multi-agent system**:

```text
Continuous Event Stream
        ↓
Event-Time State Processing
        ↓
Working Memory / State Board
        ↓
Manager
        ↓
Specialized Agents
        ↓
Synthesis / Correlation
        ↓
Debate if disagreement
        ↓
Action Agent
        ↓
Critique Agent
        ↓
Deterministic Guardrails
        ↓
HITL
        ↓
Bounded Action / No Action
        ↓
Trace + Evidence + Audit
        ↓
Checkpoint JSON
```

The eight agents are:

1. Life-Event Agent
2. Investment Analyst Agent
3. KYC/Compliance Agent
4. Sentiment/Support Agent
5. User/Engagement Agent
6. Synthesis/Correlation Agent
7. Action Agent
8. Critique Agent

The system will maintain:

- **Working memory** for current customer state;
- **Episodic memory** for past situations, interventions and outcomes;
- **Semantic memory** for policies, rules and shared knowledge.

Specialist agents will operate in parallel where possible and publish structured findings. The Synthesis Agent will correlate these findings. If agents disagree, the system will explicitly invoke debate/arbitration rather than silently selecting one result. The Action Agent will propose only actions from the bounded evaluation set, and the Critique Agent will validate the proposal before it reaches the safety/HITL layer.

Every final decision will be accompanied by evidence references and a complete trace of the prompts, retrievals, tool calls, agent outputs, guardrails and HITL decision.

---

## Source Material

- Inter IIT Tech Meet 15.0 — **Natural Language Processing: Agentic Customer 360 — Proactive Intervention Desk**
- **Customer 360 Dataset Schema & Guidelines**

## Current Status

- Architecture: **Preliminary**
- Memory schemas: **Defined**
- Agent roster: **Defined**
- Coordination strategy: **Defined**
- Out-of-order strategy: **Defined**
- Safety/HITL model: **Preliminary**
- Implementation: **Yet to start**
