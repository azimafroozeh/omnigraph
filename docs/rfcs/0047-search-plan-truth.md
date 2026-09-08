---
rfc: "0047"
title: "Search plan truth: projectable ranking, deterministic order, and loud search failures"
track: public
status: draft
implementation: in-progress
authors:
  - Ragnor Comerford (@ragnorc)
  - Azim Afroozeh (@azimafroozeh)
created: 2026-09-01
updated: 2026-09-08
discussion: "https://github.com/ModernRelay/omnigraph/pull/606"
supersedes: []
superseded_by: []
blocked_on: []
---

# RFC 0047: Search plan truth: projectable ranking, deterministic order, and loud search failures

## Summary

Ranked reads state what they executed, and search constructs that silently
do nothing become errors or warnings:

1. `fuzzy()` is retired with a stable `T26` compile diagnostic. Its hit set
   depends on the length of the stemmed index term and on letter case,
   because Lance tokenizes a fuzzy query without the index analyzer; that is
   not a contract a query can rely on. The first release after acceptance
   reports it as a warning; the next release refuses it.
2. A search filter or rank target on a binding that no scan owns is a stable
   `T27` compile diagnostic. Today the predicate or ranking is silently
   dropped and plausible rows come back in table order. The same rule refuses
   `rrf()` arms on two different bindings (`T28`) and a second text predicate
   on one binding (`T29`), two shapes that today fuse or filter garbage.
3. The executed retrieval is stated once in the lowered plan
   (`QueryIR::retrieval`), where a retrieval is the source that selects and
   ranks a window of rows (`bm25`, `nearest`, or an `rrf()` fusion of the
   two). The executor's re-inference from `order_by[0]` is deleted.
4. `bm25(...)`, `nearest(...)`, and `rrf(...)` become projectable in
   `return`, observing the exact value the ordering used; a projected rank
   expression that is not structurally identical to the executed retrieval
   is `T33`, never a NULL column.
5. Every ranked result has a total order given the pinned snapshot: score,
   then trailing `order` keys inside score ties on single-source reads, then
   stable ids. Membership at the cut is made independent of physical layout
   for `bm25` (a score plateau touching the scan cap retries uncapped); for
   `nearest` the substrate cuts k-boundary ties by row id, and this RFC says
   so instead of claiming otherwise.
6. The canonical read envelope gains two additive arrays: `warnings`
   (first code: `full_text_search_unindexed`, a full-text function on a
   column with no FTS index, which today serves through a case-sensitive
   flat scan) and `retrievals` (one row per executed source, carrying the
   projected column when there is one, the served window, whether the
   under-fill retry ran, and, when the caller asks for it, ready/pending
   embedding coverage for `@embed`-backed vectors).

Boundaries that do not change: no schema surface or storage-format change,
no change to BM25 or vector scoring math, the deprecated `POST /read`
envelope stays byte-stable. Observable row order changes only where scores
tie and where an `order` clause was previously ignored; both are listed
under Compatibility.

## Motivation

Three bug classes on `main` and one structural gap motivate this. The bug
classes are size S/M fixes under `GOVERNANCE.md`; the structural gap and the
new observable surfaces are why an RFC exists.

**Confident false negatives from silent text-search fallback.** On a real
graph `match_text($o.name, "Anthropic")` returns 2 rows while
`"anthropic"` returns 0, with no signal. A full-text function on a column
with no FTS index runs Lance's flat scan with `default_text_tokenizer()`, a
bare `SimpleTokenizer` with no lowercasing, no stemming, and no stop-word
removal, while the indexed path lowercases and stems (Lance 11.0.0
`lance/src/io/exec/fts.rs`, tokenizer selection when no segment exists).
On the engine's own fixture, unindexed `search($d.title, "deep")` and
`"introduction"` return nothing while `"Deep"` and `"Introduction"` match.
The fallback changes matching, not only cost.

**Silently dropped search on non-scan-rooted targets.** A
`search()`/`match_text()` filter or `nearest()`/`bm25()` target whose
binding is introduced by a traversal is removed from the pipeline by the
hoisting pre-pass and never attached to any scan (`exec/query.rs`,
`execute_pipeline` pre-pass): the query returns unfiltered or unranked rows
with no error. The same pre-pass drops a search on an outer binding inside
`not { }`, so "docs not mentioning `$q`" returns no rows. Two more shapes
in this class: `rrf()` arms on two different bindings fuse on the primary
arm's id and read the secondary arm's rank from cross-join row order; a
`search()` filter beside a `bm25()` ordering on the same binding is
replaced by the ordering, because both go through one
`scanner.full_text_search` setter.

