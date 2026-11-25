# ODH-ADR-SO-0001: Evaluating Semantic Operator for Governed Analytical Metrics in RHOAI

|                |            |
| -------------- | ---------- |
| Date           | 2026-11-25 |
| Scope          | ODH / RHOAI Data Strategy |
| Status         | Draft |
| Authors        | [Nikhil Kathole](@ntkathole) |
| Supersedes     | N/A |
| Superseded by  | N/A |
| Tickets        | TBD |
| Other docs     | [ODH-ADR-DR-0001 Data Registry](../data-registry/ODH-ADR-DR-0001-data-registry.md), [ODH-ADR-0001 Data Connect Hub](../data-connect-hub/ODH-ADR-0001-data-connect-hub.md), [ODH-ADR-0008 Align Catalogs and Registries](../ODH-ADR-0008-align-catalogs-and-registries.md) (AI assets only; data assets out of scope there) |

## What

This ADR **evaluates** whether RHOAI should add a **governed analytical metrics** plane for open dimensional business-number questions (certified measures/dimensions, deterministic SQL, agent access without LLM-written SQL).

It records the gap vs Registry/Feast, how the market keeps **metrics and features together but separate**, when Feast alone is enough vs when Semantic Operator (Ossie) is worth evaluating, evaluation options (including defer), and boundaries if we proceed. It does **not** commit to shipping SO.

## Why

### The problem agents expose

Agents ask natural questions such as “How many paying customers?” or “How much subscription revenue by region last quarter?” The common shortcut — LLM discovers tables and writes SQL — fails on **business meaning**. Different phrasings produce different joins and aggregations; both may run; only one matches Finance.

Agreed numbers must be defined once, governed, and executed deterministically. The LLM may map language to **certified names**; it must not invent the calculation (SQL or formula).

Data Registry closes **asset discoverability**. Feast closes **feature context** for models and many apps. Neither is a **dimensional metrics** plane (one formula, open-ended breakdowns). Semantic Operator is a candidate for that gap — not for vectors, unstructured docs, or BI visualization.

### How the market uses metrics and features together

This is the main frame for **Feast vs SO** — not an afterthought integration.

