---
marp: true
paginate: true
theme: default
title: Semantic Operator for RHOAI
description: Evaluate SO for governed analytical metrics
style: |
  section { font-size: 28px; }
  h1 { font-size: 40px; }
  h2 { font-size: 32px; }
  table { font-size: 22px; }
  code { font-size: 20px; }
  footer { font-size: 14px; }
---

<!-- _paginate: false -->

# Semantic Operator
## Evaluate a governed metrics plane for RHOAI

**Status:** Evaluate and discuss — **not** a ship / adopt decision  
**ADR:** ODH-ADR-SO-0001

---

# What this discussion is

We are evaluating whether RHOAI should add a **governed analytical metrics** plane for open dimensional business-number questions:

- certified measures + dimensions  
- deterministic SQL (LLM does not invent the calculation)  
- agent access without LLM-written SQL  

**Candidate:** Kubeflow Semantic Operator + Apache Ossie  

This records the gap vs Registry/Feast, how the market keeps **metrics and features together but separate**, when Feast alone is enough vs when SO is worth evaluating, options (including defer), and boundaries if we proceed.

---

# Agenda

1. Problem
2. Metrics + features together (market — how Feast and SO relate)
3. When Feast alone is enough
4. When SO is needed (closed vs open)
5. Bottom line + example questions
6. How the products differ / who calls which
7. What SO is / where it fits
8. Recommendation + discussion

---

# The problem

Agents ask:

> “How much **subscription revenue by region** last quarter?”  
> “How many **paying customers** do we have?”

**Common path today:** LLM discovers tables → writes SQL → runs OK → **wrong business meaning**

| Issue | Effect |
|-------|--------|
| “Revenue” ambiguous | All sales vs only recurring subscriptions |
| Joins invented | Same customer counted twice |
| Different phrasing | Different SQL, different numbers |

Agreed numbers once → LLM picks **names**, not formulas.

**Registry** = find assets · **Feast** = feature context · neither = open “same metric, many slices”

---

# Metrics + features together (market)

**Main frame for Feast vs SO** — how the market uses them (AtScale + Feast, UC Metric Views + feature store):

| Metrics plane | Features plane |
|---------------|----------------|
| Trusted company numbers; open slices for BI / agents | Entity features for train / serve (online, PIT) |
| “Subscription revenue” by region / month | `customer_id → revenue_30d` for churn scoring |
| AtScale, UC Metric Views, **SO (candidate)** | **Feast**, Tecton |

**Together:** same business meaning for humans **and** models — not two “revenues.”  
**Not the same product:** compose — do not merge. Semantic layer ≠ feature catalog.

We are **not** evaluating AtScale-style codegen tooling. We **are** using this pattern to explain how SO would sit **beside** Feast.

---

# “Isn’t Feast already the truth for the LLM?”

**Often yes.** Feature engineering = *make the numbers the app/model needs available under a trusted name.*

**Example — Feast alone is enough**

You already served `revenue` or `revenue_by_region_last_quarter`.  
User asks “what’s revenue?” → LLM **retrieves** that feature. **No SO.**

Same for: scoring customer 42, training, RAG, chatbots with a **fixed list** (`revenue_30d`, `is_paying_customer`).

**Do not evaluate SO to fix those paths.** Approach **E** (defer) is valid here.

---

# Where SO becomes necessary

### Closed vs open (the real distinction)

| Feast (closed) | SO (open) |
|----------------|-----------|
| You shipped `revenue_by_region_last_quarter` (entity can be `region`). Chatbot fetches that one value. | Finance defines **subscription revenue** once. User asks by region, then by product × month, then only enterprise — **same definition, no new FeatureView each time**. |

**“Can’t Feast use region as the entity?”** Yes — still a **closed** ask. Next slice needs another FeatureView.

### Who owns “revenue”

| Feature eng | Analytics / Finance |
|-------------|---------------------|
| `txn_sum_7d` for the fraud model | “Subscription revenue” the company reports |

Same word, two contracts. Every BI grain in Feast = choice, not free.

---

# Bottom line

| If… | Then… |
|-----|-------|
| AI only shows a **fixed menu** feature eng already serves | **Feast is enough** — SO adds little |
| Agents need trusted metrics they can **re-slice** without baking every breakdown as features | **SO matters** |

Next: examples → product differences → who calls which → what SO is → recommendation.

---

# Example questions — which plane?

| Better on **Feast** | Better on **Semantic Operator** |
|---------------------|----------------------------------|
| Features for customer 42 | **Subscription revenue by region** last quarter |
| PIT join for 10k drivers at noon | How many **paying customers** (Finance’s rules)? |
| Training feature rows last week | **Customers who left**, by product and month |
| Similar claim notes (vectors) | Booked sales vs recognized revenue by segment |
| ODFV fraud features on this request | Same **subscription revenue**, also by month — no new FeatureView |
| Fixed chatbot metrics already in Feast | *(Stay on Feast)* |

**ODFV** = still Feast (this request / these entities) — not open “by region” from Finance’s formula.

