---
rfc: "0048"
title: "Search contracts and retrieval algebra"
track: public
status: draft
implementation: not-started
authors:
  - Ragnor Comerford (@ragnorc)
  - Azim Afroozeh (@azimafroozeh)
created: 2026-09-03
updated: 2026-09-08
discussion: "https://github.com/ModernRelay/omnigraph/pull/606"
supersedes: []
superseded_by: []
blocked_on:
  - "RFC 0047 (search plan truth) acceptance: this RFC extends its retrieval IR, projectable metrics, and the retrievals envelope"
  - "RFC 0040 (system column namespace) acceptance: its schema feature-name rule is how this RFC versions accepted SchemaIR"
  - "RFC 0043 amendment entry: a dated decision-log entry in 0043 recording the three amendments listed under Design, landed with this RFC's acceptance PR"
  - "A checked-in relevance-judgment corpus for the NDCG/MRR/Recall baseline (stage 3)"
  - "Recall/latency evaluation fixing the bounds of the ann_default_v1 profile (stage 3; interim bounds are stated in Design)"
---

# RFC 0048: Search contracts and retrieval algebra

## Summary

Search becomes three separate, composable contracts:

1. **Exact value predicates**: `=`, `starts_with`, String `contains`;
   case-sensitive, never analyzed, on any field.
2. **Analyzed lexical membership**: `match_terms(field, query [, mode:
   all|any])`, legal only on fields that declare analyzed semantics, where
   analyzed means the text is passed through a named analyzer (a tokenizer
   plus lowercasing, folding, stemming and stop-word rules) before
   matching; `mode: all` is the default, so adding a term can only narrow
   a filter used as fact.
3. **Ranked retrieval**: `bm25` (lexical, any-term, positive score), exact
   `knn`, approximate `ann` (with one typed, index-family-agnostic recall
   dial, `oversample: N`, the exact-rescore refine factor), fused by N-arm
   weighted `rrf(arm(source, candidates: N [, weight: W]), …, k: K)`.

The semantics move into schema, versioned and immutable:

- `@analyzed(analyzer="standard_v1" [, scorer="bm25_v1"])` declares analyzed
  matching; the scorer is optional, so a field can be filterable without
  being rankable. A profile is a named, immutable parameter table whose
  fingerprint (a SHA-256 over a canonical byte string defined under Design)
  includes the identity of the Lance tokenizer and stemmer crates, so a
  dependency bump that changes analysis marks the affected indexes
  degraded until rebuilt, never silent drift.
- `Vector(dim, distance="l2"|"cosine"|"dot")` makes geometry schema; there is
  no query-time distance argument anywhere. `Vector(dim)` without a
  distance means `l2`, today's constant.
- `@embed("source", model="…")` records the embedding model per property
  (RFC 0012's identity, already recorded on `main`); the embedding space
  is derived from (model, dimensions, normalization), because equal
  dimensions never prove two spaces compatible and one model name can
  serve several dimensions.

Accepted SchemaIR carries the new semantics under a schema feature name,
`analyzed-search`, per RFC 0040's rule (a name, never a version number).
There is no rebuild: a new binary reads existing graphs by stated implicit
rules, the declarations arrive through ordinary `schema apply` as additive
metadata steps, and an older binary refuses a graph carrying the name.

The ambiguous surfaces retire: `search`/`match_text` get one deprecation
release as `match_terms(..., mode: any)`; `nearest` is a deprecated alias of
`ann`, removed at stage 4; positional `rrf(a, b, k)` is deprecated with a
mechanical rewrite whose loss is stated in the warning; `fuzzy` (retired by
RFC 0047) loses its grammar at stage 4. Projected metrics gain typed domains
(`Score<bm25_v1>`, `Distance<l2|cosine|dot>`, `Score<rrf_v1>`) that refuse
cross-domain comparison, aggregates, and raw-score thresholds.

Boundaries that do not change: BM25 math stays pinned to the substrate's
`k1 = 1.2`, `b = 0.75`, and IDF
`ln(1 + (N − n + 0.5) / (n + 0.5))`; one `/query` surface; lexical
membership and `knn` results are identical across every physical index
state, and BM25 scores are identical at full index coverage; graph
publication, branches, commit history, and recovery are untouched.

## Motivation

RFC 0047 makes today's search surface honest; this RFC makes it right. The
residual problems are structural and cannot be fixed without schema-owned
semantics:

- **Analyzed matching has no owner.** Whether text matches
  case-insensitively today depends on whether a physical FTS index exists;
  RFC 0047 warns and then fails closed on that cliff; only a schema-declared
  analyzer removes it. The substrate's stemmer replacement (Lance 10 to 11,
  `rust-stemmers` to `frostem`) changed the implementation behind identical
  parameters; RFC 0043 answered that at the artifact level with a code
  constant, and this RFC gives the same identity a schema home.