**`fuzzy()` is not a contract.** With the fixture's title index,
`fuzzy($d.title, $q, 2)` matches `introductio`, `Deep`, `deep`, `dep`,
`guide`, `Guide`, and misses `Introductio`, `Introduction`, `introduction`,
`learning`. The pattern is edit distance to the stemmed, lowercased index
term (`Introductio` against stem `introduct` is 3 edits; `dep` against
`deep` is 1), because Lance tokenizes a fuzzy query with a bare
`SimpleTokenizer` while the index carries the analyzer. Unindexed,
`fuzzy()` ignores its edit budget entirely.

**Rank is not data.** The compiler types rank expressions as `F64`, but the
executor rejects them in projection; RRF computes a fused score, sorts by
it, then discards it; equal-score orders depend on arrival order; retrieval
is re-discovered from `order_by[0]` at execution. The deny-list in
`docs/dev/invariants.md` rejects "side channels for query semantics or
discarded retrieval rank"; the discarded fused score is that side channel.

An issue-sized fix cannot close the structural gap: it spans the compiler,
the executor, and the public read contract, and the cure requires new
observable surfaces (diagnostics, warnings, response metadata) designed
once. The three bug classes need no RFC and can land as issue-referenced
PRs before acceptance (see Rollout).

## User and operational behavior

**Compile diagnostics.** Every code below is stable and is the `.gqt` pin
for its behavior (RFC 0045 pins diagnostics by code). Codes `T22` to `T25`
are taken on `main`; this RFC allocates `T26` to `T34`. Stage 6a (the
single-source projection PR) took `T35` to `T37` on `main` ahead of
acceptance; the table lists them so no later allocation reuses them.

| Code | Condition | Message shape |
|---|---|---|
| `T26` | any `fuzzy(...)` use | retired: its hit set depends on stem length and letter case; use `search()` or `match_text()` (not typo tolerant) |
| `T27` | a search filter or rank target on a binding no scan owns: (a) introduced by a traversal, (b) bound outside the enclosing `not { }` block | (a) target the scan-rooted binding of this match component; (b) inside a negation a search may target only bindings declared in the block |
| `T28` | `rrf()` arms target different bindings | every arm of one `rrf()` targets the same scan-rooted binding |
| `T29` | a second text predicate on one binding, or a text predicate beside a `bm25` retrieval on the same binding | one text predicate per binding until analyzed composition exists (RFC 0048) |
| `T30` | a rank expression outside `return` and `order` (for example `bm25(...) > 0.5` in `match`) | rank expressions are legal only in `return` and `order` |
| `T31` | an `order` clause whose rank expression is not the leading key, carries a direction, or is an `rrf()` followed by trailing keys | the rank expression leads `order`, takes no direction, and `rrf()` takes no trailing keys |
| `T32` | a rank expression in `order` of an aggregate query, or wrapped by an aggregate in `return` | a retrieval selects the rows an aggregate counts; state the filter instead |
| `T33` | a projected rank expression that is not structurally identical to the executed retrieval | the projection must repeat the retrieval stated in `order` |
| `T34` | `rrf(a, b, k)` with `k` that is not a positive integer literal | `k` is a positive integer literal (today a non-integer parameter silently becomes 60 and a negative one `u32::MAX`) |
| `T35` | a search predicate (`search`, `fuzzy`, `match_text`) in `return` | a search predicate belongs in `match` |
| `T36` | an alias projected a second time (`$d.slug as s, s as t`) | an alias is resolved in `order`, not projected again |
| `T37` | `rrf(...)` in `return` before stage 6b | order by `rrf(...)` and project plain columns; retired when `_fused` lands |

A binding declared twice (`$a: Doc { kind: "x" }  $a: Doc { lang: "en" }`)
is a supported constraint since #605: the second declaration filters the
scan instead of rescanning, so it is not a search shape this RFC refuses.
The engine refuses, rather than drops, every shape above if reached with
hand-built IR.