Industry practice ([AtScale + Feast](https://www.atscale.com/press/atscale-accelerates-machine-learning-with-support-for-feast-feature-stores/), UC Metric Views + feature store): **one business meaning**, two products.

| | Metrics plane (semantic layer) | Features plane (feature store) |
| - | ------------------------------ | ------------------------------ |
| Job | Trusted company numbers; open slices for BI / agents | Entity features for train / serve (incl. online, PIT) |
| Example | “Subscription revenue” (Finance) by region / month | `customer_id → revenue_30d` for churn scoring |
| Market | AtScale, UC Metric Views, LookML, **SO (candidate)** | Feast, Tecton |

**Together:** the same “subscription revenue” meaning can feed a dashboard/agent ask **and** (when useful) a Feast feature for serving — so humans and models do not invent two revenues.

**Not the same product:** semantic layer ≠ feature catalog; feature store ≠ open metrics planner. Compose — do not merge. AtScale’s Feast support is exactly that bridge of **meaning**, not “AtScale replaces Feast.”

RHOAI already has Feast. This ADR asks whether to evaluate a **metrics** plane (SO) that can sit **beside** Feast the same way. Optional pipelines that turn Ossie metrics into FeatureViews are a *way to realize* that market pattern later — not something this ADR must evaluate or adopt as a product.

### Easy confusion: “Feast already gives the LLM the truth”

Yes — for many apps. Feature engineering’s job is: **make the numbers the app/model needs available under a trusted name.**

**Example — Feast alone is enough**

Feature eng already defined and served `revenue` or even `revenue_by_region_last_quarter` at the grain the chatbot needs. User asks “what’s revenue?” The LLM picks that feature name and **retrieves** it. No SO planner. No new SQL.

Same for scoring (“features for customer 42”), training frames, RAG vectors, and chatbots that only ever ask a **fixed list** of numbers (`revenue_30d`, `is_paying_customer`, …).

**Do not evaluate SO to “fix” those paths.** They are already Feast’s job. Approach **E** (defer SO) is valid here.

### Where that stops — “SO is unnecessary”

#### 1. Closed vs open asks (this is the real distinction)

| | Example |
| - | ------- |
| **Feast (closed)** | You foresaw the ask and shipped it: `revenue_by_region_last_quarter` as a feature (entity can be `region` — region is in the DB either way). Chatbot retrieves that one value. |
| **SO (open)** | Finance defines **subscription revenue** once. User asks “by region”, then “by product × month”, then “only enterprise”. Same definition — **no new FeatureView each time**. SO plans the SQL for each slice. |

Feast covers what you **foresaw and shipped**. SO is for the **same certified definition**, many slices you did **not** pre-bake.

**“But can’t Feast just use region as the entity?”** Yes. `region → revenue_last_quarter` is a normal FeatureView. That is still a **closed** ask: you baked that grain. If the next question is “by product × month” or “only enterprise,” you need another FeatureView (or another pre-baked table). Same closed-vs-open point — not a separate Feast limitation.

**Why people talk about “customer vs report”:** most ML FeatureViews are per customer (`customer_id → revenue_30d`). That shape answers “tell me about customer 42,” not “revenue by region,” unless you also ship a region-grain (or other) FeatureView. Different grains in Feast = more closed menu items, not open slicing.

#### 2. Who owns the definition

| | Example |
| - | ------- |
| **Feature eng** | Owns `txn_sum_7d` for fraud model — ML feature contract. |
| **Analytics / Finance** | Owns “subscription revenue” the company reports — company metric contract. |

Same word **“revenue”** can mean two different contracts. Putting every BI metric (and every grain) into Feast is a **choice**, not free: many FeatureViews per slice, wrong product role, online-store cost.

### Bottom line (decision in one glance)

| Situation | What to use |
| --------- | ----------- |
| AI only surfaces a **fixed menu** of numbers feature eng already serves | **Feast is enough** — SO adds little |
| Agents need **trusted, re-sliceable** business metrics without materializing every breakdown as features | **SO matters** |

**Example questions — which plane fits**

| Better on **Feast** | Better on **Semantic Operator** |
| ------------------- | --------------------------------- |
| “Features for customer 42 (risk, tenure, …).” | “**Subscription revenue by region** last quarter.” |
| “For these 10k drivers at noon, join stats without future leak.” | “How many **paying customers** (Finance’s rules)?” |
| “All driver feature rows last week for training.” | “**Customers who left**, by product and month.” |
| “Find notes similar to this claim” (vectors). | “**Booked sales vs recognized revenue** by segment.” |
| “On this request, compute extra fraud features” (ODFV). | Same **subscription revenue**, now also by month — **no new FeatureView**. |
| Chatbot only ever needs `revenue_30d` / `is_paying_customer` already in Feast. | *(Stay on Feast — SO not needed.)* |

**ODFV note:** On-demand FeatureViews compute on the fly for **this request / these entities** (e.g. fraud score from amount on the request). Still Feast — not “subscription revenue by region” from Finance’s formula.

The sections below do **not** restart the decision. They answer, in order: (1) how Feast and SO differ as products (same meaning, different contract), (2) who calls which, (3) catalog vs metrics vs features layers, (4) RHOAI pillars.

### How Feast and SO differ as products

Same pattern (named trusted defs, versions, authz, optional MCP). Different product — and in the market they are used **together** when both BI/agents and ML need the same business number:

* **Feast:** “What do I know about **this customer**?” or “Give me this **fixed list** of features we already ship.”
* **SO:** “What is the **company number we agreed on**, broken down **how I ask now**?”
* **Together (AtScale-style):** same certified metric meaning → open asks via SO **and/or** entity features via Feast — not two conflicting “revenues.”

| | Feast | Semantic Operator |
| - | ----- | ----------------- |
| **You ask for** | Features for these entities (or a fixed feature menu) | A business measure + how to break it down |
| **Typical input** | Customer / driver IDs (or a time window of them) | Metric name + dimensions + filters |
| **How the number is made** | Already stored or computed for that entity/request (incl. ODFV) | Server plans **one SQL** from the locked formula |
| **Serve path** | Offline + online store (+ vectors) | Interactive SQL engine (Trino / StarRocks) |
| **Does not do** | Open “same metric, any new slice” without a new FeatureView | Online features, PIT training, vectors |

Folding open metrics into Feast means shipping (or rebuilding) a metrics planner inside the feature store. That is why we keep them separate — not because Feast cannot store a pre-baked `revenue_by_region` feature (it can; that is the closed case above).

### Who calls which (agents and workloads)

SO is **not** a Feast replacement. Dual MCP is **only** for agents that need **both** open business numbers and features/vectors in one tool loop.

| Need | Plane | Typical access |
| ---- | ----- | -------------- |
| Closed menu of features already served for chat/ML | **Feast only** | Feature server / SDK (SO not required) |
| “Subscription revenue by region?” (open slice) | **SO** | SO MCP (or REST) |
| “Features for customer 42” / online scoring | **Feast** | Feature server / SDK (often no MCP) |
| Training / AutoML feature inputs | **Feast** | SDK / pipelines |
| Embeddings / RAG | **Feast** or AutoRAG | Feature/vector or RAG stack |
| Open business numbers **and** features in one agent | **Both** | SO MCP + Feast MCP |

```text
Workload
  ├─ Closed feature-menu agent / app           → Feast only
  ├─ Open dimensional business-number agent    → SO MCP
  ├─ Scoring / feature / RAG                   → Feast
  ├─ Training / AutoML / Spark pipelines       → Feast SDK + Registry/DCH
  └─ Combined agent (open metrics + features)  → dual MCP (optional)
```

### Market pattern: catalog + semantics + feature store

Three layers (metrics and features are peers that share meaning — see “How the market uses metrics and features together” above):

| Layer | Role | Market examples | RHOAI |
| ----- | ---- | --------------- | ----- |
| Data catalog | What assets exist | UC metastore, Glue, Polaris | **Data Registry** |
| Business metrics / meaning | What numbers mean; slice at ask time | **UC Metric Views**, AtScale, LookML | **SO (candidate)** |
| Feature store | Entity features for train/serve | Feast, Tecton | **Feast** |

A catalog is not a metrics layer. Metrics and features stay separate products that **compose**. Path: **Registry** → **DCH** → **SO** and/or **Feast**.

### Fit to RHAI data-strategy pillars

| Pillar | SO impact |
| ------ | --------- |
| P1 Connectivity | Uses Connections/DCH; does not replace them |
| P2 Compute (Spark/Ray) | Pipelines **prepare** tables SO binds to; Spark/Ray are **not** the SO query path unless a future dialect+client exists |
| P3 Catalog | Would complement Data Registry (meaning on top of assets), not replace it |
| P4 Lineage/governance | Would consume governed access; full lineage remains Pillar 4 work |
| P5 Unified AI experience | Would add metrics MCP for KPI agents; Feast MCP only where agents need features over MCP |

This ADR does **not** create a BI/visualization pillar.

## RHOAI recommendation (evaluate, then decide)

**Recommendation:** Treat Kubeflow Semantic Operator as the **leading candidate to evaluate** for the dimensional-KPI gap. Prefer an **MCP-first evaluation** (Approach A). Do **not** treat this ADR as “RHOAI will ship SO.” Do **not** lead with a Genie-like product. Do **not** require Feast MCP on every SO trial.

| Approach | What users get | Cost / risk | Fit for evaluation |
| -------- | -------------- | ----------- | ------------------ |
| **A. SO via MCP** — trial agents against **SO MCP** (add **Feast MCP** only if that agent also needs features/vectors over MCP) | Proves certified KPI tools; optional compose with Feast MCP | Lower; matches MCP direction | **Preferred evaluation path** |
| **B. Genie-like product on SO** — first-party chat UX for KPIs | Opinionated assistant | High (UX, grounding, eval, support) | **Later**, only if A proves demand **and** productization is approved |
| **C. Genie-like without SO** — chat invents SQL from Registry | Demoware; wrong numbers | Trust failure | **Reject** |
| **D. Full BI / viz suite** | Dashboards, S3 explorers | Very high; off pillars | **Reject** |
| **E. Do nothing / defer metrics plane** | No new open-slice metrics capability; Feast+Registry unchanged | Zero near-term cost; gap remains only if open dimensional asks are required | **Valid** if agents/apps only need **closed Feast feature menus** (or demand is weak) |

**Why evaluate A before committing**

1. Confirms demand is **open dimensional business numbers**, not “we already serve those as Feast features.”
2. Surfaces Trino/engine and Ossie operational cost before DSC commitment.
3. Avoids Genie/BI scope creep and dual-MCP overreach.
4. Leaves room for **E** (defer — Feast-only is enough) or an alternate metrics backend (e.g. UC Metric Views later) if SO fails evaluation.

**Metrics + features together** is how we **explain** Feast vs SO (market pattern above). Building automated Ossie→FeatureView tooling is **not** in scope to evaluate or adopt in this ADR; shared meaning + process consistency is enough unless demand appears later.

**Exit criteria for a later adopt/productize ADR:** successful Approach A spikes (KPI golden prompts via SO; engine + Registry bootstrap viable); clear demand; engine strategy (SO-0002); packaging plan (SO-0003). Until then, status remains **evaluate**.

## Goals

* Frame Feast vs SO the way the market does: **metrics and features together** (same meaning), separate products (compose, don’t merge).
* Make the choice **easy with examples** (fixed menu → Feast; re-sliceable company metrics → SO).
* Keep boundaries: **Registry = catalog**, **Feast = features**, **SO (if adopted) = business metrics**, **DCH = connectivity**.
* Prefer **Approach A** (SO MCP + BYO Trino) if we evaluate; prefer **E** if a fixed Feast feature menu is enough.
* Do **not** evaluate/adopt Ossie→FeatureView productization in this ADR; shared meaning + process is the integration story.
* If a chat UX is ever built for company metrics, it must use a certified metrics plane — not LLM-written SQL.

## How

### Architectural shape under evaluation

If evaluation succeeds, SO sits **beside** the stack — it does not replace Registry, DCH, or Feast:

| Concern | Owning component |
| ------- | ---------------- |
| Discover data assets | **Data Registry** ([ODH-ADR-DR-0001](../data-registry/ODH-ADR-DR-0001-data-registry.md)) |
| Connect / credentials | **Data Connect Hub** / Connections |
| Features, online/offline, vectors, ODFVs | **Feast** |
| Certified metrics + dimensional SQL | **Semantic Operator** (candidate) |
| Deploy MCP servers | **MCP Lifecycle Operator** |

```text
  Business-number agents/apps     Feature / ML / RAG workloads
         │                                    │
         ▼                                    ▼
   SO MCP (or REST)              Feast SDK / feature server
         │                       (Feast MCP only if agent needs MCP)
         ▼                                    │
   SQL engine (Trino / …)                     │
         └────────────────┬───────────────────┘
                          ▼
           Data Registry + DCH / Connections
           Spark/Ray prepare tables
```

**Hard dependency for any trial:** Apache **Ossie** + SO + an SO-capable SQL engine (**Trino** or **StarRocks** today). Spark/Ray prepare data; they are **not** SO query backends. BYO Trino is fine for evaluation.

Preferred trial shape: **Approach A** (SO MCP). Feast MCP optional. Not a BI viz product. Genie-like UX out of scope for evaluation. Go/no-go for DSC/GA is a **follow-on** decision.

### Runtime path (metrics)

1. Apply an Ossie model as a `SemanticModel` CR (optionally bootstrapped from Data Registry).
2. Operator validates bindings, publishes a versioned compiled model; failed validation keeps the last good model.
3. Agent discovers allowed metrics/dimensions via MCP and calls `query_metric` (certified names only — no LLM SQL).
4. Server authorizes, plans one SQL statement, executes on the configured engine, returns rows with model version and request hash.

### Query engine

* Upstream SO = **dialect + client** per engine; compiler stays engine-agnostic. Only **Trino** and **StarRocks** ship today; there is **no tracked upstream Spark engine issue** — Spark support means contributing those adapters.
* **RHOAI evaluation stance:** prefer **Trino** for trials (Iceberg-friendly, already in data-strategy “evaluate” set). **BYO Trino** is preferred for evaluation to avoid premature platform ownership.
* Spark/Ray remain Pillar 2 for pipelines and Feast offline compute; they **prepare** data SO queries.

### Integration with Data Registry and DCH

* Registry is system of record for tables/volumes; Ossie `source` bindings should resolve to Registry assets where possible.
* `maturity_stage=semantic-layer` is a catalog hint, not a live metrics runtime.
* `connection_ref` / Connections / DCH reach storage; SO is not a second connection manager.
* Optional later: list published semantic models in Registry for discovery — query planning stays in SO.

### Integration with Feast and agents

**Primary idea (market):** metrics and features are used **together** — same business meaning, two planes (see Why). That is the Feast ↔ SO relationship story.

How that shows up in RHOAI (do not conflate):

| # | How they relate | What it is | Scope for this ADR |
| - | --------------- | ---------- | ------------------ |
| **1** | **Side-by-side asks** | Open business numbers → SO; entity features → Feast. Dual MCP only if one agent needs both. | In scope for Approach **A** (SO trial **without** Feast MCP is fine) |
| **2** | **Same meaning** | Shared lake + naming/process so Ossie “subscription revenue” and Feast `revenue_*` features do not drift | Expected whenever both planes exist — **this is the AtScale lesson**, not optional trivia |
| **3** | **Metric → FeatureView tooling** | Codegen / pipelines that turn a certified Ossie measure into entity-grain Feast features (optional score write-back) | **Not** evaluate/adopt here; only if later demand needs automation beyond (2) |

* KPI/dimensional intents → **SO MCP** (or REST). Feature/vector/ODFV intents → **Feast** via SDK/feature server by default; **Feast MCP** when the consumer is an MCP agent.
* Dual MCP only for combined agents (1). Do not imply every SO install ships or requires Feast MCP.
* A single façade MCP URL is optional later packaging; engines stay separate.

## Open Questions

* Do target customers need **open dimensional** business-number agents, or is a **closed Feast feature menu** enough (Approach E)?
* Default Trino topology: shared platform service vs per-metrics deployment vs strict BYO-only?

## Alternatives

### Alternative 1: Extend Feast into a dimensional metrics engine (rejected)

One access surface is attractive; cost is owning a full semantic layer inside Feast and blurring ML serving. Market keeps semantics and features separate. **Rejected.**

### Alternative 2: Agents generate SQL from Data Registry schemas only (rejected)

No new component; restores non-deterministic, non-certified SQL. **Rejected.**

### Alternative 3: Precompute every needed slice as Feast FeatureViews (rejected as *substitute* for a metrics plane)

**Valid when** the product only needs a **closed, known set** of numbers — that is normal feature engineering, and Feast alone is the right answer (see “When Feast alone is enough”).

**Rejected as a substitute for SO** when agents need **open-ended** breakdowns of the same certified measure: definitions explode by grain; entity-keyed FeatureViews do not match report-shaped asks; Finance/analytics metrics get forced into an ML serving product.

### Alternative 4: Unity Catalog OSS Metric Views instead of SO (deferred)

Strong if UC semantics become productizable on our path. Catalog Phase 1 already deferred heavy UC OSS. SO is Kubeflow/MCP-native and Ossie-portable — **preferred candidate to evaluate**. Keep UC Metric Views as a parallel watch if SO evaluation fails or Spark-native semantics win.

### Alternative 5: Replace Feast with SO for all “data agent” / ML data access (rejected)

SO does not provide online features, PIT frames, or vectors. KPI-only agents may use SO **without** Feast; that is valid. Using SO as a **Feast replacement** for scoring/RAG/training is not. **Rejected as a substitute for Feast.**

### Alternative 6: Full BI / visualization platform in RHOAI (rejected)

Competes with Power BI/Tableau/Superset; outside data-strategy pillars; huge opportunity cost vs Registry/Feast/agents. SO is metrics-for-AI, not a viz suite. **Rejected.**

### Alternative 7: Lead with Genie-like “ask your data” product (deferred / rejected as first step)

High UX and support cost; without SO it becomes unsafe text-to-SQL. **Reject as the initial delivery.** Revisit only as Approach B after Approach A (SO MCP) proves demand; backend must be SO for KPIs.

### Alternative 8: Single façade MCP that proxies to SO and optionally Feast (optional later)

Convenience packaging for combined agents only (integration area 1). Not required for KPI-only SO MCP. Engines stay separate.

### Alternative 9: Automate Ossie → Feast FeatureViews (out of scope to evaluate/adopt here)

The **market idea** (same metric meaning for humans and models) is already the Feast↔SO frame above. Automated codegen/pipelines are only one way to enforce it; process + shared naming (integration row 2) is enough for this ADR. **Not** a substitute for SO, **not** merging products, **not** required for Approach A.

## References

* [Introducing Semantic Operator (Kubeflow blog)](https://blog.kubeflow.org/introducing-semantic-operator/)
* [Kubeflow Semantic Operator](https://github.com/kubeflow/semantic-operator) — [Adding a query engine](https://kubeflow.github.io/semantic-operator/guides/adding-an-engine/)
* [Apache Ossie core spec — Semantic Model](https://github.com/apache/ossie/blob/main/core-spec/spec.md#semantic-model)
* [Semantic Operator — agent to SQL](https://kubeflow.github.io/semantic-operator/architecture/agent-to-sql/)
* [Databricks: What is a semantic layer?](https://answers.databricks.com/semantic-layer)
* [Unity Catalog semantics / Metric Views](https://docs.databricks.com/aws/en/uc-semantics/)
* [AtScale + Feast](https://www.atscale.com/press/atscale-accelerates-machine-learning-with-support-for-feast-feature-stores/)
* [ODH-ADR-DR-0001 Data Registry](../data-registry/ODH-ADR-DR-0001-data-registry.md)
* [ODH-ADR-0001 Data Connect Hub](../data-connect-hub/ODH-ADR-0001-data-connect-hub.md)
* Feast (feature store in RHOAI); MCP docs when relevant: `feast/docs/reference/feature-servers/mcp-server.md`
* Architecture-context: `overlays/0031-data-registry-architecture.md`; MCP lifecycle under `architecture/newest/`

## Reviews

| Reviewed by | Date | Notes |
| ----------- | ---- | ----- |
|             |      |       |