- **Recall is not in the contract.** `nearest` does not say whether
  approximate recall is permitted; vector geometry is a hard-coded engine
  constant (L2); a caller cannot ask for exact top-k on purpose.
- **The precision/recall choice is stage-specific.** Identity lookups want
  exact predicates; factual filters want all-term analyzed membership;
  candidate retrieval wants any-term ranking; fusion wants explicit windows
  and weights. One overloaded function cannot mean all four.
- **Fusion is under-specified.** Two unweighted arms, a `k` that silently
  falls back to 60 on a non-integer parameter, and arm depths inherited
  from the final limit make recall and cost unreviewable.

An issue-sized change cannot deliver this: it spans the schema language,
accepted SchemaIR, the query grammar, and the planner's capability model.

## User and operational behavior

**Schema.**

```pg
node Organization {
  slug: String @key                                     // exact only
  name: String @index @analyzed(analyzer="standard_folded_v1", scorer="bm25_v1")
  notes: String? @analyzed                              // filterable, not rankable
  embedding: Vector(1536, distance="cosine")?
    @embed("name", model="openai/text-embedding-3-small") @index
}
```

Type and annotation rules (all refused at schema apply with a catalog error
naming the property):

| Declaration | Meaning |
|---|---|
| `String @analyzed` (bare) | profile `standard_v1`, no scorer |
| `String @analyzed(analyzer=…)` | that profile; unknown name refused |
| `String @analyzed(…, scorer="bm25_v1")` | rankable with `bm25` |
| `String @index @analyzed(…)` | a physical FTS index is requested; reconciliation populates it |
| `String @analyzed(…)` without `@index` | an empty index shell carries the analyzer (Design); every query is an analyzer-correct flat scan; RFC 0047's `full_text_search_unindexed` fires on coverage |
| `@analyzed` on a non-String, a list, or an edge property | refused (text search runs on node String properties; `T23`) |
| `Vector(N)` | `distance="l2"` |
| `Vector(N, distance=…)` outside {`l2`, `cosine`, `dot`} | refused |
| `Vector` without `@embed` | no embedding space; raw-vector queries only (Design, law 7) |

Initial immutable analyzer profiles, with their full parameter tables in the
appendix: `standard_v1` (simple tokenizer, lowercase, no folding, no
stemming, no stop words; the safe default), `standard_folded_v1` (adds ASCII
folding), `english_v1` (lowercase, ASCII folding, English stemming, English
stop words, 40-character token cap: exactly today's index build, Lance
`InvertedIndexParams::default()`). Adding a profile or scorer version
requires an RFC; none is ever mutated. Changing a field's profile marks its
index `analyzer_profile_mismatch` (RFC 0046 `degraded` reason) until
`rebuild-indexes` runs. Query-time analyzer, scorer, or distance overrides
do not exist.

**Queries.**

```gq
query hybrid($q: String) {
  match {
    $d: Doc
    match_terms($d.title, $q, mode: any)
  }
  return {
    $d.slug,
    bm25($d.body, $q) as lexical_score,
    ann($d.embedding, $q, oversample: 4) as semantic_distance,
    rrf(arm(ann($d.embedding, $q, oversample: 4), candidates: 100),
        arm(bm25($d.body, $q), candidates: 100, weight: 1.5),
        k: 60) as fusion
  }
  order { fusion }
  limit 20
}
```