**Structural identity (`T33`).** The fingerprint of a rank expression is the
tuple (kind, variable, property, query argument), where the query argument is
compared as an AST after parameter references are resolved to their names,
not their values: `bm25($d.body, $q)` and `bm25($d.body, "x")` are
different fingerprints even when `$q = "x"`, so one lowered plan serves every
parameterization. Two `nearest` on one binding with different query
arguments are distinct. A projected `rrf()` fingerprint is the ordered tuple
of its arm fingerprints plus `k`.

**Metric projection.**

```gq
query ranked($q: String) {
  match { $d: Doc }
  return { $d.slug, bm25($d.body, $q) as score }
  order { bm25($d.body, $q) }
  limit 20
}
```

projects the score the ordering used: one computation, observed twice. The
leading `order` key may also be an alias of a projected rank expression
(`return { rrf(...) as fusion } order { fusion }`); lowering resolves the
alias through the projection, and the retrieval is the projected expression.
Projecting an `rrf()` yields the fused score. Projecting an individual arm
inside an `rrf()` query is out of scope here and is specified by RFC 0048
(arm-level projection with a defined value for entities outside the arm's
window).

Default column names when a rank expression is projected without an alias:
`{var}._score` for `bm25`, `{var}._distance` for `nearest`, `{var}._fused`
for `rrf`. The fused score gets its own column so the primary arm's
`_score` is never overwritten. `_score`, `_distance`, and `_fused` are
schema-reserved property names.

**Retrieval selects rows.** A `bm25` ordering restricts the result to the
documents Lance's full-text query matched (a document with no matching
term is absent, not ranked last); a `nearest` ordering restricts it to the
top-k entities by distance, where `k` is the query limit; `rrf()` restricts
it to the union of its arms' windows. `limit` then cuts that set. This is
what `T32` protects: `count($d)` under a search ordering would count a
retrieval-selected subset, so the shape is refused rather than warned about.

**Determinism.** Ranked output order is total given the pinned snapshot:

- single-source reads (`bm25`, `nearest`): score, then trailing `order`
  keys inside score ties, then every binding's id (name-sorted);
- `rrf()` reads: fused score descending, then the fused binding's id; `T31`
  refuses trailing keys after `rrf()` instead of applying them inside ties
  (the tie plateau at the limit boundary would otherwise need retention and
  a post-cut re-sort over fanout rows). RFC 0048 decides whether trailing
  keys return with its `rrf` redesign.

Membership at the cut:

