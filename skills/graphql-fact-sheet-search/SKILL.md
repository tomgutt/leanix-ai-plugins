---
name: graphql-fact-sheet-search
description: Answer any natural language question about LeanIX fact sheets by choosing the best retrieval approach — structured GraphQL query or vector search — then executing it.
meta:
  tools: [get_fact_sheet_types, get_allFactSheets_schema, get_workspace_data_model, execute_allFactSheets_query, vector_search]
---

# GraphQL Fact Sheet Search

Answer questions about LeanIX fact sheets by choosing the right retrieval approach and executing it.

**Always call `get_fact_sheet_types`, `get_allFactSheets_schema`, and `get_workspace_data_model` first, before reasoning about the query.**

---

## Choosing the retrieval approach

After loading the workspace context, decide which approach fits the query.

**Always use `execute_allFactSheets_query`** for:
- Lifecycle phase (phasing out, end of life, retiring)
- Approval / workflow state (rejected, approved, draft, archived)
- Ownership / subscription (missing responsible, has accountable)
- Relation presence or absence (has IT components, missing owner, dependent on X)
- Structured field values (technicalSuitability, businessCriticality, completion %, maturity level)
- Date ranges (created after, updated before)
- Exact full name, brand name, code, or UUID lookup — use `fullTextSearch`
- Any query where the answer depends on relations, field values, or metadata not stored in text descriptions

**Use `vector_search`** when the query is an **abbreviation, acronym, or partial name** that you cannot expand with confidence — e.g. "ac mgmt", "hr sys", "fin tool". Do NOT guess what the abbreviation stands for; let vector search find matches by semantic similarity.

**Do NOT use `vector_search`** for:
- Concept or business intent queries ("marketing automation system", "invoice data apps") — embeddings only contain displayName, description, type, lxState, status, and category; whether a fact sheet matches a business concept is not reliably encoded
- Queries about relations ("apps dependent on IBM", "apps used in Spain") — relations are **not** in embeddings
- Queries about field values (lifecycle, completion %, maturity, TCO) — field values are **not** in embeddings
- Queries about subscription/ownership — not in embeddings
- Exact full name / code / UUID lookups — GraphQL `fullTextSearch` is more precise
- Any query where the structured approach already has a clear filter path

When in doubt, use `execute_allFactSheets_query` first. Only use `vector_search` if the query is an abbreviated or partial name and GraphQL `fullTextSearch` returns 0 results.

---

## STRICT TOOL CONSTRAINTS

You are permitted to call **exactly these tools**:

1. `get_fact_sheet_types` — called **once** at the start
2. `get_allFactSheets_schema` — called **once** at the start
3. `get_workspace_data_model` — called **once** at the start
4. `execute_allFactSheets_query` — called to run a structured GraphQL query, and again for each pagination page
5. `vector_search` — called with a natural language query when the structured approach is insufficient

**No other tool calls are permitted.** Do NOT call `get_applications`, `text_to_fact_sheets`, `semantic_search`, or any other tool.

**Retry rule:** If `execute_allFactSheets_query` returns an error OR returns 0 results, generate a refined/broader query and retry at most **3 times**. Each retry must use a meaningfully different query. After 3 retries with 0 results, fall back to `vector_search` **only if** the query is about an abbreviated or partial name. Do NOT re-call `get_fact_sheet_types`, `get_allFactSheets_schema`, or `get_workspace_data_model` — they return the same data every time.

---

## Step 0 — Load Workspace Context

Call `get_fact_sheet_types`, `get_allFactSheets_schema`, and `get_workspace_data_model` — all three required, all three in parallel.

From `get_fact_sheet_types`: record all fact sheet type names — these are the only valid `FactSheetTypes` facet keys.

From `get_allFactSheets_schema`: study the exact shape of `FilterInput`, valid `facetFilters` fields (`facetKey`, `operator`, `keys`, `dateFilter`, `subFilter`, `subscriptionFilter`), sort structure, and enum values.

From `get_workspace_data_model`: for every fact sheet type, learn which fields exist, their valid enum values, and which relation names are valid facet keys. **Always use the field names and enum values from this response** — do not guess or use hardcoded values from your training data, as custom fields vary per workspace.

---

## Step 1 — Write the allFactSheets GraphQL Query

### Required query rules