The example is legal under RFC 0047's rules as amended by this RFC: the
retrieval is stated by the leading `order` key, which is an alias resolved
through the projection (0047); the `match_terms` filter and the `bm25`
retrieval on `$d` compose (this RFC lifts 0047's `T29`, stage 1); arm-level
projection is legal (stage 2), and for an entity outside an arm's window
the projected value is null (`bm25` for a document the lexical window did
not reach, `ann`/`knn` for one the vector window did not reach), with the
column descriptor marked nullable. `order { fusion }` takes no direction and
no trailing keys (0047's `T31`); the direction is the domain's (Design).

Named arguments, one table per function. Values are literals only, checked
at typecheck; a parameter in a named-argument position is `T41`.

| Function | Argument | Production | Domain | Default |
|---|---|---|---|---|
| `match_terms(field, query, mode:)` | `mode` | `all` or `any` (keyword) | | `all` |
| `bm25(field, query)` | | | | |
| `knn(field, query)` | | | | |
| `ann(field, query, oversample:)` | `oversample` | integer literal | 1..=16 (profile `ann_default_v1`) | 1 |
| `arm(source, candidates:, weight:)` | `candidates` | integer literal | 1..=10000 | required |
| | `weight` | float or integer literal | > 0, finite | 1.0 |
| `rrf(arm, …, k:)` | arms | 2 to 16 `arm(...)` | | |
| | `k` | integer literal | ≥ 1 | 60 |

Product bound: `candidates × oversample ≤ 20000` per `ann` arm (interim,
from a 240 MiB raw-vector budget at 3072 dimensions; the `ann_default_v1`
evaluation may move it). Grammar production added: `named_arg = ident ":"
(integer | float | ident)`; an unknown argument name, a positional spelling
of a named argument, `oversample` on `knn`, or an out-of-domain value is
`T41`.

- A String query argument to `knn`/`ann` is legal only when the field's
  `@embed` records a model (else `T40`) and the resolved query embedder
  matches it exactly (else execution error `embedding_model_mismatch`, as
  on `main` today); a raw Vector argument is an explicit same-space
  assertion by the caller.
- `oversample` is the refine factor: the index fetches `k × oversample`
  candidates and rescores them exactly from raw vectors, so widening it can
  only widen the exactly re-scored window. Probe policy is owned by the
  profile and never appears in the query language; `OMNIGRAPH_ANN_NPROBES`
  (RFC 0047 names it) is absorbed into `ann_default_v1` at stage 2.
- `knn` is exact under every index state (`scanner.use_index(false)`);
  `ann` reports `approximate` as its contract even when the plan happened
  to run exactly, and evaluates exactly any population segment not covered
  by a compatible artifact, never failing and never building an index
  inline. Both carry a cost signal: warning `knn_flat_scan` or
  `ann_uncovered_scan` when the exactly scanned rows exceed the profile's
  threshold (interim 100,000 rows), and the RFC 0047 `retrievals` row
  reports `index_coverage: { "covered": N, "uncovered": M }`.
- A distance-incompatible or space-incompatible ANN artifact is treated as
  no coverage: exact fallback with `coverage_reason: "artifact_incompatible"`
  on the row. It is never an error, because an error there would be a
  logical precondition on physical state.
- For `@embed` fields, ready/pending representation coverage (RFC 0047's
  opt-in mechanism) is reported per source; pending rows are missing data,
  never an approximation.

**Errors and warnings**, each with its code, phase, and pin:

| Condition | Phase | Code | Pin |
|---|---|---|---|
| `match_terms` on a field without `@analyzed` | typecheck | `T38` | `.gqt` |
| `bm25` on a field without a BM25-family scorer | typecheck | `T39` | `.gqt` |
| String vector query on a field with no recorded model | typecheck | `T40` | `.gqt` |
| named-argument production, domain, or product-bound violation; arm count; `oversample` on `knn` | typecheck | `T41` | `.gqt` |
| typed-domain violation (cross-domain compare, aggregate over a metric, raw threshold) | typecheck | `T42` | `.gqt` |
| query embedder differs from the recorded model | execution | `embedding_model_mismatch` | Rust (needs a configured embedder) |
| analysis yields zero searchable terms (stop words only, empty query) | execution | warning `match_terms_no_terms`; the predicate matches no rows | `.gqt` |
| exact scan above the profile threshold | execution | warnings `knn_flat_scan`, `ann_uncovered_scan` | Rust (scale) |
| deprecated spelling | typecheck (lint, `queries validate`) | warnings `deprecated_search`, `deprecated_match_text`, `deprecated_nearest`, `deprecated_positional_rrf`, `fuzzy_retired` | `.gqt` |

A zero-term analysis is a warning and an empty match, not an error: a
stored query with an empty `$q` must not fail at runtime where `search`
returned nothing today, and the warning names the analyzed input.

**Deprecation timeline.** Warnings in the first minor release after this
RFC's acceptance (v0.12.0 if accepted before it ships); removal in the
following minor release, advertised as breaking. `omnigraph lint` and
`omnigraph queries validate --cluster <dir>` report every deprecated
spelling with its code; `--deny-warnings` makes either exit 1 on any
warning so CI catches the whole set during the warning release.

Migration per (function, field state); the rewrite tool applies the
mechanical rows and reports the others:

| Today | Field state | Rewrite | Behavior change |
|---|---|---|---|
| `search(f, q)`, `match_text(f, q)` | `@index`/`@key` String (implicit `english_v1`) | `match_terms(f, q, mode: any)` | none |
| `search(f, q)`, `match_text(f, q)` | plain String, no index | reported, not rewritten: add `@analyzed` to the field first; `match_terms` on it is `T38` | the flat fallback that RFC 0047 fails closed on |
| `bm25(f, q)` | `@index` String | unchanged (implicit `bm25_v1`) | none |
| `bm25(f, q)` | plain String | reported: add `@analyzed(…, scorer="bm25_v1")` | |
| `nearest(f, q)` | any Vector | `ann(f, q)` | none (`recall` becomes contractual) |
| `rrf(nearest(...), bm25(...), k)` | | `rrf(arm(ann(...), candidates: <limit>), arm(bm25(...), candidates: 10000), k: <k>)` | the `ann` arm is faithful (today's window is the limit); the `bm25` arm is uncapped today, so a corpus with more than 10,000 matches loses candidates, and the warning says so |
| `fuzzy(f, q, n)` | | reported (RFC 0047 `T26`) | typo tolerance is not offered |

Consumers by stage: warning stage rewrites the user docs
(`docs/user/search/index.md`, `queries/index.md`, `search/embeddings.md`),
the agent skill (`skills/omnigraph/SKILL.md`, `references/queries.md`,
`references/search.md`, so agent-written queries are not born deprecated),
the engine fixture `search.gq`, the `.gqt` corpus bodies (file names keep
their issue numbers), `benches/scenarios.rs`, the server stored-query
fixtures, and the company dev graph's `search_*` stored queries. The
cross-version upgrade test (`crossversion_upgrade.rs`) keeps the old
spelling on the old-binary side by construction. The DST harness issues no
search query today; stage 3 adds a search operation so "identical across
every physical index state" has a simulation witness.

**Operators** cross no format boundary. On a new binary an existing graph
reads under the implicit rules: an `@index`/`@key` String is `english_v1`
with `bm25_v1` (its index already carries that analysis), a Vector is `l2`,
an `@embed` field's model is the one recorded per property. Declaring the
intent explicitly is an ordinary `schema apply`; a rewrite tool produces
the explicit schema for review:

```bash
omnigraph schema rewrite --schema schema.pg --out schema.explicit.pg --json
```

Output: the rewritten `.pg` plus a JSON `reasons` list per property
(`english_v1 because @index`, `l2 default`, `model from recorded
identity`). Exit 0 when every property is resolved; exit 1 with the list of
`@embed` fields that have no recorded model (the one population the tool
cannot resolve; the operator declares `model=` for them or leaves them
raw-vector only); exit 2 on a parse error. Stored queries are rewritten by
the parsed `.gq` rewrite tool RFC 0040 specifies, extended with the table
above:

```bash
omnigraph queries rewrite --cluster <dir> --json [--write]
```

`schema show` reports each analyzed field's profile and fingerprint, and
`index status` (RFC 0046) gains a `profile` column and the
`analyzer_profile_mismatch` reason. The full-text rebuild command
generalizes to a `rebuild-indexes` family (`--type`, `--property`,
`--kind fts|vector` selectors), schema-profile-targeted, extended to
vector artifacts under the same certified-rebuild pattern RFC 0043
established; `rebuild-full-text-indexes` stays as an alias for one release.

**Evasion and honest routes:**

| Rule | Evasion | What stops it | Honest route |
|---|---|---|---|
| `T38` on plain Strings | keep `search()` through the warning release | removal release refuses it; `--deny-warnings` in CI | add `@analyzed`; run `schema apply` |
| deprecation warnings | never run `lint` or `queries validate` | boot-time registry validation refuses at removal | `queries validate --deny-warnings` in CI |
| `@embed` with no recorded model | declare any `model=` to silence the tool | the declared model is checked against the query embedder at execution | declare the model that produced the vectors, or keep the field raw-vector only |
| profile change without rebuild | edit `analyzer=` and skip `rebuild-indexes` | the index is `degraded: analyzer_profile_mismatch` and refuses per RFC 0043 | rebuild |

## Design

**Design laws** (normative; each names its enforcement point):

1. Retrievers select and rank a window; predicates decide the population
   the window is drawn from; projection never changes membership, window,
   or order. Enforcement: lowering (retrieval stated once, RFC 0047) and
   `T30`/`T32`.
2. Filters precede retrieval: `limit k` means the best k within the
   qualifying population. Enforcement today: the #587 prefilter gate pushes
   a traversal-derived population into `bm25` arms as a structured `Expr`
   in-list up to 100,000 ids and 10 % of the table, else the postfilter plan
   with the uncapped retry. For `knn`/`ann` the law holds by construction
   only with a row-mask prefilter on the vector scan (stage 2; a substrate
   dependency on Lance's prefilter path); until then a vector arm inside a
   traversal-derived population ranks the corpus and the traversal filters,
   which the `retrievals` row reports as `prefiltered: false`.
3. Bounds do not leak across stages: final limit, arm candidate windows,
   and future reranker inputs are distinct values. Enforcement: `T41` and
   the under-fill retry keyed on the arm's own window (RFC 0047).
4. Metric identity is explicit: RFC 0047's structural fingerprints,
   extended with typed domains `Score<bm25_v1>`, `Distance<l2|cosine|dot>`,
   `Score<rrf_v1>`. Placement: a domain wrapper on the compiler's
   `ResolvedType`, so `T7` (comparison), `T8` (aggregate operands), and
   `T21` (rrf arm types) read the domain; enforcement `T42`. Allowed and
   refused operations:

   | Operation | Score / Distance |
   |---|---|
   | project in `return` | allowed |
   | lead `order` (direction is the domain's: `Score` descending, `Distance` ascending; a written direction is `T31`) | allowed |
   | compare with a literal or a parameter (`> 0.5`) | `T42` |
   | compare two metrics, same or different domain | `T42` |
   | aggregate (`avg`, `max`, `count` over a metric) | `T42` |
   | `is null` | allowed (arm-level projection yields null outside the window) |
   | parameter of a metric type | no such parameter type exists |
   | arithmetic | GQ has no arithmetic operators |

   The descriptor surface gains `domain` (`QueryResultFieldDescriptor`),
   and the RFC 0047 `retrievals` row gains `domain`; both are additive.
5. Physical state cannot weaken an exact contract: lexical membership,
   `knn`, and BM25 at full coverage are index-independent; only `ann`
   advertises approximation, and artifact absence improves it to exact.
   Enforcement: the index shell (below) and `knn`'s `use_index(false)`.
6. Defaults are immutable contracts: profiles, directions, tie rules, and
   `rrf_v1` change only under a new versioned name. `rrf_v1`: fused score
   `Σ_i w_i / (k + rank_i)` over arms in declaration order, `rank_i` 1-based
   within arm i's served window, an absent arm contributes 0; a tie is
   equality of the two `f64` sums computed in that fixed order; bounds as
   in the argument table. Distance formulas are what Lance 11 computes:

   | `distance` | Value | Range | Nearest |
   |---|---|---|---|
   | `l2` | squared Euclidean distance (no square root) | [0, ∞) | smallest |
   | `cosine` | `1 − cos(x, y)` | [0, 2] | smallest |
   | `dot` | `1 − x · y` | unbounded | smallest |

   `VectorSpec` carries the `lance-linalg` crate version as the distance
   implementation identity; a change re-marks vector artifacts
   `degraded` like an analyzer change does. Enforcement: golden tests per
   formula and the fingerprint guard.
7. Coordinates have a declared space; equal dimensions are not evidence.
   `embedding_space := sha256(model, dimensions, normalization)` for
   `@embed` fields (one model at two dimensions is two spaces); a Vector
   without `@embed` has `embedding_space: None`, which means
   caller-asserted: `knn`/`ann` accept raw vectors only and a String query
   is `T40`. Enforcement: typecheck and the capability probe.
8. The planner cannot guess capabilities: exact/ANN support, distance,
   coverage, and budget ranges come from a typed capability probe over
   observable substrate state (built on RFC 0046's index status);
   a missing fact means exact fallback with a stated reason, never a
   heuristic downgrade and never an error on physical state. Enforcement:
   the planner probe.
9. Recall dials are typed, family-agnostic, and monotone: `oversample`
   only. The under-fill retry (RFC 0047) is an escalation step inside the
   `approximate` contract: `ann_default_v1` states it (retry uncapped when
   the arm served fewer rows than its window), so recall is a function of
   the dial plus one stated step, reported as `window.retried`.
   Enforcement: the profile definition and the `retrievals` row.
10. Profile identity includes substrate behavior identity; a substrate
    change that alters analysis re-marks indexes degraded until rebuilt or
    until parity evidence is recorded. Enforcement: the fingerprint guard
    and the open-time check below.

**Accepted SchemaIR** gains logical search semantics only, never physical
index state: resolved analyzer/scorer profiles with their fingerprints,
`VectorSpec { dimensions, distance, distance_impl, embedding_space }`, and
the feature name `analyzed-search` registered beside this RFC's registry
row per RFC 0040. Three additive annotation kinds (`@analyzed`, `distance=`,
`model=`) are classified by the schema planner's `annotation_change_kind`
as metadata-only steps (no table rewrite, no index build), the one planner
rule that refuses them today.

**Profile fingerprint.** Preimage: the line `analyzer_profile_fingerprint/v1`
followed by sorted `key=value` lines, one per parameter: `name`,
`tokenizer`, `lowercase`, `ascii_folding`, `stemmer` (`none` or the
algorithm), `stop_words` (`none` or the SHA-256 of the sorted list),
`max_token_length`, `normalization`, `tokenizer_crate`,
`tokenizer_version`, `stemmer_crate`, `stemmer_version`; newline
terminated; the fingerprint is the SHA-256 hex of that byte string. The
crate versions come from a build-time constant generated by `build.rs`
from `Cargo.lock`, and a CI guard test asserts the constant equals the
lock file. A golden test pins the hex per shipped profile. Identity is by
content and name together: two profiles with equal tables and different
names have different fingerprints.

**Open-time check and mismatch procedure.** At graph open the running
binary recomputes each accepted profile's fingerprint. A mismatch on a
field marks that field's FTS index `degraded: analyzer_profile_mismatch`
(RFC 0046) and queries on it refuse per RFC 0043's fail-closed rule; the
graph opens, other fields serve, `rebuild-indexes` clears the state. A
graph never refuses to open over a fingerprint. Parity evidence (a matched
set fixture run under the old and new substrate, recorded beside the
certificate) lets an operator clear the state without a rebuild.

**Amendments to RFC 0043** (recorded in 0043's decision log with this
RFC's acceptance): (1) the rebuild targets the field's accepted profile
instead of engine-default English analysis; (2) the certificate's analyzer
generation records the profile fingerprint, so `ANALYZER_GENERATION` as a
hand-bumped code constant is replaced by the generated substrate identity;
(3) the one authority is the accepted profile, the certificate is the
derived proof. Full-text indexes stay node-only (`@analyzed` on edge
properties is refused).

**Index shell at `@analyzed` acceptance.** Accepting `@analyzed` on a field
creates an empty FTS index carrying the resolved analyzer, so the flat path
never falls back to the substrate's default tokenizer. Procedure, on the
branch the `schema apply` targets: (1) the planner classifies the step as
metadata-only; (2) the engine creates the index shell with Lance
`train(false)`, which persists the analyzer parameters on an empty index
(Lance 11 writes `params` on every index, including empty ones); (3) the
RFC 0043 certificate is written at creation with the profile fingerprint;
(4) failure of (2) or (3) fails the apply atomically, no schema change;
(5) a lazy fork shares main's datasets and inherits the shell; a merge
leaves population to reconciliation; export/load recreates the shell from
the schema on load. This is not a synchronous FTS build on a content-write
path: the shell holds no postings, and population stays with explicit
reconciliation (`optimize`), which today skips empty tables and gains this
build site.

**Analyzed composition.** `match_terms` filters and a `bm25` retrieval on
one binding compose through Lance `BooleanQuery` (`Occur::Must` for each
filter, the ranked leaf for the retrieval); this lifts RFC 0047's `T29`.
Negated analyzed search (`not { match_terms(...) }` on an outer binding)
composes as `Occur::MustNot` and lifts 0047's `T27` variant (b) for
`match_terms` only.

**Extension model.** A new retriever is a `RetrievalIR` source variant that
participates in fusion through the shared arm production; a new fusion
method consumes the same ranked-stream shape; a reranker is a
stream-to-stream stage. New behavior never arrives as a mode flag on an
unrelated function. Every retriever inherits `T27` and `T28`.

## Invariants

Strengthens invariants 5 to 9 and 11: semantics move into typed schema and
IR structures; physical acceleration stays derived (an index's absence
changes cost, never matching semantics, closing the cliff RFC 0047 could
only fail closed on); integrity failures are loud; planner facts are
explicit; resource use stays bounded (fusion windows capped, `oversample`
bounded, the product bounded, exact scans signalled). Deny-list check: no
inline index builds on write paths (the shell holds no postings); no side
channels for rank; no string-built predicates; no logical precondition on
physical coverage (incompatible artifacts fall back, never error); no
per-query substrate knobs; no second search endpoint; no shadow analyzer
authority (accepted SchemaIR is the one source; certificates are derived
proof).

## Compatibility and reversibility

- **Format:** none. SchemaIR gains the feature name `analyzed-search`; a
  binary that does not know the name refuses the graph, which is RFC 0040's
  normal forward-compatibility refusal; no manifest stamp moves. The
  grammar of `.pg` gains keyword-only annotation arguments
  (`@analyzed(analyzer="x")`) and `Vector(N, distance="…")`. Old binary
  reading new schema text: both are parse errors; bare `@analyzed` parses
  today and is retained as opaque metadata with no behavior, so the feature
  name on the graph, not the text, is what refuses. New binary reading old
  text: implicit rules above, no change on open.
- **Wire:** additive only; the RFC 0047 `retrievals` row and the result
  descriptor gain `domain`, `index_coverage`, `prefiltered`.
- **Query language:** staged deprecations as above with codes; the boot-time
  registry validation and `lint` surface every deprecated spelling.
- **Compatibility of behavior:** `english_v1` is today's index analysis, so
  rewritten `search` queries on indexed fields return the same rows;
  `bm25` scores are unchanged at full coverage; `nearest` to `ann` changes
  no rows and no order.
- **Reversibility:** grammar and annotations are reversible before 1.0;
  profile parameters, distance formulas, and exact/approximate meanings are
  deliberate near-permanent commitments, made explicit by versioned names.
  Reverting the feature name is an ordinary schema step, not a rebuild.

## Alternatives

- **A storage-format boundary crossed by export/init/load** (the original
  draft). Rejected: `main` already versions schema state and adds
  properties through `schema apply`; the only in-place obstacle is the
  planner's `annotation_change_kind` refusing non-metadata annotation
  changes, and widening it for three additive annotations is the design one
  step smaller. The rebuild would have dropped branches, commit history,
  and indexes for every graph to recover a fact (the embedding model) that
  RFC 0012 already records.
- **A SchemaIR version scalar** for the new fields. Rejected: RFC 0040 rules
  that later schema features mint a feature name, and 0044 defers to it.
- **Let index presence decide recall.** Rejected: silent result changes when
  an index appears; the pgvector README documents that approximate indexes
  return different results from the exact scan.
- **Query-time distance or analyzer arguments.** Rejected: flat and indexed
  paths diverge and stored queries silently change meaning; PostgreSQL's
  full-text documentation steers users from query-time configurations to
  generated `tsvector` columns for the same reason.
- **A single overloaded search function with option flags.** Rejected:
  reproduces the ambiguity this RFC removes.
- **Any-term default for the analyzed predicate.** Rejected: a predicate
  consumed as fact must narrow when a term is added.
- **Raw score blending for fusion.** Rejected: BM25 and distance scales are
  not comparable.
- **Raw score thresholds.** Rejected: scores are corpus- and
  version-relative.
- **Per-query substrate knobs (`ef`, `nprobes`).** Rejected: couples stored
  queries to one index family; the concession is the typed `oversample`.
- **Error on a distance-incompatible artifact.** Rejected: a precondition on
  physical state; fallback with a stated reason keeps laws 5 and 8.
- **Error on zero analyzed terms.** Rejected: a stored query with a
  stop-word query would fail at runtime where it returns nothing today.
- **A rebuild-free path only for graphs with recorded models, rebuild for
  the rest.** Subsumed: the unrecorded-model population stays raw-vector
  only until the operator declares a model; nothing else needs a rebuild.
- **Doing nothing beyond RFC 0047.** The case-sensitivity cliff stays
  failed-closed rather than fixed, recall stays implicit, and fusion stays
  unreviewable.

## Evidence and tests

`.gqt` cases (RFC 0045): `T38` to `T42` one each; `mode: all` default and
its narrowing (a two-term query returns a subset of the one-term query);
exact predicates never analyzed on an `@analyzed` field; membership parity
across the coverage-state matrix (no index, full, partial after a write)
for `match_terms`; `knn` parity across the same matrix; `bm25` score parity
at full coverage; `rrf_v1` arithmetic with three arms and weights (fixed
summation order); `T27`/`T28` inheritance by `match_terms`, `knn`, `ann`;
the rewrite equivalence `search(f, q)` before equals `match_terms(f, q,
mode: any)` after under `english_v1`; `match_terms_no_terms`; each
deprecation code. This extends RFC 0045's list of index-decision
constructs (`match_terms`, `knn`, `ann` added; `fuzzy` removed), noted
there in stage 1.

Rust owners: compiler suites for the annotation grammar, the named-argument
production, the domain wrapper, and the rewrite tools (idempotence, the
reasons list, exit codes); engine `search.rs` for profile behavior
(case/folding/stemming matrices per profile), `ann` fallback and refine
witnesses, the exact-scan warnings, `index_coverage`, and the incompatible
artifact fallback; the fingerprint golden per profile and the
`Cargo.lock` guard; `table_store` tests for the index shell (empty index
persists params, certificate at creation, apply fails atomically) and the
schema planner's additive classification; format suites for the feature
name refusal matrix; the DST search operation (stage 3); a checked-in
relevance corpus reporting NDCG@10, MRR@10, Recall@100 per modality plus a
live recall probe (`ann` versus `knn` on one filtered population) as a
maintenance operation. The `ann_default_v1` bounds are fixed by that
evaluation.

Substrate facts this RFC relies on were verified against the Lance 11.0.0
crate source and are listed in the appendix; the stemmer replacement
(`rust-stemmers` to `frostem`) is verified from `Cargo.lock`, and the
matched-set measurement behind "changed matched sets" is not in the
repository, so the Motivation states only the replacement.

## Rollout

Stages 1 and 2 ship in one release; the accepted SchemaIR of stage 1
already carries `VectorSpec`, so stage 2 mints nothing.

1. **Lexical contract:** feature name `analyzed-search`, the planner's
   additive classification, `@analyzed` profiles with fingerprints and the
   open-time check, the index shell, `match_terms`, `BooleanQuery`
   composition (lifts 0047's `T29`), the `rebuild-indexes` generalization,
   the RFC 0043 amendment entry, `schema rewrite`, deprecation warnings for
   `search`/`match_text`.
2. **Ranked contract:** `Vector(distance=)`, `embedding_space`, `knn`,
   `ann(oversample:)`, N-arm weighted `rrf`, arm-level projection, typed
   domains (`T42`), `nearest` deprecation, `queries rewrite`,
   `OMNIGRAPH_ANN_NPROBES` absorbed.
3. **Qualification:** relevance corpus and baseline, `ann_default_v1`
   bounds, capability-probe health on RFC 0046's surface, vector-artifact
   certification, the DST search operation.
4. **Breaking release:** deprecated grammar removed (`search`,
   `match_text`, `nearest`, positional `rrf`, `fuzzy`).

`implementation` advances per stage.

## Unresolved questions

1. Whether `standard_v1` stays on the substrate's simple tokenizer or waits
   for a pinned ICU word-break profile. Decider: the RFC author. Forcing
   event: multilingual matched-set fixtures in the relevance corpus
   (stage 3).
2. Whether mutable provider model aliases need a revision suffix in
   `embedding_space`. Decider: the RFC 0012 owner. Forcing event: the first
   provider alias observed to change dimensions or normalization.

## Decision log

- 2026-09-03: published as public draft RFC 0048 alongside RFC 0047 for
  review under the RFC-first process.
- 2026-09-08: diagnostic codes renumbered `T38` to `T42` (were `T35` to
  `T39`): #687 allocated `T32`, `T33`, `T35` to `T37` on `main` for RFC
  0047's stage 6a.
- 2026-09-05: review applied. Format boundary replaced by an additive
  schema feature name per RFC 0040 (no rebuild); RFC 0043 amendments
  declared; the fingerprint preimage, profile tables, distance formulas,
  and `rrf_v1` arithmetic written; named arguments made literal-only with
  codes; the `hybrid` example made legal under 0047 (alias-resolved
  retrieval, composition, arm-level projection with null outside the
  window); incompatible artifacts fall back instead of erroring; zero-term
  analysis warns instead of erroring; migration and consumer tables added;
  `nearest` given one fate (deprecated alias, removed at stage 4).

## Appendix: agent context (non-normative)

Supporting context for implementers and coding agents; the sections above
are authoritative.

**Profile parameter tables** (values are Lance 11 `InvertedIndexParams`
fields; `english_v1` equals `InvertedIndexParams::default()`, which is
what `main` builds today).

| Parameter | `standard_v1` | `standard_folded_v1` | `english_v1` |
|---|---|---|---|
| `tokenizer` | `simple` | `simple` | `simple` |
| `lowercase` | true | true | true |
| `ascii_folding` | false | true | true |
| `stemmer` | none | none | English (`frostem`) |
| `stop_words` | none | none | Lance's English list |
| `max_token_length` | 40 | 40 | 40 |
| `normalization` | none | none | none |

Fingerprint inputs beyond the table: `tokenizer_crate=lance-index`,
`tokenizer_version=<Cargo.lock>`, `stemmer_crate=frostem`,
`stemmer_version=<Cargo.lock>` (11.0.0 pins `frostem 1.20260821.3`).

**Relationship to RFC 0047.** 0047 supplies retrieval stated in the plan
(`QueryIR::retrieval`, extended here with `MatchTerms`, `Knn`, `Ann`, and
N-arm `FuseRrf`), projectable metric columns (typed `F64` there; the
domains here wrap them), the `warnings`/`retrievals` envelope (this RFC
adds `domain`, `index_coverage`, `prefiltered`), the `T27`/`T28` rules
(inherited by every retriever), and `T29`/`T31` (lifted or kept as stated
above).

**Substrate facts (Lance 11.0.0, crates.io pin; validate against the
pin).**

- Flat-path analyzer: with no FTS segments the substrate flat-scans with a
  bare case-sensitive tokenizer; with segments present, including an empty
  index, the persisted analyzer resolves (`write()` always persists
  `params`). The index shell is what makes law 5 true, and it is
  Lance-11-dependent.
- A partially covered index scores uncovered rows with a different scorer
  (RFC 0045 records this), which is why score parity is stated at full
  coverage only while membership parity holds at every coverage state.
- Exact rescore: the scanner's refine path drops quantizer distances,
  recomputes exactly from raw vectors, and sorts `(distance, rowid)` with
  `fetch = k × refine_factor`; `oversample` is that refine factor. Partial
  coverage merges indexed and flat candidates (`knn_combined`) before an
  exact finish. Exactness on an indexed column is `use_index(false)`.
- Mismatched caller metric: the substrate brute-forces on the auto path
  (with a process-log warning only) or errors on the explicit path; the
  capability probe decides before planning.
- Distances: `l2` has no square root (`lance-linalg` `l2.rs`); `cosine` is
  `1 − cos`; `dot` is `1 − dot` (`dot.rs`).
- Lexical ties: the plain match path compares score alone and drops
  equal-score boundary candidates by arrival order; RFC 0047's plateau
  retry and id tie-break remain necessary.
- BM25: `K1 = 1.2`, `B = 0.75`, `idf = ln((N − n + 0.5) / (n + 0.5) + 1)`
  (`lance-index` `scalar/inverted/scorer.rs`); `bm25_v1` freezes them. An
  upstream change requires a new scorer version, not a fork.

**Cross-RFC composition.**

- RFC 0043: amended as listed under Design; the `rebuild-indexes`
  generalization keeps 0043's staging, recovery identity, and
  single-publication properties.
- RFC 0040: the feature-name rule and the parsed `.gq` rewrite tool.
- RFC 0012: the recorded `(provider, model, dim)` identity is reused
  unchanged; its open backfill question is answered here (unrecorded
  model: raw-vector only until declared).
- RFC 0015: the pending-row predicate is the definition behind
  `embedding_coverage`.
- RFC 0046: the capability probe builds on its state vocabulary; its
  `degraded` reason set gains `analyzer_profile_mismatch` and the vector
  artifact reasons.
