# ODH-ADR-FEAST-0001: Platform-Native RBAC for the Feast Operator via kube-rbac-proxy and Aggregated ClusterRoles

|                |            |
| -------------- | ---------- |
| Date           | 2026-09-11 |
| Scope          | OpenShift AI — Feast Operator, Feature Store |
| Status         | Draft |
| Authors        | [Nikhil Kathole](@nikhilkathole) |
| Supersedes     | N/A |
| Superseded by  | N/A |
| Tickets        | TBD |
| Other docs     | [ODH-ADR-ML-0002 — Shared Workspace for Cross-Namespace Resource Sharing in MLflow](../mlflow/ODH-ADR-ML-0002-shared-workspace-for-cross-namespace-resource-sharing.md), [ODH-ADR-DR-0001 — Data Registry](../data-registry/ODH-ADR-DR-0001-data-registry.md) |

## What

Add platform-native RBAC to the Feast operator by deploying a `kube-rbac-proxy` sidecar alongside Feast server pods and creating aggregated ClusterRoles (`feast-server-viewer`, `feast-server-editor`) for server API access — complementing the existing `featurestore-viewer-role` and `featurestore-editor-role` that gate FeatureStore CR access. This establishes a **layered authorization architecture**: `kube-rbac-proxy` (via `SubjectAccessReview`) serves as the outer gate for namespace-level access control — consistent with MLflow and Data Registry — while Feast's existing `Permission`-based RBAC remains as the inner layer for domain-specific granularity (online vs offline access, regex name patterns, tag-based filtering).

## Why

### The Gap

The Feast operator currently deploys `FeatureStore` CRs as namespace-scoped resources. By default, the operator applies `spec.authz.kubernetes` automatically (even when not explicitly specified in the CR), so Feast enforces **Kubernetes authentication** out of the box — only users with a valid token (verified via TokenReview) can access the server.

However, the authorization model follows a **gradual-adoption pattern**: if no `Permission` objects are defined, all authenticated users are granted full access. Once at least one `Permission` is added, the posture flips to **deny-unless-matched** — a single rule granting access to the owning namespace effectively blocks all others. But until that first rule is configured (via Python code and `feast apply`), cross-namespace access is allowed for any authenticated user:

```
team-beta namespace                    team-alpha namespace
┌─────────────────────┐               ┌─────────────────────────┐
│ Notebook             │    remote     │ FeatureStore CR          │
│ feature_store.yaml:  │───gRPC/REST──►│ Feast Server Pod         │
│   registry:          │               │ (authz.kubernetes set,   │
│     path: feast-     │               │  user is authenticated,  │
│       server.team-   │               │  no Permission objects   │
│       alpha.svc      │               │  → full access granted)  │
└─────────────────────┘               └─────────────────────────┘
```

A notebook in `team-beta`, if authenticated, can access `team-alpha`'s Feast server when no Permission objects are defined. However, Feast's authorization model uses a **flip-on-first-rule** approach: as soon as at least one `Permission` object is defined, the posture changes — any resource or action **not covered** by a matching Permission is **denied**. This means a single Permission rule granting access to `team-alpha` users effectively blocks all other namespaces without needing an explicit deny rule.

The gap is not that Feast lacks the ability to restrict access — it can — but that:
1. It requires Python code and `feast apply` to activate, rather than standard `oc create rolebinding`
2. The admin must know to add that first Permission rule; until then, all authenticated users have full access
3. The mechanism is Feast-specific, not consistent with how every other RHOAI component (MLflow, Data Registry, Model Registry) handles authorization

### The Inconsistency

Every other multi-tenant service in RHOAI uses the same platform-native RBAC pattern:

| RHOAI Component | Auth Mechanism | Default Posture |
|-----------------|---------------|-----------------|
| **MLflow** | `kube-rbac-proxy` sidecar + SAR | Default-deny |
| **Data Registry** | `kube-rbac-proxy` sidecar + SAR (with URL path segment extraction) + Feast Permissions fallback | Default-deny |
| **Model Registry** | `kube-rbac-proxy` sidecar | Default-deny |
| **KServe** | Authorino / Service Mesh | Default-deny |
| **Feast** | K8s TokenReview (authn) + internal Python `Permission` objects (authz) | ⚠️ **Authenticated = full access** (until Permissions added) |