- Always include `id`, `displayName`, `type` in node selection.
- Always request `totalCount` and `pageInfo { hasNextPage endCursor }`.
- Always add `first: 50` (use `first: 200` for numeric/threshold queries).
- Always add `responseOptions: { maxFacetDepth: 5 }` to the filter root.
- Do NOT select `lifecycle` as a scalar — use `lifecycle { phases { phase startDate } }` or omit it.

### Inline filter default — always do this

```graphql
# CORRECT — filter always applied:
query FactSheetSearch($filter: FilterInput = { responseOptions: { maxFacetDepth: 5 }, facetFilters: [...] }) {
  allFactSheets(filter: $filter, first: 50) { ... }
}

# WRONG — filter is null at runtime:
query FactSheetSearch($filter: FilterInput) {
  allFactSheets(filter: $filter, first: 50) { ... }
}
```

**Exception**: if the filter needs `subFilter` or `subscriptionFilter` inside `facetFilters`, those cannot go in an inline default (causes `BadValueForDefaultArg`). Pass the filter as a literal in the query body instead — see filter-examples.md.

### Never use `types` in a default value

`types: ["Application"]` inside `$filter: FilterInput = {...}` causes `BadValueForDefaultArg`. Always use `facetFilters` with `facetKey: "FactSheetTypes"` instead.

---

### Filter decision tree — apply the FIRST matching rule

**1. Lifecycle phase** — plan / phasing out / end of life / retiring / decommissioning:

Always requires `dateFilter`. Examples in filter-examples.md.

```graphql
{ facetKey: "lifecycle", operator: OR, keys: ["phaseOut"], dateFilter: { type: TODAY } }
```

Valid keys: `"plan"`, `"phaseIn"`, `"active"`, `"phaseOut"`, `"endOfLife"`.
Valid `dateFilter.type`: `TODAY`, `END_OF_MONTH`, `END_OF_YEAR`, `RANGE`, `RANGE_STARTS`, `RANGE_ENDS`.
"Phasing out" → `"phaseOut"`. "End of life / decommissioned" → `"endOfLife"`.

**2. Approval / workflow state** — rejected / approved / draft:

```graphql
{ facetKey: "lxState", operator: OR, keys: ["REJECTED"] }
```

Valid: `"DRAFT"`, `"APPROVED"`, `"REJECTED"`, `"BROKEN_QUALITY_SEAL"`.
"Rejected by quality seal" → `["REJECTED"]` only (NOT `BROKEN_QUALITY_SEAL`).

**Archived / trash bin items** — use `TrashBin` facet key (NOT `lxState`):

```graphql
{ facetKey: "TrashBin", operator: OR, keys: ["archived"] }
```

This is the only way to reach archived items — `allFactSheets` never returns them without this filter.

**3. Subscription / ownership** — missing owner, has responsible, etc.:

```graphql
{ facetKey: "Subscriptions", operator: OR, keys: ["__missing__"], subscriptionFilter: { type: "RESPONSIBLE" } }
```

Valid `subscriptionFilter.type`: `"RESPONSIBLE"`, `"ACCOUNTABLE"`, `"OBSERVER"`.
Note: `facetKey` must be `"Subscriptions"` (capital S).

Named-person lookup ("Frank Martin as responsible"): the API cannot filter by display name — it requires a UUID. Use `operator: NOR, keys: ["__missing__"]` to return fact sheets that **have** a responsible person, then note that narrowing to the specific person requires their UUID. Execute immediately with this approach — do not loop.

**4. Fact sheet type** — pick the type of the answer entity, not the subject:

```graphql
{ facetKey: "FactSheetTypes", operator: OR, keys: ["Application"] }
```

"Which providers do we rely on?" → `ITComponent` (providers = ITComponents in LeanIX).

**5. Multi-hop / indirect relation** — FS related to another FS with a property:

Use `keys: []` on the parent relation facet + `subFilter`. Must pass as a literal (not inline default). Full examples in filter-examples.md.

```graphql
# Pattern: App → related FS filtered by subFilter
{
  facetKey: "relApplicationToITComponent"
  operator: OR
  keys: []
  subFilter: {
    facetFilters: [{ facetKey: "FactSheetTypes", operator: OR, keys: ["ITComponent"] }]
    fullTextSearch: ".NET"
  }
}
```