---

# How Feast and SO differ as products

Same pattern (named defs, versions, authz, optional MCP). Different product — used **together** when both need the same number:

* **Feast:** “What do I know about **this customer**?” / “This **fixed list** of features.”
* **SO:** “The **company number we agreed on**, broken down **how I ask now**.”
* **Together:** same certified meaning → open asks **and/or** entity features — AtScale-style *relationship*, not one merged engine.

| | Feast | SO |
|-|-------|-----|
| You ask for | Features for entities / fixed menu | Measure + how to break it down |
| How the number is made | Stored or ODFV for that request | Planner → **one SQL** from locked formula |
| Serve | Offline + online (+ vectors) | SQL engine (Trino / StarRocks) |
| Does not do | Open re-slice without new FeatureView | Online features, PIT, vectors |

---

# Who calls which — and how they stay aligned

SO is **not** a Feast replacement. Dual MCP only if one agent needs **both**.

| Need | Plane |
|------|-------|
| Closed feature menu for chat/ML | **Feast only** |
| Open “subscription revenue by region?” | **SO** |
| Scoring / training / RAG | **Feast** |
| Open metrics **and** features in one agent | **Both** (dual MCP) |

| How Feast ↔ SO relate | What we mean |
|-----------------------|--------------|
| Side-by-side asks | SO and/or Feast tools |
| **Same meaning** | Keep Ossie metrics and Feast features from drifting — **this is the market lesson** |
| Automated metric→FeatureView tooling | **Not** evaluate/adopt now |

---

# What is Semantic Operator?

Kubeflow subproject: **semantic layer** on Kubernetes

- Define business numbers once in **Apache Ossie**  
  e.g. subscription revenue, paying customers
- Agent picks **approved names** (MCP / REST) — not raw columns
- Server builds **one SQL** — LLM never writes SQL
- Runs on **Trino** or **StarRocks** today

> Same question + model version + user → same SQL and same number

**Hard deps for a trial:** Ossie + SO + Trino (or StarRocks). Spark/Ray **prepare** data; not the SO query backend.

---

# How it works

```text
 Author → Ossie model (tables, joins, metrics, slices)
        │ kubectl apply SemanticModel
        ▼
 Semantic Operator → check schema → publish versioned model
        │
 Agent → ask: subscription_revenue by region, last quarter
        │
 Server → access check → plan → 1× SQL → Trino / StarRocks
```

---

# Where it fits in RHOAI

| Concern | Owner |
|---------|-------|
| Discover assets | **Data Registry** |
| Connect / credentials | **DCH / Connections** |
| Features, vectors, ODFVs | **Feast** |
| Certified metrics + dimensional SQL | **SO** (candidate) |
| Deploy MCP servers | **MCP Lifecycle** |

```text
 Business-number agents          Feature / ML / RAG
        │                              │
        ▼                              ▼
  SO MCP (if evaluate)          Feast SDK / feature server
        │                              │
  SQL engine (e.g. Trino)              │
        └────────────┬─────────────────┘
                     ▼
          Data Registry + DCH
          Spark / Ray → prepare tables
```

---

# Market: three layers

| Layer | Market | RHOAI |
|-------|--------|-------|
| Catalog | UC tables, Glue, Polaris | **Data Registry** |
| Business metrics | UC Metric Views, AtScale, LookML | **Gap** (SO candidate) |
| Feature store | Feast, Tecton | **Feast** |

Compose metrics + features — **do not merge**. Catalog ≠ metrics layer.  
Genie-style chat sits **on top of** agreed metrics — definitions first.

---

# Recommendation

**Evaluate** SO — not “we adopt / ship today.”

| Path | Fit |
|------|-----|
| **A. SO via MCP** (BYO Trino) | Preferred **evaluation** |
| **B.** First-party chat on SO | Later, if A shows demand |
| **C.** Chat invents SQL from catalog | Reject |
| **D.** Full BI / charts in RHOAI | Reject |
| **E.** Defer metrics plane | Valid if only **closed Feast menus** needed |

Feast MCP only if one agent needs **both** feature lookup and open business-number questions.

---

# Discussion

1. Do we need **open** *revenue by region*-style asks — or is a **closed Feast menu** enough (**E**)?
2. Default Trino: shared platform vs per-metrics vs **BYO-only** for a spike?
3. Any risk of confusing customers (SO vs Feast messaging)?
4. Prefer **evaluate A now** vs **defer**?

---

# Backup — one-liner

**Feast:** *“What do I know about **this customer** (or this fixed feature menu)?”*  

**Semantic Operator:** *“What is the **business number we agreed on**, broken down how I ask?”*

Same lake ≠ same product.

---

# References

- ADR: `ODH-ADR-SO-0001-evaluating-semantic-operator-for-governed-analytical-metrics.md`
- https://blog.kubeflow.org/introducing-semantic-operator/
- https://kubeflow.github.io/semantic-operator/
- https://github.com/apache/ossie — Semantic Model
- UC Metric Views · AtScale + Feast pattern