Feast provides Kubernetes authentication out of the box (via `spec.authz.kubernetes`), but it is the only RHOAI component where:
1. **Authorization** is handled inside the application process (Python `Permission` objects), not at the infrastructure level
2. Authenticated users get full access until `Permission` objects are explicitly defined — a gradual-adoption model that contrasts with the default-deny posture of other components
3. Fine-grained permissions require Python code and `feast apply`, instead of standard `oc create rolebinding`
4. The admin UX for access control differs from every other RHOAI component

## Goals

* Deploy `kube-rbac-proxy` as a sidecar container for all `FeatureStore` CRs using the default `spec.authz.kubernetes` auth mode (which the operator applies automatically), matching the MLflow operator pattern
* Create aggregated `ClusterRole` resources (`feast-server-viewer`, `feast-server-editor`) with `aggregate-to-view`/`aggregate-to-edit` labels so that existing namespace RoleBindings automatically grant Feast server API access — complementing the existing `featurestore-viewer-role`/`featurestore-editor-role` that gate CR access
* Establish a **default-deny** outer gate — users without a RoleBinding in the Feast server's namespace are rejected before reaching the Feast process
* Preserve Feast's existing `Permission`-based RBAC as the inner layer for domain-specific access control (online/offline separation, regex name matching, tag filtering, result filtering)
* Require no changes to Feast's Python SDK or server code — the sidecar is purely an operator/deployment concern
* Provide a consistent admin UX across RHOAI: `oc create rolebinding` for namespace-level access, `feast apply` only for fine-grained rules

## Non-Goals

* **Replacing Feast's internal Permission model.** Feast Permissions provide capabilities that K8s RBAC cannot express (see §Layered Architecture). This ADR adds an outer gate, not a replacement.
* **Modifying the `FeatureStore` CRD schema.** The `spec.authz.kubernetes` field already exists. No CRD changes are needed.
* **OPA or Authorino integration.** External policy engines are a potential future layer but are not in scope for this ADR.

## How

### Layered Authorization Architecture

The solution uses two complementary authorization layers, each handling what it does best:

```
Incoming Request (gRPC / REST)
        │
        ▼
┌───────────────────────────────────────────────────────────┐
│ Layer 1: kube-rbac-proxy sidecar (OUTER GATE)             │
│                                                           │
│ • Extracts bearer token from request                      │
│ • Extracts namespace/project from URL path segment        │
│   (e.g., /v1/{project}/namespaces → project = team-alpha) │
│ • Issues SubjectAccessReview (SAR) to K8s API server      │
│   "Can user X access feast.dev/featureviews:get           │
│    in namespace team-alpha?"                              │
│ • No RoleBinding → 403 Forbidden (default-deny)          │
│ • Has RoleBinding → forward to Feast server               │
│                                                           │
│ Handles: namespace-level access, default-deny,            │
│          K8s audit trail, enterprise IdP passthrough       │
└───────────────────┬───────────────────────────────────────┘
                    │ allowed
                    ▼
┌───────────────────────────────────────────────────────────┐
│ Layer 2: Feast Permissions (INNER LAYER)                  │
│                                                           │
│ • Evaluates Permission objects from Feast registry        │
│ • Checks roles, groups, namespaces, name patterns, tags   │
│ • Supports online vs offline action separation            │
│ • Supports filter_only mode (soft deny)                   │
│ • Handles cases where URL path doesn't contain enough     │
│   info for resource-level SAR (fallback authorization)    │
│                                                           │
│ Handles: per-resource access, regex name patterns,        │
│          tag-based filtering, online/offline split,        │
│          result-set filtering                             │
└───────────────────────────────────────────────────────────┘
```

### Why Two Layers — What Each Handles