- `bm25`: the single-source scan asks Lance for `limit × 4` rows when the
  query has a limit, no aggregate return, and one order key (the scan cap
  from #563); Lance's plain match path keeps the top-N by score alone and
  drops equal-score boundary candidates by arrival order. When the lowest
  score in the capped batch equals the score at the final cut, the engine
  retries uncapped, so the cut is decided over the full plateau. A trailing
  key removes the cap (a bounded scan would choose which tied rows exist),
  and the query carries the warning `bm25_scan_unbounded`.
- `nearest`: Lance cuts the top-k inside the index by `(distance, row_id)`;
  row id is physical, so which of two entities at equal distance survives
  the k boundary can change after compaction. This RFC states that limit;
  it does not remove it.
- `limit` counts rows after fanout reconstruction (fusion ranks entities,
  a downstream traversal may fan one entity into several rows, and the
  limit slices the final batch), so the cut may split one entity's rows.

**Response envelope (canonical `/query` and stored-query reads; additive).**

```json
"warnings": [
  { "code": "full_text_search_unindexed", "variable": "d", "property": "title",
    "message": "full-text search on Doc.title runs the case-sensitive flat scan: no FTS index (index status: missing; run optimize)" }
],
"retrievals": [
  { "index": 0, "kind": "bm25", "variable": "d", "property": "body",
    "query": "$q", "recall": "exact", "column": "score", "descending": true,
    "window": { "requested": 80, "served": 80, "retried": false } },
  { "index": 1, "kind": "nearest", "variable": "d", "property": "embedding",
    "query": "$q", "recall": "approximate", "column": null, "descending": false,
    "window": { "requested": 20, "served": 20, "retried": true },
    "embedding_coverage": null, "coverage_reason": "not_requested" },
  { "index": 2, "kind": "rrf", "variable": "d", "arms": [0, 1], "k": 60,
    "recall": "approximate", "column": "fusion", "descending": true }
]
```

- Both arrays are omitted when empty (`skip_serializing_if`), so a read with
  no ranked source and no warning is byte-identical to today. Field value
  sets are closed enums in OpenAPI with unknown-string tolerance on
  deserialization: `kind` in {`bm25`, `nearest`, `rrf`}, `recall` in
  {`exact`, `approximate`}, `coverage_reason` in {`not_requested`,
  `not_embed_backed`}.
- `warnings` never change rows, membership, or order. Each row carries
  `code`, `variable`, `property` (null when not per-property), `message`.
  Rows are deduplicated on `(code, variable, property)` and listed in first
  emission order. Warnings belong to the pass whose rows are served: when
  the under-fill retry replaces the first pass's rows, the first pass's
  warnings are discarded with them. Codes this RFC adds:
  `full_text_search_unindexed` (fires per property when the full-text
  function ran on a column whose FTS index covers fewer rows than the
  scanned population, including no index at all), `bm25_scan_unbounded`
  (fires when a trailing key removed the `bm25` scan cap),
  `embedding_coverage_pending` (fires when requested coverage has
  `pending > 0`).
- `retrievals` rows are keyed by `index`, the plan order of the executed
  sources; an `rrf` row references its arms by index. `column` is the
  projected column name (null when the retrieval is not projected) and is
  unique per response (`T25`). `recall` is `approximate` when an
  index-accelerated plan was permitted for the source (a vector index
  existed on the property at execution), `exact` otherwise; an `rrf` row
  reports the weakest of its arms. RFC 0048 replaces this index-derived
  value with the contractual `knn`/`ann` split.
- `window.requested` is the arm's own window (the scan cap for `bm25`, `k`
  for `nearest`); `window.served` is the row count the source returned;
  `window.retried` is true when the engine re-ran the source uncapped
  because the first pass served fewer rows than its window. The retry is
  engine execution policy, keyed on the stage's own window, never on the
  query limit; it is visible here rather than in a log.