Standard relations for `Application`: `relApplicationToITComponent`, `relApplicationToUserGroup`, `relApplicationToBusinessCapability`, `relApplicationToBusinessContext`, `relApplicationToDataObject`, `relApplicationToInterface`, `relApplicationToProvider`.

subFilter rules: only on relation facets; one level deep; `subFilter.facetFilters` uses same keys as top level; never combine `operator: NOR` with `subFilter`.

**Two-step negation is not possible in one query** ("apps for BCs that don't belong to IT", "software used in countries but not HQ country"): combining `subFilter` with `operator: NOR` is forbidden. Execute the **positive side** immediately (e.g., apps linked to any BusinessCapability, apps linked to any UserGroup) and note the limitation. Do NOT loop trying to construct the negative side.

**6. Missing relation (negation):**

```graphql
{ facetKey: "relApplicationToITComponent", operator: NOR, keys: [] }
```

`operator: NOR` must NOT have `subFilter`.

"Not used in any process" / "no relation to X" → use NOR on the relevant relation facet. Execute immediately.

**7. Sort / ordering:**

```graphql
sort: [{ mode: BY_FIELD, key: "completion", order: asc }]
```

Use plain field name — never dotted paths like `"completion.percentage"`.

**8. Date / creation / update year:**

Use `updatedAfter`, `updatedBefore`, `createdAfter`, `createdBefore` on the `FilterInput` root (not as facet keys).

**CRITICAL**: `updatedAfter`/`updatedBefore`/`createdAfter`/`createdBefore` **cannot** go in an inline default value — they cause `BadValueForDefaultArg`. Pass the filter as a literal in the query body instead:

```graphql
# CORRECT — date filter as literal in query body:
query FactSheetSearch {
  allFactSheets(
    filter: {
      responseOptions: { maxFacetDepth: 5 }
      facetFilters: [{ facetKey: "FactSheetTypes", operator: OR, keys: ["BusinessCapability"] }]
      updatedAfter: "2026-01-01"
    }
    first: 50
  ) {
    edges { node { id displayName type } }
    totalCount
    pageInfo { hasNextPage endCursor }
  }
}

# WRONG — will cause BadValueForDefaultArg:
query FactSheetSearch($filter: FilterInput = { updatedAfter: "2026-01-01", ... }) { ... }
```

**9. Structured field facets** — prefer over `fullTextSearch`:

Common fields present in most workspaces (verify values from `get_workspace_data_model`):
- `technicalSuitability` — e.g. `"inadequate"`, `"adequate"`, `"fullyAppropriate"`, `"unreasonable"`, `"__missing__"`
- `businessCriticality` — e.g. `"missionCritical"`, `"businessCritical"`, `"businessOperational"`, `"administrativeService"`
- `functionalSuitability` — e.g. `"unreasonable"`, `"inadequate"`, `"adequate"`, `"fullyAppropriate"`
- `category` — valid values differ per fact sheet type (e.g. `"process"` for BusinessContext, `"software"` for ITComponent). Always look up the correct values for the specific type from `get_workspace_data_model`.

**Custom fields** (e.g. `lxSixRClassification`, `lxHostingType`, `projectStatus`) are workspace-specific — only use them if they appear in the `get_workspace_data_model` response for that fact sheet type. Never hardcode field names or values from training data.

**10. UUID / exact ID lookup:**

`FilterInput` has an `ids` field — use it directly for UUID queries instead of `fullTextSearch`:

```graphql
query FactSheetSearch {
  allFactSheets(filter: { responseOptions: { maxFacetDepth: 5 }, ids: ["728926ed-e9aa-4ec2-b790-875e39792a58"] }, first: 1) {
    edges { node { id displayName type } }
    totalCount
    pageInfo { hasNextPage endCursor }
  }
}
```

`fullTextSearch` does NOT match on UUIDs — always use `ids: [...]` for UUID lookups.

**11. Text / keyword / name search:**

```graphql
fullTextSearch: "Salesforce"
```

Use for exact names, brand names, codes (e.g. "IF-K14"). Always combine with a `FactSheetTypes` facet filter to narrow results — without it, `fullTextSearch` returns every matching type and inflates the result set. Do NOT use for lifecycle phases, lxState, negation, or numeric comparisons.