| Capability | Layer 1: kube-rbac-proxy (SAR) | Layer 2: Feast Permissions |
|---|---|---|
| Namespace-level access | ✅ RoleBindings in namespace | ✅ `NamespaceBasedPolicy` |
| Role-based access | ✅ ClusterRoles/Roles | ✅ `RoleBasedPolicy` |
| Default security posture | ✅ **Default-deny** | ⚠️ Full access until first Permission added; then deny-unless-matched |
| Per-resource access | ✅ Exact names via `resourceNames` in Role | ✅ Regex patterns via `name_patterns=["fraud_.*"]` |
| Tag-based filtering | ❌ Not supported in K8s RBAC | ✅ `required_tags={"sensitivity": "pii"}` |
| Online vs Offline separation | ❌ Only CRUD verbs | ✅ `READ_ONLINE`, `READ_OFFLINE`, `WRITE_ONLINE`, `WRITE_OFFLINE` |
| Soft deny (filter results) | ❌ Binary allow/deny | ✅ `filter_only=True` removes unauthorized items from results |
| Type hierarchy awareness | ❌ Flat resources | ✅ `FeatureView` permission covers `StreamFeatureView` subclass |
| Admin UX | ✅ `oc create rolebinding` | ⚠️ Python code + `feast apply` |
| Audit trail | ✅ K8s API server audit logs | ⚠️ Python application logs |
| Cross-tool consistency | ✅ Same as MLflow, Data Registry | ❌ Feast-specific |

Neither layer alone is sufficient. Together they provide:
- **Default-deny** namespace gating (Layer 1) — closes the security gap
- **Domain-specific** fine-grained control (Layer 2) — preserves Feast's unique RBAC capabilities

### Pseudo-Resources and SAR

The `kube-rbac-proxy` sidecar issues a **SubjectAccessReview (SAR)** — to the K8s API server. SAR is a server-side check: given a user identity (extracted from the bearer token via TokenReview), can this user perform action Y on resource Z in namespace N? No CRDs are registered for the pseudo-resources — the K8s API server evaluates SAR requests for any API group string.

| Resource | Verbs | What It Gates |
|----------|-------|---------------|
| `featureviews` | `get`, `list`, `create`, `update`, `delete` | Feature view CRUD, online/offline reads |
| `entities` | `get`, `list`, `create`, `update`, `delete` | Entity CRUD |
| `datasources` | `get`, `list`, `create`, `update`, `delete` | Data source CRUD |
| `featureservices` | `get`, `list`, `create`, `update`, `delete` | Feature service CRUD |

### URL Path Segment Extraction