- `embedding_coverage` is computed only when the request sets
  `include_coverage: true`; otherwise it is null with `coverage_reason`
  `not_requested`, and null with `not_embed_backed` for a vector property
  with no `@embed`. When computed it is `{ "population": "prefiltered",
  "ready": N, "pending": M, "complete": bool }` where `pending` uses RFC
  0015's pending-row predicate (`vector IS NULL AND source IS NOT NULL`)
  restricted to the population, `complete` is `pending == 0`, and
  `population` is the set after the predicates the engine pushed into the
  scan (an engine choice; it may exceed the query's qualifying population
  when a filter is not pushable). The count is two filtered `id` scans over
  that population per `@embed` retrieval per execution, which is why it is
  opt-in; a bounded source (a per-column pending count maintained by RFC
  0015's reconciler) may make it default-on later, recorded in this
  decision log.
- CLI: full-JSON output carries both arrays in-band; the JSONL metadata
  record carries `warnings`; every human format prints each warning as one
  stderr line `warning [<code>]: <message>` and leaves the exit code
  unchanged. The MCP surface carries the arrays in its structured result.
- The deprecated `POST /read` envelope carries none of these fields.

**Operators** see a `tracing` warning per `full_text_search_unindexed`
occurrence. The message names the RFC 0046 index state and the clearing
command: `missing` (declared, never built) clears with `optimize`; an
uncertified index refuses with HTTP 409 `full_text_index_rebuild_required`
before this warning can fire and clears with `rebuild-full-text-indexes`.
The first release after acceptance warns; the release after it fails closed:
`full_text_search_unindexed` becomes an execution error on `/query` and on
stored-query reads, because the flat fallback changes matching and
invariant 7 says missing coverage may change cost but not correctness. RFC
0048 makes the failure unnecessary by giving the flat path the declared
analyzer; until then the error is the honest posture (the same posture
ParadeDB takes for an unindexed `@@@` search).

**Upgrade gate.** Stored queries are typechecked at server boot; one stored
`fuzzy()` or `T27` shape makes the graph open return an error, the graph is
quarantined, and strict mode aborts startup. So the break unit of `T26` and
`T27` is the server, not the query. Before upgrading, run

```bash
omnigraph queries validate --cluster <dir> --json --deny-warnings
```

with the new binary; exit code 1 lists every stored query that would fail.
`--deny-warnings` is new: today the command exits 1 only on breakages.
During the warning release `T26` is reported by registry validation and
`lint` as warning code `fuzzy_retired` and the query still runs; from the
next release it is the `T26` error.

**Evasion and honest routes** for the rules above:

| Rule | Evasion | What stops it | Honest route |
|---|---|---|---|
| `T26` | keep `fuzzy()` in a stored query and skip validation | boot refuses the graph; `queries validate --deny-warnings` in CI | rewrite to `search()`/`match_text()` and accept the loss of typo tolerance, or wait for an analyzer-correct fuzzy form |
| `T27` | move the search to a `match` filter on the scan-rooted binding after the traversal | that is the correct spelling; the rule accepts it | target the component root |
| `T28`/`T29` | split into two queries and fuse client-side | nothing; the engine no longer produces silent garbage | one binding per `rrf()`; one text predicate per binding until RFC 0048 |
| fail-closed unindexed search | declare `@index` and never run `optimize` | the index is `missing`; the error names `optimize` | declare `@index`, run `optimize` |
| `include_coverage` | poll coverage on every read | the cost is stated; the operator owns it | ask on the reads that need it |

## Design

- **Retrieval in the plan.** `QueryIR` gains `retrieval: Option<RetrievalIR>`
  (`Nearest { variable, property, query, k }`, `Bm25 { variable, property,
  query, scan_cap }`, `FuseRrf { arms: [RetrievalIR; 2], k }`). Lowering
  fixes the retrieval shape once, parameter values and String-query
  embedding stay execution-time. The shape is lowering-owned; execution
  policy is engine-owned: the ANN probe budget (`AnnProbeBudget`, #591) and
  the under-fill retry are applied by the executor over the stated shape
  and reported in `retrievals[].window`, never stored in the IR. Lowering
  asserts `retrieval.is_some()` exactly when `order_by[0]` is a rank
  expression (or an alias resolving to one), and the executor refuses a
  hand-built IR where the two disagree. The executor's `order_by[0]`
  inference is deleted; the resolved retrieval feeds the existing scan and
  fusion machinery, which is what lets the #587 prefilter gate compose
  unchanged.
- **Scan-rooted targets (`T27`).** For each traversal-connected component
  of the match clause, lowering scans exactly one binding, the component
  root, computed by `scan_root_variables`; typecheck's `T27` pass calls
  that function, so the rule and the plan cannot drift. A `not { }` block
  computes its roots over its own bindings; an outer binding is never a
  root inside the block, hence message variant (b). Negated text search
  (`not { search($d.body, $q) }` on an outer `$d`) is refused here;
  supporting it needs Lance `BooleanQuery` `Occur::MustNot` composition
  and is RFC 0048's call.
- **Text predicate composition (`T29`).** Every hoisted text filter and the
  `bm25` retrieval on a binding go through `scanner.full_text_search`, whose
  setter is an assignment. Until composition exists (Lance `BooleanQuery`,
  `Occur::Must` for the filter plus the ranked leaf, RFC 0048 stage 1),
  typecheck refuses the second occupant. A `.gqt` case pins the class with
  a term absent from some rows.
- **Advisories.** Execution threads one notice sink per pass; the served
  pass's sink is the response's `warnings`, deduplicated on `(code,
  variable, property)` across the two `rrf` arms (which run sequentially,
  so emission order is deterministic).
- **Fusion.** The fused score is materialized as `{var}._fused` on the
  fused rows before projection; winner selection is `(fused score desc,
  entity id asc)` with the fused score computed in arm declaration order;
  fanout rows of a winning entity are taken from one arm batch in pipeline
  order; `limit` applies after reconstruction. `rrf` arithmetic today:
  1-based rank within the served window, contribution `1 / (k + rank)`, an
  absent arm contributes 0; RFC 0048 versions this as `rrf_v1`.
- **Coverage.** When requested, ready/pending counts reuse the scan's own
  structured `filter_expr` (DataFusion `ident()`, never `col()`, which
  lowercases unquoted identifiers) through a sealed, streaming count on the
  storage boundary (`TableStore::count_rows_matching`, registered read-only
  in `forbidden_apis`); no SQL strings, no retained batches.
- **Existing process-global dial.** `OMNIGRAPH_ANN_NPROBES` (default 20)
  sets the probe cap for every `nearest` and is not per-query; it stays as
  is here and RFC 0048 absorbs it into its ANN profile.

## Invariants

- **Loud integrity failures (8):** strengthened. The motivating silent
  false negatives and silent drops become diagnostics, warnings, or (after
  the warning release) errors; the `rrf` cross-binding and text-composition
  shapes that produced plausible unranked output are refused.
- **Query semantics are typed structures (9):** strengthened. Retrieval
  moves from execution-time re-inference into the typed lowered plan; the
  fused score becomes an ordinary projected column.
- **Physical acceleration is derived (7):** preserved. The unindexed
  condition warns for one release, then fails closed, because the flat
  fallback changes correctness, which invariant 7 forbids; `recall` is
  reported from the plan the executor was permitted, and RFC 0048 makes it
  contractual.
- **Bounded, observable resource use (11):** the single-source `bm25` scan
  keeps the #563 cap and its retry, and the plateau retry runs at most once;
  a trailing key removes the cap and says so (`bm25_scan_unbounded`); the
  `bm25` arm inside `rrf()` is uncapped today (RFC 0048's `candidates:`
  bounds it); coverage costs two scans and is opt-in; the retry is reported,
  not silent.
- Deny-list: no side channel for discarded rank remains; no new endpoint; no
  string-built predicates; no logical precondition on index coverage
  during the warning release. The fail-closed release introduces one
  precondition on coverage, stated here as the deliberate exception the
  deny-list allows an RFC to argue: a flat scan with a different analyzer
  is a different query, not a slower one.

## Compatibility and reversibility

- **Wire:** both response arrays are additive, serde-defaulted, absent when
  empty; unknown-string tolerance holds for every enum. The legacy `/read`
  envelope is untouched. OpenAPI regenerates. `include_coverage` is a new
  optional request field, default false.
- **Language:** `T26` to `T34` reject queries that previously returned
  wrong or arbitrary rows. `T26` is a warning for one release. The `fuzzy`
  grammar form is retained so the diagnostic is a typecheck error, not a
  parse error; RFC 0048 stage 4 removes the grammar.
- **Order:** single-source ties were run-dependent and become total; an
  aggregate query with a search ordering, previously silently unordered, is
  refused (`T32`); trailing keys after `rrf()`, previously ignored, are
  refused (`T31`); a direction on a rank expression, previously ignored, is
  refused (`T31`). `bm25` membership at a capped plateau, previously
  layout-dependent, becomes content-determined.
- **Migration.** In-repo users of `fuzzy()`: the user docs
  (`docs/user/search/index.md`), the agent skill
  (`skills/omnigraph/SKILL.md`, `references/search.md`), and the engine
  fixture `search.gq`; all are edited in the `T26` stage. The company dev
  graph's stored queries use `nearest` and `search` and are unaffected by
  this RFC.
- **Reverting** requires no storage or format work: the response fields are
  additive, the IR field is internal, and the diagnostics can be relaxed, at
  the cost of restoring the silent-failure classes this exists to remove.

## Alternatives

- **Route `fuzzy()` through the index analyzer instead of retiring it.**
  Not chosen: the stem-then-fuzz form is a new contract (typo tolerance over
  analyzed terms) and belongs with the analyzer profiles of RFC 0048, not in
  a plan-truth RFC. The retirement costs working `fuzzy` users a capability,
  which Compatibility states.
- **Keep warning on unindexed text search forever.** Rejected: the fallback
  serves case-sensitive, unstemmed matches, so a warning leaves invariant 7
  violated. One warning release, then fail closed.
- **A pure `retrieval_of(&QueryIR)` instead of a stored `retrieval`
  field.** The minus-one design: one authority, nothing to drift. Not
  chosen because RFC 0048's N-arm `rrf` with per-arm windows and weights
  is a structure `order_by[0]` cannot carry; the stored field plus the
  lowering assertion and executor refusal give one authority in practice.
- **Retain the boundary tie plateau and apply trailing keys inside fused
  ties.** Rejected in favor of `T31`: it needs a plateau retention bounded
  only by the plateau width, a post-cut re-sort over fanout rows, and a
  float-equality tie test that RFC 0048's weighted N-arm sum invalidates.
  Refusing is loud and reversible.
- **Two arrays (`metrics` for projected columns, `retrievals` for executed
  sources).** Rejected: every metric row was 1:1 with a retrieval row
  except the `rrf` row, `kind` meant two things, and the join key was not
  unique. One keyed array carries the projected column as a field.
- **Compute embedding coverage on every read.** Rejected: two filtered scans
  over the population per `@embed` retrieval per execution violate
  invariant 11 on a `limit 10` read over a million rows.
- **Warn on a search-ordered aggregate instead of refusing it.** Rejected:
  the retrieval still cuts the aggregated population, so the warning "the
  rank was not applied" would hide that the count is over a subset.
- **Silently NULL (or best-effort match) mismatched metric projections.**
  Rejected: a NULL column invites misreading.
- **A separate search/rank response endpoint.** Rejected: one GQ surface.
- **Doing nothing.** The bug classes continue to produce confident wrong
  answers, and rank stays unprojectable.

## Evidence and tests

Query behavior is pinned in `.gqt` logic tests (RFC 0045; the code is the
pin). Cases to add under `crates/omnigraph-gqt/cases/`, each named after
the issue it closes once filed:

- `T26` on inline and stored `fuzzy()`; the warning form in the warning
  release;
- `T27` (a) traversal-introduced target, (b) outer binding inside
  `not { }`; `T28` two-binding `rrf`; `T29` two text predicates and text
  predicate beside `bm25`;
- `T30` to `T34` one case each;
- single-source tie order with `expect ordered` (score, trailing key, id);
  `rrf` order `(fused desc, id asc)` with `expect ordered`, which lifts RFC
  0045's refusal of `ordered` on `rrf()`-led orders (stage 3 adds the
  cross-RFC note to 0045); the aggregate refusal keeps 0045's aggregate
  rule;
- `bm25` plateau at the scan cap: a corpus of 100 equal-score documents,
  `limit 10`, membership asserted against the full plateau;
- projected `bm25`/`nearest`/`rrf` values (12-decimal normalization makes
  them assertable; this lifts RFC 0045's "ranking scores stay unprojected"
  rule, noted there in the same stage).

Rust owners for what the format cannot express: `search.rs` for the
warning carrier (dedup across arms, per-pass sink, `bm25_scan_unbounded`),
the `retrievals` window fields under the retry, coverage counts at the
pinned snapshot under a concurrent write (a failpoint), and characterization
goldens captured before the executor refactor; `rrf_prefilter_gate.rs` (the
`ranked_var_is_expand_dst` fixture becomes the `T27` shape); server
`data_routes`/`openapi` for envelope shape, absence-when-empty
byte-stability of `POST /read`, enum tolerance and the serde-default round
trip; `omnigraph-server` `queries.rs` for the boot refusal and the warning
form; CLI tests for `queries validate --deny-warnings`, stderr carriage, and
the JSONL `warnings` field; `forbidden_apis` for the read-only
classification of `count_rows_matching`.

The prototype branch `search-contracts-p0-p1` @ `3e459aad` (closed PR
#595) demonstrates the retrieval IR, the projectable metrics, and the
warning carrier; it is cited as a demonstration, not as pinned evidence:
its base `0b8481ad` predates #591, #603, #604, #608, #621, the GQ logic
harness, and RFC 0046, its review-fix commit spans several stages, and the
stages are re-cut from the current tree.

## Rollout

Stages 1 to 3 are size S/M under `GOVERNANCE.md` and land as
issue-referenced PRs before acceptance; stages 4 to 7 reference this RFC
once accepted.

1. **Warning carrier** and `full_text_search_unindexed` (engine, API, CLI,
   OpenAPI), with `queries validate --deny-warnings`.
2. **`T27`, `T28`, `T29`** (compiler pass,
   shared `scan_root_variables`, engine backstops); the `rrf_prefilter_gate`
   fixture port lands in the same PR.
3. **Deterministic ties** for single-source reads and `rrf` (`T31`, `T32`),
   the `bm25` plateau retry, `bm25_scan_unbounded`; cross-RFC notes in RFC
   0045.
4. **`T26` fuzzy retirement**: warning form in the first release after
   acceptance, error in the next.
5. **Characterization goldens, then the retrieval-IR refactor**
   (behavior-equivalent on the fixture corpus; goldens pin that corpus).
6. **Projectable metrics**, two PRs: 6a single-source scores (`T33`, the
   `_distance`/`_score` columns, `T32`/`T35`/`T36` at typecheck, `T37` as
   the placeholder refusal); 6b `T30`, `T34`, alias-resolved retrieval,
   `_fused` (retires `T37`).
7. **`retrievals` metadata**, window reporting, opt-in coverage.
8. **Fail closed** on unindexed text search, one release after stage 1
   shipped its warning.

`implementation` advances to `in-progress` at stage 4 and `complete` when
stage 8 ships.

## Unresolved questions

1. Whether embedding coverage becomes default-on from a bounded source.
   Decider: the RFC 0015 owner. Forcing event: a reconciler-maintained
   per-column pending count exists.

## Decision log

- 2026-09-01: RFC opened as the public proposal for the search plan-truth
  slice; prototype PR #595 closed under the size-L rule and retained as a
  demonstration.
- 2026-09-08: stage 6 split into 6a/6b; 6a landed as #687 ahead of
  acceptance and allocated `T32`, `T33`, `T35` to `T37` on `main`
  (implementation `in-progress`). The duplicate-declaration rule dropped:
  a binding declared twice is a supported constraint since #605, so it is
  not a shape this RFC refuses.
- 2026-09-05: review applied. Retirement ground for `fuzzy()` corrected
  (not inert; analyzer bypass); `T26`/`T27` blast radius stated (server
  boot) with the `queries validate` gate; two arrays folded into one keyed
  `retrievals` array; coverage made opt-in; trailing keys after `rrf()`
  and search-ordered aggregates refused instead of plateau retention and a
  warning; the `bm25` plateau retry added; fail-closed on unindexed text
  search scheduled one release after the warning; codes renumbered because
  `main` took `T25` (#621); Lance update-then-optimize fence stage dropped
  (RFC 0043's domain).

## Appendix: agent context (non-normative)

Supporting context for implementers and coding agents. Nothing here is a
contract; the sections above are authoritative.

**Prototype map.** Branch `search-contracts-p0-p1` @ `3e459aad`. The
remainder of the search-contracts program, schema-owned analyzed search
(`@analyzed`), schema-bound vector distance, and the schema feature name
they require, is [RFC 0048](0048-search-contracts.md), blocked on this RFC.

**Substrate facts (Lance 11.0.0, crates.io pin; validate against the pin,
not the GitHub tag).**

- `full_text_search` on a column with no FTS segments plans a flat scan with
  `default_text_tokenizer()` (`lance/src/io/exec/fts.rs`); with segments
  present, including empty ones, the index analyzer is used, and a
  partially covered index analyzes its uncovered tail with the index
  tokenizer, so `full_text_search_unindexed` is keyed on coverage, not on
  artifact presence.
- A fuzzy query is tokenized with a bare `SimpleTokenizer` while index
  terms carry the analyzer (`fts.rs`, `tokenizer_for_match_query`).
- The plain match path compares score alone and drops equal-score boundary
  candidates by arrival order; compound paths tie-break on row id, which is
  not the logical entity id.
- The vector index cuts top-k with a `SortExec` on `(_distance, _rowid)`
  and `fetch = k × refine` (`dataset/scanner.rs`).
- RFC 0043's fail-closed FTS certification gates uncertified indexes; this
  RFC's warning covers absent or partial coverage. Both can fire on one
  graph.
- `search_score_orderings` (#544) already synthesizes `{var}._distance` /
  `{var}._score` orderings; projection resolves the same columns.

**Cross-PR composition.**

- #587 (rrf prefilter gate, merged): composes untouched because the gate
  operates on the resolved retrieval and this RFC only changes where it
  comes from. Its `ranked_var_is_expand_dst` fixture is the `T27` shape.
- #591 (bounded ANN probe expansion, merged after the prototype's base):
  `AnnProbeBudget` is execution policy over the stated shape; the uncapped
  re-run is reported as `window.retried`.
- RFC 0040 (system columns): the `__` namespace is where engine-owned
  metric columns migrate.