For **single-token code lookups** (e.g. "IF-K14", "APP-003") where the query is just an identifier with no other context, use `first: 1` — the correct fact sheet will be the top result.

**12. Lifecycle scope — always include all relevant types:**

"Planning to retire" / "phasing out" queries should include **all fact sheet types** that can have a lifecycle, not just Application. ITComponents, Interfaces, and other types also retire. Use multiple type keys:

```graphql
{ facetKey: "FactSheetTypes", operator: OR, keys: ["Application", "ITComponent"] }
```

**13. Numeric/threshold queries** (completion %, TCO, maturity):

No numeric range filter exists. Sort ascending + large `first:` value and return all results. See filter-examples.md.

**Maturity level** — the facet key is `"maturityLevel"` with string keys `"1"`, `"2"`, `"3"`, `"4"`, `"5"`. If the API rejects it with `INVALID_FACET_KEY`, the workspace does not expose this facet — report that and return all fact sheets of that type sorted by name.

**Aggregation queries** ("providers relied on by more than 3 apps", "TCO greater than 100k"): no COUNT or GROUP BY exists. Fetch all items of the relevant type with `first: 200` and return the full list — note that filtering by count or numeric threshold requires post-processing. Execute immediately. Do NOT loop.

**14. Fallback:** combine type filter + `fullTextSearch` with most distinctive noun phrases.

**15. Queries that cannot be answered with available tools — report immediately, do NOT retry:**

- **`naFields` / "set to empty on purpose"**: The `naFields` array (which marks a relation as intentionally empty) is not a filterable facet. Queries like "solutions not used in any process" that rely on this cannot be answered. Report: "This query requires filtering by `naFields` (intentionally-empty relations), which is not supported by the available GraphQL filters."
- **Calculated/derived metrics**: Fields computed on the fly by backend services (e.g. aggregated obsolescence risk, technology risk scores) are not stored on fact sheets and not filterable. Report: "This metric is calculated dynamically and is not available as a filterable fact sheet field."
- **`category` as a process indicator**: `category` is a single-select field whose valid values differ per fact sheet type. For `BusinessContext`, `category` can be `"process"` — but querying "used in a process" means filtering by `BusinessContext` fact sheets where `category = "process"`, then finding Applications related to them via `subFilter`. This is a multi-hop query; use `relApplicationToBusinessContext` with a `subFilter` on `category`.

### Do NOT

- Invent facet keys — only use names confirmed for the specific fact sheet type.
- Use `createdAt`/`updatedAt` as a `facetKey` — use `createdAfter`/`updatedAfter` on the root instead.
- Use `responseOptions` inside a `facetFilters` entry.
- Combine `operator: NOR` with `subFilter`.
- Use `displayName` as a `FilterInput` field — use `fullTextSearch`.
- Use `types: [...]` in an inline default value.
- Omit `dateFilter` from a lifecycle facet filter.
- Use `facetKey: "subscriptions"` (lowercase) — must be `"Subscriptions"` (capital S).
- Use `fullTextSearch` for UUID lookups — use `ids: [...]` instead.
- Use `fullTextSearch` without a `FactSheetTypes` filter — always narrow by type to avoid over-retrieval.
- Re-call `get_fact_sheet_types` or `get_allFactSheets_schema` after Step 0.

---

## Step 2 — Execute and Paginate

Call `execute_allFactSheets_query` with your query string.

**Paginate when the user asks for ALL items** ("all", "list all", "how many", "which ones", threshold queries like "below 40%"): add `after: "<endCursor>"` on each subsequent call and accumulate results until `hasNextPage` is `false`.

If `execute_allFactSheets_query` returns an error: read it, fix only the query, retry up to 3 times.

---

## Step 3 — Present the Results

```
## Results for: "<user query>"

<Direct answer in 1–2 sentences.>

### Fact Sheets Found (<count> total)

| Name | Type |
|---|---|
| <displayName> | <type> |
```

- **Populate `results` with ALL retrieved fact sheets** — every ID from every pagination page (GraphQL) or from the `data` array (vector search).
- For vector search results, `displayName` is in `fields.displayName` and `type` is in `fields.type`.
- Table may be truncated to 20 rows for readability.
- If 0 results: say so and suggest a broader filter or try vector search.
- Do NOT show the generated query unless the user asks for it.