For REST endpoints, the proxy extracts the **namespace/project** from the URL path using named path captures — the same approach already merged for the Data Registry in [opendatahub-io/kube-rbac-proxy#28](https://github.com/opendatahub-io/kube-rbac-proxy/pull/28). Standard clients (PyIceberg, Spark, Trino, SDK) authenticate with just `Authorization: Bearer <token>` — no custom `X-Namespace` headers needed.

Example kube-rbac-proxy configuration for Feast REST endpoints:

```yaml
authorization:
  endpoints:
    # Feast REST: GET /v1/{project}/feature-views
    - path: /v1/{project}/feature-views
      mappings:
        - methods: [get]
          resources:
            - resourceAttributes:
                namespace: '{{ index .PathParams "project" }}'
                apiGroup: feast.dev
                resource: featureviews
                verb: get
        - methods: [post, put]
          resources:
            - resourceAttributes:
                namespace: '{{ index .PathParams "project" }}'
                apiGroup: feast.dev
                resource: featureviews
                verb: create
```

For endpoints where the namespace/project **cannot be determined from the URL** (e.g., gRPC calls, or REST paths without project in the path), the proxy performs a namespace-level SAR check, and **Feast Permissions** handles the fine-grained authorization at Layer 2. This is the same pattern used by the Data Registry: SAR gates what can be determined from the URL; Feast Permissions covers the rest.

The `kube-rbac-proxy` sidecar performs a SAR check per request. Namespace-level gating (e.g., "can user X access `featureviews` in `team-alpha`?") is the baseline. Per-resource granularity is also supported via `resourceNames` in the ClusterRole or Role — for example, restricting a user to specific FeatureViews:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: feast-pii-reader
  namespace: team-alpha
rules:
  - apiGroups: ["feast.dev"]
    resources: ["featureviews"]
    resourceNames: ["customer_credit_features", "fraud_detection_features"]
    verbs: ["get"]
```

This allows platform admins to combine namespace-level gating (via aggregated ClusterRoles) with per-resource restrictions (via custom Roles) — all using standard `oc create role` / `oc create rolebinding`, without any Feast-side configuration.

### Aggregated ClusterRoles

The Feast operator **already creates** two aggregated ClusterRoles for the FeatureStore CRD:

| Existing ClusterRole | Aggregates to | Resource | Purpose |
|---|---|---|---|
| `featurestore-viewer-role` | `view` | `feast.dev/featurestores` (CRD) | View/list FeatureStore **Custom Resources** |
| `featurestore-editor-role` | `edit`, `admin` | `feast.dev/featurestores` (CRD) | CRUD on FeatureStore **Custom Resources** |

These control who can manage the `FeatureStore` CR itself (the K8s object). This ADR adds **new** ClusterRoles for the **pseudo-resources** that gate access to the Feast **server API** (feature views, entities, etc.) via `kube-rbac-proxy`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: feast-server-viewer
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
rules:
  - apiGroups: ["feast.dev"]
    resources: ["featureviews", "entities", "datasources", "featureservices"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: feast-server-editor
  labels:
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
rules:
  - apiGroups: ["feast.dev"]
    resources: ["featureviews", "entities", "datasources", "featureservices"]
    verbs: ["get", "list", "create", "update", "delete"]
```

The naming distinguishes the two layers: `featurestore-*-role` gates the **CR** (who can deploy/configure a FeatureStore), while `feast-server-*` gates the **server API** (who can read/write feature data).

**Aggregation effect:** Users who already have `view`, `edit`, or `admin` RoleBindings in the Feast server's namespace **automatically** receive both CR access (existing) and server API access (new) — zero additional configuration needed.

| Existing RoleBinding | CR Access (existing) | Server API Access (new) | Server API Verbs |
|---------------------|---------------------|------------------------|-----------------|
| `view` in namespace | `featurestore-viewer-role` | `feast-server-viewer` | `get`, `list` (read-only) |
| `edit` in namespace | `featurestore-editor-role` | `feast-server-editor` | `get`, `list`, `create`, `update`, `delete` |
| `admin` in namespace | `featurestore-editor-role` | `feast-server-editor` | `get`, `list`, `create`, `update`, `delete` |

### Operator Implementation

When the operator reconciles a `FeatureStore` CR with `spec.authz.kubernetes` set, it:

1. **Injects a `kube-rbac-proxy` sidecar** into the Feast server Deployment (registry, online store, offline store pods)
2. **Reconfigures port routing** — external traffic hits the proxy port; the proxy forwards to the Feast server on localhost
3. **Creates the aggregated ClusterRoles** (`feast-server-viewer`, `feast-server-editor`) if they don't already exist
4. **Preserves the existing `authz.kubernetes.roles` behavior** — Feast's internal `KubernetesTokenParser` continues to extract roles for Layer 2 Permission evaluation

```yaml
# Existing CRD field — no changes needed
apiVersion: feast.dev/v1
kind: FeatureStore
metadata:
  name: my-feature-store
  namespace: team-alpha
spec:
  feastProject: team_alpha
  authz:
    kubernetes:
      roles:
        - feast-admin
        - feast-reader
  # ... rest of spec
```

Since `spec.authz.kubernetes` is the default auth mode (applied automatically by the operator even when not explicitly specified), the `kube-rbac-proxy` sidecar is deployed for all standard FeatureStore CRs. CRs explicitly configured with OIDC auth (`spec.authz.oidcAuthz`) or `spec.authz.noAuth: true` do not get the sidecar.

### Pod Topology

```
Feast Server Pod (with kube-rbac-proxy)
┌──────────────────────────────────────────────────────┐
│                                                      │
│  ┌──────────────────────┐  ┌──────────────────────┐  │
│  │ kube-rbac-proxy      │  │ feast-server         │  │
│  │ (sidecar)            │  │ (main container)     │  │
│  │                      │  │                      │  │
│  │ Port 8443 (external) │──│ Port 8080 (localhost) │  │
│  │ TLS termination      │  │ gRPC + REST          │  │
│  │ SAR check            │  │ Feast Permissions     │  │
│  └──────────────────────┘  └──────────────────────┘  │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### User Experience — Before and After

**Before (today):**

| Step | Action | By whom |
|------|--------|---------|
| 1 | Deploy FeatureStore CR | Team |
| 2 | Write Python `Permission` objects | Team |
| 3 | `feast apply` to register permissions | Team |
| 4 | If not done → everyone has full access ⚠️ | — |

**After (with this ADR):**

| Step | Action | By whom |
|------|--------|---------|
| 1 | Deploy FeatureStore CR with `authz.kubernetes` | Team |
| 2 | **Done** — existing RoleBindings grant access automatically | — |
| 3 | (Optional) Write Feast `Permission` objects for fine-grained rules | Team |

For teams that only need namespace-level access control (the majority), **step 2 is automatic** — the aggregated ClusterRoles mean anyone who can already use the namespace can use Feast. Fine-grained Feast Permissions become opt-in, not mandatory.

## Alternatives

### 1. Rely Solely on Feast's Built-in Permission Model

Continue using Feast's Python-based `Permission` objects as the only authorization mechanism.

**Pros:**
- No operator changes needed.
- Feast Permissions are more expressive than K8s RBAC for domain-specific rules (tag filtering, online/offline separation).

**Cons:**
- Gradual-adoption model — Kubernetes authentication is enforced, and once at least one `Permission` is defined the posture flips to deny-unless-matched. But until that first rule is added, all authenticated users get full access — and adding that rule requires Python code + `feast apply`.
- Requires Python code (`feast apply`) to manage permissions — inconsistent with every other RHOAI component's admin UX.
- No K8s audit trail for access decisions.
- No cross-tool consistency — Feast uses a different auth mechanism than MLflow, Data Registry, Model Registry, and KServe.
- Cross-namespace access is unrestricted unless explicit `NamespaceBasedPolicy` Permissions are written and applied.

**Decision: Rejected as the sole mechanism.** Feast Permissions are preserved as the inner layer. Once at least one Permission is defined, the model is effectively deny-unless-matched — which is sound. But the mechanism requires Python code + `feast apply` (inconsistent admin UX), the default posture before the first rule is open, and there is no platform-level audit trail for authorization decisions.

### 2. Replace Feast RBAC Entirely with kube-rbac-proxy

Remove Feast's internal `Permission` evaluation and rely solely on `kube-rbac-proxy` SAR for all authorization.

**Pros:**
- Simpler — one authorization mechanism.
- Fully consistent with MLflow, which has no internal auth logic.

**Cons:**
- Loses domain-specific capabilities that K8s RBAC cannot express:
  - **Online vs Offline access separation** — K8s RBAC has generic CRUD verbs, not `READ_ONLINE`/`READ_OFFLINE`
  - **Regex name patterns** — K8s `resourceNames` requires exact matches, no wildcards
  - **Tag-based filtering** — K8s RBAC cannot match on object metadata
  - **Result-set filtering** — K8s RBAC is binary allow/deny, cannot filter list results
- Would require splitting Feast API endpoints into multiple pseudo-resources to approximate the current action model (e.g., separate `featureviews-online` and `featureviews-offline` resources)

**Decision: Rejected.** Feast's domain-specific RBAC capabilities have genuine value for teams with fine-grained requirements. The layered approach preserves these while adding the default-deny outer gate.

### 3. Use Authorino Instead of kube-rbac-proxy

Deploy Authorino as the external authorization layer.

**Pros:**
- More expressive — Authorino supports OPA policies, JSON pattern matching, and external auth delegation.
- Could potentially replace both kube-rbac-proxy AND Feast Permissions in a single layer.

**Cons:**
- Authorino is not currently deployed for MLflow, Data Registry, or Model Registry — introducing it for Feast alone creates inconsistency.
- Adds an infrastructure dependency (Authorino operator) that kube-rbac-proxy does not require.
- More complex to configure than kube-rbac-proxy for the namespace-level gating use case.

**Decision: Deferred.** If RHOAI adopts Authorino as a platform-wide auth layer in the future, Feast can migrate from kube-rbac-proxy. For now, kube-rbac-proxy provides consistency with MLflow and requires no new infrastructure.

### 4. NetworkPolicy-Only Approach

Use Kubernetes NetworkPolicy to restrict cross-namespace traffic to Feast server pods.

**Pros:**
- Zero code changes — purely infrastructure.
- Blocks cross-namespace access at the network level.

**Cons:**
- Blunt instrument — blocks ALL cross-namespace traffic, including legitimate remote access (e.g., shared registry model).
- No fine-grained control — cannot say "team-beta can read but team-gamma cannot."
- No audit trail — blocked requests are silently dropped.
- Not the pattern used by any other RHOAI component.

**Decision: Rejected.** NetworkPolicy is a useful defense-in-depth measure but cannot serve as the primary authorization mechanism.

## Security and Privacy Considerations

- **Immediate default-deny without requiring Feast Permission setup.** Today, a FeatureStore CR with `authz.kubernetes` enforces authentication. Once at least one `Permission` object is defined, Feast switches to deny-unless-matched — but until that first rule is added via Python code and `feast apply`, all authenticated users have full access. With the kube-rbac-proxy sidecar, namespace-level authorization is enforced **immediately** at deploy time via K8s RoleBindings — no Feast-side configuration needed. This matches the default-deny posture of every other RHOAI component from day one.
- **Cross-namespace access is gated.** The SAR check is namespace-scoped. A notebook in `team-beta` connecting to `team-alpha`'s Feast server will be denied unless an explicit RoleBinding exists in `team-alpha` for that user or ServiceAccount.
- **K8s audit trail.** Every SAR evaluation is logged by the K8s API server. This provides a compliance-grade audit trail for all Feast access decisions — unlike Feast's current Python-level logging.
- **Enterprise IdP integration.** Because SAR evaluates the user identity (extracted from the bearer token via TokenReview) against K8s RBAC, enterprise IdP integration (via OpenShift OAuth → Entra ID, AD, Okta) is automatic. User deprovisioning in the IdP immediately revokes Feast access.
- **No credential exposure.** The kube-rbac-proxy sidecar handles TLS termination and token validation. Feast's application code never sees raw bearer tokens.

## Risks

- **Performance overhead of SAR calls.** Each request incurs a SAR round-trip to the K8s API server. *Mitigation:* kube-rbac-proxy caches SAR results with configurable TTLs (default: 5 minutes for allow, 30 seconds for deny). The MLflow and Data Registry deployments use the same proxy with no reported performance issues in production. For online feature serving with sub-2ms latency requirements, the cache absorbs the overhead after the first request.
- **Cross-namespace access with kube-rbac-proxy.** With the sidecar deployed, a user in `team-beta` connecting to the Feast server in `team-alpha` needs a RoleBinding in `team-alpha`'s namespace for the SAR check to pass. However, the aggregated ClusterRoles minimize manual work:
  - If the user is added to `team-alpha`'s Data Science Project via the dashboard (which creates a `dashboard-permissions-*` RoleBinding with `view`/`edit` role), the aggregated `feast-server-viewer`/`feast-server-editor` ClusterRoles automatically grant Feast server API access — **no additional RoleBinding needed**.
  - The Feast operator already creates a **client ConfigMap** (`feast-<name>-client`) containing the `feature_store.yaml` in the server's namespace, and maintains a **namespace registry** ConfigMap that tracks all FeatureStore instances cluster-wide (readable by all authenticated users). The dashboard/workbench creation flow can use this registry to auto-deploy the client config, allowing users to connect to any Feast server they have access to without manual configuration.
  - For users who are NOT added to the server's namespace, access is denied — this is the intended default-deny behavior. An admin must explicitly grant access via `oc create rolebinding` or the dashboard's project sharing UI.
  
  *Mitigation for existing setups:* Since `spec.authz.kubernetes` is already the default, most existing deployments already use Kubernetes authentication. The sidecar adds namespace-level authorization on top. Migration documentation will guide teams through verifying that their cross-namespace users have appropriate RoleBindings (most already will if they were added via the dashboard).
- **Group-based Feast Permissions blocked by outer gate.** Teams using Feast's `GroupBasedPolicy` (e.g., granting `data-scientists` group access to feature views) may find that the kube-rbac-proxy outer gate denies requests before they reach Feast's inner layer — if the group does not have a corresponding RoleBinding in the server's namespace. For example, a user in the `data-scientists` group connecting from `team-beta` to `team-alpha`'s Feast server: Layer 1 checks for a RoleBinding in `team-alpha` → no RoleBinding for the group → 403 Forbidden, even though Layer 2 would have allowed it.
  
  *Mitigation:* K8s RoleBindings support `Group` subjects, so the group-based access can be mirrored at the K8s level:
  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: data-scientists-feast-access
    namespace: team-alpha
  subjects:
    - kind: Group
      name: data-scientists
      apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: ClusterRole
    name: feast-server-viewer
    apiGroup: rbac.authorization.k8s.io
  ```
  This aligns both layers: Layer 1 passes the group's RoleBinding, Layer 2 passes the `GroupBasedPolicy`. Migration documentation will include a mapping guide: for each Feast `GroupBasedPolicy`, create a corresponding K8s RoleBinding with the same group name in the server's namespace. The Feast operator could also automate this — when it detects `GroupBasedPolicy` Permission objects in the registry, it could create the corresponding RoleBindings.

- **Dual authorization complexity.** Two authorization layers may confuse operators troubleshooting access issues (is the denial from Layer 1 or Layer 2?). *Mitigation:* Layer 1 returns a clear `403 Forbidden` with the namespace and pseudo-resource in the response. Layer 2 returns a Feast-specific `FeastPermissionError` with the Permission name and explanation. The error messages are distinguishable. Operator documentation will include a troubleshooting decision tree.

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| Feast / Feature Store         |                  |            | Yes — operator changes (sidecar injection, ClusterRole creation) |
| Data Registry                 |                  |            | Yes — shared FeatureStore CR gets consistent auth |
| AI Core Platform / Operator   |                  |            | Yes — ClusterRole aggregation labels must not conflict with existing roles |
| Product Management            |                  |            | Yes — security posture improvement, admin UX improvement |
| Architects Team               | @opendatahub-io/architects |  | Yes — establishes auth consistency pattern |

## References

* [ODH-ADR-ML-0002 — Shared Workspace for Cross-Namespace Resource Sharing in MLflow](../mlflow/ODH-ADR-ML-0002-shared-workspace-for-cross-namespace-resource-sharing.md) — precedent for SAR-based authorization, Auth CR controller, aggregated ClusterRoles
* [ODH-ADR-DR-0001 — Data Registry](../data-registry/ODH-ADR-DR-0001-data-registry.md) — Data Registry SAR authorization with kube-rbac-proxy + Feast Permissions fallback
* [opendatahub-io/kube-rbac-proxy#28 — URL path segment extraction for SAR](https://github.com/opendatahub-io/kube-rbac-proxy/pull/28) — named path captures (`{project}`) for extracting namespace from URL paths, merged for Data Registry support. Enables standard Iceberg/REST clients to authenticate with `Authorization: Bearer` only — no custom headers needed.
* [Feast Permissions documentation](https://docs.feast.dev/getting-started/architecture/rbac) — Feast's built-in RBAC model (roles, groups, namespaces, tag filtering, action types)
* [opendatahub-io/kube-rbac-proxy](https://github.com/opendatahub-io/kube-rbac-proxy) — ODH fork of kube-rbac-proxy used by MLflow, Data Registry, and other RHOAI components for SAR-based authorization
* [Kubernetes RBAC aggregation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#aggregated-clusterroles) — the mechanism that auto-inherits feast-server-viewer/editor from view/edit roles

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
|                               |            |       |
